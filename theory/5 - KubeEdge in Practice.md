# KubeEdge in Practice

## From theory to deployment

This module focuses on what KubeEdge looks like in practice: how a deployment is structured, what the full synchronization flow looks like, what happens during a network outage, and how common real-world patterns are implemented.

All examples assume the following setup:

- **Cloud node**: a VM running K3s as the Kubernetes control plane, with KubeEdge's CloudCore installed;
- **Edge node**: a second VM (simulating a remote edge device) running KubeEdge's EdgeCore;
- **Communication**: WebSocket tunnel between the two VMs over port 10000;

---

## Typical deployment topology

A production KubeEdge deployment typically looks like this:

```
                       [ Internet / WAN ]
                               |
                  +------------+------------+
                  |       Cloud Node        |
                  |  K3s control plane      |
                  |  CloudCore (port 10000) |
                  |  kubectl access         |
                  +------------+------------+
                               |
                WebSocket tunnel (port 10000)
                               |
             +-----------------+-----------------+
             |                                   |
  +----------+------+               +------------+------+
  |  Edge Node 01   |               |  Edge Node 02     |
  |  EdgeCore       |               |  EdgeCore         |
  |  nginx workload |               |  sensor app       |
  |  temp-sensor    |               |  camera feed      |
  +-----------------+               +-------------------+
             |                                   |
       [ IoT Devices ]                   [ IoT Devices ]
```

---

## Deploying a workload to the edge

Deploying an application to an edge node in KubeEdge is identical to deploying to any Kubernetes node. The only additions are a `nodeSelector` (or `nodeAffinity`) to target the edge node and a `toleration` for the edge taint.

### Step 1 — Label the edge node

```bash
# Add a descriptive label for flexible scheduling
kubectl label node edge-node-01 location=factory-floor environment=production

# Verify
kubectl get node edge-node-01 --show-labels
```

### Step 2 — Write the Deployment manifest

```yaml
# deployment-nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-edge
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-edge
  template:
    metadata:
      labels:
        app: nginx-edge
    spec:
      # Target the edge node by label
      nodeSelector:
        location: factory-floor

      # Required: tolerate the edge taint added by KubeEdge
      tolerations:
        - key: "node-role.kubernetes.io/edge"
          operator: "Exists"
          effect: "NoSchedule"

      containers:
        - name: nginx
          image: nginx:1.25-alpine     # alpine: smaller image, faster pull
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "32Mi"
              cpu: "50m"
            limits:
              memory: "64Mi"
              cpu: "100m"

          # Liveness probe: restart the container if it stops responding
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 30
```

### Step 3 — Apply and verify

```bash
# Deploy from the cloud node
kubectl apply -f deployment-nginx.yaml

# Watch the Pod reach Running state (image pull may take a moment)
kubectl get pods -w
# NAME                          READY   STATUS              NODE
# nginx-edge-7d6f9b8c4-xk2p9   0/1     ContainerCreating   edge-node-01
# nginx-edge-7d6f9b8c4-xk2p9   1/1     Running             edge-node-01

# Confirm placement
kubectl get pods -o wide
# NAME                          READY   STATUS    IP           NODE
# nginx-edge-7d6f9b8c4-xk2p9   1/1     Running   10.42.1.10   edge-node-01
```

The pod is running on the edge node. The cloud control plane scheduled it, but the actual container is managed locally by Edged — the cloud is not involved in keeping it running.

---

## What happens during normal operation

During normal operation, the system maintains a continuous synchronization loop between cloud and edge:

```
kubectl apply  →  API server (etcd)  →  EdgeController
                                              ↓
                                          EdgeHub
                                              ↓
                             ┌────────────────┴───────────────────┐
                             ↓                                    ↓
                        MetaManager                             Edged
                      (persist spec to                    (pull image, start
                         SQLite DB)                          container)
                             ↑                                    ↑
                             └─────────────── status ────────────┘
                                              ↓
                                         EdgeController
                                              ↓
                                      API server (status update)
```

1. Operator applies a manifest via `kubectl` → the API server stores the desired state in etcd
2. **EdgeController** detects the change and forwards the resource spec to the relevant edge node via the WebSocket tunnel;
3. **EdgeHub** on the edge node receives the message and routes it to the appropriate component;
4. **MetaManager** persists the resource spec in the local SQLite database;
5. **Edged** picks up the spec from MetaManager and acts on it (pull image, create container via containerd);
6. Edged reports the Pod status back through EdgeHub → EdgeController → API server;
7. `kubectl get pods` reflects the updated status;

---

## What happens during a network outage

When the WebSocket tunnel between EdgeCore and CloudCore is interrupted:

1. **EdgeHub** detects the disconnection and begins buffering outgoing messages locally;
2. **Edged** continues managing running containers normally: it reads Pod specs from MetaManager's local cache, not from the cloud;
3. If a container crashes, **Edged** restarts it using the cached Pod spec and the cloud is not consulted;
4. **DeviceTwin** continues synchronizing device state locally, collecting sensor readings and maintaining the twin;
5. From the cloud side, the edge node transitions to `NotReady` after the heartbeat timeout,  but its Pods are **not evicted**;

### Why are pods not evicted?

Standard Kubernetes evicts pods from `NotReady` nodes after `tolerationSeconds` expires (default: 300 seconds). KubeEdge overrides this by automatically setting an extremely large `tolerationSeconds` value on edge nodes when they join the cluster. This ensures that edge pods are never evicted due to connectivity loss — they are expected to keep running indefinitely at the edge.

```bash
# From the cloud, the node appears NotReady during an outage
$ kubectl get nodes
NAME            STATUS     ROLES        AGE
cloud-node      Ready      master       10d
edge-node-01    NotReady   agent,edge   2d

# Pod status shows Unknown (cloud cannot verify edge state)
$ kubectl get pods -o wide
NAME                          READY   STATUS    NODE
nginx-edge-7d6f9b8c4-xk2p9   1/1     Unknown   edge-node-01

# BUT on the edge node, the container is still running:
$ sudo crictl ps
CONTAINER   IMAGE              STATE     NAME
a1b2c3d4e   nginx:1.25-alpine  Running   nginx
```

The workload continues serving traffic locally, completely unaffected by the cloud outage.

---

## Reconnection and state reconciliation

When connectivity is restored:

1. **EdgeHub** re-establishes the WebSocket tunnel to CloudCore;
2. **MetaManager** flushes its queued outgoing messages to CloudCore;
3. **EdgeController** compares the edge state with the desired state stored in etcd;
4. Any divergences are reconciled: missing resources are re-deployed, stale configurations are updated, deleted resources are removed;
5. Pod statuses are written back to the API server — `kubectl get pods` becomes accurate again;

```bash
# Watch the edge node return to Ready
watch kubectl get nodes
# NAME          STATUS   ROLES        AGE    ← after ~10 seconds
# cloud-node    Ready    master       10d
# edge-node-01  Ready    agent,edge   2d

# Pod status is corrected automatically
kubectl get pods -o wide
# NAME                          READY   STATUS    NODE
# nginx-edge-7d6f9b8c4-xk2p9   1/1     Running   edge-node-01
```

This reconciliation is **fully automatic** and requires no operator intervention.

---

## Managing IoT devices

KubeEdge's device management flow involves three steps: defining a DeviceModel, registering a Device instance, and reading/writing device state from the cloud.

### Step 1 — Define a DeviceModel

```yaml
# devicemodel-thermometer.yaml
apiVersion: devices.kubeedge.io/v1alpha2
kind: DeviceModel
metadata:
  name: thermometer
  namespace: default
spec:
  properties:
    - name: temperature
      description: Temperature reading from the sensor
      type:
        float:
          accessMode: ReadOnly
          unit: Celsius
    - name: unit
      description: Display unit (C or F)
      type:
        string:
          accessMode: ReadWrite
          defaultValue: "C"
    - name: alarm-enabled
      description: Whether the over-temperature alarm is active
      type:
        boolean:
          accessMode: ReadWrite
          defaultValue: "true"
```

### Step 2 — Register a Device instance

```yaml
# device-thermometer-01.yaml
apiVersion: devices.kubeedge.io/v1alpha2
kind: Device
metadata:
  name: thermometer-01
  namespace: default
spec:
  deviceModelRef:
    name: thermometer
  nodeSelector:
    nodeSelectorTerms:
      - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
              - edge-node-01
```

```bash
kubectl apply -f devicemodel-thermometer.yaml
kubectl apply -f device-thermometer-01.yaml

# Verify
kubectl get device -n default
# NAME               AGE
# thermometer-01     30s
```

### Step 3 — Read device state from the cloud

```bash
kubectl get device thermometer-01 -o yaml

# Status section shows the twin state:
# status:
#   twins:
#     - propertyName: temperature
#       reported:
#         value: "22.7"
#         metadata:
#           timestamp: "1704067200"
#     - propertyName: unit
#       desired:
#         value: "C"
#       reported:
#         value: "C"
#     - propertyName: alarm-enabled
#       desired:
#         value: "true"
#       reported:
#         value: "true"
```

### Step 4 — Update device configuration from the cloud

```bash
# Change the display unit to Fahrenheit
kubectl patch device thermometer-01 --type=merge \
  -p '{"status":{"twins":[{"propertyName":"unit","desired":{"value":"F"}}]}}'

# DeviceTwin on edge-node-01 will:
# 1. Detect the desired state change
# 2. Publish the new configuration to the physical device via MQTT
# 3. Update the reported state once the device acknowledges it
```

---

## Observability

### Node and Pod status

```bash
# All nodes with IP and OS info
kubectl get nodes -o wide

# All Pods across all namespaces, with node placement
kubectl get pods -A -o wide

# Full node details: conditions, capacity, allocated resources, events
kubectl describe node edge-node-01

# EdgeCore service logs (stream live)
sudo journalctl -u edgecore -f

# EdgeCore logs since a specific time
sudo journalctl -u edgecore --since "30 minutes ago"
```

### Tunnel status

```bash
# On the cloud node: check CloudCore logs for connection events
kubectl logs -n kubeedge -l app=cloudcore -f | grep -i "edge-node"
# edge node edge-node-01 connected
# edge node edge-node-01 disconnected
# edge node edge-node-01 connected   ← reconnection after outage
```

### Container runtime (on the edge node)

```bash
# List running containers
sudo crictl ps

# List all containers including stopped ones
sudo crictl ps -a

# Container logs (equivalent to kubectl logs, but local)
sudo crictl logs <container-id>

# Pod details
sudo crictl pods
sudo crictl inspectp <pod-id>
```

---

## Common patterns

### Pattern 1 — Sensor data collection

An edge node runs a containerized application that reads from sensors via MQTT, aggregates data locally, and periodically sends summaries to the cloud for storage and analysis.

```
Sensor  →  MQTT PUBLISH  →  EventBus  →  DeviceTwin (reported state)
                                                  ↓
                                         EdgeController
                                                  ↓
                                     Cloud storage / analytics
```

Local aggregation reduces cloud bandwidth usage by orders of magnitude compared to sending every raw reading.

### Pattern 2 — GitOps at the edge

KubeEdge integrates naturally with GitOps tools (ArgoCD, Flux) because it reuses the standard Kubernetes API. A GitOps pipeline that manages cloud workloads can be extended to manage edge workloads simply by adding edge-targeted manifests to the Git repository.

```
Git repo (manifest change)
    ↓
ArgoCD / Flux detects drift
    ↓
kubectl apply (runs on cloud node)
    ↓
EdgeController propagates to edge nodes
    ↓
Edged applies changes locally
```

The entire GitOps flow works without modification. Edge nodes are just another set of targets in the cluster.

---

## Summary

In practice, KubeEdge delivers on its architectural promises:

- **Workload deployment** uses standard `kubectl` and YAML manifests — no new tools to learn, existing GitOps pipelines work unchanged;
- **Offline operation** is transparent: edge workloads continue running during cloud outages, pods are never evicted, state is automatically reconciled on reconnection;
- **IoT device management** uses Kubernetes-native custom resources (DeviceModel, Device) that integrate cleanly into existing workflows;
- **Observability** relies on standard Kubernetes tooling, supplemented by direct log inspection on the edge node via `journalctl` and `crictl`;

---

*Previous: [4 - KubeEdge Architecture](./4%20-%20KubeEdge%20Architecture.md)*  
*Next: [6 - KubeEdge (Lab)](../lab/6%20-%20KubeEdge%20(Lab).md)*

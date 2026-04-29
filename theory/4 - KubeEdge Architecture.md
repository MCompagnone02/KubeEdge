# KubeEdge Architecture

## Overview

KubeEdge is an open-source project hosted by the CNCF (Cloud Native Computing Foundation) that extends Kubernetes to edge environments. It's the de-facto standard for Kubernetes-based IoT and edge deployments.

Its architecture is built around a clear separation between three layers:

- **Cloud layer**: the standard Kubernetes control plane;
- **Edge layer**: lightweight edge nodes running EdgeCore (replaces kubelet);
- **Device layer**: physical IoT devices communicating over MQTT;

```
+----------------------------------------------------------+
|                      Cloud Layer                         |
|                                                          |
|   [ Kubernetes API Server ]       [ CloudCore ]          |
|          ↑ ↓                           ↑ ↓               |
|     Standard K8s API           ┌──────────────────┐      |
|     (etcd, scheduler,          │ EdgeController   │      |
|      controller-manager)       │ DeviceController │      |
|                                └──────────────────┘      |
+---------------------------↕ WebSocket / QUIC ↕-----------+
                      (port 10000, edge-initiated)
+----------------------------------------------------------+
|                       Edge Layer                         |
|                                                          |
|   [ EdgeCore ]                                           |
|   ┌──────────┬───────────────┬─────────────┬──────────┐  |
|   │  Edged   │  MetaManager  │ DeviceTwin  │ EdgeHub  │  |
|   └──────────┴───────────────┴─────────────┴──────────┘  |
|                        ↑ ↓  EventBus (MQTT)              |
+----------------------------------------------------------+
|                      Device Layer                        |
|                                                          |
|   [ Temperature Sensors ]  [ Actuators ]  [ Cameras ]    |
+----------------------------------------------------------+
```

---

## Cloud layer

The cloud layer consists of a standard Kubernetes cluster (often K3s for lightweight deployments) augmented by **CloudCore**, the KubeEdge component that bridges the control plane and the edge nodes.

### CloudCore

CloudCore runs as a Pod inside the Kubernetes cluster and exposes a secure endpoint on port 10000 that edge nodes connect to. It contains two main controllers.

#### EdgeController

The EdgeController maintains bidirectional synchronization between the Kubernetes API server and the edge nodes.

**Downward flow** (cloud → edge): watches the API server for changes to resources assigned to edge nodes (Pods, ConfigMaps, Secrets, Services) and forwards them to the relevant EdgeCore instances via the WebSocket tunnel.

**Upward flow** (edge → cloud): receives status updates from edge nodes and writes them back to the API server, so `kubectl get pods` accurately reflects the state of edge workloads.

```
kube-apiserver  →  EdgeController  →  EdgeCore (edge node)
     ↑                                       │
     └────────────── status updates ─────────┘
```

The EdgeController tracks which resources belong to which edge node using standard Kubernetes labels and node selectors; no special routing configuration is needed.

#### DeviceController

The DeviceController manages the full lifecycle of IoT devices registered in KubeEdge. It synchronizes **Device** and **DeviceModel** custom resources between the cloud and the edge, enabling operators to configure and monitor physical devices using standard Kubernetes manifests.

```yaml
# Example: registering an IoT device
apiVersion: devices.kubeedge.io/v1alpha2
kind: Device
metadata:
  name: temperature-sensor-01
spec:
  deviceModelRef:
    name: temperature-sensor-model
  nodeSelector:
    nodeSelectorTerms:
      - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
              - edge-node-01
```

The DeviceController detects changes to Device resources (e.g., a new desired state) and propagates them to the corresponding edge node via the tunnel. It also receives device state reports from DeviceTwin and updates the Device resource's status in the API server.

### The communication tunnel

CloudCore and EdgeCore communicate over a **WebSocket** tunnel (with **QUIC** as an optional alternative for high-latency or packet-loss-prone links).

| Property | WebSocket | QUIC |
|----------|-----------|------|
| Transport | TCP | UDP |
| Multiplexing | Single connection | Multiple streams over one connection |
| Head-of-line blocking | Yes (TCP) | No |
| Best for | Stable, low-latency links | High-latency, lossy links (satellite, LTE) |
| Standard port | 10000/TCP | 10000/UDP |

Two critical design decisions:

1. **Firewall-friendly**: the edge node *initiates* the outbound connection to CloudCore. No inbound ports need to be open on the edge side: this is crucial when edge nodes are behind NAT or corporate firewalls;

2. **Resilient to disconnection**: the tunnel reconnects automatically after network outages. Messages are queued locally (in MetaManager and EdgeHub) to avoid data loss during interruptions;

### Security: certificate-based authentication

Every edge node authenticates to CloudCore using **TLS mutual authentication**. The join flow:

1. `keadm gettoken` generates a time-limited bootstrap token on the cloud node;
2. `keadm join` on the edge node uses the token to request a certificate from CloudCore;
3. CloudCore signs and returns a node certificate;
4. Subsequent connections use the node certificate for mutual TLS — the bootstrap token is no longer needed;

```bash
# Token is valid for a limited time (default: 24 hours)
keadm gettoken --kube-config=/root/.kube/config
# → eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# Edge node uses token once to get its certificate
keadm join --cloudcore-ipport=<ip>:10000 --token=<token>
```

---

## Edge layer

The edge layer runs on each edge node. Its core component is **EdgeCore**, which is a single binary that bundles all edge-side functionality and replaces the standard Kubernetes node agent (`kubelet`).

### EdgeCore

EdgeCore contains five sub-components, each responsible for a specific aspect of edge node management.

#### Edged

Edged is a streamlined replacement for `kubelet`. It manages the full Pod lifecycle on the edge node:

- Pulling container images from registries (or local cache);
- Starting and stopping containers via the local container runtime (`containerd` or `CRI-O`);
- Monitoring container health via liveness and readiness probes;
- Restarting failed containers according to the Pod's `restartPolicy`;
- Mounting volumes (ConfigMaps, Secrets, emptyDir, hostPath);

**Critical difference from standard kubelet**: Edged does not require a live connection to the API server to manage running Pods. It works entirely from the local state provided by MetaManager. When the cloud is unreachable, running pods continue normally and crashed containers are restarted from the cached spec.

#### MetaManager

MetaManager is the **offline-autonomy engine** of KubeEdge. It acts as a local persistence layer, storing all resource metadata in a **SQLite database** on the edge node.

When Edged or DeviceTwin need information about a resource, they query MetaManager first. MetaManager serves the request from its local cache, and only goes to the cloud if the resource is not cached.

```
Normal operation:
  Edged  →  MetaManager  →  (served from local SQLite)
                         ↘  (cache miss)  →  CloudCore → API server

During outage:
  Edged  →  MetaManager  →  (served from local SQLite, no cloud needed)
                            (cloud requests queued for reconnection)
```

When connectivity is restored, MetaManager synchronizes its state with CloudCore and flushes its outgoing queue. This reconciliation is automatic and transparent to running workloads.

**What MetaManager stores locally:**
- Pod specs (so containers can be restarted without cloud access);
- ConfigMaps and Secrets (so applications have their configuration);
- Node status updates (queued for delivery when the tunnel recovers);
- Device twin state (replicated from DeviceTwin);

#### DeviceTwin

DeviceTwin implements the **digital twin pattern** for IoT devices. For each physical device connected to the edge node, DeviceTwin maintains a lightweight in-memory and on-disk representation with two halves:

```
Cloud (desired state)
         ↓
    DeviceTwin
    ↙          ↖
Physical device    (reported state)
```

- **Desired state**: the configuration pushed from the cloud;
- **Reported state**: the actual current state of the physical device;

DeviceTwin continuously works to reconcile the two, pushing configuration updates to the device via EventBus (MQTT) and collecting sensor readings to report back to the cloud.

This pattern decouples the cloud's management logic from the physical device's real-time behavior. The cloud sets intentions; DeviceTwin handles the low-level protocol details and the reconciliation loop. Device state is persisted locally, so configurations survive edge node restarts and network outages.

#### EventBus

EventBus is KubeEdge's **MQTT integration layer**. It connects DeviceTwin to the physical world via MQTT — the lightweight publish-subscribe protocol that is the de-facto standard for IoT communication (ISO/IEC 20922).

EventBus can operate in two modes:

- **Internal MQTT broker**: KubeEdge runs an embedded Mosquitto broker on the edge node (default port 1883). Suitable for small deployments;
- **External MQTT broker**: KubeEdge connects to an existing broker (e.g., EMQX, HiveMQ, AWS IoT Core). Suitable for production deployments with multiple edge nodes sharing a broker;

```
Physical device  →  MQTT PUBLISH  →  EventBus  →  DeviceTwin  →  Cloud
Physical device  ←  MQTT SUBSCRIBE ←  EventBus  ←  DeviceTwin  ←  Cloud
```

#### EdgeHub

EdgeHub manages the WebSocket (or QUIC) connection to CloudCore. It is the **single communication gateway** between all internal EdgeCore components and the cloud.

No other component communicates directly with the cloud. Edged, MetaManager, and DeviceTwin all pass their outgoing messages through EdgeHub, which handles:

- Connection establishment and TLS handshake;
- Automatic reconnection with exponential backoff after network outages;
- Message queuing and delivery guarantees;
- Routing incoming cloud messages to the correct component;

This design ensures that connectivity logic is centralized in one place — any component can assume reliable message delivery without worrying about connection state.

---

## The KubeEdge object model

KubeEdge extends the standard Kubernetes API with two custom resource types for IoT device management.

### DeviceModel

A **DeviceModel** defines the template for a *class* of devices — their properties, data types, units, and access modes. It is analogous to a class definition in object-oriented programming. A single DeviceModel can be instantiated by many Device resources.

```yaml
apiVersion: devices.kubeedge.io/v1alpha2
kind: DeviceModel
metadata:
  name: temperature-sensor-model
spec:
  properties:
    - name: temperature
      description: Current temperature reading
      type:
        float:
          accessMode: ReadOnly       # device reports this, cloud cannot set it
          defaultValue: 0.0
          unit: Celsius
    - name: alarm-threshold
      description: Temperature threshold that triggers an alarm
      type:
        float:
          accessMode: ReadWrite      # cloud can set this, device reports current value
          defaultValue: 80.0
          unit: Celsius
    - name: sampling-interval
      description: How often the sensor takes a reading (milliseconds)
      type:
        int:
          accessMode: ReadWrite
          defaultValue: 5000
          unit: ms
```

### Device

A **Device** is an instance of a DeviceModel, representing a specific physical device connected to a specific edge node. Its `status.twins` section holds the current desired and reported values for each property.

```yaml
apiVersion: devices.kubeedge.io/v1alpha2
kind: Device
metadata:
  name: temperature-sensor-01
spec:
  deviceModelRef:
    name: temperature-sensor-model
  nodeSelector:
    nodeSelectorTerms:
      - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
              - edge-node-01
  protocol:
    modbus:
      slaveID: 1
status:
  twins:
    - propertyName: temperature
      desired:
        value: "0"             # read-only property: desired value is ignored
      reported:
        value: "23.4"          # current reading from the physical sensor
        metadata:
          timestamp: "1704067200"
    - propertyName: alarm-threshold
      desired:
        value: "75.0"          # operator lowered the threshold from 80 to 75
      reported:
        value: "75.0"          # device has applied and confirmed the new threshold
    - propertyName: sampling-interval
      desired:
        value: "2000"          # operator increased sampling frequency
      reported:
        value: "5000"          # device has not yet applied the change (transitioning)
```

The `desired != reported` state for `sampling-interval` above represents a device that is in the process of applying a configuration change: DeviceTwin will keep retrying until the reported state converges.

---

## Workload scheduling to edge nodes

KubeEdge reuses the standard Kubernetes scheduling model. Edge nodes appear as normal nodes in the cluster (with the `agent,edge` role) and workloads are assigned to them using standard node selectors or node affinity rules.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inference-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: inference
  template:
    metadata:
      labels:
        app: inference
    spec:
      # Target a specific edge node
      nodeSelector:
        kubernetes.io/hostname: edge-node-01
      # Edge nodes carry this taint by default — tolerate it to allow scheduling
      tolerations:
        - key: "node-role.kubernetes.io/edge"
          operator: "Exists"
          effect: "NoSchedule"
      containers:
        - name: inference
          image: myregistry/inference:latest
          resources:
            requests:
              memory: "256Mi"
              cpu: "500m"
            limits:
              memory: "512Mi"
              cpu: "1"
```

This means that existing Kubernetes tooling, such as Helm charts, GitOps pipelines, `kubectl`, work without modification for edge workloads.

### The edge node taint

KubeEdge automatically taints edge nodes with `node-role.kubernetes.io/edge:NoSchedule` when they join the cluster. This prevents the Kubernetes scheduler from accidentally placing cloud-only workloads on edge nodes. Workloads explicitly intended for edge nodes must add the matching toleration.

---

## Installation: keadm

KubeEdge provides a dedicated CLI called `keadm` for cluster setup, node management, and upgrades.

```bash
# Cloud node: initialize CloudCore
keadm init \
  --advertise-address=<cloud-node-ip> \
  --kube-config=/root/.kube/config \
  --kubeedge-version=v1.15.0

# Cloud node: get the join token (valid 24h by default)
keadm gettoken --kube-config=/root/.kube/config

# Edge node: join the cluster
keadm join \
  --cloudcore-ipport=<cloud-node-ip>:10000 \
  --token=<token> \
  --kubeedge-version=v1.15.0
```

After joining, the edge node appears alongside standard nodes:

```bash
$ kubectl get nodes
NAME            STATUS   ROLES          AGE   VERSION
cloud-node      Ready    control-plane  10d   v1.27.0
edge-node-01    Ready    agent,edge     2d    v1.13.0
```

The `agent,edge` role is assigned automatically by KubeEdge. The version difference (`v1.27.0` vs `v1.13.0`) reflects the internal version reported by Edged, which is intentionally lower; Edged implements only the subset of the kubelet API relevant to edge operation.

---

## Summary

KubeEdge's architecture is built around two main components:

- **CloudCore** (cloud side): bridges the Kubernetes control plane and the edge nodes, synchronizing workloads (EdgeController) and IoT device state (DeviceController) via a secure WebSocket/QUIC tunnel;

- **EdgeCore** (edge side): a lightweight agent that manages the full Pod lifecycle (Edged), caches all resource state for offline operation (MetaManager), implements the digital twin pattern for IoT devices (DeviceTwin), communicates with physical devices via MQTT (EventBus) and manages the cloud tunnel (EdgeHub);

The result is a unified management plane where edge nodes behave as standard Kubernetes nodes from the operator's perspective, while internally providing the offline autonomy and IoT device management capabilities that standard Kubernetes lacks.

---

*Previous: [3 - Edge Tools Overview](./3%20-%20Edge%20Tools%20Overview.md)*  
*Next: [5 - KubeEdge in Practice](./5%20-%20KubeEdge%20in%20Practice.md)*

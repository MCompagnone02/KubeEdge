# KubeEdge (Lab)

## Overview

This lab walks through a complete KubeEdge setup on two local VMs, simulating a minimal cloud-to-edge deployment. It is structured as three progressive exercises:

- **Lab 1** — Set up the environment (K3s + KubeEdge)
- **Lab 2** — Deploy a workload to the edge node
- **Lab 3** — Simulate a network outage and observe offline resilience

Each lab builds on the previous one. By the end, you will have experienced the full lifecycle of a KubeEdge deployment: cluster setup, workload management, and fault tolerance.

### Prerequisites

| Requirement | Version |
|-------------|---------|
| Two Linux VMs (Ubuntu 22.04 recommended) | — |
| containerd | ≥ 1.6 |
| kubectl | ≥ 1.27 |
| keadm | ≥ 1.15 |
| Internet access on both VMs | — |

Throughout this lab:
- **`cloud-node`** refers to the VM running K3s and CloudCore (IP: `192.168.1.10`)
- **`edge-node`** refers to the VM running EdgeCore (IP: `192.168.1.20`)

> **Tip**: if you are using a different network range, replace `192.168.1.10` and `192.168.1.20` throughout with your actual IPs. The two VMs must be able to reach each other on port 10000/TCP.

---

## Lab 1 — Environment setup

**Objective**: install K3s on the cloud node, install KubeEdge's CloudCore on the cloud node, and join the edge node to the cluster.

### 1.1 Install K3s on the cloud node

K3s provides the lightweight Kubernetes control plane on the cloud side. We disable Traefik (the default ingress controller) since we don't need it here.

```bash
# Run on: cloud-node

# Install K3s
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik" sh -

# Verify the cluster is up
kubectl get nodes
# NAME         STATUS   ROLES                  AGE   VERSION
# cloud-node   Ready    control-plane,master   1m    v1.27.x

# Copy kubeconfig to the standard location
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

# Verify kubectl works
kubectl cluster-info
```

> **What happened**: K3s started a single-node Kubernetes cluster using SQLite as the data store. The kubeconfig was copied to `~/.kube/config` so `kubectl` can find it without specifying `--kubeconfig` on every command.

### 1.2 Install keadm on both nodes

`keadm` is the KubeEdge installation and cluster management CLI.

```bash
# Run on: cloud-node AND edge-node

export KUBEEDGE_VERSION=v1.15.0

# Download the keadm archive
wget https://github.com/kubeedge/kubeedge/releases/download/${KUBEEDGE_VERSION}/keadm-${KUBEEDGE_VERSION}-linux-amd64.tar.gz

# Extract and install
tar -zxvf keadm-${KUBEEDGE_VERSION}-linux-amd64.tar.gz
sudo cp keadm-${KUBEEDGE_VERSION}-linux-amd64/keadm /usr/local/bin/keadm

# Verify
keadm version
# KubeEdge version: v1.15.0
```

### 1.3 Initialize CloudCore on the cloud node

```bash
# Run on: cloud-node

keadm init \
  --advertise-address=192.168.1.10 \
  --kube-config=/root/.kube/config \
  --kubeedge-version=${KUBEEDGE_VERSION}

# This command:
# 1. Downloads and installs the KubeEdge CRDs (Device, DeviceModel, etc.)
# 2. Deploys CloudCore as a Pod in the kubeedge namespace
# 3. Opens port 10000/TCP for edge node connections

# Verify CloudCore is running
kubectl get pods -n kubeedge
# NAME                        READY   STATUS    AGE
# cloudcore-xxxxxxxxx-xxxxx   1/1     Running   1m

# Retrieve the join token for the edge node
keadm gettoken --kube-config=/root/.kube/config
# eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...   ← copy this value
```

> **About the token**: this is a time-limited JWT that the edge node uses *once* to obtain its permanent TLS certificate from CloudCore. After joining, communication is secured with mutual TLS using the issued certificate — the token is not needed again.

### 1.4 Join the edge node to the cluster

```bash
# Run on: edge-node

keadm join \
  --cloudcore-ipport=192.168.1.10:10000 \
  --token=<token-from-step-1.3> \
  --kubeedge-version=${KUBEEDGE_VERSION}

# This command:
# 1. Downloads and installs EdgeCore
# 2. Requests a TLS certificate from CloudCore using the bootstrap token
# 3. Writes the EdgeCore configuration to /etc/kubeedge/config/edgecore.yaml
# 4. Starts and enables the edgecore systemd service

# Verify EdgeCore is running
sudo systemctl status edgecore
# ● edgecore.service - KubeEdge Edge Node
#    Loaded: loaded (/etc/systemd/system/edgecore.service; enabled)
#    Active: active (running)
```

### 1.5 Verify the edge node has joined the cluster

```bash
# Run on: cloud-node

kubectl get nodes
# NAME          STATUS   ROLES          AGE   VERSION
# cloud-node    Ready    control-plane  5m    v1.27.x
# edge-node     Ready    agent,edge     1m    v1.13.x
```

The `agent,edge` role confirms that the node is managed by KubeEdge. Note the version difference: EdgeCore reports a lower version (`v1.13.x`) because Edged implements only the subset of the kubelet API relevant to edge operation.

**Lab 1 complete.** You now have a functional KubeEdge cluster with one cloud node and one edge node.

---

## Lab 2 — Workload deployment to the edge

**Objective**: deploy a containerized workload (nginx) to the edge node using standard Kubernetes tooling, and verify it is actually running on the edge node.

### 2.1 Label the edge node

Labels allow flexible workload targeting without hardcoding hostnames in manifests.

```bash
# Run on: cloud-node

kubectl label node edge-node location=factory-floor

# Verify
kubectl get node edge-node --show-labels
# NAME        STATUS   ROLES        LABELS
# edge-node   Ready    agent,edge   ...,location=factory-floor,...
```

### 2.2 Apply the Deployment manifest

```yaml
# File: lab/manifests/deployment-nginx.yaml

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
      # Schedule on nodes labeled location=factory-floor
      nodeSelector:
        location: factory-floor

      # KubeEdge taints edge nodes with this key — we must tolerate it
      tolerations:
        - key: "node-role.kubernetes.io/edge"
          operator: "Exists"
          effect: "NoSchedule"

      containers:
        - name: nginx
          image: nginx:1.25-alpine    # alpine variant: ~40 MB, faster to pull
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "32Mi"
              cpu: "50m"
            limits:
              memory: "64Mi"
              cpu: "100m"

          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 30
```

```bash
# Run on: cloud-node

# Create the manifest file
cat > deployment-nginx.yaml << 'EOF'
# (paste the YAML above)
EOF

kubectl apply -f deployment-nginx.yaml

# Watch the Pod come up
kubectl get pods -w
# NAME                          READY   STATUS              NODE
# nginx-edge-7d6f9b8c4-xk2p9   0/1     ContainerCreating   edge-node
# nginx-edge-7d6f9b8c4-xk2p9   1/1     Running             edge-node
```

### 2.3 Verify the workload is running on the edge node

```bash
# Run on: cloud-node

# Confirm placement and IP address
kubectl get pods -o wide
# NAME                          READY   STATUS    IP           NODE
# nginx-edge-7d6f9b8c4-xk2p9   1/1     Running   10.42.1.10   edge-node

# Inspect the Pod's events (confirm image was pulled and container started)
kubectl describe pod nginx-edge-7d6f9b8c4-xk2p9
# Events:
#   Type    Reason     Message
#   ----    ------     -------
#   Normal  Pulling    Pulling image "nginx:1.25-alpine"
#   Normal  Pulled     Successfully pulled image
#   Normal  Created    Created container nginx
#   Normal  Started    Started container nginx
```

```bash
# Run on: edge-node

# Verify the container is running locally
sudo crictl ps
# CONTAINER   IMAGE              STATE     NAME
# a1b2c3d4e   nginx:1.25-alpine  Running   nginx

# Test the nginx response locally
curl http://10.42.1.10:80
# <!DOCTYPE html>
# <html>
# <head><title>Welcome to nginx!</title></head>
# ...
```

**What to observe**: the container is running on the edge node, but it was scheduled entirely from the cloud using standard `kubectl apply`. From the cloud's perspective this is no different from deploying to any other Kubernetes node. From the edge's perspective, Edged is managing the container locally via containerd.

**Lab 2 complete.** You have deployed and verified a workload on the edge node.

---

## Lab 3 — Offline resilience

**Objective**: demonstrate that edge workloads continue running during a cloud outage, and that state is automatically reconciled when connectivity is restored.

### 3.1 Confirm the baseline state

```bash
# Run on: cloud-node

# Both nodes should be Ready, Pod should be Running
kubectl get nodes && kubectl get pods -o wide
# NAME          STATUS   ROLES          AGE
# cloud-node    Ready    control-plane  15m
# edge-node     Ready    agent,edge     11m
#
# NAME                          READY   STATUS    NODE
# nginx-edge-7d6f9b8c4-xk2p9   1/1     Running   edge-node
```

### 3.2 Simulate a network outage

```bash
# Run on: edge-node

# Block all outbound traffic from edge-node to cloud-node
# This completely severs the WebSocket tunnel
sudo iptables -A OUTPUT -d 192.168.1.10 -j DROP

echo "Network outage simulated. Cloud node is unreachable from edge-node."
```

### 3.3 Observe the cluster from the cloud

```bash
# Run on: cloud-node

# After ~40 seconds (heartbeat timeout), the edge node appears NotReady
watch kubectl get nodes
# NAME          STATUS     ROLES          AGE
# cloud-node    Ready      control-plane  20m
# edge-node     NotReady   agent,edge     16m   ← after ~40 seconds

# Pod status from the cloud shows Unknown (cloud cannot verify edge state)
kubectl get pods -o wide
# NAME                          READY   STATUS    NODE
# nginx-edge-7d6f9b8c4-xk2p9   1/1     Unknown   edge-node
```

> **Important**: the Pod status shows `Unknown` — not `Terminating` or `Error`. This is deliberate: KubeEdge sets an extremely large `tolerationSeconds` on edge nodes, so pods are never evicted. The cloud simply cannot verify the edge state while disconnected.

### 3.4 Verify the workload is still running at the edge

```bash
# Run on: edge-node

# The container is still running — completely unaffected by the cloud outage
sudo crictl ps
# CONTAINER   IMAGE              STATE     NAME
# a1b2c3d4e   nginx:1.25-alpine  Running   nginx

# Nginx is still serving traffic locally
curl http://10.42.1.10:80
# <!DOCTYPE html><html><head><title>Welcome to nginx!</title></head>...

# EdgeCore logs show the disconnection — and continued operation
sudo journalctl -u edgecore --since "5 minutes ago" | grep -i "connect\|offline\|disconnect"
# ... connection to cloud lost
# ... running in offline mode
# ... (EdgeCore continues managing workloads from local cache)
```

### 3.5 Simulate a container crash during the outage

This step demonstrates that Edged can restart crashed containers entirely from the local cache, without any cloud interaction.

```bash
# Run on: edge-node

# Find and kill the nginx container process
CONTAINER_ID=$(sudo crictl ps | grep nginx | awk '{print $1}')
sudo crictl stop $CONTAINER_ID

# Wait a few seconds, then check — Edged restarts it from the cached spec
sleep 5
sudo crictl ps
# CONTAINER   IMAGE              STATE     NAME
# b2c3d4e5f   nginx:1.25-alpine  Running   nginx   ← new container ID, freshly started
```

The container restarted without any cloud involvement. Edged read the Pod spec from MetaManager's SQLite cache and recreated the container via containerd.

### 3.6 Restore connectivity and observe reconciliation

```bash
# Run on: edge-node

# Remove the iptables rule to restore connectivity
sudo iptables -D OUTPUT -d 192.168.1.10 -j DROP

echo "Connectivity restored."
```

```bash
# Run on: cloud-node

# Watch the edge node return to Ready (~10 seconds)
watch kubectl get nodes
# NAME          STATUS   ROLES          AGE
# cloud-node    Ready    control-plane  30m
# edge-node     Ready    agent,edge     26m   ← back to Ready

# Pod status is corrected automatically
kubectl get pods -o wide
# NAME                          READY   STATUS    NODE
# nginx-edge-7d6f9b8c4-xk2p9   1/1     Running   edge-node

# CloudCore logs confirm the reconnection
kubectl logs -n kubeedge -l app=cloudcore --tail=20
# edge node edge-node connected
# sync complete
```

When the tunnel re-established, MetaManager flushed its queued status updates to CloudCore. The API server was updated automatically. No manual intervention was required.

### 3.7 Cleanup

```bash
# Run on: cloud-node

# Remove the deployment
kubectl delete -f deployment-nginx.yaml

# Verify the Pod is terminated
kubectl get pods
# No resources found in default namespace.
```

---

## Lab summary

| Lab | Demonstrated | Key component |
|-----|-------------|---------------|
| Lab 1 | K3s + KubeEdge setup on two VMs; edge node joins via `keadm` | CloudCore, EdgeCore, EdgeHub |
| Lab 2 | Workload scheduled from cloud runs on edge node; verified with `kubectl` and `crictl` | Edged, MetaManager |
| Lab 3 | Network outage: edge workload keeps running; container restart from cache; automatic reconciliation | MetaManager, Edged, EdgeHub |

These three labs cover the full lifecycle of a KubeEdge deployment: setup, workload management, and fault tolerance — the three properties that distinguish KubeEdge from a standard Kubernetes installation.

---

## Troubleshooting reference

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `keadm init` fails with kubeconfig error | Wrong kubeconfig path | Run `kubectl get nodes` first to confirm K3s is working; use `--kube-config=/root/.kube/config` |
| Edge node stays `NotReady` after `keadm join` | Port 10000 blocked by firewall | Run `sudo ufw allow 10000/tcp` on cloud-node; verify with `nc -zv 192.168.1.10 10000` from edge-node |
| Pod stuck in `ContainerCreating` on edge | Image pull failed (no internet on edge) | Pre-pull on edge: `sudo crictl pull nginx:1.25-alpine`; or set up a local image registry |
| EdgeCore crashes on start | Token expired or already used | Re-run `keadm gettoken` and rejoin with the new token |
| `crictl` command not found | containerd CLI not in PATH | Install: `sudo apt install -y containerd`; or use `ctr -n k8s.io c ls` |
| `keadm join` fails with "certificate verification" | Time drift between VMs | Sync clocks: `sudo timedatectl set-ntp true` on both VMs |
| Pod shows `Unknown` status from cloud | Expected during outage | This is correct behavior; status resolves when connectivity is restored |
| EdgeCore service not starting after reboot | Service not enabled | Run `sudo systemctl enable edgecore` |

---

## Further exploration

After completing the three labs, consider these extensions:

1. **Add a second edge node** — repeat steps 1.4 and 1.5 with a third VM, then deploy the nginx workload to both edge nodes using a `DaemonSet` instead of a `Deployment`

2. **Deploy a DeviceModel and Device** — apply the YAML manifests from [5 - KubeEdge in Practice](../theory/5%20-%20KubeEdge%20in%20Practice.md) and observe the twin state with `kubectl get device -o yaml`

3. **Explore EdgeCore configuration** — inspect `/etc/kubeedge/config/edgecore.yaml` on the edge node; note the MetaManager SQLite path, the MQTT broker address, and the CloudCore endpoint

4. **Monitor with Prometheus** — KubeEdge exposes metrics endpoints on both CloudCore and EdgeCore; configure a Prometheus scrape target to collect them

---

*Previous: [5 - KubeEdge in Practice](../theory/5%20-%20KubeEdge%20in%20Practice.md)*

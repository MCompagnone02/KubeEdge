# KubeEdge Lab
## Complete Guide — Setup, Execution, Errors and Fixes
*Distributed Edge Programming | Mattia Compagnone*

---

## 0. Prerequisites — Multipass Setup

Multipass creates lightweight Ubuntu VMs on Windows without manually configuring VirtualBox. It is the simplest way to simulate two Linux machines on the same laptop.

**Installation**

Download the `.exe` installer from [multipass.run](https://multipass.run) and install it. Requires Windows 10/11 with Hyper-V (Pro/Education) or VirtualBox (Home).

**Creating the VMs**

Open PowerShell as Administrator and run:

```powershell
multipass launch 22.04 --name cloud-node --cpus 2 --memory 2G --disk 10G
multipass launch 22.04 --name edge-node  --cpus 1 --memory 2G --disk 10G

# Verify both are Running and note the IPs
multipass list
```

> **Note** — Write down the IPv4 of cloud-node (e.g. `172.26.39.147`). You will use this IP in all subsequent commands.

**Transferring files to the VMs**

If you have special characters in the path (°, apostrophes), copy files to `C:\temp\` first, then transfer them:

```powershell
New-Item -ItemType Directory -Force -Path C:\temp
Copy-Item 'C:\...\KubeEdge\deployment-nginx.yaml' C:\temp\deployment-nginx.yaml

multipass transfer C:\temp\deployment-nginx.yaml cloud-node:/home/ubuntu/deployment-nginx.yaml
```

---

## Lab 1 — Cluster Setup

**Objective**: install K3s on the cloud node, start CloudCore, and join the edge node to the cluster.

### 1.1 Install K3s on the cloud node

```bash
multipass shell cloud-node

# Install K3s (disable Traefik, not needed here)
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC='--disable traefik' sh -
```

> **Error**: `Unable to read /etc/rancher/k3s/k3s.yaml — permission denied`
> **Fix**: `sudo chmod 644 /etc/rancher/k3s/k3s.yaml`

After the fix, configure kubectl:

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
sudo chmod 644 /etc/rancher/k3s/k3s.yaml

# Verify
kubectl get nodes
# NAME         STATUS   ROLES           VERSION
# cloud-node   Ready    control-plane   v1.35.x+k3s1
```

> **What happened**: K3s started a single-node Kubernetes cluster using SQLite as the data store. The kubeconfig was copied to `~/.kube/config` so `kubectl` can find it without specifying `--kubeconfig` on every command.

### 1.2 Install keadm on both nodes

`keadm` is the KubeEdge installation and cluster management CLI. Run these steps on **both VMs**.

```bash
# Download the binary directly (explicit version to avoid ${} issues)
wget https://github.com/kubeedge/kubeedge/releases/download/v1.15.0/keadm-v1.15.0-linux-amd64.tar.gz
tar -zxf keadm-v1.15.0-linux-amd64.tar.gz
```

> **Error**: `cp: -r not specified; omitting directory 'keadm-v1.15.0-linux-amd64/keadm'`
> **Fix**: The binary is nested inside a subdirectory. Use `find keadm-v1.15.0-linux-amd64 -name 'keadm' -type f` to locate it.

```bash
# Correct binary path
sudo cp keadm-v1.15.0-linux-amd64/keadm/keadm /usr/local/bin/keadm
sudo chmod +x /usr/local/bin/keadm
keadm version   # should print v1.15.0
```

### 1.3 Initialize CloudCore on the cloud node

CloudCore is the KubeEdge cloud component. It manages communication with all edge nodes.

```bash
keadm init \
  --advertise-address=<CLOUD-NODE-IP> \
  --kube-config=$HOME/.kube/config \
  --kubeedge-version=v1.15.0

# This command:
# 1. Downloads and installs the KubeEdge CRDs (Device, DeviceModel, etc.)
# 2. Deploys CloudCore as a Pod in the kubeedge namespace
# 3. Opens port 10000/TCP for edge node connections

# Verify CloudCore is running
kubectl get pods -n kubeedge
# NAME                      READY   STATUS    AGE
# cloudcore-xxxxx-xxxxx     1/1     Running   1m

# Retrieve the join token for the edge node (copy it!)
keadm gettoken --kube-config=$HOME/.kube/config
```

> **Note** — The token expires after a few hours. If the edge node fails to join, regenerate the token with `keadm gettoken` and retry.

> **About the token**: this is a time-limited JWT that the edge node uses *once* to obtain its permanent TLS certificate from CloudCore. After joining, communication is secured with mutual TLS — the token is not needed again.

### 1.4 Join the edge node to the cluster

Open a second PowerShell window and enter the edge node:

```bash
multipass shell edge-node
```

Before joining, install the required dependencies:

```bash
# 1. containerd
sudo apt-get update -q && sudo apt-get install -y containerd

# 2. CNI plugins (required for container networking)
sudo mkdir -p /opt/cni/bin
wget -q https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz
sudo tar -zxf cni-plugins-linux-amd64-v1.3.0.tgz -C /opt/cni/bin

# 3. CNI configuration
sudo mkdir -p /etc/cni/net.d
cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
{"cniVersion":"0.4.0","name":"bridge","type":"bridge",
 "bridge":"cni0","isGateway":true,"ipMasq":true,
 "ipam":{"type":"host-local","subnet":"10.244.0.0/16",
 "routes":[{"dst":"0.0.0.0/0"}]}}
EOF

# 4. Disable SystemdCgroup (incompatible with this version of containerd)
sudo sed -i 's/SystemdCgroup = true/SystemdCgroup = false/' /etc/containerd/config.toml
sudo systemctl restart containerd

# 5. keadm (same procedure as cloud-node)
wget https://github.com/kubeedge/kubeedge/releases/download/v1.15.0/keadm-v1.15.0-linux-amd64.tar.gz
tar -zxf keadm-v1.15.0-linux-amd64.tar.gz
sudo cp keadm-v1.15.0-linux-amd64/keadm/keadm /usr/local/bin/keadm
sudo chmod +x /usr/local/bin/keadm
```

Now join the edge node to the cluster:

```bash
sudo keadm join \
  --cloudcore-ipport=<CLOUD-NODE-IP>:10000 \
  --token=<TOKEN-FROM-STEP-1.3> \
  --kubeedge-version=v1.15.0

# Expected output:
# KubeEdge edgecore is running, For logs visit: journalctl -u edgecore.service -xe
```

**Common errors:**

| Error | Fix |
|-------|-----|
| `cni plugin not initialized` | Install CNI plugins and create `/etc/cni/net.d/10-bridge.conf` (see above) |
| `expected cgroupsPath to be of format slice:prefix:name` | `sed -i 's/SystemdCgroup = true/SystemdCgroup = false/' /etc/containerd/config.toml && systemctl restart containerd` |
| `management directory is not clean` | `sudo rm -rf /etc/kubeedge/` then retry the join |
| `token expired / certificate verification failed` | Regenerate the token: `keadm gettoken --kube-config=$HOME/.kube/config` |
| `port 10000 unreachable` | Check cloud-node IP with `multipass list` and verify both VMs are Running |

### 1.5 Verify the edge node has joined the cluster

```bash
# Run on: cloud-node
kubectl get nodes
# NAME         STATUS   ROLES          VERSION
# cloud-node   Ready    control-plane  v1.35.x+k3s1
# edge-node    Ready    agent,edge     v1.26.7-kubeedge-v1.15.0
```

---

## Lab 2 — Workload Deployment to the Edge

**Objective**: deploy a containerized workload (nginx) to the edge node using standard Kubernetes tooling, and verify it is actually running on the edge node.

### 2.1 Label the edge node

Labels allow flexible workload targeting without hardcoding hostnames in manifests.

```bash
# Run on: cloud-node
kubectl label node edge-node location=factory-floor
kubectl get node edge-node --show-labels
```

### 2.2 Apply the Deployment manifest

> **⚠ Common issue** — Copy-pasting YAML from the terminal may introduce tabs instead of spaces. To avoid this, transfer the `deployment-nginx.yaml` file from your KubeEdge folder using `multipass transfer` (see Section 0).

```powershell
# Transfer from your KubeEdge folder (PowerShell)
Copy-Item '...\KubeEdge\deployment-nginx.yaml' C:\temp\deployment-nginx.yaml
multipass transfer C:\temp\deployment-nginx.yaml cloud-node:/home/ubuntu/deployment-nginx.yaml
```

```bash
# Apply from cloud-node
kubectl apply -f deployment-nginx.yaml
```

The manifest includes two elements that are essential for KubeEdge:
- `nodeSelector: location: factory-floor` — schedules the Pod only on nodes with this label
- `toleration` for `node-role.kubernetes.io/edge:NoSchedule` — KubeEdge automatically taints edge nodes with this key; without the toleration the Pod will not be scheduled

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
      nodeSelector:
        location: factory-floor
      tolerations:
        - key: "node-role.kubernetes.io/edge"
          operator: "Exists"
          effect: "NoSchedule"
      containers:
        - name: nginx
          image: nginx:1.25-alpine
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

### 2.3 Verify the workload is running on the edge node

```bash
# Run on: cloud-node — watch the Pod come up
kubectl get pods -o wide -w
# NAME                         READY   STATUS    IP           NODE
# nginx-edge-6d75c7c5f-92wct   1/1     Running   10.244.0.5   edge-node

# Run on: edge-node — verify the container is running locally
sudo ctr -n k8s.io containers ls
# Should show nginx:1.25-alpine with state Running

# Test that nginx responds
curl http://10.244.0.5:80
```

**What to observe**: the container is running on the edge node, but it was scheduled entirely from the cloud using standard `kubectl apply`. From the cloud's perspective this is no different from deploying to any other Kubernetes node. From the edge's perspective, Edged is managing the container locally via containerd.

---

## Lab 3 — Offline Resilience

**Objective**: simulate a network outage between edge and cloud, verify that nginx keeps running, then restore connectivity and observe automatic reconciliation.

### 3.1 Confirm the baseline state

```bash
# Run on: cloud-node — both nodes Ready, Pod Running
kubectl get nodes && kubectl get pods -o wide
```

### 3.2 Simulate a network outage

Block traffic from edge-node to cloud-node with iptables:

```bash
# Run on: edge-node
sudo iptables -A OUTPUT -d <CLOUD-NODE-IP> -j DROP
echo "Network outage simulated. Cloud node is unreachable."
```

### 3.3 Observe the cluster from the cloud

```bash
# Run on: cloud-node
# After ~40 seconds (heartbeat timeout), the edge node appears NotReady
watch kubectl get nodes
# NAME         STATUS     ROLES
# cloud-node   Ready      control-plane
# edge-node    NotReady   agent,edge    ← after ~40 seconds

kubectl get pods -o wide
# STATUS: Unknown or Running (depends on K3s/KubeEdge version)
# Either way, the cloud CANNOT verify the actual edge state
```

> **Why not Terminating?** — The `Unknown` status is the expected behavior. The important thing is that Pods are never evicted: KubeEdge sets an extremely large `tolerationSeconds` on edge nodes precisely for this reason.

### 3.4 Verify the workload is still running at the edge

```bash
# Run on: edge-node — nginx is still alive despite the cloud being unreachable
sudo ctr -n k8s.io containers ls
# docker.io/library/nginx:1.25-alpine   Running   ← still alive!

curl http://10.244.0.5:80
# <!DOCTYPE html>... nginx responds normally

# EdgeCore logs show the disconnection — and continued operation
sudo journalctl -u edgecore --since "5 minutes ago" | grep -i "connect\|offline\|disconnect"
# ... connection to cloud lost
# ... running in offline mode
# ... (EdgeCore continues managing workloads from local cache)
```

The container is running on the edge node completely unaffected by the cloud outage. Edged read the Pod spec from MetaManager's SQLite cache and manages the container via containerd — no cloud interaction required.

### 3.5 Restore connectivity and observe reconciliation

```bash
# Run on: edge-node — remove the iptables rule
sudo iptables -D OUTPUT -d <CLOUD-NODE-IP> -j DROP
echo "Connectivity restored."
```

```bash
# Run on: cloud-node — watch the edge node return to Ready (~10 seconds)
watch kubectl get nodes
# NAME         STATUS   ROLES
# cloud-node   Ready    control-plane
# edge-node    Ready    agent,edge     ← back to Ready automatically

# Pod status is corrected automatically
kubectl get pods -o wide
# NAME                         READY   STATUS    NODE
# nginx-edge-6d75c7c5f-92wct   1/1     Running   edge-node

# CloudCore logs confirm the reconnection
kubectl logs -n kubeedge -l app=cloudcore --tail=20
# edge node edge-node connected
# sync complete
```

When the tunnel re-established, MetaManager flushed its queued status updates to CloudCore. The API server was updated automatically. No manual intervention was required.

### 3.6 Cleanup

```bash
# Run on: cloud-node
kubectl delete -f deployment-nginx.yaml

# Verify the Pod is terminated
kubectl get pods
# No resources found in default namespace.
```

---

## Lab Summary

| Lab | Demonstrated | Key component |
|-----|-------------|---------------|
| Lab 1 | K3s + KubeEdge setup on two VMs; edge node joins via `keadm` | CloudCore, EdgeCore, EdgeHub |
| Lab 2 | Workload scheduled from cloud runs on edge node; verified with `kubectl` and `ctr` | Edged, MetaManager |
| Lab 3 | Network outage: edge workload keeps running; automatic reconciliation on reconnect | MetaManager, Edged, EdgeHub |

These three labs cover the full lifecycle of a KubeEdge deployment: setup, workload management, and fault tolerance — the three properties that distinguish KubeEdge from a standard Kubernetes installation.

---

---

## Troubleshooting Reference

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `keadm init` fails with kubeconfig error | Wrong kubeconfig path | Verify K3s is working with `kubectl get nodes`; use `--kube-config=$HOME/.kube/config` |
| Edge node stays `NotReady` after `keadm join` | Port 10000 blocked | Run `sudo ufw allow 10000/tcp` on cloud-node; verify with `nc -zv <cloud-ip> 10000` from edge-node |
| `cni plugin not initialized` | CNI plugins not installed | Install CNI plugins and create `/etc/cni/net.d/10-bridge.conf` (see Lab 1.4) |
| `expected cgroupsPath` error | SystemdCgroup enabled | `sed -i 's/SystemdCgroup = true/SystemdCgroup = false/' /etc/containerd/config.toml && systemctl restart containerd` |
| `management directory is not clean` | Stale EdgeCore files | `sudo rm -rf /etc/kubeedge/` then retry `keadm join` |
| Pod stuck in `ContainerCreating` on edge | Image pull failed | Pre-pull on edge: `sudo ctr images pull docker.io/library/nginx:1.25-alpine` |
| EdgeCore crashes on start | Token expired or already used | Re-run `keadm gettoken` and rejoin with the new token |
| `keadm join` fails with "certificate verification" | Time drift between VMs | Sync clocks: `sudo timedatectl set-ntp true` on both VMs |
| Pod shows `Unknown` status from cloud | Expected during outage | Correct behavior; status resolves when connectivity is restored |
| EdgeCore service not starting after reboot | Service not enabled | Run `sudo systemctl enable edgecore` |

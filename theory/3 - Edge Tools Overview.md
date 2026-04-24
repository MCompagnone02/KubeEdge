# Edge Tools Overview

## The ecosystem

Several projects extend Kubernetes for edge environments. Each makes different trade-offs between simplicity, resource footprint, IoT support, and compatibility with the standard Kubernetes API.

---

## K3s

**K3s** is a lightweight, fully conformant Kubernetes distribution created by Rancher (now part of SUSE). Its goal is to reduce the resource requirements of a standard Kubernetes installation while maintaining full API compatibility.

K3s packages the entire control plane into a **single binary** (~70 MB), replacing or removing several heavy components:

| Standard K8s component | K3s equivalent |
|------------------------|----------------|
| `etcd` | SQLite (default) or embedded etcd |
| Multiple control-plane binaries | Single `k3s` binary |
| Cloud provider integrations | Removed (optional add-ons) |
| `kube-proxy` | Replaced by `flannel` + `iptables` |
| Container runtime (Docker) | `containerd` (bundled) |

### Key characteristics

- **Minimum requirements**: 512 MB RAM, 1 CPU core;
- **Installation**: simple using a single command;
- **Full K8s API compatibility**: all standard `kubectl` commands and YAML manifests work without modification;
- **ARM support**: native support for ARM64 and ARMv7 architectures;
- **Auto-updating**: built-in support via the Rancher update controller;
- **Startup time**: cluster ready in under 30 seconds on modest hardware;

### Security model

K3s uses the same RBAC, TLS, and service account mechanisms as standard Kubernetes. Nodes authenticate to the API server using auto-generated bootstrap tokens and node certificates. All inter-component communication is encrypted by default.

### Limitations for edge

K3s is a lightweight Kubernetes distribution, but it is not specifically designed for edge scenarios. It still assumes:

- Reasonably stable connectivity to the control plane;
- No native IoT device management;
- Nodes are expected to stay online; offline autonomy is not a built-in feature: a disconnected K3s agent cannot restart crashed pods without API server access;

K3s is best used as the **cloud-side cluster** in an edge deployment, providing the lightweight control plane that KubeEdge or other tools extend toward the edge.

```bash
# Install K3s as server (control plane)
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik" sh -

# Verify the cluster is ready
kubectl get nodes

# Install K3s as agent (worker node)
curl -sfL https://get.k3s.io | K3S_URL=https://<server>:6443 \
  K3S_TOKEN=<node-token> sh -
```

---

## MicroK8s

**MicroK8s** is a lightweight Kubernetes distribution developed and maintained by Canonical (the company behind Ubuntu). It is designed for developer workstations, CI/CD pipelines, and small-scale edge deployments.

MicroK8s is distributed as a **snap package** — Canonical's containerized software packaging format. This means it installs in strict isolation from the host system and updates automatically through the snap store without disrupting the running cluster.

```bash
# Install MicroK8s
sudo snap install microk8s --classic

# Check status
microk8s status

# Enable add-ons
microk8s enable dns dashboard ingress metrics-server

# Alias kubectl for convenience
sudo snap alias microk8s.kubectl kubectl
```

### Key characteristics

- **Developer-friendly**: single-node cluster ready in under a minute on any Ubuntu machine;
- **Add-on ecosystem**: optional components (DNS, dashboard, Istio, Knative, Prometheus) enabled with a single command;
- **High availability**: supports multi-node HA clusters with automatic etcd clustering via `microk8s add-node`;
- **Strict confinement**: snap packaging isolates MicroK8s from the rest of the system — clean uninstall, no residue;
- **Automatic updates**: managed through the snap store channels (stable, candidate, beta, edge);
- **GPU support**: `microk8s enable gpu` exposes NVIDIA GPUs to workloads via the NVIDIA operator;

### Limitations for edge

Like K3s, MicroK8s is a general-purpose lightweight distribution. It does not natively address:

- Offline node autonomy;
- IoT device management;
- Geographically distributed, intermittently connected nodes;

MicroK8s is best suited for **developer environments** and small, well-connected edge clusters where the full IoT management features of KubeEdge are not needed.

---

## OpenYurt

**OpenYurt** is an open-source project by Alibaba Cloud that adds edge autonomy capabilities on top of a standard Kubernetes cluster, rather than replacing it.

OpenYurt's core design principle is **non-invasiveness**: it does not modify the Kubernetes control plane or any of its components. Instead, it adds a transparent proxy layer between the control plane and the edge nodes.

```
[ kube-apiserver ]
       ↓
[ YurtHub ]          ← local HTTP proxy running on each edge node
       ↓
[ kubelet ]
```

**YurtHub** is the central component: a local HTTP/HTTPS proxy that intercepts all communication between `kubelet` and the API server, caching every response in a local store. If connectivity is lost, `kubelet` continues to send requests to `YurtHub` — which transparently serves the cached state, making the node behave as if the API server is still reachable.

### Components

| Component | Function |
|-----------|----------|
| **YurtHub** | Edge node proxy with local cache; provides offline autonomy |
| **YurtTunnel** | Secure, multiplexed tunnel for cloud-to-edge communication (e.g., `kubectl exec`, `kubectl logs`) |
| **YurtManager** | Cloud-side controller for node pools and workload lifecycle |
| **Node Pools** | Groups edge nodes by location or function for topology-aware scheduling |

### Node Pools

One of OpenYurt's distinguishing features is the **NodePool** abstraction, which groups edge nodes geographically or logically:

```yaml
apiVersion: apps.openyurt.io/v1beta1
kind: NodePool
metadata:
  name: factory-floor-beijing
spec:
  type: Edge
  selector:
    matchLabels:
      apps.openyurt.io/nodepool: factory-floor-beijing
```

Workloads can be distributed to specific NodePools, enabling topology-aware deployments at scale.

### Key characteristics

- **Non-invasive**: works with any standard Kubernetes cluster (EKS, GKE, on-prem) without modification;
- **Node autonomy**: YurtHub provides offline operation through local HTTP response caching;
- **Node pools**: topology-aware workload management at scale;
- **Tunnel**: secure channel for `kubectl exec/logs` over NAT, even to offline-recovered nodes;
- **Cloud vendor integration**: deep integration with Alibaba Cloud ACK;

### Limitations

- No native IoT device management model (no Device/DeviceModel CRDs);
- Requires YurtHub to be running on each edge node — adds a process dependency;
- Tighter coupling to Alibaba Cloud tooling in some components;
- More complex initial setup than K3s or MicroK8s;

---

## Comparison

| Feature | K3s | MicroK8s | OpenYurt | KubeEdge |
|---------|-----|----------|----------|----------|
| Lightweight footprint | ✓ | ✓ | — | ✓ |
| Full K8s API compatibility | ✓ | ✓ | ✓ | ✓ |
| Offline node autonomy | — | — | ✓ | ✓ |
| Native IoT device management | — | — | — | ✓ |
| ARM support | ✓ | ✓ | ✓ | ✓ |
| MQTT / device protocol support | — | — | — | ✓ |
| Easy installation | ✓ | ✓ | — | — |
| Multi-node edge management | — | ✓ | ✓ | ✓ |
| Non-invasive (no K8s fork) | ✓ | ✓ | ✓ | — |
| Node grouping / topology | — | — | ✓ | ✓ |
| Minimum RAM (edge node) | ~512 MB | ~512 MB | ~256 MB* | ~128 MB |

*YurtHub alone; requires a running kubelet underneath.

### Key takeaways

**K3s and MicroK8s** reduce Kubernetes' resource footprint but do not address the specific challenges of edge computing (offline autonomy, device management). They are excellent choices for lightweight clusters in well-connected environments, or as the cloud-side control plane.

**OpenYurt** adds offline autonomy to a standard Kubernetes cluster non-invasively, making it attractive when you already have a Kubernetes investment and want to extend it to the edge. Its NodePool model scales well. It stops short of native IoT device management.

**KubeEdge** is the only tool in this comparison that addresses all three core edge challenges simultaneously: lightweight footprint, offline autonomy, and native IoT device management. This comes at the cost of a more complex setup and a steeper learning curve. Its MQTT integration and DeviceTwin model make it the natural choice for IoT deployments.

---

## Choosing the right tool

| Scenario | Recommended tool |
|----------|-----------------|
| Developer laptop or CI/CD pipeline | MicroK8s |
| Lightweight production cluster, reliable connectivity | K3s |
| Existing K8s cluster, need edge autonomy at scale | OpenYurt |
| IoT deployments, offline nodes, device management | KubeEdge |
| Lightweight cloud control plane + IoT edge layer | **K3s + KubeEdge** |

### When to combine tools

It is not uncommon to use multiple tools in a single deployment:

- **K3s (cloud)** manages the cluster control plane, keeps the cloud VM footprint small;
- **KubeEdge (edge)** manages the actual edge nodes and IoT devices;
- **OpenYurt-style node pools** logical grouping of KubeEdge edge nodes, if needed at scale;

---

## Summary

The edge Kubernetes ecosystem offers a spectrum of tools, from lightweight distributions (K3s, MicroK8s) to purpose-built edge platforms (KubeEdge, OpenYurt). No single tool is the right choice for every scenario.

KubeEdge stands out for deployments that require all three core edge capabilities simultaneously: resource efficiency, offline autonomy, and IoT device management. The next module explores its architecture in depth.

---

*Previous: [2 - Kubernetes at the Edge](./2%20-%20Kubernetes%20at%20the%20Edge.md)*  
*Next: [4 - KubeEdge Architecture](./4%20-%20KubeEdge%20Architecture.md)*

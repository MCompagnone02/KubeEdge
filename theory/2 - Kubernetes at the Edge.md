# Kubernetes at the Edge

## What is edge computing?

**Edge computing** is a distributed computing paradigm that brings computation and data storage closer to the sources of data, rather than relying on a centralized cloud datacenter.

The "edge" refers to the boundary between the physical world and the digital infrastructure — it includes devices such as: industrial sensors and actuators on factory floors, cameras and vision systems for real-time analysis, IoT gateways in smart buildings or cities and many more.

```
        [ Cloud Datacenter ]
               ↑ ↓
        [ Edge Node / Gateway ]
               ↑ ↓
   [ IoT Devices / Sensors / Actuators ]
```

The edge sits between the raw devices and the cloud, processing data locally and only sending relevant results upstream.

---

## Why not just use the cloud?

Cloud computing offers virtually unlimited resources and a mature ecosystem. However, several fundamental constraints make it unsuitable as the sole infrastructure for edge scenarios.

### Latency

Network round-trips to a cloud datacenter typically take **50–200ms**. For many applications, this is acceptable. For others, it is not:

| Application | Acceptable latency |
|---|---|
| Video streaming | ~200ms |
| Web browsing | ~100ms |
| Industrial automation | < 10ms |
| Autonomous vehicles | < 5ms |
| Robotic surgery | < 1ms |

When a robotic arm on a factory floor needs to react to a sensor reading, waiting for a cloud response is not an option. The decision must happen locally.

### Bandwidth

Modern edge deployments generate enormous volumes of raw data. A single high-resolution industrial camera can produce **1–5 GB/s** of uncompressed video. Sending all of this to the cloud is economically and technically unfeasible.

Edge nodes filter, compress, and analyze data locally, sending only structured results (e.g., "anomaly detected at position X") to the cloud rather than raw streams.

### Data sovereignty and privacy

Regulations such as GDPR (Europe) and HIPAA (United States) restrict where certain types of data can be stored and processed. Medical records, personal images, and financial data may legally need to stay within a specific geographic boundary or on-premises infrastructure. Processing at the edge avoids sending sensitive data to external cloud providers.

---

## The problem with vanilla Kubernetes at the edge

Kubernetes was designed for stable, well-connected datacenter environments. Applying it directly to edge scenarios reveals several structural limitations.

### Assumption of reliable connectivity

Standard Kubernetes requires persistent communication between worker nodes and the control plane. The `kubelet` on each node continuously reports status to the `kube-apiserver`. If connectivity is lost:

- The node is marked `NotReady`;
- The scheduler may evict Pods from that node;
- Workloads that were running fine locally get disrupted by the cluster's own management logic;

This is the opposite of what an edge system needs: a node that is temporarily offline should continue running its workloads undisturbed.

### Resource requirements

A standard Kubernetes worker node runs several system components (`kubelet`, `kube-proxy`, container runtime, CNI plugins) that together consume a large amount of memory and CPU.

Edge devices, however, may have severely constrained resources:

| Device type | Typical RAM | Typical CPU |
|---|---|---|
| Cloud VM (standard) | 8–64 GB | 4–32 cores |
| Edge gateway | 2–8 GB | 2–4 cores |
| Raspberry Pi 4 | 1–8 GB | 4 cores (ARM) |
| Industrial microcontroller | 256 MB | 1 core |

A full Kubernetes node simply cannot run on many of these devices.

### No native IoT device management

Kubernetes manages containers — it has no concept of physical devices such as temperature sensors, motors, or cameras. There is no native mechanism to represent device state, push configuration to hardware, or react to device telemetry within the Kubernetes object model.

### Network topology assumptions

Standard Kubernetes assumes nodes can communicate directly with each other and with the control plane over a low-latency LAN. Edge nodes may be:

- Behind NAT or firewalls with no inbound connectivity;
- Connected over high-latency WAN links (4G, satellite);
- Geographically distributed across thousands of locations;

---

## Requirements for edge orchestration

Given these constraints, a Kubernetes-based edge orchestration system must satisfy the following requirements:

1. **Offline autonomy**: Edge nodes must continue running scheduled workloads even when disconnected from the control plane. State must be cached locally;

2. **Lightweight footprint**: the edge agent must run on resource-constrained hardware (ARM processors, limited RAM);

3. **Device management**: the system must provide a native way to represent, configure, and monitor physical IoT devices alongside containerized workloads;

4. **Reliable synchronization**: when connectivity is restored after an outage, the edge node and the control plane must reconcile their states without data loss or workload disruption;

5. **Standard Kubernetes API compatibility**: operators should be able to use familiar tools (`kubectl`, Helm, standard YAML manifests) without learning a completely new system;

6. **Secure communication**: the link between edge nodes and the cloud control plane must be encrypted and authenticated, especially since it crosses untrusted public networks;

---

## The edge computing landscape

Several projects have emerged to extend Kubernetes toward edge environments, each with a different philosophy:

| Project | Approach | Best suited for |
|---|---|---|
| **KubeEdge** | Extends K8s with a dedicated edge runtime and IoT device management | Large-scale IoT and industrial edge deployments |
| **K3s** | Lightweight K8s distribution, same API | Resource-constrained nodes, simple edge clusters |
| **MicroK8s** | Minimal K8s by Canonical, easy install | Developer environments, small edge clusters |
| **OpenYurt** | Adds edge autonomy on top of standard K8s | Geographically distributed edge nodes |

These tools are not mutually exclusive. A common pattern is to use K3s as the cloud-side cluster and KubeEdge to extend it with edge-specific capabilities — combining the simplicity of K3s with the IoT management features of KubeEdge.

---

## The edge-cloud continuum

Modern systems rarely place all logic at one extreme. Instead, they distribute computation across a **continuum**:

```
[ IoT Device ]  →  [ Edge Node ]  →  [ Regional Fog ]  →  [ Cloud ]
  Raw sensing       Local inference    Aggregation          Storage
  Actuation         Filtering          Analytics            ML training
  < 1ms latency     < 10ms latency     < 50ms latency       unlimited
```

- **IoT devices** perform basic sensing and actuation with minimal onboard processing;
- **Edge nodes** run containerized inference models, filter data, and make time-critical decisions;
- **Fog nodes** (optional intermediate layer) aggregate data from multiple edge nodes and run heavier analytics;
- **Cloud** stores long-term data, trains machine learning models, and provides the management control plane;

KubeEdge targets the edge node layer of this continuum, managing it with the same Kubernetes primitives used to manage cloud workloads.

---

## Summary

Standard Kubernetes was not designed for edge environments. The key mismatches are:

- It assumes reliable, low-latency connectivity to the control plane;
- Its resource footprint is too heavy for constrained devices;
- It has no native model for physical IoT device management;

Edge orchestration systems extend Kubernetes to address these gaps, enabling the same declarative, self-healing model to work across the entire cloud-to-edge continuum.

---

*Previous: [1 - Introduction to Kubernetes](./1%20-%20Introduction%20to%20Kubernetes.md)*  
*Next: [3 - Edge Tools Overview](./3%20-%20Edge%20Tools%20Overview.md)*

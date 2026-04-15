# Introduction to Kubernetes

## What is Kubernetes?

Kubernetes is an open-source platform for automating the deployment, scaling, and management of containerized applications. It was originally developed by Google and released in 2014 and it is now maintained by the Cloud Native Computing Foundation (CNCF).

---

## Some history about Kubernetes

Understanding why Kubernetes exists requires a look at how software deployment has evolved:

1. **Bare metal era**: applications ran directly on physical servers, scaling required buying new hardware and isolation between applications was minimal;

2. **Virtual machines (VMs)**: Hypervisors (VMware, KVM) allowed multiple isolated OS instances on a single machine improving utilization. The problem with VMs is that they are heavy: each one runs a full OS kernel;

3. **Containers**: Docker popularized Linux containers: lightweight, fast-starting, and portable. A container shares the host OS kernel but isolates the application's filesystem, processes, and network;

4. **Container orchestration**: this is the problem that Kubernetes solves, because running hundreds of containers across multiple machines, with load balancing, health checks, and rolling updates, isn't easy;

---

## Core concepts

### Cluster

A Kubernetes **cluster** is the top-level unit which consists of a set of machines (physical or virtual) that collectively run containerized workloads.

```
+---------------------------+
|        Kubernetes         |
|          Cluster          |
|  +--------+ +----------+  |
|  | Control| | Worker   |  |
|  | Plane  | | Nodes    |  |
|  +--------+ +----------+  |
+---------------------------+
```

### Node

A **node** is a single machine (VM or physical server) in the cluster. There are two types of node:

- **Control plane node** manages the cluster state and makes scheduling decisions;
- **Worker node** runs the actual application workloads;

### Pod

A **Pod** is the smallest deployable unit in Kubernetes. It wraps one or more containers that share: the same network namespace (same IP address), the same storage volumes and the same lifecycle;
In practice, most Pods contain a single container while multi-container Pods are used for tightly coupled helpers.

```yaml
# Example: a simple single-container Pod
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
```

### Namespace

A **Namespace** is a logical partition within a cluster that allows multiple teams or projects to share the same cluster while keeping their resources isolated.

```bash
kubectl get pods --namespace=production
kubectl get pods -n development
```

---

## The control plane

The control plane is the brain of the cluster: it maintains the desired state of the system and reacts to changes.

| Component | Role |
|---|---|
| `kube-apiserver` | It exposes the Kubernetes API; all communication go through here |
| `etcd` | It's a distributed key-value store which holds the entire cluster state |
| `kube-scheduler` | It assigns Pods to nodes based on resource availability and constraints |
| `kube-controller-manager` | It runs controllers that reconcile actual state with desired state |

### The reconciliation loop

Kubernetes is declarative: you describe the **desired state**, and the control plane continuously works to match the **actual state** to it. This loop is the foundation of Kubernetes' self-healing behavior.

```
Desired state  →  kube-apiserver  →  etcd
                                       ↓
                              controller-manager
                                       ↓
                         reconcile actual ↔ desired
```

---

## The worker node

Each worker node runs the following components:

| Component | Role |
|---|---|
| `kubelet` | It's an agent that communicates with the control plane and ensures containers described in PodSpecs are running and healthy |
| `kube-proxy` | It manages network rules on the node and enables communication to/from Pods |
| Container runtime | It actually runs the containers (e.g., `containerd`). Docker was historically common but is no longer directly supported |

---

## Key Kubernetes objects

### Deployment

A **Deployment** manages a set of identical Pods (replicas). It handles rolling updates and rollbacks automatically.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3 # Keep 3 identical Pods running
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
```

### Service

Pods are ephemeral: it means that they can be created and destroyed at any time, and their IP addresses change. A **Service** provides a stable network endpoint to reach a set of Pods.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx # Routes traffic to Pods with this label
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP  # Internal-only access (default)
```

Service types:
- `ClusterIP` is accessible only within the cluster (default);
- `NodePort` exposes the service on a static port on each node;
- `LoadBalancer` provisions an external load balancer (cloud providers);

### ConfigMap and Secret

- **ConfigMap** stores non-sensitive configuration data as key-value pairs (e.g., environment variables, config files);
- **Secret** stores sensitive data (passwords, tokens, TLS certificates) in base64-encoded form;

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
```

---

## Interacting with the cluster: kubectl

`kubectl` is the command-line tool for interacting with Kubernetes. All commands communicate with the `kube-apiserver`.

```bash
# List all nodes in the cluster
kubectl get nodes

# List all Pods in all namespaces
kubectl get pods -A

# Apply a manifest file
kubectl apply -f deployment.yaml

# View logs of a Pod
kubectl logs nginx-pod

# Open a shell inside a running container
kubectl exec -it nginx-pod -- /bin/bash

# Describe a resource (useful for debugging)
kubectl describe pod nginx-pod

# Delete a resource
kubectl delete -f deployment.yaml
```

---

## Summary

Kubernetes solves the problem of running containerized applications at scale across multiple machines. Its key ideas are:

- **Declarative configuration**: you describe what you want, Kubernetes figures out how to achieve it;
- **Self-healing**: if a Pod crashes or a node fails, Kubernetes automatically reschedules workloads;
- **Horizontal scaling**: adding replicas is a one-line change;
- **Portability**: the same manifests work on any Kubernetes cluster, regardless of the underlying infrastructure;

These properties made Kubernetes the standard for cloud-native deployments and the foundation on which KubeEdge extends orchestration to the edge.

---

*Next: [2 - Kubernetes at the Edge](./2%20-%20Kubernetes%20at%20the%20Edge.md)*

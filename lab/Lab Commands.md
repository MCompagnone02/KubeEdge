# KubeEdge Lab Commands
---

## Quick start (if VMs were suspended)

```powershell
# PowerShell
multipass start cloud-node
multipass start edge-node
multipass list
```

```bash
multipass shell cloud-node # to enter cloud-node
multipass shell edge-node # to enter edge-node
```

---

## 1. Show the cluster is working (cloud-node)

```bash
kubectl get nodes
# cloud-node   Ready   control-plane
# edge-node    Ready   agent,edge
```

If edge-node stays NotReady after restart, wait 30 seconds.
If it still doesn't reconnect: `sudo systemctl restart edgecore` on edge-node.

---

## 2. Show nginx is running on the edge (cloud-node)

```bash
kubectl get pods -o wide
# nginx-edge-xxxxx   1/1   Running   10.244.0.x   edge-node
```

---

## 3. Show the container is managed locally (edge-node)

```bash
sudo ctr -n k8s.io containers ls
# nginx:1.25-alpine   Running
```

---

## 4. Simulate network outage (edge-node)

```bash
sudo iptables -A OUTPUT -d 172.26.41.55 -j DROP
```

---

## 5. Observe edge-node become NotReady (cloud-node)

```bash
watch kubectl get nodes
# edge-node   NotReady   agent,edge   ← after ~40 seconds
```

```bash
kubectl get pods -o wide
# nginx-edge-xxxxx   Running/Unknown   edge-node
# STATUS: Unknown or Running (depends on K3s/KubeEdge version)
# Either way, the cloud CANNOT verify the actual edge state
```

---

## 6. Show nginx is still running during the outage (edge-node)

```bash
sudo ctr -n k8s.io containers ls
# nginx:1.25-alpine   Running   ← still alive! # Check here for IP address

curl http://10.244.0.8:80
# <!DOCTYPE html>... nginx responds normally
```

---

## 7. Restore connectivity (edge-node)

```bash
sudo iptables -D OUTPUT -d 172.26.41.55 -j DROP
```

---

## 8. Observe automatic reconciliation (cloud-node)

```bash
watch kubectl get nodes
# edge-node   Ready   agent,edge   ← back to Ready in ~10 seconds
```

---

## 9. Show Prometheus metrics (cloud-node)

Open a new terminal, execute `multipass shell cloud-node`, then:

```bash
# Start port-forward (if not already running)
kubectl port-forward -n monitoring svc/prometheus-server 9090:80 --address 0.0.0.0
```

Open in browser: `http://172.26.41.55:9090`

Useful queries:
- `up`: all monitored targets
- `node_memory_MemAvailable_bytes`: available RAM per node
- `100 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100)`: memory usage %

---

## 10. Show Grafana dashboard (cloud-node)

Open a new terminal, execute `multipass shell cloud-node`, then:

```bash
# Start port-forward (if not already running, new terminal)
kubectl port-forward -n monitoring svc/grafana 3000:80 --address 0.0.0.0
```

Open in browser: `http://172.26.41.55:3000`

- Login: `admin` / (retrieve password with command below if needed)
- Show the **Node Exporter Full** dashboard (CPU, RAM, disk, network per node)
- During Lab 3 outage: edge-node metrics disappear → reappear on reconnect

```bash
# Retrieve Grafana admin password if needed
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

---

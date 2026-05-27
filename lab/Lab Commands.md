# KubeEdge Lab Commands
---

## 1. Show the cluster is working (cloud-node)

```bash
kubectl get nodes
# cloud-node   Ready   control-plane
# edge-node    Ready   agent,edge
```

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
# nginx-edge-xxxxx   Unknown   edge-node   ← cloud cannot verify edge state
```

---

## 6. Show nginx is still running during the outage (edge-node)

```bash
sudo ctr -n k8s.io containers ls
# nginx:1.25-alpine   Running   ← still alive!

curl http://10.244.0.5:80
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

## Quick start (if VMs were suspended)

```powershell
# PowerShell
multipass start cloud-node
multipass start edge-node
multipass list
```


If edge-node stays NotReady after restart, wait 30 seconds.
If it still doesn't reconnect: `sudo systemctl restart edgecore` on edge-node.

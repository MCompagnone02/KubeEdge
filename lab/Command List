```bash
# Mostra il cluster
kubectl get nodes
```

```bash
# Mostra CloudCore in esecuzione
kubectl get pods -n kubeedge
```

```bash
# Deploya nginx sull'edge
kubectl apply -f deployment-nginx.yaml
kubectl get pods -o wide -w
```

```bash
# Verifica locale sull'edge
# (passare alla finestra edge-node)
sudo ctr -n k8s.io containers ls
curl http://<POD-IP>:80
```

```bash
# Simula outage
# (sempre sull'edge-node)
sudo iptables -A OUTPUT -d <IP-CLOUD-NODE> -j DROP

# cloud-node — aspetta 40s
watch kubectl get nodes   # edge-node → NotReady
kubectl get pods -o wide  # status Unknown / Running in cache
```

```bash
# Dimostra autonomia edge durante outage
# (dall'edge-node — nonostante la connessione sia tagliata)
sudo ctr -n k8s.io containers ls   # nginx ancora Running
curl http://<POD-IP>:80            # nginx risponde
```

```bash
# Ripristina e mostra riconciliazione
# (sull'edge-node)
sudo iptables -D OUTPUT -d <IP-CLOUD-NODE> -j DROP

# (sul cloud-node)
watch kubectl get nodes   # edge-node → Ready in ~10s automaticamente
```

```bash
# Comandi di troubleshooting utili
# Logs EdgeCore (edge-node)
sudo journalctl -u edgecore.service --since '5 minutes ago'

# Logs CloudCore (cloud-node)
kubectl logs -n kubeedge -l app=cloudcore --tail=20

# Stato servizi
sudo systemctl status edgecore      # edge-node
kubectl get pods -n kubeedge        # cloud-node

# Riavvio EdgeCore se necessario
sudo systemctl restart edgecore
```

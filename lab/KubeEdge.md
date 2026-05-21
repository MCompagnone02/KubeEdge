# KubeEdge Lab
## Guida Completa — Setup, Esecuzione, Errori e Soluzioni
*Distributed Edge Programming | Mattia Compagnone*

---

## 0. Prerequisiti — Installazione Multipass

Multipass crea VM Ubuntu leggere su Windows senza configurare VirtualBox a mano. È il modo più semplice per simulare due macchine Linux sullo stesso portatile.

**Installazione**

Scarica l'installer `.exe` da [multipass.run](https://multipass.run) e installalo. Richiede Windows 10/11 con Hyper-V (Pro/Education) oppure VirtualBox (Home).

**Creazione delle VM**

Apri PowerShell come Amministratore ed esegui:

```powershell
multipass launch 22.04 --name cloud-node --cpus 2 --memory 2G --disk 10G
multipass launch 22.04 --name edge-node  --cpus 1 --memory 2G --disk 10G

# Verifica che siano Running e annota gli IP
multipass list
```

> **ℹ Info** — Annota l'IPv4 di cloud-node (es. `172.26.39.147`). Userai questo IP in tutti i comandi successivi.

**Trasferimento file nelle VM**

A causa di caratteri speciali nel percorso (°, apostrofi), copia prima i file in `C:\temp\` poi trasferiscili:

```powershell
New-Item -ItemType Directory -Force -Path C:\temp
Copy-Item 'C:\...\KubeEdge\setup-cloud.sh' C:\temp\setup-cloud.sh
Copy-Item 'C:\...\KubeEdge\setup-edge.sh'  C:\temp\setup-edge.sh

multipass transfer C:\temp\setup-cloud.sh cloud-node:/home/ubuntu/setup-cloud.sh
multipass transfer C:\temp\setup-edge.sh  edge-node:/home/ubuntu/setup-edge.sh
```

---

## Lab 1 — Setup del Cluster KubeEdge

**Obiettivo**: installare K3s sulla cloud-node, avviare CloudCore, unire l'edge-node al cluster.

### 1.1 Installazione K3s sulla cloud-node

```bash
multipass shell cloud-node

# Installa K3s (disabilita Traefik, non necessario)
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC='--disable traefik' sh -
```

> **❌ Errore**: `Unable to read /etc/rancher/k3s/k3s.yaml — permission denied`
> **✅ Fix**: `sudo chmod 644 /etc/rancher/k3s/k3s.yaml`

Dopo il fix, configura kubectl:

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
sudo chmod 644 /etc/rancher/k3s/k3s.yaml

# Verifica
kubectl get nodes
# NAME         STATUS   ROLES           VERSION
# cloud-node   Ready    control-plane   v1.35.x+k3s1
```

### 1.2 Installazione keadm

`keadm` è il CLI di KubeEdge per gestire il cluster. Va installato su **entrambe le VM**.

```bash
# Scarica il binario (versione esplicita per evitare problemi con ${})
wget https://github.com/kubeedge/kubeedge/releases/download/v1.15.0/keadm-v1.15.0-linux-amd64.tar.gz
tar -zxf keadm-v1.15.0-linux-amd64.tar.gz
```

> **❌ Errore**: `cp: -r not specified; omitting directory 'keadm-v1.15.0-linux-amd64/keadm'`
> **✅ Fix**: Il binario è annidato in una sottocartella. Usa `find keadm-v1.15.0-linux-amd64 -name 'keadm' -type f` per trovarlo.

```bash
# Percorso corretto del binario
sudo cp keadm-v1.15.0-linux-amd64/keadm/keadm /usr/local/bin/keadm
sudo chmod +x /usr/local/bin/keadm
keadm version   # deve stampare v1.15.0
```

### 1.3 Avvio di CloudCore

CloudCore è il componente cloud di KubeEdge. Gestisce la comunicazione con tutti gli edge-node.

```bash
keadm init \
  --advertise-address=<IP-CLOUD-NODE> \
  --kube-config=$HOME/.kube/config \
  --kubeedge-version=v1.15.0

# Verifica che CloudCore sia Running
kubectl get pods -n kubeedge
# NAME                      READY   STATUS    AGE
# cloudcore-xxxxx-xxxxx     1/1     Running   1m

# Recupera il token per il join dell'edge-node (copialo!)
keadm gettoken --kube-config=$HOME/.kube/config
```

> **⚠ Attenzione** — Il token scade dopo alcune ore. Se l'edge-node non riesce a fare il join, rigenera il token con `keadm gettoken` e riprova.

### 1.4 Join dell'edge-node

Apri una seconda finestra PowerShell ed entra nell'edge-node:

```bash
multipass shell edge-node
```

Prima di fare il join, installa le dipendenze:

```bash
# 1. containerd
sudo apt-get update -q && sudo apt-get install -y containerd

# 2. Plugin CNI (necessari per la rete dei container)
sudo mkdir -p /opt/cni/bin
wget -q https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz
sudo tar -zxf cni-plugins-linux-amd64-v1.3.0.tgz -C /opt/cni/bin

# 3. Configurazione CNI
sudo mkdir -p /etc/cni/net.d
cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
{"cniVersion":"0.4.0","name":"bridge","type":"bridge",
 "bridge":"cni0","isGateway":true,"ipMasq":true,
 "ipam":{"type":"host-local","subnet":"10.244.0.0/16",
 "routes":[{"dst":"0.0.0.0/0"}]}}
EOF

# 4. Disabilita SystemdCgroup (incompatibile con la versione di containerd)
sudo sed -i 's/SystemdCgroup = true/SystemdCgroup = false/' /etc/containerd/config.toml
sudo systemctl restart containerd

# 5. keadm (stessa procedura della cloud-node)
wget https://github.com/kubeedge/kubeedge/releases/download/v1.15.0/keadm-v1.15.0-linux-amd64.tar.gz
tar -zxf keadm-v1.15.0-linux-amd64.tar.gz
sudo cp keadm-v1.15.0-linux-amd64/keadm/keadm /usr/local/bin/keadm
sudo chmod +x /usr/local/bin/keadm
```

Ora unisci l'edge-node al cluster:

```bash
sudo keadm join \
  --cloudcore-ipport=<IP-CLOUD-NODE>:10000 \
  --token=<TOKEN-DA-STEP-1.3> \
  --kubeedge-version=v1.15.0

# Output atteso:
# KubeEdge edgecore is running, For logs visit: journalctl -u edgecore.service -xe
```

**Tabella errori comuni:**

| Errore | Soluzione |
|--------|-----------|
| `cni plugin not initialized` | Installa CNI plugins e crea `/etc/cni/net.d/10-bridge.conf` (vedi sopra) |
| `expected cgroupsPath to be of format slice:prefix:name` | `sed -i 's/SystemdCgroup = true/SystemdCgroup = false/' /etc/containerd/config.toml && systemctl restart containerd` |
| `management directory is not clean` | `sudo rm -rf /etc/kubeedge/` poi riprova il join |
| `token expired / certificate verification failed` | Rigenera il token: `keadm gettoken --kube-config=$HOME/.kube/config` |
| `port 10000 unreachable` | Verifica IP cloud-node con `multipass list` e che le VM siano entrambe Running |

### 1.5 Verifica finale

```bash
# Dalla cloud-node
kubectl get nodes
# NAME         STATUS   ROLES          VERSION
# cloud-node   Ready    control-plane  v1.35.x+k3s1
# edge-node    Ready    agent,edge     v1.26.7-kubeedge-v1.15.0
```

> **✅ Lab 1 Completato** — La versione diversa è normale: EdgeCore implementa solo il subset del kubelet necessario per l'edge.

---

## Lab 2 — Deploy del Workload sull'Edge

**Obiettivo**: deployare nginx sull'edge-node usando `kubectl` dalla cloud-node, verificare che giri localmente.

### 2.1 Etichetta il nodo

```bash
kubectl label node edge-node location=factory-floor
kubectl get node edge-node --show-labels
```

### 2.2 Crea e applica il manifest

> **⚠ Consiglio** — Problema comune: il copy-paste dal terminale introduce tab invece di spazi nel YAML. Per evitarlo, trasferisci il file `deployment-nginx.yaml` dalla cartella KubeEdge con `multipass transfer` (vedi Sezione 0).

```powershell
# Trasferisci il file dalla cartella KubeEdge (PowerShell)
Copy-Item '...\KubeEdge\deployment-nginx.yaml' C:\temp\deployment-nginx.yaml
multipass transfer C:\temp\deployment-nginx.yaml cloud-node:/home/ubuntu/deployment-nginx.yaml
```

```bash
# Applica dalla cloud-node
kubectl apply -f deployment-nginx.yaml
```

Il manifest include due elementi fondamentali per KubeEdge:
- `nodeSelector: location: factory-floor` — schedula il Pod solo sui nodi con questa etichetta
- `toleration` per `node-role.kubernetes.io/edge:NoSchedule` — KubeEdge aggiunge automaticamente questo taint agli edge-node; senza la toleration il Pod non viene schedulato

### 2.3 Verifica

```bash
# Dalla cloud-node — osserva il Pod partire
kubectl get pods -o wide -w
# NAME                         READY   STATUS    IP           NODE
# nginx-edge-6d75c7c5f-92wct   1/1     Running   10.244.0.5   edge-node

# Dalla edge-node — verifica il container localmente
sudo ctr -n k8s.io containers ls
# Deve apparire nginx:1.25-alpine con stato Running

# Test che nginx risponda
curl http://10.244.0.5:80
```

> **✅ Lab 2 Completato** — Il Pod è stato schedulato dalla cloud ma il container viene gestito localmente da Edged sulla edge-node, tramite containerd. Il cloud non è coinvolto nel mantenerlo in esecuzione.

---

## Lab 3 — Offline Resilience

**Obiettivo**: simulare un'interruzione di rete tra edge e cloud, verificare che nginx continui a girare, poi ripristinare la connettività e osservare la riconciliazione automatica.

### 3.1 Verifica stato baseline

```bash
# Dalla cloud-node — entrambi i nodi Ready, Pod Running
kubectl get nodes && kubectl get pods -o wide
```

### 3.2 Simula il network outage

Blocca il traffico dall'edge-node verso la cloud-node con iptables:

```bash
# Sulla edge-node
sudo iptables -A OUTPUT -d <IP-CLOUD-NODE> -j DROP
echo 'Outage simulato. Cloud irraggiungibile.'
```

### 3.3 Osserva dalla cloud-node

```bash
# Aspetta ~40 secondi (timeout heartbeat)
watch kubectl get nodes
# NAME         STATUS     ROLES
# cloud-node   Ready      control-plane
# edge-node    NotReady   agent,edge    ← dopo ~40 secondi

kubectl get pods -o wide
# STATUS: Unknown oppure Running (dipende dalla versione K3s/KubeEdge)
# In ogni caso il cloud NON può verificare lo stato reale dell'edge
```

> **ℹ Perché non Terminating?** — Lo status `Unknown` è il comportamento atteso. L'importante è che i Pod non vengano evicted: KubeEdge imposta un `tolerationSeconds` molto alto sugli edge-node proprio per questo.

### 3.4 Verifica sull'edge-node

```bash
# Sulla edge-node — nginx è ancora attivo nonostante il cloud sia irraggiungibile
sudo ctr -n k8s.io containers ls
# docker.io/library/nginx:1.25-alpine   Running   ← ancora vivo!

curl http://10.244.0.5:80
# <!DOCTYPE html>... nginx risponde normalmente
```

### 3.5 Ripristina la connettività

```bash
# Sulla edge-node — rimuovi la regola iptables
sudo iptables -D OUTPUT -d <IP-CLOUD-NODE> -j DROP
echo 'Connettività ripristinata.'

# Dalla cloud-node — osserva la riconciliazione automatica (~10 secondi)
watch kubectl get nodes
# NAME         STATUS   ROLES
# cloud-node   Ready    control-plane
# edge-node    Ready    agent,edge     ← torna Ready automaticamente
```

> **✅ Lab 3 Completato** — Nessun intervento manuale necessario. EdgeHub ha re-stabilito il tunnel WebSocket, MetaManager ha inviato gli aggiornamenti in coda, l'API server è stato aggiornato automaticamente.

---

## Guida Esame — Comandi da Mostrare

Le VM Multipass rimangono salvate sul portatile tra una sessione e l'altra. Per l'esame basta riattivarle e rieseguire i comandi dimostrativi.

### Avvio rapido (VM già configurate)

```powershell
# PowerShell — avvia le VM (se erano sospese)
multipass start cloud-node
multipass start edge-node
multipass list   # verifica che siano Running

# Entra nella cloud-node
multipass shell cloud-node
```

> **⚠ Attenzione** — Se dopo il riavvio delle VM l'edge-node risulta `NotReady`, aspetta 30 secondi che EdgeCore si riconnetta automaticamente. Se non si riconnette: `sudo systemctl restart edgecore` sulla edge-node.

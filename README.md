# Cluster Kubernetes HA Multi-Nodo & Pipeline GitOps

Infrastruttura HA su 3 nodi Ubuntu Server gestita via Ansible da un PC locale. Kubernetes (Control Plane, etcd) e PostgreSQL HA sono distribuiti su tutti i nodi per la massima stabilità. Il nodo 1 gestisce la CI/CD (Jenkins, ArgoCD), il nodo 2 eroga le App Web e il nodo 3 cura l'osservabilità (InfluxDB, Grafana) con Telegraf su ogni nodo.

L'infrastruttura è composta da 3 nodi fisici/virtuali Ubuntu Server coordinati dal PC locale di gestione:

```text
===================================================================
                       [ TUO PC PRINCIPALE ]
   Ansible Playbook | Git Push | kubectl | Web Browsing
===================================================================
          |                      |                      |
          v                      v                      v
.--------------------. .--------------------. .--------------------.
|       node-1       | |       node-2       | |       node-3       |
|   192.168.1.101    | |   192.168.1.102    | |   192.168.1.103    |
|--------------------| |--------------------| |--------------------|
| * ControlPlane     | | * ControlPlane     | | * ControlPlane     |
| * etcd             | | * etcd             | | * etcd             |
| * Postgres-ha1     | | * Postgres-ha2     | | * Postgres-ha3     |
| * ArgoCD           | | * App Web Pod      | | * InfluxDB         |
| * Jenkins          | | * Telegraf         | | * Grafana          |
| * Telegraf         | |                    | | * Telegraf         |
'--------------------' '--------------------' '--------------------'



🖥️ FASE 1: Configurazione di Base dei Nodi Ubuntu Server

Questa fase va eseguita su ciascuno dei 3 nodi (node-1, node-2, node-3) prima di avviare le automazioni.
Step 1.1: Configurazione IP Statico con Netplan
Modifica il file di rete di Netplan per assegnare l'IP statico al nodo:

Bash
sudo nano /etc/netplan/00-installer-config.yaml
Inserisci la configurazione (adatta il nome dell'interfaccia, es. eth0 o enp1s0, in base al tuo sistema):

YAML
----------
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: no
      addresses:
        - 192.168.1.101/24  # Su node-2 usa .102, su node-3 usa .103
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8

Applica la configurazione:

Bash
sudo netplan apply

Step 1.2: Configurazione /etc/hosts
Sui 3 nodi e sul tuo PC di gestione, aggiungi la mappatura dei nomi:

Bash
sudo nano /etc/hosts

192.168.1.101 node-1
192.168.1.102 node-2
192.168.1.103 node-3

Step 1.3: Abilitazione sudo senza password (visudo)
Per permettere ad Ansible di eseguire i comandi amministrativi in modo trasparente, aggiungi la regola NOPASSWD per l'utente server1,2,3 accedi in ssh:

Bash
sudo visudo

Aggiungi questo in fondo alla fine del file:

server1 ALL=(ALL) NOPASSWD: ALL

al server 2 accedi in ssh 
server2 ALL=(ALL) NOPASSWD: ALL

al server3 accedi in ssh 
server3 ALL=(ALL) NOPASSWD: ALL

Step 1.4: Disabilitazione Swap e Configurazione Moduli Kernel
K3s e Kubernetes richiedono che la memoria Swap sia disabilitata e che i moduli di bridging/overlay siano caricati:

Bash
# 1. Disabilita la swap nell'immediato e al riavvio
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# 2. Carica i moduli del kernel richiesti per il networking di Kubernetes
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/k3s.conf
overlay
br_netfilter
EOF

# 3. Configura i parametri sysctl per il bridging di rete
cat <<EOF | sudo tee /etc/sysctl.d/99-k3s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system

🔑 FASE 2: Autenticazione SSH Key-Based dal PC Locale
Sul TUO PC PRINCIPALE, crea ed invia le chiavi ai 3 nodi per consentire la connessione remota ad Ansible senza digitare la password:

Bash
# 1. Generazione della chiave SSH ED25519
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519

# 2. Copia della chiave pubblica sui 3 nodi
ssh-copy-id server1@192.168.1.101
ssh-copy-id server1@192.168.1.102
ssh-copy-id server1@192.168.1.103

☸️ FASE 3: Deployment Cluster K3s HA (etcd) e Setup kubectl
Installazione del cluster K3s con etcd embedded distribuito sui 3 nodi per garantire l'Alta Affidabilità del Control Plane.
Step 3.1: Configurazione Inventario Ansible (hosts.ini)
Sul TUO PC PRINCIPALE, crea o verifica la cartella del progetto:

Bash
mkdir -p ~/k3s-gitops-lab/ansible
cd ~/k3s-gitops-lab/ansible

nano hosts.ini

Inserisci il file di inventario:


[master_first]
node-1 ansible_host=192.168.1.101 ansible_user=server1

[master_others]
node-2 ansible_host=192.168.1.102 ansible_user=server1
node-3 ansible_host=192.168.1.103 ansible_user=server1

[k3s_cluster:children]
master_first
master_others


Step 3.2: Playbook Ansible di Installazione (preparazione-e-installare-k3s.yml)
Crea il playbook per automatizzare l'installazione di K3s:

Bash

nano preparazione-e-installare-k3s.yml

YAML
---
- name: Inizializzazione Primo Nodo Master K3s (cluster-init)
  hosts: master_first
  become: true
  tasks:
    - name: Inizializza K3s Server con etcd
      shell: |
        curl -sfL https://get.k3s.io | K3S_TOKEN="secret-cluster-token" exec sh -s - server \
          --cluster-init \
          --node-ip=192.168.1.101 \
          --tls-san=192.168.1.101
      args:
        creates: /usr/local/bin/k3s

- name: Join degli Altri Nodi Master al Cluster etcd
  hosts: master_others
  become: true
  tasks:
    - name: Esegui Join a K3s HA
      shell: |
        curl -sfL https://get.k3s.io | K3S_TOKEN="secret-cluster-token" exec sh -s - server \
          --server https://192.168.1.101:6443 \
          --node-ip={{ ansible_host }} \
          --tls-san={{ ansible_host }}
      args:
        creates: /usr/local/bin/k3s


Esegui il playbook dal PC locale:

Bash
ansible-playbook -i hosts.ini preparazione-e-installare-k3s.yml

Step 3.3: Recupero e Configurazione del File kubeconfig , Scarica il file di configurazione dal primo nodo per gestire il cluster direttamente dal tuo PC principale tramite kubectl:

Bash
# 1. Copia il file sul nodo remoto con permessi letti dall'utente server1
ssh server1@192.168.1.101 "sudo cp /etc/rancher/k3s/k3s.yaml ~/k3s.yaml && sudo chown server1:server1 ~/k3s.yaml"

# 2. Scarica il file nel PC locale
mkdir -p ~/.kube
scp server1@192.168.1.101:~/k3s.yaml ~/.kube/config

# 3. Pulisci il file temporaneo sul nodo
ssh server1@192.168.1.101 "rm ~/k3s.yaml"

# 4. Aggiorna l'IP da 127.0.0.1 a 192.168.1.101 e imposta i permessi corretti
sed -i 's/127.0.0.1/192.168.1.101/g' ~/.kube/config
chmod 600 ~/.kube/config

Verifica lo stato di salute dei nodi:
Bash
kubectl get nodes -o wide
Esito atteso: Tutti e 3 i nodi mostrano STATUS: Ready e ruoli control-plane,master.

🗄️ FASE 4: Deployment PostgreSQL HA (CloudNativePG)
Configurazione dell'operator CloudNativePG per gestire un cluster database a 3 istanze (1 Primary + 2 Repliche) con failover automatico.

Step 4.1: Installazione Operator CloudNativePG
Bash
kubectl apply --server-side -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.22/releases/cnpg-1.22.1.yaml

Attendi il completo avvio del pod del controller:
Bash
kubectl get pods -n cnpg-system

Step 4.2: Deploy del Cluster PostgreSQL
Crea il file di configurazione per il cluster database:
Bash
cd ~/k3s-gitops-lab
---
nano postgres-cluster.yaml
---
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgres-ha
  namespace: default
spec:
  instances: 3
  storage:
    size: 5Gi
  bootstrap:
    initdb:
      database: appdb
      owner: appuser
---

kubectl apply -f postgres-cluster.yaml

Verifica lo stato del database:
Bash
kubectl get cluster postgres-ha

Esito atteso: INSTANCES: 3, READY: 3, STATUS: Cluster in healthy state.

🔄 FASE 5: Installazione della Suite CI/CD (Jenkins & ArgoCD)
Step 5.1: Deployment ArgoCD (Continuous Delivery - GitOps)

Bash
# 1. Creazione namespace e applicazione dei manifest ufficiali
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. Recupero della password di amministrazione
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

• Accesso Web UI: Esegui dal PC locale kubectl port-forward svc/argocd-server -n argocd 8080:443 e naviga su https://localhost:8080.
    • Credenziali: User admin | Password ottenuta dal comando sopra.

Step 5.2: Deployment Jenkins (Continuous Integration)

Bash
# 1. Creazione Namespace
kubectl create namespace jenkins

# 2. Manifest Deployment e Service
nano ~/k3s-gitops-lab/jenkins.yaml

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts
        ports:
        - containerPort: 8080
        - containerPort: 50000
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: jenkins
spec:
  type: NodePort
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30800
  selector:
    app: jenkins


kubectl apply -f ~/k3s-gitops-lab/jenkins.yaml

# 3. Recupero token di sblocco iniziale
kubectl logs -n jenkins deployment/jenkins | grep -A 5 "Jenkins initial setup is required"
    • Accesso Web UI: Naviga su [http://192.168.1.101:30800](http://192.168.1.101:30800) ed inserisci la chiave alfanumerica estratta dai log per sbloccare i plugin consigliati.

🚀 FASE 6: Configurazione dell'Applicazione GitOps su ArgoCD
Applica la risorsa Application in ArgoCD per avviare la sincronizzazione automatica da Git con tolleranza alle modifiche manuali (selfHeal: true).

Bash
nano ~/k3s-gitops-lab/argocd-app.yaml

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: k3s-gitops-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/argoproj/argocd-example-apps.git'
    targetRevision: HEAD
    path: guestbook
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true


kubectl apply -f ~/k3s-gitops-lab/argocd-app.yaml

Verifica lo stato dell'applicazione:

Bash
kubectl get application -n argocd
Esito atteso: SYNC STATUS: Synced e HEALTH STATUS: Healthy.

📊 FASE 7: Monitoraggio Observability (TIG Stack: Telegraf + InfluxDB + Grafana)
Installazione della suite per raccogliere metriche time-series e visualizzarle tramite dashboard.
Step 7.1: Namespace & InfluxDB Deployment

Bash
kubectl create namespace monitoring
nano ~/k3s-gitops-lab/influxdb.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: influxdb
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: influxdb
  template:
    metadata:
      labels:
        app: influxdb
    spec:
      containers:
      - name: influxdb
        image: influxdb:1.8
        ports:
        - containerPort: 8086
---
apiVersion: v1
kind: Service
metadata:
  name: influxdb
  namespace: monitoring
spec:
  ports:
  - port: 8086
    targetPort: 8086
  selector:
    app: influxdb


kubectl apply -f ~/k3s-gitops-lab/influxdb.yaml

Step 7.2: Deploy Telegraf DaemonSet
Deploy dell'agente distribuito su tutti e 3 i nodi del cluster per raccogliere metriche hardware (CPU, RAM, Disco):
Bash
nano ~/k3s-gitops-lab/telegraf.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: telegraf-config
  namespace: monitoring
data:
  telegraf.conf: |
    [agent]
      interval = "10s"
      round_interval = true
    [[outputs.influxdb]]
      urls = ["http://influxdb.monitoring.svc:8086"]
      database = "k3s_metrics"
    [[inputs.cpu]]
      percpu = true
      totalcpu = true
    [[inputs.mem]]
    [[inputs.disk]]
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: telegraf
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: telegraf
  template:
    metadata:
      labels:
        app: telegraf
    spec:
      containers:
      - name: telegraf
        image: telegraf:latest
        volumeMounts:
        - name: config
          mountPath: /etc/telegraf
      volumes:
      - name: config
        configMap:
          name: telegraf-config


kubectl apply -f ~/k3s-gitops-lab/telegraf.yaml
Step 7.3: Deploy Grafana
Bash
nano ~/k3s-gitops-lab/grafana.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:latest
        ports:
        - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
spec:
  type: NodePort
  ports:
  - port: 3000
    targetPort: 3000
    nodePort: 30300
  selector:
    app: grafana


kubectl apply -f ~/k3s-gitops-lab/grafana.yaml
    • Accesso Web Grafana: Naviga su [http://192.168.1.101:30300](http://192.168.1.101:30300) (User: admin | Password: admin).
    • Configurazione Data Source:
        1. Vai su Connections > Data Sources > Add Data Source.
        2. Seleziona InfluxDB.
        3. Imposta URL: [http://influxdb.monitoring.svc:8086](http://influxdb.monitoring.svc:8086) e Database: k3s_metrics.
        4. Clicca su Save & Test.


Fix e Configurazione Observability (TIG Stack)
1. Diagnostica del Database e delle Metriche
Abbiamo verificato che l'agente Telegraf stesse scrivendo correttamente le metriche nel database InfluxDB k3s_metrics:

Bash
kubectl exec -it -n monitoring deployment/influxdb -- influx -database k3s_metrics -execute "SHOW MEASUREMENTS"

Esito: Confermata la presenza delle metriche base (cpu, disk, mem).
2. Risoluzione dei Valori "N/A" su Grafana
I widget della dashboard (Uptime, RAM, Processi, Load Average) mostravano N/A o No Data per due motivi:
    1. Mancavano alcuni plugin d'ingresso nel file di configurazione di Telegraf.
    2. La variabile di selezione Server in cima alla dashboard di Grafana non era impostata.
3. Estensione dei Moduli Telegraf (telegraf.yaml)

Abbiamo aggiornato la ConfigMap di Telegraf aggiungendo gli input di sistema (system, processes, swap):

Bash
nano ~/k3s-gitops-lab/telegraf.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: telegraf-config
  namespace: monitoring
data:
  telegraf.conf: |
    [agent]
      interval = "10s"
      round_interval = true
    [[outputs.influxdb]]
      urls = ["http://influxdb.monitoring.svc:8086"]
      database = "k3s_metrics"
    [[inputs.cpu]]
      percpu = true
      totalcpu = true
    [[inputs.mem]]
    [[inputs.disk]]
    [[inputs.system]]
    [[inputs.processes]]
    [[inputs.swap]]
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: telegraf
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: telegraf
  template:
    metadata:
      labels:
        app: telegraf
    spec:
      containers:
      - name: telegraf
        image: telegraf:latest
        volumeMounts:
        - name: config
          mountPath: /etc/telegraf
      volumes:
      - name: config
        configMap:
          name: telegraf-config


# Applica e riavvia i pod del DaemonSet su tutti i nodi
kubectl apply -f ~/k3s-gitops-lab/telegraf.yaml
kubectl rollout restart daemonset telegraf -n monitoring

4. Selezione del Nodo su Grafana
Nella Dashboard di Grafana ([http://192.168.1.101:30300](http://192.168.1.101:30300)):
    1. È stato selezionato il server dal menu a tendina in alto (es. telegraf-hjn9r o All).
    2. Risultato finale: Tutti i contatori in tempo reale si sono popolati correttamente (CPU: 3.17%, RAM: 46.0%, RootFS: 68%, Uptime: 1.1 hours).




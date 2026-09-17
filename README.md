# K3s-GitOps-Observability-Platform
Infrastruttura HA su 3 nodi Ubuntu Server gestita via Ansible da un PC locale. Kubernetes (Control Plane, etcd) e PostgreSQL HA sono distribuiti su tutti i nodi per la massima stabilità. Il nodo 1 gestisce la CI/CD (Jenkins, ArgoCD), il nodo 2 eroga le App Web e il nodo 3 cura l'osservabilità (InfluxDB, Grafana) con Telegraf su ogni nodo.  

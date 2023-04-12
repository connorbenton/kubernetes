# Kubernetes Cluster Homelab
###3 Nodes Cluster Homelab with Metallb via BGP and Traefik as Ingress Controller. Cert Manager is used for automated Letsencypt certificate and DNS challenge over Cloudflare ###

Hardware Used:

   - Pfsense box
   - ASUS mini PC (Proxmox with 2 vm as worker nodes)
   - Raspberrypi 3B+ (Tainted Master Node)
   - Cisco switch
   - Ubiquity AP

This is a small project I created to learn kubernetes with only one Asus mini PC and an old Raspberrypi 3B+.
The Asus mini PC has 2x SSD installed and 32GB of RAM shared with Proxmox running 2 VM with Oracle Linux installed and 14GB RAM allocated for each node.
Since only one mini pc is not enough to run a HA cluster, I installed each node on separate SSD at least in case of a disk failure one of the node should still running.
All the pods persistent data is retained in a external 256GB NVME volume attached to the Asus mini pc via 3.1 gen2 USB, shared between the 2 worker nodes for easy backup and replacement in case of failure.
The external NVME is attached to Proxmox and the worker nodes as a NFS volume.
The Rasperrypi 3B+ has a 128GB SD card installed with raspbian OS that act as a tainted master node as all the pods are allocated in the workers nodes only.

---- INSTALLATION STEPS ----

Step 1:
##K3s installation##

curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik" sh -  (Traefik is installed later via helm with custom values)

----------------------------

Step 2:
###Metallnstallation with BGP values file##

kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.9/config/manifests/metallb-frr.yaml -f bgpconfig.yaml
(bgpconfig values includes pfsense neightbor configuration for the metallb frr yaml)

----------------------------

Step 3:
###Traefik installation###

1. kubectl create namespace traefik

2. helm install --namespace=traefik traefik traefik/traefik --values=values.yaml

----------------------------

Step 4:
###Certmanager Installation###

1. kubectl create namespace cert-manager

2. Install correct version:
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.9.1/cert-manager.crds.yaml

3. Apply values:
helm install cert-manager jetstack/cert-manager --namespace cert-manager --values=values.yaml --version v1.9.1

----------------------------

Step 5:
- Install Letsencrypt certificate for staging/production with cloudflare DNS

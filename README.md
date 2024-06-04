# K3s Mini Cluster Homelab
The project involves the configuration of a 3-node mini cluster, incorporating a Metallb load balancer via BGP and NGINX as the Ingress Controller. Cert Manager is employed to automate Let's Encrypt certificate management, utilizing the DNS challenge method over Cloudflare.

Hardware components:

- Router (eventually OPNSense applicance)
- Server/master node (also set up in this cluster to run pods, not just manage the cluster)
- Any number of agents (worker nodes)

This undertaking focuses on establishing a K3s cluster, where pods have PVs that are stored locally (to maximize performance). Future goal would be to move all persistent storage to one single location (either on a single pod, or via a distributed storage across all pods) and to benchmark to see if there are large performance drops when using such a network-based storage. Pods are configured to run on specific nodes, so that resource-heavy applications/pods can be located specifically on high-performance nodes, but another future goal would be to benchmark actual performance across nodes to see if there would be performance issues were one high perf node to fail and its pods passed to a lower perf node.

Original setup is to run everything in IPv4, but depending on ISP if IPv6 is instead accessible, K3S is set up as dual stack so that IPv6 can be configured as well. This repo contains all pods for a 'baseline' cluster setup, further pods (for example, ones that only come with a docker/docker-compose setup) can be set up via the 'generic docker app example' subdir.  

LoadBalancer/Ingress/SSL/Auth:
- Metallb
- Nginx
- Cert-manager 
- Authentik

Utilities/tools:
- Heimdall 
- Webtop

Applications Database:
- Mariadb

Monitoring/Logs:
- Grafana
- InfluxDB
- Prometheus
- Node Exporter
- Telegraf

Test webserver (to see if cluster is working correctly and accessible from outside):
- Nginx

CI/CD
- ArgoCD

----------------------------

---- INSTALLATION STEPS ----

Step 1:
  K3s installation

1. Install K3s on the master node :
```curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik,servicelb" sh -```  (Traefik and metallb will be installed later with custom values, IPv6 must be enabled now otherwise it cannot be added later!)
(If needed, copy /etc/rancher/k3s/k3s.yaml to your user area ~/.kube/config (permissions 600), and add ```export KUBECONFIG=~/.kube/config``` to your ~/.bashrc.

2. Get your token for worker nodes deployment:
```cat /var/lib/rancher/k3s/server/node-token```

3. Install K3s on the worker nodes:
```curl -sfL https://get.k3s.io | K3S_URL=https://myserver:6443 K3S_TOKEN=mynodetoken sh -```

https://docs.k3s.io/quick-start

----------------------------

Step 2:
   Install Metallb desired version with BGP values file 

```kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.9/config/manifests/metallb-frr.yaml -f bgpconfig.yaml```
(bgpconfig values includes pfsense neightbor configuration for the metallb frr yaml)

https://metallb.universe.tf/installation/

----------------------------

Step 3:
   Traefik installation via helm with customized values

1. ```helm repo add traefik https://traefik.github.io/charts```

2. ```helm repo update``` 

3. ```kubectl create namespace traefik```

4. ```helm install --namespace=traefik traefik traefik/traefik -f values.yaml``` 
   
https://artifacthub.io/packages/helm/traefik/traefik

----------------------------

Step 4:
   Certmanager Installation

1. ```kubectl create namespace cert-manager```

2. Add helm repo:
```helm repo add jetstack https://charts.jetstack.io```

3. Update helm repo:
```helm repo update```

4. Install certmanager and apply values with cdrs:
```helm install cert-manager jetstack/cert-manager --namespace cert-manager --values=values.yaml --version v1.9.1```

https://artifacthub.io/packages/helm/cert-manager/cert-manager

----------------------------

Step 5:
   Deploy Cert-manager secret and Letsencrypt certificate for staging/production with cloudflare DNS
   
1. Install ```certmanager-secret.yaml``` manifest with your DNS Cloudflare token

2. Deploy  ```prod-deploy.yaml``` and ```yourdomain-prod-deploy.yaml``` with your chosen domain (for testing purposes, you can use the ```staging-deploy.yaml``` and ```yourdomain-stage-deploy.yaml```)

How to create a DNS token with Cloudflare: 
https://cert-manager.io/docs/configuration/acme/dns01/cloudflare/

----------------------------

Step 6 (Optional):
   Authentik installation via helm with customized values

1. ```helm repo add authentik https://charts.goauthentik.io```
2. ```helm repo update```
3. ```helm install authentik authentik/authentik -f values.yaml```

https://goauthentik.io/docs/installation/kubernetes
https://artifacthub.io/packages/helm/goauthentik/authentik

----------------------------

Step 7 (Optional):
   Install ArgoCD via helm with customized values
   
1. ```helm repo add argo https://argoproj.github.io/argo-helm```
2. ```helm repo update```
3. ```kubectl create namespace argocd```
4. ```helm install argocd argo/argo-cd  --namespace argocd -f argocd-values.yaml```
5. Apply traefik ingress

https://argo-cd.readthedocs.io/en/stable/getting_started/

----------------------------

Step 8 (Optional):
   Install keel via helm with customized values for automatic container image updates
   
1. ```helm repo add keel https://charts.keel.sh ```
2. ```helm repo update```
3. ```helm install keel --namespace=kube-system keel/keel -f keel-values.yaml```
4. Add annotations for keel as needed on your deployment yaml

https://keel.sh/docs/#deploying-with-kubectl

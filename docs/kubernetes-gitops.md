# Kubernetes GitOps Workflow met ArgoCD en k3s

## Inleiding

In dit document wordt beschreven hoe ik een Kubernetes cluster heb opgezet met **k3s** en hoe ik **ArgoCD** heb geïnstalleerd om een applicatie automatisch te beheren via **GitOps**.  

Met **GitOps** wordt de volledige infrastructuur en applicatieconfiguratie beheerd via een **Git-repository**. Dit zorgt voor **automatische synchronisatie**, versiebeheer en een consistente omgeving.

---

## 1. Vereisten

### **1.1 Hardware**
- **Raspberry Pi Cluster**: Dit project draait op **Raspberry Pi's 5 en 4** met Ubuntu server OS.
- **Minimaal 4GB RAM per node** (voor de Control Plane heb ik 8GB RAM).

### **1.2 Software**
- **k3s** (lichtgewicht Kubernetes distro, ideaal voor Raspberry Pi).
- **ArgoCD** (voor GitOps based applicatiemanagement).
- **Docker Hub** (om container images te hosten).
- **GitHub** (voor versiebeheer van Kubernetes manifests en ArgoCD configuratie).

### **1.3 Tools & CLI's**
- **kubectl** (om te communiceren met Kubernetes).
- **Docker CLI** (voor het bouwen en pushen van Docker images).
- **ArgoCD CLI en UI** (voor het beheren van ArgoCD applicaties).
- **GitHub Actions** (voor CI/CD automatisering).

---

## 2. Installeren van k3s

### **2.1 Installeren van de k3s Kubernetes Cluster**
**Op de Control plane (eerste Raspberry Pi)**:
```bash
curl -sfL https://get.k3s.io | sh -
```

Dit installeert k3s en start de cluster automatisch.

Om de kubeconfig beschikbaar te maken voor kubectl:
```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

Op een worker node (extra Raspberry Pi’s):

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<control-plane-ip>:6443 K3S_TOKEN=<secret-token> sh -
```
Je vindt de <secret-token> op de control plane in:
```bash
cat /var/lib/rancher/k3s/server/node-token
```

### **2.2 Controleren of Kubernetes draait**
```bash
kubectl get nodes
```
Output:
NAME           STATUS   ROLES                  AGE   VERSION
controlplane   Ready    control-plane,master   13d   v1.31.5+k3s1
node1          Ready    <none>                 13d   v1.31.5+k3s1
node2          Ready    <none>                 13d   v1.31.5+k3s1

## 3. Installeren van ArgoCD
ArgoCD is een GitOps tool die ervoor zorgt dat onze Kubernetes-applicaties automatisch gesynchroniseerd worden met Git.

### **3.1 Installeren van ArgoCD en ArgoCD CLI**
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```
### **3.2 Inloggen op ArgoCD**
ArgoCD gebruikt een standaard admin-wachtwoord. Haal het op met:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```
Forward de poort om toegang te krijgen tot de UI:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:80
```
### 4. Build en Push Docker Image naar Docker Hub
Mijne applicaties worden als Docker-container opgeslagen in Docker Hub.

### **4.1 Eerste applicatie: Docker Image Bouwen en Pushen**
```bash
sudo docker login
sudo docker buildx build --platform linux/amd64,linux/arm64 -t dockerhub-user/nginx-app:latest ./apps/nginx --push
```

## 5. Configureren van ArgoCD om de Applicatie te Beheren

Maak een ArgoCD Application aan via YAML:
```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-app
spec:
  destination:
    name: ''
    namespace: default
    server: 'https://kubernetes.default.svc'
  source:
    path: manifests/nginx
    repoURL: 'https://github.com/username/reponame'
    targetRevision: main
  project: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```
Voeg deze toe aan je Git-repo en push naar GitHub.

ArgoCD zal nu automatisch controleren of de configuratie in Git overeenkomt met wat er draait in Kubernetes.

## 6. Automatiseren met GitHub Actions
GitHub Actions kan Docker-images automatisch bouwen en pushen.

```yaml
---
name: Build and Push Docker Image

on:
  push:
    branches:
      - main
  workflow_dispatch: {}

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Log in to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_PASSWORD }}

      - name: Build and Push Docker Image for NGINX
        run: |
          docker build -t ${{ secrets.DOCKERHUB_USERNAME }}/nginx-app:latest \
          ./apps/nginx
          docker push ${{ secrets.DOCKERHUB_USERNAME }}/nginx-app:latest
```
Dit workflowbestand plaats je in .github/workflows/build-and-push.yml

## 8. Conclusie

Dit document beschrijft hoe ik Kubernetes met k3s, ArgoCD en GitOps heb opgezet.
Met deze workflow kan ik applicaties betrouwbaar implementeren en beheren via versiebeheer in Git.

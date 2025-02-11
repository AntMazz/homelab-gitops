# 🏠 Homelab GitOps with K3s, ArgoCD & GitLab CI/CD

## 🔹 Overview
This repo manages my **Raspberry Pi Kubernetes cluster** using **GitOps**.  
Everything is automated with **ArgoCD, GitLab CI/CD, and Helm**.  

## 🛠️ Tools Used
- Kubernetes (K3s)
- ArgoCD for GitOps
- GitHub Actions for automation
- Docker Hub (image registry)
- Helm for package management
- Prometheus & Grafana (Monitoring)

## 🚀 Deployment Workflow
1️⃣ **Developers push updates** to GitHub  
2️⃣ **GitHub Actions validate YAML & Helm charts**  
3️⃣ **ArgoCD detects changes** and deploys them  
4️⃣ **Kubernetes updates services** using the new images from Harbor  

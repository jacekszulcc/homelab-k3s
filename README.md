# 🚀 Hybrid Homelab & K3s High Availability Cluster

![Status](https://img.shields.io/badge/Status-Active-success?style=flat-square)
![Kubernetes](https://img.shields.io/badge/Kubernetes-K3s-blue?logo=kubernetes&style=flat-square)
![Infrastructure](https://img.shields.io/badge/Infrastructure-Hybrid-orange?style=flat-square)

Welcome to the configuration repository for my home infrastructure. This project demonstrates a production-grade **Kubernetes (K3s)** cluster running in **High Availability (HA)** mode on bare-metal hardware via Proxmox VE, integrated with an **Unraid** storage server.

The main goal of this project is to simulate a real-world DevOps environment using modern Cloud Native technologies.

## 🌐 Live Demo
My portfolio website hosted on this cluster: **[https://jacek.szulc.cc](https://jacek.szulc.cc)**

---

## 🏗️ Architecture Overview

The infrastructure is split into two logical segments: the Compute Cluster (K3s) and the Storage/Utility Server (Unraid).

### 1. Compute Layer (Proxmox VE Cluster)
Running on **2x Lenovo ThinkCentre M920q** nodes (i5-8500T, 32GB RAM each).
- **Hypervisor:** Proxmox VE 8.x (Clustered)
- **Orchestrator:** K3s (Lightweight Kubernetes)
- **Topology:** 3x Control Plane nodes + 2x Worker nodes (HA Etcd backend)
- **Networking:** Static IPs, internal bridge networking.

### 2. Storage & Utility Layer (Unraid)
Running on a custom server (i5-10400, 32GB RAM, ~16TB Storage).
- **Role:** Persistent Storage, Media Server, Network Utilities.
- **Services:** Nginx Proxy Manager, AdGuard Home (DNS), PostgreSQL, Vaultwarden, Arr-stack.

### 3. Connectivity & Security
- **Ingress:** Cloudflare Tunnel (Zero Trust) - No open ports on the router.
- **DNS:** Split-DNS architecture (AdGuard Home for local, Cloudflare for public).
- **SSL:** Fully automated via Cloudflare and Let's Encrypt.

---

## 🛠️ Tech Stack

| Category | Technology | Usage |
| :--- | :--- | :--- |
| **Orchestration** | **Kubernetes (K3s)** | Main cluster management |
| **Virtualization** | **Proxmox VE** | VM hosting for K8s nodes |
| **Storage** | **Unraid** | NFS/SMB storage, Docker containers |
| **Networking** | **Cloudflare Tunnel** | Secure external access |
| **GitOps** | **Git & GitHub** | Infrastructure as Code (IaC) source |
| **Web Server** | **Nginx** | Serving the portfolio application |
| **Config Mgmt** | **K8s ConfigMaps** | Managing application configuration |

---

## 📂 Repository Structure

```text
.
├── nauka/               # Learning environment & Portfolio app
│   ├── nginx-pro.yaml   # Deployment & Service definition
│   └── strona.yaml      # ConfigMap containing HTML source
├── cloudflare/          # Connectivity configs (Tunnels)
└── README.md            # Project documentation
```

## 🚀 Deployment Workflow

This project follows the **Infrastructure as Code** principles. To deploy changes to the portfolio website:

1. **Code:** Edit the HTML in `strona.yaml`.
2. **Apply:** Push changes to the cluster:
   ```bash
   kubectl apply -f nauka/strona.yaml
   ```
3. **Refresh:** Restart the deployment to pick up config changes:
   ```bash
   kubectl rollout restart deployment nginx-pro -n nauka
   ```

## 🔮 Roadmap

- [x] High Availability K3s Cluster setup
- [x] Cloudflare Tunnel integration
- [x] Portfolio website deployment
- [x] Unraid infrastructure integration
- [ ] **GitOps:** Implementation of ArgoCD for automated sync
- [ ] **Monitoring:** Prometheus & Grafana stack
- [ ] **Storage:** Longhorn implementation for K8s persistent volumes

---

*Created by **Jacek Szulc** | 2025*

# Hybrid Homelab & K3s High Availability Cluster

![Status](https://img.shields.io/badge/Status-Archived-lightgrey?style=flat-square)
![Kubernetes](https://img.shields.io/badge/Kubernetes-K3s-blue?logo=kubernetes&style=flat-square)
![Infrastructure](https://img.shields.io/badge/Infrastructure-Hybrid-orange?style=flat-square)

> **This project is finished and the cluster no longer exists.**
>
> The manifests and documentation in this repository date from **November 2025**. The cluster was decommissioned in 2026, when the hardware was repurposed for other tasks. Nothing here is running.
>
> The repository is kept as architecture documentation: what was built, how it was wired together, and why those particular choices were made.

This repository held the configuration for a self-hosted **Kubernetes (K3s)** cluster running in **High Availability** mode on bare-metal hardware under Proxmox VE, alongside an **Unraid** storage server. The goal was to learn Cloud Native tooling by running it on real hardware rather than in a managed environment.

---

## Architecture Overview

The infrastructure was split into two segments: a compute cluster (K3s on Proxmox) and a storage/utility server (Unraid).

### 1. Compute Layer (Proxmox VE)

Two **Lenovo ThinkCentre M920q** nodes, each with an Intel Core i5 T-series (35 W) CPU and 32 GB RAM.

- **Hypervisor:** Proxmox VE 8.x, clustered
- **Orchestrator:** K3s
- **Topology:** 3 control-plane nodes + 2 worker nodes, embedded etcd
- **Networking:** static IPs on `192.168.50.0/24`, internal bridge networking

The five K3s VMs were distributed across the two hosts as follows:

| Physical host | K3s nodes |
| :--- | :--- |
| **PVE1** | `k3s-server-01`, `k3s-server-03`, `k3s-worker-02` |
| **PVE2** | `k3s-server-02`, `k3s-worker-01` |

This distribution mattered more than the node count. See [Design decisions](#design-decisions).

### 2. Storage & Utility Layer (Unraid)

A custom-built server (Intel Core i5-10400, 32 GB RAM, ~16 TB storage).

- **Role:** persistent storage, media services, network utilities
- **Services (Docker):** Nginx Proxy Manager, AdGuard Home, PostgreSQL, Vaultwarden, Home Assistant, Arr stack

### 3. Connectivity & Security

- **Ingress:** Cloudflare Tunnel (Zero Trust), running as two `cloudflared` replicas in `kube-system`, with no inbound ports open on the router
- **DNS and certificates:** split-DNS (AdGuard Home for the local network, Cloudflare for public names) and TLS termination were configured **outside this repository**, in the Cloudflare dashboard and on the Unraid host. No configuration for either is stored here.

### 4. Monitoring

A monitoring stack was deployed on the cluster: **Prometheus**, **Grafana**, and **node_exporter** scraping the cluster nodes. Grafana panels were embedded directly into the portfolio page via `d-solo` iframes. Those `<iframe>` sources are still visible in [apps/strona.yaml](apps/strona.yaml) and are the remaining evidence of that stack. No Prometheus or Grafana manifests were committed to this repository; they were installed directly against the cluster.

**Stakater Reloader** was also deployed, to restart Deployments automatically when a mounted ConfigMap changed.

---

## Tech Stack

| Category | Technology | Usage |
| :--- | :--- | :--- |
| **Orchestration** | Kubernetes (K3s) | Cluster management |
| **Virtualization** | Proxmox VE | VM hosting for K8s nodes |
| **Storage** | Unraid | NFS/SMB storage, Docker services |
| **Networking** | Cloudflare Tunnel | External access without open ports |
| **Monitoring** | Prometheus, Grafana, node_exporter | Node and cluster metrics |
| **Config reload** | Stakater Reloader | Automatic rollout on ConfigMap change |
| **Web Server** | Nginx | Serving the static portfolio page |
| **Config Mgmt** | K8s ConfigMaps | Application content and configuration |

---

## Repository Structure

```text
.
├── apps/                # Application workloads
│   ├── nginx-pro.yaml   # Namespace, Deployment & Service definitions
│   └── strona.yaml      # ConfigMap containing the HTML source
├── cloudflare/          # Cloudflare Tunnel deployment
│   └── tunnel.yaml      # cloudflared Deployment (token redacted)
└── README.md
```

The Kubernetes namespace is still named `nauka` ("learning") in the manifests. That name is left as-is, because it records what actually ran on the cluster.

---

## Deployment Workflow

Changes to the portfolio page were applied by hand:

```bash
kubectl apply -f apps/strona.yaml
```

Initially a rollout restart was needed for Nginx to pick up the new ConfigMap contents. After Stakater Reloader was deployed, the `reloader.stakater.com/auto: "true"` annotation on the Deployment handled that automatically, and the manual restart was no longer part of the workflow.

---

## Design decisions

These are the reasons behind the architecture as it is recorded in this repository.

### Three control-plane nodes on two physical hosts, and why that was the weak point

K3s with embedded etcd needs an odd number of control-plane nodes to hold quorum, so the cluster ran three of them. With three servers, etcd tolerates the loss of one member.

The constraint was that there were only **two** physical hosts. Three control-plane VMs cannot be spread evenly across two machines: PVE1 ended up carrying two of them (`k3s-server-01` and `k3s-server-03`), PVE2 carrying one. Losing PVE2 was survivable. **Losing PVE1 took two etcd members down at once and the cluster lost quorum.** The control plane would have gone read-only until the host came back.

So the setup was formally HA at the Kubernetes level while remaining single-point-of-failure at the hardware level. The takeaway is that **HA topology is planned against physical failure domains, not against a count of virtual machines**. Genuine tolerance of one host failure would have required a third physical host (or an external/off-cluster etcd quorum member).

### Cloudflare Tunnel instead of port forwarding

Exposing the cluster through a Cloudflare Tunnel meant no inbound ports had to be opened on the home router, and the home IP address was never published in DNS. Certificate issuance and TLS termination sat with the provider, which removed a whole class of renewal and expiry work from the cluster.

The cost is a hard external dependency: the tunnel is the only ingress path, so a Cloudflare outage or an account problem takes the services offline, and there is no fallback route. `cloudflared` ran with two replicas to cover the failure of a single pod, but that is redundancy inside one dependency, not a second path.

### NodePort instead of an Ingress controller

The application was exposed with a Service of type `NodePort` on port `30090`, with no Ingress resource and no ingress controller in the repository. Because Cloudflare Tunnel was already terminating TLS and routing hostnames on the provider side, an in-cluster ingress controller would have duplicated routing that was configured elsewhere.

The trade-off is that the routing configuration then lived in the Cloudflare dashboard rather than in this repository. The manifests alone do not describe how a request reached the pod.

### A static site served from a ConfigMap

The portfolio page was stored as HTML inside a ConfigMap and mounted into the Nginx container, instead of being baked into a container image.

**This was a shortcut to make learning easier, not a production pattern.** It removed the image build and registry step entirely, so a content change was a single `kubectl apply`. The reasons not to do this in production are the reasons it was convenient here: content is not versioned as an immutable artifact, there is no build or scanning stage, ConfigMaps have a 1 MiB size limit, and application content ends up mixed into cluster configuration. A container image built in CI and referenced by tag is the correct approach.

---

## Scope and limitations

This was an educational project, and its limits should be stated plainly.

- **Provisioning was manual.** Proxmox VMs, the operating systems, and the K3s installation were all set up by hand. There is no Terraform, Ansible, cloud-init, or any other infrastructure automation anywhere in this repository.
- **Manifests were applied with `kubectl apply` from a workstation.** There was no pipeline and no automated validation.
- **This repository was not GitOps.** Keeping manifests in Git is version control, not GitOps. GitOps requires a controller running in the cluster that continuously reconciles live state against the repository and corrects drift. There was no such controller, so nothing prevented the cluster from diverging from what is committed here, and in fact it did: Prometheus, Grafana, and Reloader all ran on the cluster without ever being represented in this repository.
- **Cluster state was not backed up.** No etcd snapshot policy and no Velero.
- **Not everything that ran is captured here.** The manifests cover the application and the tunnel; the rest of the cluster was configured directly.

---

## Not implemented

- **Longhorn.** Distributed block storage for Kubernetes persistent volumes was planned but never deployed. No workload in this repository requested a PersistentVolume; the only state was the HTML in a ConfigMap.

**ArgoCD** was evaluated during the project (one commit in this repository's history was made specifically to test a sync), but no ArgoCD manifests or `Application` resources remain here, and continuous reconciliation was never part of the working setup.

The project was wound down before these were taken further.

---

## Status

The cluster was decommissioned deliberately, not abandoned. It had served its purpose as a learning environment, and the two ThinkCentre nodes were more useful assigned to other work than kept running an idle Kubernetes control plane.

The Unraid server is still in service and still hosts the workloads listed above, including a self-hosted **n8n** instance used for automation.

---

*Jacek Szulc | 2025*

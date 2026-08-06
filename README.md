# Hybrid Homelab & K3s High Availability Cluster

![Status](https://img.shields.io/badge/Status-Archived-lightgrey?style=flat-square)
![Kubernetes](https://img.shields.io/badge/Kubernetes-K3s-blue?logo=kubernetes&style=flat-square)
![Infrastructure](https://img.shields.io/badge/Infrastructure-Hybrid-orange?style=flat-square)

> **Two layers, two different statuses.**
>
> The **K3s cluster** documented here no longer exists. It was decommissioned in 2026 and its hardware was repurposed; the manifests and documentation date from **November 2025**.
>
> The **Unraid** storage and utility layer described alongside it is a separate machine, and it is still in service.
>
> This repository is kept as architecture documentation: what was built, how it was wired together, and why those particular choices were made.

| Layer | Hardware | Status |
| :--- | :--- | :--- |
| **Compute** (K3s on Proxmox VE) | 2x Lenovo ThinkCentre M920q | **Decommissioned.** Shut down in 2026, hardware reassigned. |
| **Storage & utility** (Unraid) | Custom build, Intel Core i5-10400 | **In service.** Hosts an n8n container; no active workflows yet. |

This repository held the configuration for a self-hosted **Kubernetes (K3s)** cluster running in **High Availability** mode on bare-metal hardware under Proxmox VE, alongside an **Unraid** storage server. The goal was to learn Cloud Native tooling by running it on real hardware rather than in a managed environment.

The two layers were built together but did not depend on each other, which is why one could be shut down while the other kept running. The n8n container described under [Automation environment](#automation-environment) was never part of the Kubernetes cluster; it runs on the Unraid host, and shutting the cluster down did not affect it.

---

## Architecture Overview

The infrastructure was split into two segments: a compute cluster (K3s on Proxmox) and a storage/utility server (Unraid).

### 1. Compute Layer (Proxmox VE) [decommissioned]

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

### 2. Storage & Utility Layer (Unraid) [still in service]

A custom-built server (Intel Core i5-10400, 32 GB RAM, ~16 TB storage). This layer was never part of the Kubernetes cluster and was not affected by its shutdown. It is the layer that still runs today.

- **Role:** persistent storage, media services, network utilities, Docker host for n8n
- **Services (Docker):** **n8n**, Nginx Proxy Manager, AdGuard Home, PostgreSQL, Vaultwarden, Home Assistant, Arr stack

### 3. Connectivity & Security

- **Ingress:** Cloudflare Tunnel (Zero Trust), running as two `cloudflared` replicas in `kube-system`, with no inbound ports open on the router
- **DNS and certificates:** split-DNS (AdGuard Home for the local network, Cloudflare for public names) and TLS termination were configured **outside this repository**, in the Cloudflare dashboard and on the Unraid host. No configuration for either is stored here.

### 4. Monitoring

A monitoring stack was deployed on the cluster: **Prometheus**, **Grafana**, and **node_exporter** scraping the cluster nodes. Grafana panels were embedded directly into the portfolio page via `d-solo` iframes. Those `<iframe>` sources are still visible in [apps/strona.yaml](apps/strona.yaml) and are the remaining evidence of that stack. No Prometheus or Grafana manifests were committed to this repository; they were installed directly against the cluster.

**Stakater Reloader** was also deployed, to restart Deployments automatically when a mounted ConfigMap changed.

---

## Automation environment

**n8n** runs as a Docker container on the Unraid host, separate from the Kubernetes cluster. It was never part of that cluster, so decommissioning K3s did not affect it, and the container is still up.

**There are no active workflows in this instance yet.** What exists is the environment rather than a running automation system: the container is deployed on the Unraid host and ready for workflows that are still being built. Nothing is executing on a schedule and nothing is processing traffic today.

**No n8n configuration is stored in this repository.** The container is managed through the Unraid Docker interface. What this repository documents is the infrastructure, not workflow definitions.

### Access

n8n is not exposed publicly. It is reachable from the LAN or over VPN only. There is no Cloudflare Tunnel and no reverse proxy in front of this service, so it is not routable from the internet at all. Authentication on the instance itself is basic auth.

### Data and backups

The instance uses n8n's default SQLite database, held in the Unraid appdata volume.

Backups are handled by the **Appdata Backup** plugin on Unraid:

- **Method:** each container is stopped, backed up, and started again, so the copy is consistent rather than taken from a live volume
- **Sources:** `/mnt/user/appdata` and `/mnt/cache/appdata`, which covers every container on the host, n8n included
- **Destination:** `/mnt/user/backups_unraid/daily`
- **Retention:** copies older than 7 days are removed, with a minimum of 3 kept
- **Compression:** enabled
- **Also captured:** the Unraid boot flash drive and VM metadata
- **Notifications:** on failure only

### Credentials

n8n encrypts stored credentials with a key held in `N8N_ENCRYPTION_KEY`. That variable is not set explicitly, so n8n generated the key itself and keeps it in the appdata volume alongside the database. Vaultwarden is running on the same host but is not used for this.

No credentials are stored in the instance today, because there are no workflows to hold them.

See [Scope and limitations](#scope-and-limitations) for what this arrangement does not cover.

---

## Tech Stack

| Category | Technology | Usage |
| :--- | :--- | :--- |
| **Orchestration** | Kubernetes (K3s) | Cluster management |
| **Virtualization** | Proxmox VE | VM hosting for K8s nodes |
| **Storage** | Unraid | NFS/SMB storage, Docker host |
| **Automation** | n8n (Docker, on Unraid) | Automation environment; no active workflows yet |
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

### n8n environment on Unraid

- **The backups stay on the machine they came from.** They are written to `/mnt/user/backups_unraid/daily`, on the same Unraid server that holds the originals. There is no copy anywhere else, so a failure of the array destroys the original and the backup at the same time.
- **Restoring from a backup has never been tested.** The job runs and produces archives, but no archive has ever been restored, so the recovery path is unverified.
- **The n8n encryption key sits in the same volume as the database**, and the same backup captures both. That makes a restore straightforward, and it also makes the backup archive sensitive material: whatever can read it holds the encrypted credentials and the key that opens them. This has no consequence today, because no credentials are stored. It starts to matter with the first connected integration.

---

## Not implemented

- **Longhorn.** Distributed block storage for Kubernetes persistent volumes was planned but never deployed. No workload in this repository requested a PersistentVolume; the only state was the HTML in a ConfigMap.

**ArgoCD** was evaluated during the project (one commit in this repository's history was made specifically to test a sync), but no ArgoCD manifests or `Application` resources remain here, and continuous reconciliation was never part of the working setup.

The project was wound down before these were taken further.

---

## What I would do differently

**Size the platform to the workload.** The only things this cluster ever served were a static page and a monitoring stack that largely existed to watch the cluster itself. A five-node HA Kubernetes cluster is a great deal of platform for that. The workload I actually intend to run, automation in a single n8n container, needs a Docker host and nothing more.

**Plan high availability around physical failure domains.** Three control-plane VMs spread across two physical machines is high availability on paper only, for the reasons set out under [Design decisions](#design-decisions). With two hosts available, there were two honest options: run a single control-plane node and describe it as such, or add a third machine. Splitting three VMs across two hosts produced the appearance of HA without the property.

**Keep the whole platform in the repository, or do not call it Infrastructure as Code.** Prometheus, Grafana and Stakater Reloader all ran on this cluster without ever appearing in these manifests. A repository that describes only part of a system cannot rebuild that system, and the gap is invisible until you need it.

**Why the cluster was retired and the Unraid layer was not.** The cluster was built to learn Kubernetes, and it served that purpose. Once the ThinkCentre hardware was needed for other tasks, keeping an idle control plane running was not worth the power and maintenance it cost. The Unraid host stayed because the services my household depends on run on it, and because it is where the automation work continues. The lesson I take from that split is to be clear about which parts of an environment exist to learn from and which exist to do work, because the two justify their upkeep differently.

---

## Status

The cluster was decommissioned deliberately, not abandoned. It had served its purpose as a learning environment, and the two ThinkCentre nodes were more useful assigned to other work than kept running an idle Kubernetes control plane.

The Unraid server is still in service and continues to host the workloads listed above.

---

*Jacek Szulc | 2025*

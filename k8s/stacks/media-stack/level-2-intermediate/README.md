# Level 2 (Intermediate): Production Homelab Architecture with Ingress & Kustomize

This directory implements the **Level 2 (Intermediate)** standard for the homelab media stack. It eliminates manual port mappings, adds automated health monitoring, enforces native Kubernetes user security, provisions dynamic storage, and organizes the deployment using **Kustomize base & overlays**.

---

## What's New in Level 2 vs. Level 1

| Feature | Level 1 (Beginner) | Level 2 (Intermediate) |
| :--- | :--- | :--- |
| **Networking** | 10+ NodePorts (`:30080`, `:30898`, etc.) | Unified **Ingress Controller** with custom domains (`*.home.lab`) |
| **Service Type** | NodePort | **ClusterIP** (Internal only) + Ingress |
| **Configuration** | Static raw manifests | **Kustomize** (`base/` and `overlays/homelab/`) |
| **Health Monitoring** | Basic Docker restarts | Kubernetes **Liveness & Readiness Probes** for self-healing |
| **Permissions** | Container env `PUID`/`PGID` | Native Pod **`securityContext`** (`runAsUser`, `fsGroup`) |
| **Resource Control** | Unbounded CPU/RAM | Defined **`resources.requests`** & **`resources.limits`** |
| **Storage** | Static `hostPath` | Dynamic **StorageClass** with `ReadWriteMany` (RWX) |

---

## Directory Structure

```text
k8s/stacks/media-stack/level-2-intermediate/
├── base/
│   ├── namespace.yaml
│   ├── storage.yaml            # Dynamic PVCs (RWX for data, RWO for config)
│   ├── ingress.yaml            # Unified routing table for all web UIs
│   ├── apps/                   # Microservices with Probes, securityContext & Limits
│   │   ├── dashy.yaml
│   │   ├── stremio.yaml
│   │   ├── qbittorrent.yaml
│   │   ├── sabnzbd.yaml
│   │   ├── radarr.yaml
│   │   ├── sonarr.yaml
│   │   ├── prowlarr.yaml
│   │   ├── bazarr.yaml
│   │   ├── jellyseerr.yaml
│   │   ├── emby.yaml
│   │   ├── unpackerr.yaml
│   │   ├── profilarr.yaml
│   │   ├── qui.yaml
│   │   └── flaresolverr.yaml
│   └── kustomization.yaml
└── overlays/
    └── homelab/
        ├── kustomization.yaml  # Environment orchestrator
        ├── common.env          # Non-sensitive variables
        ├── secrets.env.example # Secret generator template
        └── patches/
            └── storageclass-patch.yaml # Custom StorageClass patch
```

---

## Ingress Hostname Directory

All web traffic routes through your cluster's Ingress controller (e.g., Traefik or NGINX Ingress):

| Domain Name | Target Service | Purpose |
| :--- | :--- | :--- |
| `dashy.home.lab` | `dashy:8080` | Dashboard Homepage |
| `qbit.home.lab` | `qbittorrent:8080` | Torrent Downloader WebUI |
| `sabnzbd.home.lab` | `sabnzbd:8080` | Usenet Downloader WebUI |
| `radarr.home.lab` | `radarr:7878` | Movies Automation |
| `sonarr.home.lab` | `sonarr:8989` | TV Shows Automation |
| `prowlarr.home.lab` | `prowlarr:9696` | Indexers & Trackers |
| `bazarr.home.lab` | `bazarr:6767` | Subtitles Downloader |
| `jellyseerr.home.lab` | `jellyseerr:5055` | Media Requests |
| `emby.home.lab` | `emby:8096` | Media Streaming Server |
| `stremio.home.lab` | `stremio-server:11470` | Stremio Media Server |
| `profilarr.home.lab` | `profilarr:6868` | Configuration Profile Sync |
| `qui.home.lab` | `qui:7476` | Autobrr Web UI |

> [!NOTE]
> `qbittorrent` maintains a dedicated NodePort (`30681` TCP / `30682` UDP) solely for BitTorrent peer-to-peer traffic so that peers can reach you outside the HTTP Ingress.

---

## Prerequisites & DNS Setup

To resolve `*.home.lab` hostnames on your network:

1. **Local DNS (Recommended)**: In Pi-hole, AdGuard Home, or pfSense/OPNsense, create a wildcard DNS rewrite:
   ```text
   *.home.lab  ->  <YOUR_KUBERNETES_INGRESS_IP>
   ```
2. **Local `/etc/hosts` (Quick Testing)**:
   Add entries on your client machine:
   ```text
   <INGRESS_IP> dashy.home.lab qbit.home.lab sonarr.home.lab radarr.home.lab emby.home.lab
   ```

---

## Deployment Guide

### 1. Configure Secrets
Navigate to the homelab overlay and copy the secrets template:
```bash
cd k8s/stacks/media-stack/level-2-intermediate/overlays/homelab
cp secrets.env.example secrets.env
# Edit secrets.env with your real API keys
```

### 2. Configure StorageClass Patch
Open `patches/storageclass-patch.yaml` and set your cluster's RWX storage class:
```yaml
spec:
  storageClassName: nfs-client # Or democratic-csi, truenas, ceph-fs, etc.
```

### 3. Preview Manifests (Dry Run)
Inspect the generated output before applying:
```bash
kubectl kustomize .
```

### 4. Deploy to Cluster
Apply the entire overlay:
```bash
kubectl apply -k .
```

### 5. Verify Ingress & Self-Healing Probes
```bash
# Check Ingress routing rules
kubectl get ingress -n media

# Check pod health and probe status
kubectl get pods -n media -o wide
```

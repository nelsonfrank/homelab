# Level 1 (Beginner): Kubernetes Media Stack Manifests

This directory contains the beginner-level implementation of the Portainer `media-stack` Docker Compose migrated to Kubernetes.

---

## Architecture Overview

- **Namespace**: All resources run in the isolated `media` namespace (`00-namespace.yaml`).
- **Configuration**: Shared environment variables (`TZ`, `PUID`, `PGID`) are consolidated in a `ConfigMap`, while sensitive API keys live in a `Secret` (`01-configmap-and-secrets.yaml`).
- **Storage**:
  - The shared library (`/data`) uses a `ReadWriteMany` `PersistentVolume` (`pv-media-data`) backed by `hostPath` so all downloaders and Arr services access the identical directory structure.
  - Each service has an isolated `ReadWriteOnce` `PersistentVolumeClaim` for its configuration directory (`/config`) containing SQLite databases and settings.
- **Service Isolation**: **1 Service = 1 Deployment = 1 Pod**. Each container operates in its own network and process boundary, preventing port conflicts between services like qBittorrent and SABnzbd (which both listen on port 8080 internally).
- **External Access**: Services are exposed using standard Kubernetes `NodePort` definitions (ports `30000`–`32767`), enabling direct browser access without requiring an Ingress controller.

---

## Service Port & Access Matrix

Access any web UI by navigating to `http://<YOUR_KUBERNETES_NODE_IP>:<NODE_PORT>`:

| Service | Internal Port | Service Type | NodePort | Description |
| :--- | :--- | :--- | :--- | :--- |
| **Dashy** | `8080` | `NodePort` | `30400` | Dashboard & Homelab Homepage |
| **Stremio** | `11470` / `12470` | `NodePort` | `31470` / `31471` | Streaming media server |
| **qBittorrent** | `8080` | `NodePort` | `30880` | Torrent downloader WebUI (Bittorrent: `30681`) |
| **SABnzbd** | `8080` | `NodePort` | `30881` | Usenet downloader WebUI |
| **Radarr** | `7878` | `NodePort` | `30787` | Movie automation |
| **Sonarr** | `8989` | `NodePort` | `30898` | TV show automation |
| **Prowlarr** | `9696` | `NodePort` | `30969` | Indexer manager |
| **Bazarr** | `6767` | `NodePort` | `30676` | Subtitles manager |
| **Jellyseerr** | `5055` | `NodePort` | `30505` | Media request platform |
| **Emby** | `8096` / `8920` | `NodePort` | `30896` / `30920` | Media streaming server |
| **Profilarr** | `6868` | `NodePort` | `30686` | Profile sync utility |
| **QUI** | `7476` | `NodePort` | `30747` | Autobrr Web UI |
| **FlareSolverr** | `8191` | `ClusterIP` | *Internal Only* | Cloudflare bypass proxy |
| **Unpackerr** | *None* | *Worker* | *N/A* | Background extraction daemon |

---

## Inter-Service Communication (Kubernetes DNS)

Within the `media` namespace, applications communicate directly using Kubernetes service names instead of host IP addresses:

- **Unpackerr to Sonarr**: `http://sonarr:8989`
- **Unpackerr to Radarr**: `http://radarr:7878`
- **Sonarr/Radarr to qBittorrent**: `http://qbittorrent:8080`
- **Sonarr/Radarr to SABnzbd**: `http://sabnzbd:8080`
- **Prowlarr to FlareSolverr**: `http://flaresolverr:8191`
- **Jellyseerr to Emby**: `http://emby:8096`

---

## Deployment Instructions

### 1. Configure Host Storage Paths
Open `02-storage.yaml` and verify that the `hostPath` points to the physical directory where your media is stored on your host:
```yaml
hostPath:
  path: /mnt/storage/media # Update to your actual host media path
```

### 2. Configure API Keys
Edit `01-configmap-and-secrets.yaml` with your preferred timezone and pre-configured API keys if applicable.

### 3. Deploy to the Cluster
Deploy using `kubectl apply` with the provided `kustomization.yaml`:

```bash
# From this directory:
kubectl apply -k .

# Or using standard file apply:
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-configmap-and-secrets.yaml
kubectl apply -f 02-storage.yaml
kubectl apply -f .
```

### 4. Verify Pod & Service Status
```bash
# Check running pods
kubectl get pods -n media -o wide

# Check services and allocated NodePorts
kubectl get svc -n media

# View logs for any service
kubectl logs -n media -l app.kubernetes.io/name=qbittorrent -f
```

---

## Transitioning to Level 2
When you are ready to eliminate manual NodePorts, manage SSL certificates, and use domain names (e.g., `http://sonarr.home.lab`), see the [Migration Guide](../../../doc/docker-compose-to-kubernetes.md).

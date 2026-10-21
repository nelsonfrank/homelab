# Migrating Docker Compose to Kubernetes: A 3-Level Engineering Guide

This guide provides a comprehensive, developer-focused walkthrough on migrating a Docker Compose-based homelab media stack into Kubernetes. Whether you are running K3s, Talos, MicroK8s, or vanilla Kubernetes, this document walks through the architectural mental shift and breaks down the migration into three progressive skill levels: **Beginner**, **Intermediate**, and **Advanced**.

---

## Table of Contents
- [1. The Architectural Paradigm Shift](#1-the-architectural-paradigm-shift)
  - [1.1 The "One Giant Pod" Anti-Pattern](#11-the-one-giant-pod-anti-pattern)
  - [1.2 Docker Compose vs. Kubernetes Concept Mapping](#12-docker-compose-vs-kubernetes-concept-mapping)
- [2. Level 1: Beginner — Direct Translation with Raw Manifests](#2-level-1-beginner--direct-translation-with-raw-manifests)
  - [2.1 Architecture & Strategy](#21-architecture--strategy)
  - [2.2 Shared Namespaces & Config Storage](#22-shared-namespaces--config-storage)
  - [2.3 Storage: The RWO vs. RWX Reality Check](#23-storage-the-rwo-vs-rwx-reality-check)
  - [2.4 Beginner Manifests (Working Examples)](#24-beginner-manifests-working-examples)
  - [2.5 Pros, Cons & When to Move On](#25-pros-cons--when-to-move-on)
- [3. Level 2: Intermediate — Clean Architecture, Ingress & Kustomize](#3-level-2-intermediate--clean-architecture-ingress--kustomize)
  - [3.1 Eliminating Port Spaghetti with Ingress](#31-eliminating-port-spaghetti-with-ingress)
  - [3.2 DRY Configurations with Kustomize](#32-dry-configurations-with-kustomize)
  - [3.3 Native User Permissions (`securityContext`)](#33-native-user-permissions-securitycontext)
  - [3.4 Auto-Healing with Liveness & Readiness Probes](#34-auto-healing-with-liveness--readiness-probes)
  - [3.5 Dynamic Storage Provisioning (NFS / CSI)](#35-dynamic-storage-provisioning-nfs--csi)
  - [3.6 Intermediate Kustomize Blueprint](#36-intermediate-kustomize-blueprint)
- [4. Level 3: Advanced — Production Homelab, GitOps & Hardware Acceleration](#4-level-3-advanced--production-homelab-gitops--hardware-acceleration)
  - [4.1 The VPN Sidecar Pattern (Gluetun + qBittorrent)](#41-the-vpn-sidecar-pattern-gluetun--qbittorrent)
  - [4.2 Hardware Transcoding (Intel QuickSync / NVIDIA GPU)](#42-hardware-transcoding-intel-quicksync--nvidia-gpu)
  - [4.3 GitOps Deployment with Argo CD](#43-gitops-deployment-with-argo-cd)
  - [4.4 Automated TLS via cert-manager & Cloudflare DNS](#44-automated-tls-via-cert-manager--cloudflare-dns)
  - [4.5 Backup & State Protection (VolSync / Velero)](#45-backup--state-protection-volsync--velero)
  - [4.6 The Helm App-Template Pattern (`bjw-s`)](#46-the-helm-app-template-pattern-bjw-s)
- [5. Comparison Matrix: Beginner vs. Intermediate vs. Advanced](#5-comparison-matrix-beginner-vs-intermediate-vs-advanced)
- [6. Step-by-Step Migration Checklist](#6-step-by-step-migration-checklist)

---

## 1. The Architectural Paradigm Shift

Moving from Docker Compose to Kubernetes is not merely converting YAML syntax; it is shifting from a **single-node container supervisor** to a **distributed declarative orchestration system**.

```
Docker Compose Paradigm:
+-------------------------------------------------------+
| Single Host (e.g., 192.168.1.164)                     |
|                                                       |
|  [qBittorrent:8080] <--- bridge ---> [Sonarr:8989]    |
|         |                                  |          |
|         +-----------> /data <--------------+          |
|                   (Host Bind Mount)                   |
+-------------------------------------------------------+

Kubernetes Paradigm:
+------------------------------------------------------------------------+
| Cluster Overlay Network (Calico / Flannel / Cilium)                    |
|                                                                        |
| Node 1                                Node 2                           |
| +-----------------------------+       +------------------------------+ |
| | Pod: qBittorrent            |       | Pod: Sonarr                  | |
| | (IP: 10.42.1.15)            |       | (IP: 10.42.2.88)             | |
| +-----------------------------+       +------------------------------+ |
|                ^                                      ^                |
|                |                                      |                |
|     Service: qbittorrent:8080              Service: sonarr:8989        |
|                |                                      |                |
|                +---------> Shared Storage <-----------+                |
|                     (PV / PVC - ReadWriteMany)                         |
+------------------------------------------------------------------------+
```

### 1.1 The "One Giant Pod" Anti-Pattern

When developers first approach Kubernetes, a common instinct is:
> *"My `docker-compose.yml` has 14 containers. A Pod can hold multiple containers. Therefore, I will create one Pod with 14 containers!"*

**Why this fails catastrophically for a media stack:**
1. **Network Namespace Collision (`localhost`)**: All containers in a Pod share the identical network namespace and localhost. In your Docker Compose:
   - `qbittorrent` listens on port `8080`.
   - `sabnzbd` listens on port `8080` internally.
   Putting both in one Pod causes an immediate port bind collision (`bind: address already in use`).
2. **Coupled Lifecycle**: If Bazarr crashes or needs an image update, the entire 14-container Pod restarts.
3. **Resource Scheduling**: Kubernetes cannot schedule the containers across different nodes in your cluster; the whole monolith must fit on one node.
4. **Correct Pod Pattern**: In Kubernetes, **1 Microservice = 1 Deployment/StatefulSet = 1 Pod** (except when a true helper/sidecar pattern is needed, such as a VPN client or log shipper).

---

### 1.2 Docker Compose vs. Kubernetes Concept Mapping

| Docker Compose Concept | Kubernetes Equivalent | Purpose & Notes |
| :--- | :--- | :--- |
| `service:` | `Deployment` or `StatefulSet` + `Pod` | Declares desired state, replicas, image, and container lifecycle. |
| `ports:` | `Service` (ClusterIP / NodePort / LoadBalancer) + `Ingress` | Exposes network access internally and externally. |
| `networks:` | Kubernetes DNS (`CoreDNS`) + CNI | All Pods can talk to each other across nodes by default. |
| `volumes:` (bind mount `/data`) | `PersistentVolume` (PV) & `PersistentVolumeClaim` (PVC) | Manages durable disks. Requires `ReadWriteMany` for shared media. |
| `volumes:` (named config) | PVC with `ReadWriteOnce` | Dedicated storage for SQLite databases and application settings. |
| `environment:` | `ConfigMap` (non-sensitive) & `Secret` (sensitive) | Injects configuration, TZ, paths, and API keys. |
| `restart: unless-stopped` | `restartPolicy: Always` (default in Deployments) | Handled automatically by the Kubernetes controller loop. |
| Service Name DNS (`http://sonarr:8989`) | K8s Service DNS (`http://sonarr.media.svc.cluster.local:8989`) | Accessible as `http://sonarr:8989` within the same namespace. |

---

## 2. Level 1: Beginner — Direct Translation with Raw Manifests

The goal of the Beginner level is to get workloads up and running using vanilla, standard Kubernetes manifests (`apiVersion`, `kind: Deployment`, `kind: Service`, `kind: PersistentVolumeClaim`).

### 2.1 Architecture & Strategy
- **1 Deployment + 1 Service** per container.
- Put all media resources into a dedicated **Namespace** (e.g., `media`).
- Use **ClusterIP Services** for intra-cluster communication (e.g., Unpackerr calling Sonarr).
- Use **NodePort** or host port mappings temporarily to access Web UIs from your browser (e.g., `http://<node-ip>:30080`).
- Use **ConfigMap** for shared variables (like `TZ`) and **Secret** for API keys.

---

### 2.2 Shared Namespaces & Config Storage

First, create a namespace and a central config/secret store:

```yaml
# 00-namespace-and-configs.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: media
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: media-common-env
  namespace: media
data:
  TZ: "America/New_York"
  PUID: "1000"
  PGID: "1000"
---
apiVersion: v1
kind: Secret
metadata:
  name: media-api-secrets
  namespace: media
type: Opaque
stringData:
  SONARR_API_KEY: "your_sonarr_api_key_here"
  RADARR_API_KEY: "your_radarr_api_key_here"
```

---

### 2.3 Storage: The RWO vs. RWX Reality Check

This is the **#1 trap** in migrating a media stack to Kubernetes:

- **App Configs (`/config`)**: SQLite databases (Sonarr, Radarr, qBittorrent, etc.) lock their database files. They must be mounted with **`ReadWriteOnce` (RWO)** — mounted to exactly one node at a time.
- **Shared Media Library (`/data`)**: `qbittorrent`, `sabnzbd`, `radarr`, `sonarr`, `bazarr`, and `unpackerr` **must all access the exact same path (`/data`) simultaneously**.
  - If your cluster has more than 1 node, a standard block storage PVC (like Longhorn RWO or Local PV) will fail because Pods on Node 2 cannot mount the volume already attached to Node 1.
  - In Level 1 (single-node homelab), a `hostPath` PersistentVolume works seamlessly.
  - In Level 2/3 (multi-node), an **NFS or CephFS `ReadWriteMany` (RWX)** volume is mandatory.

```yaml
# 01-storage-beginner.yaml
# Shared Media Volume (RWX via HostPath for Single-Node Homelab)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-media-data
spec:
  capacity:
    storage: 2Ti
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/storage/media # Change to your actual host disk path
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-media-data
  namespace: media
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Ti
  volumeName: pv-media-data
---
# Dedicated Config PVC for qBittorrent (RWO)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: qbittorrent-config
  namespace: media
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

---

### 2.4 Beginner Manifests (Working Examples)

Here is how you translate `qbittorrent` and `sonarr` properly into individual Deployments and Services:

#### qBittorrent (Downloader)
```yaml
# 02-qbittorrent.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: qbittorrent
  namespace: media
  labels:
    app.kubernetes.io/name: qbittorrent
spec:
  replicas: 1
  strategy:
    type: Recreate # Required for SQLite/RWO volumes to prevent multiple pods mounting simultaneously during rollout
  selector:
    matchLabels:
      app.kubernetes.io/name: qbittorrent
  template:
    metadata:
      labels:
        app.kubernetes.io/name: qbittorrent
    spec:
      containers:
        - name: qbittorrent
          image: lscr.io/linuxserver/qbittorrent:latest
          envFrom:
            - configMapRef:
                name: media-common-env
          ports:
            - name: webui
              containerPort: 8080
            - name: bittorrent-tcp
              containerPort: 6881
              protocol: TCP
            - name: bittorrent-udp
              containerPort: 6881
              protocol: UDP
          volumeMounts:
            - name: config
              mountPath: /config
            - name: data
              mountPath: /data
      volumes:
        - name: config
          persistentVolumeClaim:
            claimName: qbittorrent-config
        - name: data
          persistentVolumeClaim:
            claimName: pvc-media-data
---
apiVersion: v1
kind: Service
metadata:
  name: qbittorrent
  namespace: media
spec:
  type: NodePort # Allows accessing from browser via NodeIP:30080
  selector:
    app.kubernetes.io/name: qbittorrent
  ports:
    - name: webui
      port: 8080
      targetPort: 8080
      nodePort: 30080
```

#### Sonarr (TV Automation)
```yaml
# 03-sonarr.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sonarr-config
  namespace: media
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sonarr
  namespace: media
  labels:
    app.kubernetes.io/name: sonarr
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app.kubernetes.io/name: sonarr
  template:
    metadata:
      labels:
        app.kubernetes.io/name: sonarr
    spec:
      containers:
        - name: sonarr
          image: lscr.io/linuxserver/sonarr:latest
          envFrom:
            - configMapRef:
                name: media-common-env
          ports:
            - name: http
              containerPort: 8989
          volumeMounts:
            - name: config
              mountPath: /config
            - name: data
              mountPath: /data
      volumes:
        - name: config
          persistentVolumeClaim:
            claimName: sonarr-config
        - name: data
          persistentVolumeClaim:
            claimName: pvc-media-data
---
apiVersion: v1
kind: Service
metadata:
  name: sonarr
  namespace: media
spec:
  type: ClusterIP # Internal only, or NodePort if direct browser access needed
  selector:
    app.kubernetes.io/name: sonarr
  ports:
    - name: http
      port: 8989
      targetPort: 8989
```

#### Connecting Services (Internal DNS)
In Docker Compose, services connected via `http://qbittorrent:8080`.
In Kubernetes, because both are in the `media` namespace, Sonarr communicates with qBittorrent using the exact same syntax:
```
Host: qbittorrent
Port: 8080
```
Or fully qualified: `qbittorrent.media.svc.cluster.local:8080`.

Notice how in Docker Compose, `unpackerr` had an explicit IP address:
```yaml
- UN_SONARR_0_URL=http://192.168.1.164:8989
```
In Kubernetes, this hardcoded IP is replaced with the reliable cluster DNS name:
```yaml
- UN_SONARR_0_URL=http://sonarr.media.svc.cluster.local:8989
```

---

### 2.5 Pros, Cons & When to Move On

- **Pros**: Easy to understand; looks familiar to Docker Compose users; works out of the box with `kubectl apply -f .`.
- **Cons**:
  - NodePort port allocation becomes a mess (memorizing ports `30080`, `30898`, `30787`).
  - Duplication of YAML boilerplate across all 14 services.
  - Manual updates and no easy staging/production environment separation.
  - Static `hostPath` binds you to a single physical machine.

---

## 3. Level 2: Intermediate — Clean Architecture, Ingress & Kustomize

Level 2 introduces production homelab standards: unified Ingress routing, structured configuration with Kustomize, dynamic storage classes, and container health monitoring.

```
                      Client Request (e.g., https://sonarr.home.lab)
                                     |
                                     v
                        [Ingress Controller (Traefik/NGINX)]
                                     |
                  +------------------+------------------+
                  |                                     |
       Path / Host: sonarr.home.lab         Path / Host: qbit.home.lab
                  |                                     |
                  v                                     v
       [Service: sonarr (ClusterIP)]        [Service: qbittorrent (ClusterIP)]
                  |                                     |
                  v                                     v
             [Pod: sonarr]                         [Pod: qbit]
```

### 3.1 Eliminating Port Spaghetti with Ingress

Instead of mapping random NodePorts or host ports for 14 different services, route everything through an Ingress Controller (like Traefik, standard in K3s, or NGINX Ingress):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: media-ingress
  namespace: media
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
    - host: qbit.home.lab
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: qbittorrent
                port:
                  number: 8080
    - host: sonarr.home.lab
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: sonarr
                port:
                  number: 8989
    - host: radarr.home.lab
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: radarr
                port:
                  number: 7878
```

---

### 3.2 DRY Configurations with Kustomize

Instead of copying 80 lines of YAML for each of your 14 services, organize your manifests using **Kustomize** (built directly into `kubectl`).

#### Directory Layout
```text
k8s/
├── base/
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── common-config.env
│   ├── shared-storage.yaml
│   ├── apps/
│   │   ├── qbittorrent/
│   │   │   ├── deployment.yaml
│   │   │   └── service.yaml
│   │   ├── sonarr/
│   │   │   ├── deployment.yaml
│   │   │   └── service.yaml
│   │   ├── radarr/
│   │   │   ├── deployment.yaml
│   │   │   └── service.yaml
│   │   └── ...
│   └── ingress.yaml
└── overlays/
    ├── homelab/
    │   ├── kustomization.yaml
    │   └── patches/
    │       └── node-affinity.yaml
```

#### Base `kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: media

configMapGenerator:
  - name: media-common-env
    envs:
      - common-config.env

secretGenerator:
  - name: media-api-secrets
    envs:
      - secrets.env

resources:
  - namespace.yaml
  - shared-storage.yaml
  - ingress.yaml
  - apps/qbittorrent/deployment.yaml
  - apps/qbittorrent/service.yaml
  - apps/sonarr/deployment.yaml
  - apps/sonarr/service.yaml
```

Deploy the entire stack with one command:
```bash
kubectl apply -k k8s/overlays/homelab/
```

---

### 3.3 Native User Permissions (`securityContext`)

In Docker Compose, LinuxServer images use `PUID=1000` and `PGID=1000` environment variables.
In Kubernetes, you can enforce this at the Pod runtime level using `securityContext`:

```yaml
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000          # Ensures any volume mounted has permissions owned by group 1000
    fsGroupChangePolicy: "OnRootMismatch" # Skips chown if already correct, dramatically speeding up Pod startup
  containers:
    - name: sonarr
      image: lscr.io/linuxserver/sonarr:latest
```

---

### 3.4 Auto-Healing with Liveness & Readiness Probes

If Radarr or Sonarr hangs or hits an out-of-memory lock on its SQLite database, Docker Compose keeps the container marked "running" indefinitely. Kubernetes can actively monitor and automatically reboot unhealthy containers:

```yaml
containers:
  - name: sonarr
    image: lscr.io/linuxserver/sonarr:latest
    readinessProbe:
      httpGet:
        path: /ping
        port: 8989
      initialDelaySeconds: 15
      periodSeconds: 10
    livenessProbe:
      httpGet:
        path: /ping
        port: 8989
      initialDelaySeconds: 30
      periodSeconds: 20
      failureThreshold: 3
```

---

### 3.5 Dynamic Storage Provisioning (NFS / CSI)

Instead of manually creating PVs with static host paths, configure a dynamic StorageClass (such as `nfs-subdir-external-provisioner` connected to your TrueNAS/Synology or `democratic-csi`):

```yaml
# pvc-media-data-dynamic.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-media-data
  namespace: media
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-client # Automatically provisions on your NAS
  resources:
    requests:
      storage: 10Ti
```

---

## 4. Level 3: Advanced — Production Homelab, GitOps & Hardware Acceleration

The Advanced level addresses the real-world operational complexities of self-hosting a media stack: torrent VPN kill-switches, GPU video transcoding, GitOps automation, automated SSL certificates, and backup pipelines.

---

### 4.1 The VPN Sidecar Pattern (Gluetun + qBittorrent)

In a home setup, routing torrent downloaders through a VPN (e.g., ProtonVPN, Mullvad, WireGuard) with a **hard kill-switch** is essential.

In Kubernetes, this is achieved by placing **Gluetun and qBittorrent into the SAME Pod**. Because containers in a Pod share the network stack (`localhost`):
1. Gluetun establishes the VPN tunnel (`tun0`).
2. qBittorrent communicates solely over Gluetun's network interface.
3. If the VPN drops, network traffic halts immediately (fail-closed kill-switch).
4. The WebUI port is exposed safely through Gluetun's port-forwarding to the Kubernetes Service.

```
+-------------------------------------------------------------+
| Pod: qbittorrent-vpn                                        |
|                                                             |
|   +-------------------+              +-------------------+  |
|   | Container:        |              | Container:        |  |
|   | gluetun           |              | qbittorrent       |  |
|   | (VPN Gateway)     |              | (App)             |  |
|   +-------------------+              +-------------------+  |
|             ^                                  |            |
|             |======= Localhost Loopback =======|            |
|             v                                               |
|      VPN Tunnel (tun0) ---> Public Internet                 |
+-------------------------------------------------------------+
```

```yaml
# qbittorrent-vpn-pod.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: qbittorrent
  namespace: media
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: qbittorrent
  template:
    metadata:
      labels:
        app: qbittorrent
    spec:
      containers:
        # VPN Sidecar
        - name: gluetun
          image: qmcgaw/gluetun:latest
          securityContext:
            capabilities:
              add:
                - NET_ADMIN
          env:
            - name: VPN_SERVICE_PROVIDER
              value: "custom"
            - name: VPN_TYPE
              value: "wireguard"
          ports:
            - name: webui
              containerPort: 8080 # qBittorrent WebUI exposed via Gluetun
          volumeMounts:
            - name: gluetun-config
              mountPath: /gluetun
        # Downloader
        - name: qbittorrent
          image: lscr.io/linuxserver/qbittorrent:latest
          env:
            - name: WEBUI_PORT
              value: "8080"
          volumeMounts:
            - name: qbit-config
              mountPath: /config
            - name: media-data
              mountPath: /data
      volumes:
        - name: gluetun-config
          persistentVolumeClaim:
            claimName: gluetun-config
        - name: qbit-config
          persistentVolumeClaim:
            claimName: qbit-config
        - name: media-data
          persistentVolumeClaim:
            claimName: pvc-media-data
```

---

### 4.2 Hardware Transcoding (Intel QuickSync / NVIDIA GPU)

In your Docker Compose stack, `emby` (or Plex/Jellyfin) transcode media on CPU unless given GPU devices. In Kubernetes, you can pass hardware devices cleanly:

#### Intel QuickSync (VA-API / `/dev/dri`)
Using host devices in the Pod spec:
```yaml
spec:
  containers:
    - name: emby
      image: emby/embyserver:latest
      securityContext:
        privileged: true # Or use the Intel Device Plugin for Kubernetes
      resources:
        requests:
          gpu.intel.com/i915: 1 # When using Intel Device Plugin
        limits:
          gpu.intel.com/i915: 1
      volumeMounts:
        - name: dri
          mountPath: /dev/dri
  volumes:
    - name: dri
      hostPath:
        path: /dev/dri
```

#### NVIDIA GPU (NVENC / NVDEC)
1. Install the `nvidia-gpu-operator`.
2. Add the resource request to your Deployment:
```yaml
resources:
  limits:
    nvidia.com/gpu: 1
```

---

### 4.3 GitOps Deployment with Argo CD

With GitOps, you never run `kubectl apply` manually. Your Git repository is the single source of truth:

```yaml
# argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: homelab-media-stack
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/nelsonfrank/homelab.git'
    targetRevision: HEAD
    path: k8s/overlays/homelab
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: media
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```
When you push a Git commit updating an image tag or configuration, Argo CD automatically validates and synchronizes the change to your cluster.

---

### 4.4 Automated TLS via cert-manager & Cloudflare DNS

Replace insecure HTTP internal URLs with automated, valid HTTPS wildcard certificates:

1. Deploy **cert-manager**.
2. Configure a `ClusterIssuer` using Cloudflare DNS-01 verification (which works even for internal private IPs behind your LAN!):

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-cloudflare
spec:
  acme:
    email: admin@yourdomain.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-account-key
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
---
# Ingress with automatic TLS
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: media-ingress
  namespace: media
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-cloudflare
spec:
  tls:
    - hosts:
        - "*.home.yourdomain.com"
      secretName: wildcard-home-tls
  rules:
    - host: sonarr.home.yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: sonarr
                port:
                  number: 8989
```

---

### 4.5 Backup & State Protection (VolSync / Velero)

Media databases (`sonarr.db`, `radarr.db`) are prone to corruption if backed up while active or if disks fail:

- Use **VolSync** to trigger automated, scheduled volume snapshots using Restic or Kopia directly to local MinIO or off-site S3 storage (Backblaze B2, AWS S3).
- Implement Kubernetes Job cron tasks to trigger SQLite backup endpoints before snapshotting:
  ```bash
  curl -d "" -X POST http://sonarr:8989/api/v3/system/backup?apiKey=$API_KEY
  ```

---

### 4.6 The Helm App-Template Pattern (`bjw-s`)

In the modern Kubernetes homelab community, writing raw Deployments for 14 simple containers is considered obsolete. Instead, practitioners use the popular **`bjw-s/app-template`** Helm chart.

A single `values.yaml` file can declare all 14 services cleanly without boilerplate:

```yaml
# values.yaml (bjw-s app-template)
sonarr:
  controllers:
    main:
      containers:
        main:
          image:
            repository: lscr.io/linuxserver/sonarr
            tag: latest
          env:
            TZ: America/New_York
          probes:
            liveness:
              enabled: true
            readiness:
              enabled: true
  service:
    main:
      ports:
        http:
          port: 8989
  ingress:
    main:
      enabled: true
      hosts:
        - host: sonarr.home.lab
          paths:
            - path: /
  persistence:
    config:
      enabled: true
      type: persistentVolumeClaim
      accessMode: ReadWriteOnce
      size: 5Gi
    media:
      enabled: true
      existingClaim: pvc-media-data
      globalMounts:
        - path: /data
```

---

## 5. Comparison Matrix: Beginner vs. Intermediate vs. Advanced

| Capability | Level 1: Beginner | Level 2: Intermediate | Level 3: Advanced |
| :--- | :--- | :--- | :--- |
| **Tooling** | Raw YAML (`kubectl apply -f`) | Kustomize (`kubectl apply -k`) | GitOps (Argo CD / Flux) or Helm |
| **Pod Architecture** | 1 Deployment per Service | 1 Deployment per Service | Pods with specialized Sidecars (VPN, Loggers) |
| **Networking** | NodePort (`:30080`, `:30898`) | Ingress Controller (Traefik / NGINX) | Ingress + cert-manager (Cloudflare TLS) + Gateway API |
| **DNS** | Raw IPs or basic CoreDNS | Unified internal hostnames (`app.home.lab`) | Wildcard Split-Horizon DNS (`*.home.domain.com`) |
| **Storage** | Static `hostPath` | Dynamic CSI / NFS RWX | CSI with automated VolSync snapshots |
| **Security** | Plaintext Environment Vars | ConfigMap + Secret manifests | External Secrets Operator / Sealed Secrets |
| **Transcoding** | Pure CPU software encoding | Host `/dev/dri` passthrough | GPU Device Plugins (Intel QSV / NVIDIA NVENC) |
| **Resilience** | Basic container restart | Liveness & Readiness Probes | Automated Rollback, Pod Disruption Budgets, HPA |

---

## 6. Step-by-Step Migration Checklist

Follow this sequence to transition your existing Portainer stack to Kubernetes without data loss:

1. **Audit Existing Volumes**:
   - Locate where `${FOLDER_FOR_CONFIGS}` and `${FOLDER_FOR_MEDIA}` live on your host machine.
2. **Setup the Media Storage Class**:
   - Ensure an NFS server or dynamic CSI provider is accessible to all K8s worker nodes for `/data`.
3. **Migrate Stateless/Low-Risk Services First**:
   - Start with `flaresolverr`, `dashy`, or `profilarr`. Verify network reachability.
4. **Deploy Downloaders (qBittorrent / SABnzbd)**:
   - Ensure download folders match the exact paths previously configured in Compose so active downloads resume cleanly.
5. **Copy Application Configurations**:
   - Stop a service in Portainer (e.g., `radarr`).
   - Copy `${FOLDER_FOR_CONFIGS}/radarr` contents into the newly provisioned PVC mount.
   - Start the Kubernetes `radarr` Deployment.
6. **Verify Cross-Service Connectivity**:
   - Update Sonarr/Radarr Download Client settings to target internal K8s hostnames (`http://qbittorrent:8080`) rather than local host IPs.
7. **Switch Ingress / DNS**:
   - Point your local DNS (Pi-hole, AdGuard Home, or router) to your Kubernetes Ingress controller VIP.

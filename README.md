# Homelab

This repository contains the configuration and setup for my personal homelab. The infrastructure is managed using **Portainer** stacks, which allow for easy deployment and management of containerized services.

## Project Structure

The project is organized into infrastructure configurations for both Portainer and Kubernetes:

- `portainer/stacks/`: Contains Docker Compose files for various service groups.
  - `media-stack/`: A comprehensive stack for media management, downloading, and streaming.
- `k8s/stacks/`: Kubernetes manifests for running the services in a cluster.
  - `media-stack/`:
    - [level-1-beginner/](k8s/stacks/media-stack/level-1-beginner/): Complete working manifests for Level 1 implementation (Deployments, Services, PVCs, ConfigMaps).
    - [level-2-intermediate/](k8s/stacks/media-stack/level-2-intermediate/): Production homelab setup with Ingress routing, Kustomize base & overlays, dynamic RWX storage, health probes, and securityContext.
- `doc/`: Architecture and migration guides.
  - [Docker Compose to Kubernetes Migration Guide](doc/docker-compose-to-kubernetes.md): Detailed 3-level guide (Beginner, Intermediate, Advanced) on transitioning homelab stacks to Kubernetes.

## Documentation & Guides

- 📘 **[Docker Compose to Kubernetes: 3-Level Migration Guide](doc/docker-compose-to-kubernetes.md)**:
  - **Level 1 (Beginner)**: Direct translation to Deployments, Services, ConfigMaps, Secrets, and avoiding the "one giant pod" trap.
  - **Level 2 (Intermediate)**: Eliminating port sprawl with Ingress, DRY deployments with Kustomize, dynamic storage classes (RWO vs. RWX), and health probes.
  - **Level 3 (Advanced)**: Production homelab practices including VPN sidecars (Gluetun + qBittorrent kill-switch), GPU hardware transcoding passthrough, GitOps with Argo CD, and automated TLS with cert-manager.

## Media Stack

The `media-stack` provides a full suite of tools for an automated media library.

### Dashboard & Requests
- **Dashy**: A self-hosted dashboard for easy access to all services.
- **Jellyseerr**: A request management and discovery tool for movies and TV shows.

### Media Server
- **Emby**: A powerful media server for organizing and playing your media collection.

### Downloaders
- **qBittorrent**: A lightweight and feature-rich BitTorrent client.
- **SABnzbd**: A versatile Usenet downloader.

### Automation & Management
- **Radarr**: Automatically manages movie downloads.
- **Sonarr**: Automatically manages TV show downloads.
- **Prowlarr**: Indexer manager that integrates with various torrent and Usenet indexers.
- **Bazarr**: Automatically downloads subtitles for your media.
- **Unpackerr**: Automatically extracts downloaded archives.
- **Profilarr**: Manages quality and release profiles across different Arrs.
- **FlareSolverr**: A proxy server to bypass Cloudflare and other anti-bot protections.

### Utilities
- **QUI**: A web interface for autobrr.

## Getting Started

### Portainer / Docker Compose
To deploy these stacks with Docker, you can copy the contents of the `docker-compose.yml` files into new stacks in your Portainer instance.

> [!NOTE]
> Make sure to configure the necessary environment variables (such as `FOLDER_FOR_CONFIGS`, `FOLDER_FOR_MEDIA`, `PUID`, `PGID`, etc.) in your Portainer stack configuration.

### Kubernetes
To migrate and run these workloads on Kubernetes, follow the step-by-step instructions in the [Docker Compose to Kubernetes Migration Guide](doc/docker-compose-to-kubernetes.md).

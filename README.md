# Homelab

This repository contains the configuration and setup for my personal homelab. The infrastructure is managed using **Portainer** stacks, which allow for easy deployment and management of containerized services.

## Project Structure

The project is organized into Portainer stacks located in the `portainer/stacks` directory.

- `portainer/stacks/`: Contains Docker Compose files for various service groups.
  - `media-stack/`: A comprehensive stack for media management, downloading, and streaming.

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

To deploy these stacks, you can copy the contents of the `docker-compose.yml` files into new stacks in your Portainer instance.

> [!NOTE]
> Make sure to configure the necessary environment variables (such as `FOLDER_FOR_CONFIGS`, `FOLDER_FOR_MEDIA`, `PUID`, `PGID`, etc.) in your Portainer stack configuration.

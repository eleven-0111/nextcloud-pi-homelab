# 🌩️ Raspberry Pi Private Cloud — Nextcloud + AI Documentation Pipeline

> A self-hosted private cloud running on a Raspberry Pi 4, with secure remote access via Cloudflare Tunnel and Tailscale, Azure SSO integration, full monitoring stack, and an AI-powered documentation pipeline using LangChain and Ollama.

---

## 📋 Table of Contents

1. [Project Overview](#-project-overview)
2. [Hardware Setup](#-hardware-setup)
3. [Operating System & Kernel](#-operating-system--kernel)
4. [Storage Configuration](#-storage-configuration)
5. [Docker Services](#-docker-services)
6. [Networking & Remote Access](#-networking--remote-access)
7. [Security](#-security)
8. [Monitoring Stack](#-monitoring-stack)
9. [System Services](#-system-services)
10. [AI Documentation Pipeline](#-ai-documentation-pipeline)
11. [Architecture Overview](#-architecture-overview)
12. [Next Steps](#-next-steps)

---

## 🔍 Project Overview

This project sets up a fully self-hosted private cloud using **Nextcloud** on a Raspberry Pi 4, accessible securely from anywhere via a **Cloudflare Tunnel** — no port forwarding required. It includes:

- **Nextcloud** (v33) deployed via Docker with a MySQL 8.0 backend
- **Cloudflare Tunnel** for secure, public HTTPS access via a custom domain
- **Tailscale** VPN for internal device access
- **Azure Entra ID (SSO)** for login via OpenID Connect
- **Prometheus + Grafana** for real-time monitoring
- **LangChain + Ollama** for automated AI documentation generation

Here's the project build log captured straight from the terminal — the story of how it all came together:

![Project Build Log](assets/01_build_log.jpg)

---

## 🖥️ Hardware Setup

| Component         | Details                          |
|-------------------|----------------------------------|
| Board             | Raspberry Pi 4 Model B           |
| RAM               | 8GB                              |
| Storage           | 512GB SSD (via USB enclosure)    |
| Cooling           | Aluminium Heatsink Case + Dual Fans |
| Connectivity      | Ethernet                         |
| Power             | Official Raspberry Pi PSU        |

---

## 🐧 Operating System & Kernel

| Property     | Value                                        |
|--------------|----------------------------------------------|
| OS           | Ubuntu 22.04.5 LTS (Jammy Jellyfish)         |
| Kernel       | `5.15.0-1098-raspi`                          |
| Architecture | `aarch64` (ARM64)                            |
| Build Date   | Thu Mar 19 11:01:01 UTC 2026                 |

---

## 💾 Storage Configuration

The system boots and runs entirely from the SSD — the USB boot was migrated to SSD for better performance and reliability.

```
NAME   SIZE    TYPE  MOUNTPOINT
sda    476.9G  disk
├─sda1   512M  part  /boot/firmware  (vfat)
└─sda2 476.4G  part  /               (ext4)
```

**`/etc/fstab` entries:**
```
LABEL=writable       /               ext4   discard,errors=remount-ro  0 1
LABEL=system-boot    /boot/firmware  vfat   defaults                   0 1
```

> `discard` flag enables TRIM passthrough for SSD longevity, complemented by a systemd `fstrim` service.

---

## 🐳 Docker Services

### Running Containers

| Container       | Image                  | Status      | Port Mapping         |
|-----------------|------------------------|-------------|----------------------|
| `nextcloud-app` | `nextcloud:33-apache`  | ✅ Up        | `0.0.0.0:8080 → 80` |
| `nextcloud-db`  | `mysql:8.0`            | ✅ Up        | `3306` (internal)    |

### Docker Images

| Image                 | Disk Usage | Content Size |
|-----------------------|------------|--------------|
| `nextcloud:33-apache` | 2.12 GB    | 524 MB       |
| `nextcloud:27-apache` | 1.78 GB    | 419 MB       |
| `mysql:8.0`           | 1.08 GB    | 243 MB       |

Verifying both services are live and responding — Nextcloud on `:8080` and Grafana on `:3000`:

![Services Running](assets/04_services_running.jpg)

> Nextcloud is exposed internally on port `8080` and tunnelled publicly via Cloudflare — no ports are opened on the router.

---

## 🌐 Networking & Remote Access

### Cloudflare Tunnel

```
Domain → Cloudflare DNS → cloudflared (tunnel) → localhost:8080 (Nextcloud)
```

- Active connections across `del06`, `maa01`, `maa04` PoPs (Delhi & Mumbai)
- Managed as a systemd service (`cloudflared.service`)
- Zero port forwarding on the home router

DNS records configured in Cloudflare — both `grafana` and `nextcloud` subdomains routed through the tunnel:

![Cloudflare DNS Setup](assets/02_cloudflare_dns.jpg)

Tunnel config mapping hostnames to local services:

![Tunnel Config](assets/03_tunnel_config.jpg)

### Tailscale VPN

| Device                  | OS      | Status                          |
|-------------------------|---------|----------------------------------|
| `server` (this Pi)      | Linux   | ✅ Active                        |
| `parrot`                | Linux   | ✅ Active (relay: Bangalore)     |
| `saqlain-mustaqs-s21`   | Android | 🔴 Offline (last seen 3d ago)   |
| `ubuntu`                | Linux   | 🔴 Offline (last seen 17d ago)  |

- Provides secure peer-to-peer access between personal devices
- SSH access over Tailscale — no public SSH exposure

---

## 🔐 Security

- **SSH** — Public key authentication only (password auth disabled)
- **Cloudflare Tunnel** — All traffic encrypted via Cloudflare; no open inbound ports
- **Tailscale** — WireGuard-based VPN for internal access
- **Azure Entra ID (SSO)** — OpenID Connect via Azure App Registration for Nextcloud login
- **Unattended Upgrades** — Automatic OS security patches (`unattended-upgrades.service`)
- **SMART monitoring** — Disk health tracked via `smartmontools.service`

---

## 📊 Monitoring Stack

| Tool                    | Role                                      |
|-------------------------|-------------------------------------------|
| **Prometheus**          | Metrics collection and time-series DB     |
| **Node Exporter**       | Exposes system-level metrics (CPU, RAM, disk, network) |
| **Grafana**             | Dashboards for visualizing metrics        |

All three run as systemd services (`prometheus.service`, `prometheus-node-exporter.service`, `grafana-server.service`).

Grafana up and running, connected to the Pi via the custom domain:

![Grafana Home](assets/05_grafana_home.jpg)

Prometheus configured as the data source in Grafana:

![Prometheus Datasource](assets/06_grafana_prometheus.jpg)

Importing the Node Exporter Full dashboard for system metrics:

![Node Exporter Dashboard Import](assets/07_node_exporter_import.jpg)

---

## 🔑 Azure Entra ID (SSO)

Grafana configured with Azure Entra ID as the OAuth2 provider — users can sign in with their Microsoft account:

![Azure SSO Configuration](assets/08_azure_sso.jpg)

User successfully synced and authenticated via AzureAD:

![Azure SSO User](assets/09_azure_sso_user.jpg)

---

## ⚙️ System Services

33 active systemd units. Key services:

| Service                          | Description                        |
|----------------------------------|------------------------------------|
| `cloudflared.service`            | Cloudflare Tunnel                  |
| `docker.service`                 | Docker Engine                      |
| `tailscaled.service`             | Tailscale VPN                      |
| `ssh.service`                    | OpenSSH Server                     |
| `prometheus.service`             | Monitoring & time-series DB        |
| `prometheus-node-exporter.service` | System metrics exporter          |
| `grafana-server.service`         | Monitoring dashboards              |
| `unattended-upgrades.service`    | Automatic OS security updates      |
| `smartmontools.service`          | SSD health monitoring              |
| `systemd-timesyncd.service`      | NTP time sync                      |

---

## 🤖 AI Documentation Pipeline

One of the unique aspects of this project is that the documentation is **automatically generated** using a local AI pipeline — no cloud APIs involved.

### Stack

| Component    | Role                                      |
|--------------|-------------------------------------------|
| **Python**   | Orchestration scripts                     |
| **LangChain**| LLM chaining and prompt management        |
| **Ollama**   | Local LLM inference on the Raspberry Pi   |

### Flow

```
Server Data Collection
        ↓
   Python Scripts
   (gather system info: uname, docker ps, lsblk, tailscale, etc.)
        ↓
    LangChain
   (structure and prompt engineering)
        ↓
     Ollama
   (local LLM running on the Pi)
        ↓
   README.md / Docs
        ↓
   GitHub Push (automated)
```

> The LLM runs entirely on-device — no data leaves the Pi. The pipeline collects live server state and produces structured Markdown documentation.

---

## 🗺️ Architecture Overview

```
                        ┌─────────────────────────────────┐
                        │         Raspberry Pi 4 (8GB)    │
                        │         Ubuntu 22.04.5 LTS       │
                        │                                  │
  Internet              │  ┌─────────────────────────┐    │
     │                  │  │  Docker                  │    │
     ▼                  │  │  ├── nextcloud:33-apache  │    │
 Cloudflare DNS         │  │  └── mysql:8.0            │    │
     │                  │  └─────────────────────────┘    │
     ▼                  │                                  │
 cloudflared ──────────►│  :8080 (Nextcloud)              │
 (Tunnel)               │                                  │
                        │  ┌──────────┐ ┌──────────────┐  │
 Tailscale VPN          │  │Prometheus│ │   Grafana    │  │
 (Personal Devices) ───►│  │+ Node    │ │  Dashboards  │  │
                        │  │Exporter  │ └──────────────┘  │
                        │  └──────────┘                   │
                        │                                  │
                        │  ┌────────────────────────────┐ │
                        │  │  AI Pipeline               │ │
                        │  │  LangChain + Ollama         │ │
                        │  │  → Auto README Generation  │ │
                        │  └────────────────────────────┘ │
                        └─────────────────────────────────┘
```

---

## 🚀 Next Steps

- [ ] Automate GitHub push from the AI pipeline
- [ ] Expand pipeline to generate structured multi-section docs
- [ ] Explore Nextcloud apps (Calendar, Notes, Talk)
- [ ] Add backup automation (restic / rclone)

---

## 📄 License

MIT — feel free to fork and adapt for your own homelab setup.

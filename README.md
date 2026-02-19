# Public Infrastructure Automation

Ansible playbook for managing public-facing VPS infrastructure. Deploys and configures web services behind an Nginx reverse proxy with Cloudflare integration.

## 🎯 Overview

This repository manages a single Oracle Cloud VPS running:
- **Nginx Gateway** — Reverse proxy with Cloudflare-only firewall, certbot wildcard SSL, and static website hosting
- **SearXNG** — Privacy-respecting metasearch engine
- **Files CDN** — Static file hosting and delivery
- **External Monitor** — Prometheus + Loki + Grafana stack for VPS metrics and log aggregation
- **Monitoring** — Node Exporter, Fail2Ban Exporter, and Promtail for VPS system metrics and log shipping

## 🔒 Security

- **Cloudflare-only ingress** — Port 443 restricted to Cloudflare IPs via ipset + UFW
- **DNS-01 wildcard certs** — Certbot with Cloudflare DNS plugin
- **Hardened base** — SSH key-only auth, fail2ban, kernel hardening, AppArmor
- **Podman Quadlet isolation** — All services run as systemd-managed Podman containers

## 📁 Repository Structure

```
.
├── README.md
├── LICENSE
├── .gitignore
└── ansible/
    ├── ansible.cfg              # Ansible configuration
    ├── site.yml                 # Main playbook
    ├── update.yml               # System update playbook
    ├── cleanup-docker.yml       # Oneshot Docker removal (pre-migration)
    ├── inventory/
    │   └── hosts.yml            # VPS host inventory
    ├── group_vars/
    │   └── all.yml              # All variables (system, services, Cloudflare)
    ├── roles/
    │   ├── base/                # System hardening, users, SSH, firewall
    │   ├── nginx_gateway/       # Nginx reverse proxy + Cloudflare + certbot
    │   ├── searxng/             # SearXNG search engine
    │   ├── filescdn/            # File CDN service
    │   ├── external-monitor/    # Prometheus + Loki + Grafana
    │   └── monitoring-vps/      # Node Exporter + Fail2Ban Exporter + Promtail
    └── credentials/             # Sensitive data (gitignored)
        ├── hosts/               # Per-host credentials
        └── ssh_keys/            # SSH keys
```

## 🚀 Quick Start

1. **Clone and configure:**
   ```bash
   git clone <your-repo-url> public-infra
   cd public-infra/ansible
   cp inventory/hosts.yml.example inventory/hosts.yml
   cp group_vars/all.yml.example group_vars/all.yml
   ```

2. **Set up SSH keys** — Place keys in `ansible/credentials/ssh_keys/` (see [ansible/credentials/ssh_keys/README.md](ansible/credentials/ssh_keys/README.md))

3. **Edit variables** — Update `ansible/group_vars/all.yml` with your domain, Cloudflare token, and credentials

4. **Run:**
   ```bash
   ansible-playbook site.yml
   ```

## 🏗️ Architecture

All services run on a single VPS behind Nginx:

```
Internet → Cloudflare → [443] Nginx Gateway
                                 ├── search.domain.com      → SearXNG container
                                 ├── cdn.domain.com         → FileCDN container
                                 ├── monitoring.domain.com  → Grafana container
                                 └── domain.com             → Static website (git clone)
                              
External Monitor (separate Podman network, Grafana also on vps-services):
  Prometheus → Node Exporter (host) + Fail2Ban Exporter (host)
  Loki ← Promtail (container logs + systemd journal)
  Grafana → Prometheus + Loki (dashboards + log exploration)
```

The `vps-services` Podman network is deployed by `nginx_gateway` via Quadlet and shared by `searxng`, `filescdn`, and `grafana`. Nginx resolves backends by container name via Podman's aardvark-dns (10.89.0.1).

> **Note:** `nginx_gateway` must appear before `searxng`, `filescdn`, and `external-monitor` in the `host_roles` list, since it deploys the shared `vps-services` network.

## 🔧 Roles

### base
System hardening applied to all hosts: users, SSH, sudoers, UFW, fail2ban, chrony, kernel hardening, AppArmor, unattended-upgrades.

### nginx_gateway
- Installs Podman and deploys shared `vps-services` network via Quadlet
- Deploys Cloudflare ipset firewall (port 443 only from CF IPs)
- Certbot with dns-cloudflare plugin for wildcard certs
- Nginx with `set_real_ip_from` for all Cloudflare ranges
- Virtual host configs for each service
- Git-cloned static website

### searxng
SearXNG metasearch engine via Podman Quadlet on the `vps-services` network.

### filescdn
Lightweight Nginx container serving static files on the `vps-services` network.

### monitoring-vps
Installs node_exporter, fail2ban_exporter, and promtail as native systemd services. Promtail ships Podman container logs and systemd journal to Loki. UFW rules allow localhost and Podman subnet access (scraped by external-monitor's Prometheus).

### external-monitor
Podman Quadlet Prometheus + Loki + Grafana stack:
- Scrapes local node_exporter and fail2ban_exporter for VPS metrics
- Loki ingests logs from Promtail (containers + systemd journal)
- Grafana dashboard accessible at `monitoring.<domain>` via nginx reverse proxy

## 📊 Monitoring

The VPS runs a self-contained monitoring stack:
- **Node Exporter** (port 9100, localhost) — system metrics
- **Fail2Ban Exporter** (port 9191, localhost) — intrusion prevention metrics
- **Promtail** (native systemd) — ships container + journal logs to Loki
- **Prometheus** (port 9090, localhost) — scrapes exporters
- **Loki** (port 3100, localhost) — log aggregation
- **Grafana** (`monitoring.<domain>`) — dashboards + log exploration, reverse-proxied through nginx

## 📄 License

See [LICENSE](LICENSE).

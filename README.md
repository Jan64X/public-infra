> **Note:** I've since moved to a serverless setup, so this repo is not currently maintained. Archiving it here in case any of it is useful or if I want to resume using it in the future.

# Public Infrastructure Automation

Ansible playbook for managing a single Oracle Cloud VPS to run my public services. Deploys and configures web services behind an Nginx reverse proxy with Cloudflare integration.

## Overview

The VPS runs the following services:
- **Nginx Gateway** - Reverse proxy with Cloudflare-only firewall, certbot wildcard SSL, and static website hosting
- **SearXNG** - Privacy-respecting metasearch engine
- **Files CDN** - Static file hosting
- **Monitoring** - Prometheus + Loki + Grafana with Node Exporter, Fail2Ban Exporter, and Promtail

## Security

- Port 443 restricted to Cloudflare IPs via ipset + UFW
- Wildcard certs via certbot with Cloudflare DNS plugin
- SSH key-only auth, fail2ban, kernel hardening, AppArmor
- All services run as systemd-managed Podman containers via Quadlet

## Repository Structure

```
ansible/
├── ansible.cfg
├── site.yml
├── update.yml
├── inventory/hosts.yml
├── group_vars/all.yml
├── roles/
│   ├── base/
│   ├── nginx_gateway/
│   ├── searxng/
│   ├── filescdn/
│   ├── external-monitor/
│   └── monitoring-vps/
└── credentials/         # gitignored
```

## Quick Start

```bash
git clone <your-repo-url> public-infra
cd public-infra/ansible
cp inventory/hosts.yml.example inventory/hosts.yml
cp group_vars/all.yml.example group_vars/all.yml
# Place SSH keys in credentials/ssh_keys/
# Edit group_vars/all.yml with your domain, Cloudflare token, and credentials
ansible-playbook site.yml
```

## Architecture

```
Internet -> Cloudflare -> [443] Nginx Gateway
                                 ├── search.domain.com   -> SearXNG
                                 ├── cdn.domain.com      -> FileCDN
                                 ├── monitoring.domain.com -> Grafana
                                 └── domain.com          -> Static site

Prometheus -> Node Exporter + Fail2Ban Exporter
Loki <- Promtail (container logs + systemd journal)
Grafana -> Prometheus + Loki
```

All backend services share the `vps-services` Podman network, deployed by `nginx_gateway` via Quadlet. Nginx resolves backends by container name via aardvark-dns.

> `nginx_gateway` must appear before `searxng`, `filescdn`, and `external-monitor` in `host_roles`, since it creates the shared network.

## Roles

**base** - System hardening: users, SSH, sudoers, UFW, fail2ban, chrony, kernel settings, AppArmor, unattended-upgrades.

**nginx_gateway** - Podman setup, shared network, Cloudflare ipset firewall, certbot wildcard certs, Nginx virtual hosts, git-cloned static site.

**searxng** - SearXNG container on the `vps-services` network.

**filescdn** - Lightweight Nginx container serving static files.

**monitoring-vps** - Node Exporter, Fail2Ban Exporter, and Promtail as native systemd services. UFW rules allow scraping from localhost and the Podman subnet.

**external-monitor** - Prometheus, Loki, and Grafana via Quadlet. Grafana is accessible at `monitoring.<domain>` through the Nginx reverse proxy.

## License 

See [LICENSE](LICENSE).

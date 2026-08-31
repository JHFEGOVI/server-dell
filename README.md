# server-dell

Personal homelab server built on a Dell Inspiron 3443 running Ubuntu Server 24.04.
Documented as a DevOps/sysadmin portfolio project.

## Hardware

- **Model:** Dell Inspiron 3443
- **CPU:** Intel Core i5-5200U
- **RAM:** 8GB
- **Storage:** 256GB SSD
- **OS:** Ubuntu Server 24.04.04 LTS

## Architecture Overview

- **VPN:** WireGuard — self-managed, full tunnel configuration
- **Reverse proxy:** Nginx — manual configuration, no GUI tools
- **Containerization:** Docker CE + Docker Compose — one stack per service
- **SSL:** Let's Encrypt wildcard certificate via Certbot + DNS challenge
- **SSL termination:** At VPS level — WireGuard already encrypts the VPS→server leg
- **Monitoring:** Prometheus + Grafana + node-exporter + cAdvisor
- **Security:** ufw, fail2ban, SSH public key authentication only

## Repository Structure

```
server-dell/
├── docs/
│   ├── decisions/     # Architecture Decision Records (ADRs)
│   └── architecture.md
├── docker/            # Docker Compose stacks, one directory per service
├── nginx/
│   ├── vps/           # Nginx config running on the VPS
│   └── dell/          # Nginx config running on the server
├── system/            # System-level configs (.example files only)
├── scripts/           # Utility scripts
├── .gitignore
└── README.md
```

## Architecture Decision Records

| ADR | Title |
|-----|-------|
| [ADR-001](docs/decisions/ADR-001.md) | SSH public key authentication only |
| [ADR-002](docs/decisions/ADR-002.md) | SSL termination at VPS level |
| [ADR-003](docs/decisions/ADR-003.md) | Manual Nginx over Nginx Proxy Manager |
| [ADR-004](docs/decisions/ADR-004.md) | Separate Docker Compose stacks per service |
| [ADR-005](docs/decisions/ADR-005.md) | Shared proxy-network Docker network |
| [ADR-006](docs/decisions/ADR-006.md) | Isolated monitoring-internal Docker network |
| [ADR-007](docs/decisions/ADR-007.md) | No host-exposed ports on monitoring stack |
| [ADR-008](docs/decisions/ADR-008.md) | HTTP basic auth for admin tools |
| [ADR-009](docs/decisions/ADR-009.md) | PasswordAuthentication override in 50-cloud-init.conf |

## Security Notes

This repository contains **no secrets, credentials, or private keys**.
Files under `system/` are `.example` versions with placeholders only.
Real configuration files are excluded via `.gitignore`.

## Author

Jhon — [jhongomez.dev](https://jhongomez.dev)

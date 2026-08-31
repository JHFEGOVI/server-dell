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

## Security Notes

This repository contains **no secrets, credentials, or private keys**.
Files under `system/` are `.example` versions with placeholders only.
Real configuration files are excluded via `.gitignore`.

## Author

Jhon — [jhongomez.dev](https://jhongomez.dev)

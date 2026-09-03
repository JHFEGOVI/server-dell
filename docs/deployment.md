# Deployment Guide

This guide documents how to reproduce this homelab setup from scratch.
It assumes a fresh **Ubuntu Server 24.04 LTS** installation with:

- A user with sudo privileges
- Internet access (ethernet or WiFi)
- A DigitalOcean account (or equivalent VPS provider)
- A registered domain and Porkbun API credentials
- WireGuard-capable clients (laptop, desktop, mobile)

> All configuration file examples are in this repository.
> Copy the relevant `.example` file, remove the `.example` extension,
> and replace placeholders with your actual values before applying.

## 1. Base System

### Network configuration

Configure static IPs for both interfaces using Netplan.
Reference: [`system/netplan/00-network.yaml.example`](../system/netplan/00-network.yaml.example)

```bash
sudo cp /etc/netplan/00-network.yaml /etc/netplan/00-network.yaml.bak
sudo nano /etc/netplan/00-network.yaml
sudo netplan apply
```

### IP forwarding

Required for WireGuard to route traffic between peers.
Reference: [`system/sysctl-dell.conf`](../system/sysctl-dell.conf)

```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### Firewall (ufw)

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2847/tcp comment 'SSH'
sudo ufw allow 51820/udp comment 'WireGuard'
sudo ufw enable
```

## 2. WireGuard

### Dell server

Install WireGuard and generate server keys:

```bash
sudo apt install wireguard -y
wg genkey | sudo tee /etc/wireguard/server_private.key | wg pubkey | sudo tee /etc/wireguard/server_public.key
sudo chmod 600 /etc/wireguard/server_private.key
```

Generate a key pair for each client (ASUS, Fedora, mobile):

```bash
wg genkey | sudo tee /etc/wireguard/CLIENT_private.key | wg pubkey | sudo tee /etc/wireguard/CLIENT_public.key
```

Apply configuration:
Reference: [`system/wireguard/server-wg0.conf.example`](../system/wireguard/server-wg0.conf.example)

```bash
sudo nano /etc/wireguard/wg0.conf
sudo chmod 600 /etc/wireguard/wg0.conf
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```

### VPS relay

Reference: [`system/wireguard/vps-wg0.conf.example`](../system/wireguard/vps-wg0.conf.example)
Reference: [`system/sysctl-vps.conf`](../system/sysctl-vps.conf)

```bash
apt install wireguard -y
wg genkey | tee /etc/wireguard/vps_private.key | wg pubkey > /etc/wireguard/vps_public.key
chmod 600 /etc/wireguard/vps_private.key
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
nano /etc/wireguard/wg0.conf
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0
ufw allow 51820/udp comment 'WireGuard'
ufw allow 22/tcp comment 'SSH'
ufw allow 80/tcp comment 'HTTP'
ufw allow 443/tcp comment 'HTTPS'
ufw enable
```

### Clients

Reference: [`system/wireguard/client-wg0.conf.example`](../system/wireguard/client-wg0.conf.example)

```bash
sudo apt install wireguard -y  # Debian/Ubuntu
sudo dnf install wireguard-tools -y  # Fedora
sudo nano /etc/wireguard/wg0.conf
sudo chmod 600 /etc/wireguard/wg0.conf
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```

For mobile (iOS/Android), generate a QR code on the Dell:

```bash
sudo apt install qrencode -y
qrencode -t ansiutf8 < /tmp/mobile.conf
```

## 3. SSH Hardening

Generate an ED25519 key pair on each client machine:

```bash
ssh-keygen -t ed25519 -C "your_device_name"
ssh-copy-id -p 2847 -i ~/.ssh/id_ed25519.pub user@server_ip
```

Apply configuration:
Reference: [`system/ssh/sshd_config.example`](../system/ssh/sshd_config.example)
Reference: [`system/ssh/50-cloud-init.conf.example`](../system/ssh/50-cloud-init.conf.example)

```bash
sudo nano /etc/ssh/sshd_config
sudo nano /etc/ssh/sshd_config.d/50-cloud-init.conf
sudo systemctl restart ssh
```

Install and configure fail2ban:

```bash
sudo apt install fail2ban -y
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

## 4. DNS Configuration

In your domain registrar (Porkbun or equivalent):

- Create an `A` record pointing `YOUR_DOMAIN` → VPS public IP
- Create an `A` record pointing `*.YOUR_DOMAIN` → VPS public IP (wildcard)
- Set TTL to 600

## 5. Docker and Stacks

### Install Docker

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
```

### Create shared network

```bash
docker network create proxy-network
```

### Monitoring stack

Reference: [`docker/monitoring/docker-compose.yml`](../docker/monitoring/docker-compose.yml)
Reference: [`docker/monitoring/prometheus/prometheus.yml`](../docker/monitoring/prometheus/prometheus.yml)

```bash
cd ~/docker/monitoring
cp .env.example .env
nano .env  # set GRAFANA_ADMIN_PASSWORD
docker compose up -d
```

### Proxy stack

Reference: [`docker/proxy/docker-compose.yml`](../docker/proxy/docker-compose.yml)

```bash
cd ~/docker/proxy
cp .env.example .env
docker compose up -d
```

## 6. SSL Certificate

On the VPS, install Certbot and obtain a wildcard certificate via DNS challenge:

```bash
apt install certbot python3-certbot-dns-porkbun -y
mkdir -p /etc/letsencrypt/secrets
nano /etc/letsencrypt/secrets/porkbun.ini  # add Porkbun API credentials
chmod 600 /etc/letsencrypt/secrets/porkbun.ini
certbot certonly \
  --authenticator dns-porkbun \
  --dns-porkbun-credentials /etc/letsencrypt/secrets/porkbun.ini \
  -d YOUR_DOMAIN \
  -d *.YOUR_DOMAIN
```

## 7. Nginx

### VPS

```bash
apt install nginx -y
nano /etc/nginx/sites-available/YOUR_DOMAIN
ln -s /etc/nginx/sites-available/YOUR_DOMAIN /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
```

Reference: [`nginx/vps/jhongomez.dev.conf`](../nginx/vps/jhongomez.dev.conf)

### Dell proxy

Add a server block for each service under `~/docker/proxy/conf.d/`.
Reference: [`nginx/dell/conf.d/grafana.conf`](../nginx/dell/conf.d/grafana.conf)

For services requiring HTTP basic auth, generate an htpasswd file:

```bash
sudo apt install apache2-utils -y
sudo mkdir -p /etc/nginx/auth
sudo htpasswd -c /etc/nginx/auth/.htpasswd YOUR_USERNAME
```

Restart the proxy stack after adding new configurations:

```bash
cd ~/docker/proxy
docker compose restart
```

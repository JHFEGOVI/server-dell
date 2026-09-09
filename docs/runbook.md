# Runbook

Operational reference for day-to-day tasks. For initial setup, see [deployment.md](./deployment.md).

---

## WireGuard

### Add a new peer

On the local server — generate keys:

```bash
wg genkey | sudo tee /etc/wireguard/<name>_private.key | wg pubkey | sudo tee /etc/wireguard/<name>_public.key
sudo chmod 600 /etc/wireguard/<name>_private.key
```

On the local server — add peer to `wg0.conf`:

```bash
sudo nano /etc/wireguard/wg0.conf
# Add [Peer] block with PublicKey and AllowedIPs = 10.0.0.X/32
sudo systemctl restart wg-quick@wg0
```

On the VPS — add the same peer:

```bash
wg set wg0 peer <PUBLIC_KEY> allowed-ips 10.0.0.X/32
nano /etc/wireguard/wg0.conf
# Persist the peer in the file
systemctl restart wg-quick@wg0
```

### Restart WireGuard

On the local server or VPS:

```bash
sudo systemctl restart wg-quick@wg0
sudo wg show
```

On clients:

```bash
sudo systemctl restart wg-quick@wg0
ping -c 3 10.0.0.1
```

### Verify tunnel status

```bash
sudo wg show
ping -c 3 10.0.0.1   # from client to local server
ping -c 3 10.0.0.5   # from client to VPS
```

### Tunnel is down and not reconnecting

```bash
sudo systemctl stop wg-quick@wg0
sudo systemctl start wg-quick@wg0
sudo wg show
```

If the endpoint changed (local server on WiFi instead of Ethernet):

```bash
sudo wg set wg0 peer <VPS_PUBLIC_KEY> endpoint 10.0.0.X:51820
```

### Emergency access if WireGuard fails

```bash
# Via local Ethernet IP
ssh -p YOUR_SSH_PORT user@YOUR_SERVER_ETHERNET_IP
# Via local WiFi IP
ssh -p YOUR_SSH_PORT user@YOUR_SERVER_WIFI_IP
# Via DigitalOcean web console if VPS fails
# cloud.digitalocean.com → Droplets → Console
```

---

## Network

### Change static IP

```bash
sudo nano /etc/netplan/00-network.yaml
sudo netplan try
# Confirm with Enter if SSH is still responding
```

### Verify routing

```bash
ip route
ip addr show
```

### Ethernet did not come up after power loss

```bash
sudo netplan apply
```

### ISP or router change

If the local network range changes (new ISP, new router), SSH will stop responding
because the server's static IPs no longer match the new network.

1. Identify the new network range from another device:

```bash
ip route | grep default
```

2. Access the local server physically (monitor + keyboard) if SSH is unreachable.

3. Update Netplan with the new IPs, gateway, and WiFi SSID if needed:

```bash
sudo nano /etc/netplan/00-network.yaml
sudo netplan apply
```

4. Verify connectivity:

```bash
ip addr show enp7s0
ping YOUR_NEW_GATEWAY
```

5. Verify WireGuard tunnel is still up:

```bash
sudo wg show
```

> **Note:** The VPS and domain DNS records do not need to change — only the local
> network configuration is affected.

---

## SSH

### Add a new client key

On the local server:

```bash
echo "ssh-ed25519 YOUR_PUBLIC_KEY your_device_name" >> ~/.ssh/authorized_keys
```

### Verify password authentication is disabled

```bash
sudo sshd -T | grep passwordauthentication
# Expected output: passwordauthentication no
```

---

## SSL Certificate

### Check certificate expiration

```bash
certbot certificates
```

### Renew manually

```bash
certbot renew
```

### Verify automatic renewal

```bash
systemctl status certbot.timer
```

---

## Nginx [VPS]

### Add a new subdomain

1. Create an `A` record in your DNS registrar pointing `subdomain.YOUR_DOMAIN` → VPS public IP
2. Add a `server` block to `/etc/nginx/sites-available/YOUR_DOMAIN`
3. Verify and reload:

```bash
nginx -t
systemctl reload nginx
```

### Verify configuration

```bash
nginx -t
```

### Reload without downtime

```bash
systemctl reload nginx
```

---

## Nginx Proxy [Local Server]

### Add a new service

1. Create a new config file:

```bash
nano ~/docker/proxy/conf.d/service-name.conf
```

2. Restart the proxy stack:

```bash
cd ~/docker/proxy
docker compose restart
```

### View proxy logs

```bash
docker logs nginx-proxy
```

---

## Docker

### Add a new service to an existing stack

```bash
# 1. Edit docker-compose.yml and add the service
# 2. Apply without touching running services
docker compose up -d
```

### Verify container network membership

```bash
docker inspect container-name --format '{{json .NetworkSettings.Networks}}'
```

### Verify communication between containers

```bash
docker exec nginx-proxy ping -c 2 service-name
```

---

## Monitoring Stack

### Check stack status

```bash
cd ~/docker/monitoring
docker compose ps
```

### Restart an individual service

```bash
docker compose restart prometheus
docker compose restart grafana
```

### View logs

```bash
docker compose logs -f
docker compose logs -f grafana
```

### Verify Prometheus targets

```bash
curl localhost:9090/api/v1/targets | python3 -m json.tool | grep -E '"job"|"health"'
```

### Stop and restart the full stack

```bash
docker compose down
docker compose up -d
```

### Reset Grafana admin password

Option A — via CLI (recommended, preserves configuration):

```bash
docker compose exec grafana grafana-cli admin reset-admin-password NEW_PASSWORD
docker compose restart grafana
```

Option B — delete volume and recreate (loses all configuration):

```bash
docker compose down
docker volume rm monitoring_grafana-data
docker compose up -d
```

> **Note:** `GF_SECURITY_ADMIN_PASSWORD` in `.env` only applies on the first
> initialization. Subsequent password changes require the CLI or volume deletion.

# Traefik

Reverse proxy with automatic HTTPS via Let's Encrypt. Acts as the single entry point for all services on the VPS.

## Prerequisites

- VPS with a public IP address
- Docker and Docker Compose installed
- A domain name (e.g. `example.com`) with DNS access
- Ports **80** and **443** open on the firewall

## 1. Configure DNS

Point your domain to the VPS public IP. You need at minimum:

| Type | Name               | Value            |
|------|--------------------|------------------|
| A    | `traefik`          | `<VPS_IP>`       |

This gives you `traefik.example.com` for the dashboard.

For every service you plan to proxy, add an additional A record:

| Type | Name               | Value            |
|------|--------------------|------------------|
| A    | `grafana`          | `<VPS_IP>`       |
| A    | `gitea`            | `<VPS_IP>`       |
| ...  | ...                | ...              |

Alternatively, use a wildcard record to cover all subdomains at once:

| Type | Name               | Value            |
|------|--------------------|------------------|
| A    | `*`                | `<VPS_IP>`       |

> DNS propagation can take a few minutes. Verify with `dig traefik.example.com` before proceeding.

## 2. Open firewall ports

Make sure your VPS firewall allows inbound traffic on ports 80 and 443. For example, with `ufw`:

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

Port 80 is required even though all traffic is redirected to HTTPS — Let's Encrypt uses it for the HTTP challenge.

## 3. Create the `.env` file

```bash
cp .env.example .env
```

Edit `.env` and set the values:

```dotenv
TRAEFIK_DOMAIN=traefik.example.com
ACME_EMAIL=admin@example.com
DASHBOARD_AUTH=admin:$$2y$$05$$...
```

### Generating `DASHBOARD_AUTH`

Use `htpasswd` to create a bcrypt-hashed password:

```bash
# Install if needed:
# Debian/Ubuntu: sudo apt install apache2-utils
# Alpine:        apk add apache2-utils

htpasswd -nB admin
```

You will be prompted for a password. The output will look like:

```
admin:$2y$05$Wz1...
```

**Important:** before pasting into `.env`, escape every `$` by doubling it to `$$`. Docker Compose treats `$` as variable interpolation. For example:

```
# htpasswd output:
admin:$2y$05$abc123

# In .env:
DASHBOARD_AUTH=admin:$$2y$$05$$abc123
```

## 4. Start Traefik

```bash
docker compose up -d
```

Verify it's running:

```bash
docker compose logs -f
```

Look for `Configuration loaded from flags` and no ACME errors. The first certificate request may take a few seconds.

Once ready, open `https://traefik.example.com` in your browser and log in with the credentials you set.

## 5. Connect other services

Traefik creates and owns the `traefik-public` Docker network. Other compose stacks join this network as external and use labels to register routes.

Example for a service running on port 3000:

```yaml
services:
  myapp:
    image: myapp:latest
    networks:
      - traefik-public
    labels:
      - traefik.enable=true
      - traefik.http.routers.myapp.rule=Host(`app.example.com`)
      - traefik.http.routers.myapp.entrypoints=websecure
      - traefik.http.routers.myapp.tls.certresolver=letsencrypt
      - traefik.http.services.myapp.loadbalancer.server.port=3000

networks:
  traefik-public:
    external: true
```

Key points:

- `traefik.enable=true` — required because `exposedbydefault=false`
- `Host(...)` — must match a DNS record pointing to the VPS
- `loadbalancer.server.port` — only needed if the container exposes multiple ports or no `EXPOSE` in the image
- Do **not** publish ports to the host (`ports:`) for proxied services — Traefik routes traffic through the Docker network

## Troubleshooting

**Certificate not issued**
- Verify DNS resolves to the VPS: `dig +short traefik.example.com`
- Verify port 80 is reachable from outside: `curl -I http://traefik.example.com`
- Check ACME logs: `docker compose logs traefik | grep acme`

**Dashboard returns 401**
- Verify `DASHBOARD_AUTH` has `$$` escaping in `.env`
- Recreate the container after changing `.env`: `docker compose up -d --force-recreate`

**Service not appearing in Traefik**
- Confirm the service is on the `traefik-public` network: `docker network inspect traefik-public`
- Confirm `traefik.enable=true` label is set
- Check service logs for startup errors

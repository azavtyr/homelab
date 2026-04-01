# Traefik

HTTPS reverse proxy for homelab services. Uses Let's Encrypt wildcard certificates via Cloudflare DNS challenge. Designed to work with Tailscale — all traffic stays inside the VPN, nothing is exposed to the public internet.

## Prerequisites

- A domain managed by Cloudflare (e.g. `example.com`)
- A Cloudflare API token with `Zone:DNS:Edit` permission
- [Tailscale](../tailscale/README.md) running on the server

## Setup

### 1. Create Cloudflare API token

Go to [Cloudflare API Tokens](https://dash.cloudflare.com/profile/api-tokens) and create a token with:

- **Permissions**: Zone → DNS → Edit
- **Zone Resources**: Include → your domain

### 2. Configure

```bash
cp .env.example .env
```

Edit `.env`:

```
DOMAIN=home.example.com
ACME_EMAIL=you@example.com
CF_DNS_API_TOKEN=your-cloudflare-api-token
```

`DOMAIN` is the base for all service subdomains. Services will be accessible at `grafana.home.example.com`, `gitea.home.example.com`, etc.

### 3. Start Traefik

```bash
docker compose up -d
```

Traefik dashboard is available at `https://traefik.home.example.com` (replace with your actual domain).

On first start, Traefik will request a certificate from Let's Encrypt via Cloudflare DNS challenge. This takes a minute or two. Certificates are stored in a Docker volume and persist across restarts.

### 4. DNS setup

You need `*.home.example.com` to resolve to the server's Tailscale IP. Two options:

#### Option A: Tailscale DNS (recommended)

In [Tailscale Admin Console](https://login.tailscale.com/admin/dns) → Nameservers → Add Split DNS:

- **Domain**: `home.example.com`
- **Nameserver**: your server's Tailscale IP (e.g. `100.x.y.z`)

Then on the server, run a lightweight DNS that resolves `*.home.example.com` to itself. The simplest way is adding a dnsmasq entry:

```
address=/home.example.com/100.x.y.z
```

Or use Tailscale's built-in DNS by adding individual records.

#### Option B: Cloudflare DNS

Add a wildcard A record in Cloudflare:

```
*.home.example.com → 100.x.y.z (Tailscale IP)
```

Set **Proxy status** to **DNS only** (grey cloud). This is simpler but the DNS record is public (though the IP is only reachable via Tailscale).

## Connect a service

Add labels to any Docker Compose service:

```yaml
services:
  grafana:
    image: grafana/grafana
    networks:
      - traefik-public
    labels:
      - traefik.enable=true
      - traefik.http.routers.grafana.rule=Host(`grafana.home.example.com`)
      - traefik.http.routers.grafana.entrypoints=websecure
      - traefik.http.routers.grafana.tls.certresolver=letsencrypt
      - traefik.http.services.grafana.loadbalancer.server.port=3000

networks:
  traefik-public:
    external: true
```

Key points:

- Use `websecure` entrypoint (not `web`) for HTTPS
- Add `tls.certresolver=letsencrypt` to get a certificate
- `loadbalancer.server.port` is needed when the container listens on a non-standard port
- Do **not** publish ports to the host — Traefik routes through the Docker network
- The service must be on the `traefik-public` network

## Using environment variables for domains

To avoid hardcoding the domain, use a variable in your service's `.env`:

```yaml
labels:
  - traefik.http.routers.grafana.rule=Host(`grafana.${DOMAIN}`)
```

With `DOMAIN=home.example.com` in the service's `.env` file.

## Troubleshooting

**Certificate not issued**
- Check Traefik logs: `docker logs traefik`
- Verify the Cloudflare API token has DNS edit permissions for the correct zone
- Let's Encrypt has [rate limits](https://letsencrypt.org/docs/rate-limits/) — if testing, add `--certificatesresolvers.letsencrypt.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/v2/directory` to use the staging server

**Service not showing in dashboard**
- Check it's on the `traefik-public` network: `docker network inspect traefik-public`
- Check `traefik.enable=true` label is set

**Browser shows certificate warning**
- Wait a few minutes for the DNS challenge to complete
- Check that the domain matches what's in the `Host()` rule

**Can't reach services remotely**
- Verify Tailscale is connected on your device: `tailscale status`
- Check DNS resolves correctly: `dig grafana.home.example.com`
- Make sure the server's firewall allows traffic from Tailscale interface

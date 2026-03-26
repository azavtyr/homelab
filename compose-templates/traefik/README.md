# Traefik

Local reverse proxy for homelab services. Routes traffic by hostname so you can access services by name instead of IP:port.

## Setup

### 1. Start Traefik

```bash
docker compose up -d
```

Traefik dashboard is available at `http://traefik.home` (or whatever you set in `TRAEFIK_DOMAIN`).

### 2. Connect a service

Add labels to any compose service to register it with Traefik. Example:

```yaml
services:
  grafana:
    image: grafana/grafana
    networks:
      - traefik-public
    labels:
      - traefik.enable=true
      - traefik.http.routers.grafana.rule=Host(`grafana.home`)
      - traefik.http.services.grafana.loadbalancer.server.port=3000

networks:
  traefik-public:
    external: true
```

Change `grafana.home` to any name you want. The `Host()` rule is what Traefik matches on.

`loadbalancer.server.port` is only needed when the container listens on a non-standard port or exposes multiple ports.

Do **not** publish ports to the host (`ports:`) for proxied services — Traefik routes traffic through the Docker network.

### 3. Configure local DNS

For the browser to know that `grafana.home` points to your server, you need DNS resolution. Pick one:

#### Option A: Pi-hole / AdGuard Home (recommended)

Add DNS records in Pi-hole (Local DNS → DNS Records):

```
grafana.home    → 192.168.1.50
gitea.home      → 192.168.1.50
portainer.home  → 192.168.1.50
```

All devices on the network will resolve these automatically.

#### Option B: Wildcard with dnsmasq

If your router supports dnsmasq (GL.iNet routers do), add one rule to resolve all `*.home` to the server:

```
address=/home/192.168.1.50
```

In GL.iNet admin panel: Network → DNS → Custom DNS Rules, or via SSH in `/etc/dnsmasq.d/`. This way any new `*.home` subdomain works without adding individual records.

#### Option C: /etc/hosts (per device)

On each device, add entries to `/etc/hosts` (Linux/Mac) or `C:\Windows\System32\drivers\etc\hosts` (Windows):

```
192.168.1.50  grafana.home gitea.home portainer.home
```

Quick but has to be repeated on every device.

## Example: full service setup

After setup, adding a new service with a custom address is just two lines in its labels:

```yaml
labels:
  - traefik.enable=true
  - traefik.http.routers.myapp.rule=Host(`myapp.home`)
```

Pick any name — `git.home`, `monitor.home`, `nas.home` — as long as DNS resolves it to the server.

## Troubleshooting

**Service not showing in dashboard**
- Check it's on the `traefik-public` network: `docker network inspect traefik-public`
- Check `traefik.enable=true` label is set

**Browser can't resolve hostname**
- Verify DNS: `dig grafana.home` or `nslookup grafana.home`
- If using Pi-hole, make sure the device uses Pi-hole as its DNS server

**Wrong service responds**
- Check for duplicate `Host()` rules in the Traefik dashboard at `:8080`

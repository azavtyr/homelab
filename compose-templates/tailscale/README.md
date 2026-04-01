# Tailscale

VPN access to homelab services. Runs on the server so all Docker services are reachable through the Tailscale network. Nothing is exposed to the public internet.

## Setup

### 1. Get an auth key

Go to [Tailscale Admin Console](https://login.tailscale.com/admin/settings/keys) and generate an auth key. Use a **reusable** key if you want the container to re-authenticate after restarts.

### 2. Configure

```bash
cp .env.example .env
```

Edit `.env`:

```
TS_AUTHKEY=tskey-auth-xxxxx   # your auth key from step 1
TS_HOSTNAME=homelab            # how this machine appears in Tailscale
```

### 3. Start

```bash
docker compose up -d
```

The server will appear in your Tailscale network. You can verify with:

```bash
docker exec tailscale tailscale status
```

### 4. Install Tailscale on your devices

Install Tailscale on your phone, laptop, etc. from [tailscale.com/download](https://tailscale.com/download). Once connected, your devices can reach the server via its Tailscale IP.

### 5. Firewall

After confirming Tailscale works, block all incoming traffic on your router except from the local network. All remote access goes through Tailscale.

## MagicDNS

Tailscale has built-in [MagicDNS](https://tailscale.com/kb/1081/magicdns) that resolves machine names automatically. Your server will be reachable at `homelab` (or whatever `TS_HOSTNAME` you set) from any device on the tailnet.

For custom domains like `grafana.home.example.com`, add a DNS record in Tailscale Admin Console pointing `*.home.example.com` to the server's Tailscale IP. See the [Traefik README](../traefik/README.md) for the full HTTPS setup.

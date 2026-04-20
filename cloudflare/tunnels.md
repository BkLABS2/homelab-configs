# Cloudflare Tunnels

Zero Trust remote access to homelab services without opening any inbound ports on the router. All services are published under `krausshomelab.com` using Cloudflare Tunnels, with traffic proxied through Cloudflare's edge before reaching the homelab.

---

## Architecture

```
Internet
    │
    │  (HTTPS — Cloudflare edge)
    ▼
Cloudflare Zero Trust
    │
    │  (Encrypted outbound tunnel — no open ports)
    ▼
cloudflared daemon (running on homelab)
    │
    ├──▶ Proxmox VE        → proxmox.krausshomelab.com
    ├──▶ Synology NAS       → sharkdrive.krausshomelab.com
    └──▶ Raspberry Pi (SSH) → pi.krausshomelab.com
```

No inbound firewall rules required. The `cloudflared` daemon initiates outbound connections to Cloudflare — traffic flows back through that tunnel.

---

## Tunnels

### Proxmox VE — `proxmox.krausshomelab.com`

Exposes the Proxmox web UI (port 8006) remotely over HTTPS.

```yaml
tunnel: <TUNNEL_ID>
credentials-file: /root/.cloudflared/<TUNNEL_ID>.json

ingress:
  - hostname: proxmox.krausshomelab.com
    service: https://localhost:8006
    originRequest:
      noTLSVerify: true
  - service: http_status:404
```

> `noTLSVerify: true` is set because Proxmox uses a self-signed certificate internally. Cloudflare handles the public-facing TLS.

---

### Synology NAS — `sharkdrive.krausshomelab.com`

Exposes the Synology DSM web interface remotely over HTTPS.

```yaml
tunnel: <TUNNEL_ID>
credentials-file: /root/.cloudflared/<TUNNEL_ID>.json

ingress:
  - hostname: sharkdrive.krausshomelab.com
    service: https://localhost:5001
    originRequest:
      noTLSVerify: true
  - service: http_status:404
```

---

### Raspberry Pi — `pi.krausshomelab.com`

Exposes SSH access to the Raspberry Pi via Cloudflare's SSH browser rendering or the `cloudflared` client. SSH key authentication only — password auth is disabled.

```yaml
tunnel: <TUNNEL_ID>
credentials-file: /home/pi/.cloudflared/<TUNNEL_ID>.json

ingress:
  - hostname: pi.krausshomelab.com
    service: ssh://localhost:2222
  - service: http_status:404
```

**SSH config on the Pi (`/etc/ssh/sshd_config`)**
```
Port 2222
PasswordAuthentication no
PubkeyAuthentication yes
```

**Connecting remotely via `cloudflared`**
```bash
# Local machine ~/.ssh/config
Host pi.krausshomelab.com
    ProxyCommand cloudflared access ssh --hostname %h
    User pi
    IdentityFile ~/.ssh/<YOUR_KEY>
    Port 2222
```

Then connect normally:
```bash
ssh pi.krausshomelab.com
```

---

## Running cloudflared as a Service

On each host, `cloudflared` runs as a system service so it starts automatically on boot.

```bash
# Install and register the service
cloudflared service install

# Check status
systemctl status cloudflared

# View logs
journalctl -u cloudflared -f
```

---

## DNS

Each tunnel hostname has a corresponding CNAME record in Cloudflare DNS pointing to the tunnel endpoint. These are managed automatically by `cloudflared` when the tunnel is created — no manual DNS configuration needed.

| Hostname | Type | Target |
|---|---|---|
| `proxmox.krausshomelab.com` | CNAME | `<TUNNEL_ID>.cfargotunnel.com` |
| `sharkdrive.krausshomelab.com` | CNAME | `<TUNNEL_ID>.cfargotunnel.com` |
| `pi.krausshomelab.com` | CNAME | `<TUNNEL_ID>.cfargotunnel.com` |

---

## Security Notes

- No ports are open on the router — all traffic flows outbound through the tunnel
- SSH password authentication is disabled on the Raspberry Pi; key auth only
- Cloudflare Zero Trust policies can be layered on top for additional access control (email OTP, service tokens, etc.)
- Tunnel credentials files are stored locally and never committed to this repo

---

## Skills Demonstrated

`Cloudflare Tunnels` `Zero Trust` `cloudflared` `SSH hardening` `DNS` `Remote access` `Linux` `Raspberry Pi` `Proxmox` `Synology DSM`

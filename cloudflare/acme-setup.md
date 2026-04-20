# TLS Certificate Automation with acme.sh

Automated Let's Encrypt certificate provisioning for homelab services using `acme.sh` with the Cloudflare DNS plugin. Certificates are issued via the DNS-01 challenge — no open ports required, and wildcard certs are supported.

---

## Why DNS-01

The standard HTTP-01 challenge requires a publicly reachable web server on port 80. Since homelab services sit behind Cloudflare Tunnels with no open inbound ports, DNS-01 is used instead. `acme.sh` temporarily creates a TXT record in Cloudflare DNS to prove domain ownership, then cleans it up automatically after the cert is issued.

---

## Setup

### 1. Install acme.sh

```bash
curl https://get.acme.sh | sh
source ~/.bashrc
```

### 2. Configure Cloudflare API credentials

Create a scoped API token in the Cloudflare dashboard with **Zone:DNS:Edit** permissions for `krausshomelab.com` only. Then export it in your shell:

```bash
export CF_Token="<CF_API_TOKEN>"
export CF_Account_ID="<CF_ACCOUNT_ID>"
export CF_Zone_ID="<CF_ZONE_ID>"
```

> Store these in `~/.bashrc` or `~/.zshrc` so they persist across sessions. Never hardcode them in scripts.

### 3. Issue a certificate

**Single hostname**
```bash
acme.sh --issue \
  --dns dns_cf \
  -d proxmox.krausshomelab.com \
  --server letsencrypt
```

**Wildcard (covers all subdomains)**
```bash
acme.sh --issue \
  --dns dns_cf \
  -d krausshomelab.com \
  -d "*.krausshomelab.com" \
  --server letsencrypt
```

---

## Certificates Issued

| Hostname | Type | Deployed To |
|---|---|---|
| `proxmox.krausshomelab.com` | Single | Proxmox VE |
| `sharkdrive.krausshomelab.com` | Single | Synology DSM |
| `pi.krausshomelab.com` | Single | Raspberry Pi |

---

## Deploying to Proxmox

Proxmox expects the cert and key at a specific path. `acme.sh` can deploy directly after issuance:

```bash
acme.sh --deploy \
  -d proxmox.krausshomelab.com \
  --deploy-hook proxmox
```

Or manually copy the files:

```bash
acme.sh --install-cert -d proxmox.krausshomelab.com \
  --cert-file     /etc/pve/local/pveproxy-ssl.pem \
  --key-file      /etc/pve/local/pveproxy-ssl.key \
  --reloadcmd     "systemctl restart pveproxy"
```

---

## Deploying to Synology DSM

Synology doesn't have a native `acme.sh` deploy hook. Certificates are copied to the DSM cert directory and the web server is restarted:

```bash
acme.sh --install-cert -d sharkdrive.krausshomelab.com \
  --cert-file     /usr/syno/etc/certificate/system/default/cert.pem \
  --key-file      /usr/syno/etc/certificate/system/default/privkey.pem \
  --fullchain-file /usr/syno/etc/certificate/system/default/fullchain.pem \
  --reloadcmd     "synoservicectl --reload nginx"
```

---

## Auto-Renewal

`acme.sh` installs a cron job automatically on install. Certificates are checked and renewed every 60 days (Let's Encrypt certs expire after 90).

Verify the cron job is in place:

```bash
crontab -l | grep acme
```

Should show something like:
```
0 0 * * * /root/.acme.sh/acme.sh --cron --home /root/.acme.sh > /dev/null
```

Test a renewal manually without actually renewing:
```bash
acme.sh --renew -d proxmox.krausshomelab.com --force --dry-run
```

---

## Security Notes

- API token is scoped to DNS edit on `krausshomelab.com` only — minimum required permissions
- Credentials are stored as environment variables, never in scripts or this repo
- TXT records created during the DNS-01 challenge are cleaned up automatically by `acme.sh`
- Private keys stay on the host they were issued for and are never committed to this repo

---

## Skills Demonstrated

`acme.sh` `Let's Encrypt` `DNS-01 challenge` `Cloudflare API` `TLS/SSL` `Certificate automation` `Linux` `Cron` `Proxmox` `Synology DSM`

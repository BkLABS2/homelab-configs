# Splunk SIEM Pipeline

An automated threat detection and response pipeline built on Splunk Enterprise. Logs from homelab VMs are forwarded into Splunk, where brute-force attempts are detected by Fail2Ban and automatically blocked at the network edge via the Cloudflare WAF API — no manual intervention required.

---

## Architecture

```
Proxmox VMs
    │
    │  (Universal Forwarder — syslog + auth logs)
    ▼
Splunk Enterprise
    │
    │  (Saved search triggers on threshold breach)
    ▼
Fail2Ban
    │
    │  (Python script reads Fail2Ban ban log)
    ▼
Cloudflare WAF API
    │
    │  (IP rule created in Firewall Rules)
    ▼
Edge Block
```

---

## Components

### 1. Splunk Universal Forwarder

Installed on each Proxmox VM. Monitors auth logs and forwards events to the Splunk indexer.

**`inputs.conf`**
```ini
[monitor:///var/log/auth.log]
index = <YOUR_INDEX>
sourcetype = linux_secure

[monitor:///var/log/syslog]
index = <YOUR_INDEX>
sourcetype = syslog
```

**`outputs.conf`**
```ini
[tcpout]
defaultGroup = splunk_indexer

[tcpout:splunk_indexer]
server = <SPLUNK_HOST>:9997
```

---

### 2. Splunk Saved Search (Alert)

A scheduled search runs every 5 minutes and triggers when a source IP exceeds the failed login threshold. The alert fires an action that writes the offending IP to a watch file consumed by the Python script.

**Search query**
```spl
index=<YOUR_INDEX> sourcetype=linux_secure "Failed password"
| stats count by src_ip
| where count > 10
| outputlookup banned_ips.csv
```

**Alert settings**
- Schedule: every 5 minutes
- Trigger condition: number of results > 0
- Action: run script → `cf_ban.py`

---

### 3. Fail2Ban

Monitors `auth.log` and SSH logs for brute-force patterns. Bans are written to a log file that the Python script tails.

**`jail.local`**
```ini
[sshd]
enabled   = true
port      = <SSH_PORT>
filter    = sshd
logpath   = /var/log/auth.log
maxretry  = 5
bantime   = 3600
findtime  = 600
action    = %(action_mw)s
```

**`action.d/cf-ban.conf`** — custom action that calls the Python script on ban
```ini
[Definition]
actionban  = python3 /opt/homelab/cf_ban.py --ban <ip>
actionunban = python3 /opt/homelab/cf_ban.py --unban <ip>
```

---

### 4. Python Script — `cf_ban.py`

Reads the banned IP from Fail2Ban's action and creates (or deletes) a block rule in Cloudflare WAF via the API.

```python
import argparse
import requests

CF_API_TOKEN = "<CF_API_TOKEN>"   # store in env var, never hardcode
CF_ZONE_ID   = "<CF_ZONE_ID>"
CF_API_URL   = f"https://api.cloudflare.com/client/v4/zones/{CF_ZONE_ID}/firewall/rules"

HEADERS = {
    "Authorization": f"Bearer {CF_API_TOKEN}",
    "Content-Type": "application/json"
}

def ban(ip: str):
    payload = [{
        "filter": {
            "expression": f'ip.src eq {ip}',
            "description": f"Auto-ban: {ip}"
        },
        "action": "block",
        "description": f"Fail2Ban block: {ip}"
    }]
    r = requests.post(CF_API_URL, headers=HEADERS, json=payload)
    print(f"[BAN] {ip} → {r.status_code}")

def unban(ip: str):
    # Fetch rules, find matching IP, delete by rule ID
    r = requests.get(CF_API_URL, headers=HEADERS)
    rules = r.json().get("result", [])
    for rule in rules:
        if ip in rule.get("filter", {}).get("expression", ""):
            rule_id = rule["id"]
            requests.delete(f"{CF_API_URL}/{rule_id}", headers=HEADERS)
            print(f"[UNBAN] {ip} → rule {rule_id} deleted")

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--ban")
    parser.add_argument("--unban")
    args = parser.parse_args()
    if args.ban:
        ban(args.ban)
    elif args.unban:
        unban(args.unban)
```

> **Security note:** `CF_API_TOKEN` is loaded from an environment variable in production — never hardcoded. Set it with `export CF_API_TOKEN=your_token` or store it in a `.env` file that is listed in `.gitignore`.

---

### 5. Cloudflare WAF

Receives the block rule from `cf_ban.py` and enforces it at the edge — before traffic ever reaches the homelab. Rules are visible in the Cloudflare dashboard under **Security → WAF → Firewall Rules**.

---

## Environment Variables

Never commit real values. Store these in your shell profile or a `.env` file:

```bash
export CF_API_TOKEN="<your_cloudflare_api_token>"
export CF_ZONE_ID="<your_cloudflare_zone_id>"
export SPLUNK_HOST="<your_splunk_ip>"
```

---

## Skills Demonstrated

`Splunk` `SPL` `Universal Forwarder` `Fail2Ban` `Python` `Cloudflare API` `Linux` `SSH hardening` `Automated threat response` `SIEM`

# homelab-configs

A collection of configurations, notes, and documentation from my personal homelab — built around Proxmox VE with a focus on virtualization, network security, and monitoring. This repo serves as both a reference and a portfolio of the infrastructure I've designed and maintained.

---

## Infrastructure Overview

| Component | Details |
|---|---|
| Hypervisor | Proxmox VE (bare metal) |
| Domain | `krausshomelab.com` (Cloudflare DNS) |
| Remote Access | Cloudflare Tunnels (Zero Trust) |
| Monitoring / SIEM | Splunk Enterprise |
| Identity | Active Directory Domain Services (Windows Server 2019) |
| Storage | Synology NAS (`sharkdrive`) |
| Edge Device | Raspberry Pi 4 |

---

## Sections

### Proxmox VE
Virtual machine and container infrastructure running on bare metal. Hosts Windows Server 2019 (AD DS), Splunk, and various Linux VMs.

- VM provisioning and resource allocation
- Network bridge and VLAN configuration
- Snapshot and backup strategies

### Cloudflare Tunnels & DNS
Zero Trust remote access without exposing open ports. All services are published under `krausshomelab.com` using Cloudflare Tunnels with DNS-01 TLS certificates via `acme.sh`.

- Tunnel configuration for Proxmox, Synology NAS, and Raspberry Pi
- Let's Encrypt cert automation with Cloudflare DNS plugin
- SSH key authentication on non-standard ports

### Splunk & SIEM Pipeline
A home SOC pipeline that ingests logs from the homelab, detects brute-force attempts via Fail2Ban, and pushes firewall rules to Cloudflare WAF automatically.

```
Proxmox VMs → Splunk → Fail2Ban → Python script → Cloudflare WAF
```

- Universal Forwarder configuration
- Custom alerts and dashboards
- Automated threat response via Cloudflare API

---

## Skills Demonstrated

`Proxmox VE` `Linux` `Windows Server 2019` `Active Directory` `Splunk` `Cloudflare` `Bash` `PowerShell` `Python` `Networking` `DNS` `TLS/SSL` `Zero Trust`

---

## About

Built and maintained by [Brandon Krauss](https://www.linkedin.com/in/brandon-krauss-a8a847282/) — IT Technician and Cybersecurity student at Farmingdale State College. CompTIA A+ Certified, pursuing Security+.

> This homelab is an ongoing project. Configs and documentation are added as new components are built out.

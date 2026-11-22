# Home Lab

This is my home lab: a small set of repurposed PCs I'm using to build hands-on infrastructure skills in virtualization, networking, backups, and self-hosting.

My background is on the support and applications side of IT: Mac repair, then help desk, and now an IT Business Analyst supporting business applications and working on projects. AI is changing IT quickly, and this lab is how I'm rounding out the infrastructure side so I understand the whole stack. These notes cover what I built, what went wrong, and what I learned from it.

## Hardware

| Device | Role |
|---|---|
| Dell OptiPlex 5050 Micro (i7-6700, 16 GB RAM, 512 GB SSD) | Main Proxmox VE server |
| Lenovo ThinkCentre M625q Tiny | Future Proxmox Backup Server |
| Seagate 5 TB USB HDD | Future media / file storage |
| Dell XPS 13 | Test client |
| Gaming tower | Management workstation |

## Network

Everything sits on my home LAN behind an AT&T fiber gateway.

| Item | Address |
|---|---|
| Subnet | 192.168.1.0/24 |
| Gateway / DNS | 192.168.1.254 |
| Proxmox server (`pve.home.lan`) | 192.168.1.10 (static, outside the DHCP range) |
| Pi-hole DNS (planned) | 192.168.1.11 |
| Backup server (planned) | 192.168.1.12 |
| Test container `test01` | 192.168.1.20 |

## Documentation

| # | Topic | Status |
|---|---|---|
| 01 | [Installing Proxmox VE](docs/01-proxmox-install.md) | Done |
| 02 | [My first LXC container](docs/02-first-container.md) | Done |
| 03 | Pi-hole and local DNS names | Next |

## Roadmap

0. Hardware check (done)
1. Proxmox VE install (done)
2. Security baseline and first VM / container (first container done)
3. Pi-hole and local DNS names
4. Jellyfin media server
5. Tailscale remote access
6. Backups (Proxmox Backup Server) and monitoring
7. File storage (Samba, Nextcloud)
8. Own domain, HTTPS reverse proxy, Vaultwarden
9. Automation with Git and Ansible
10. My own router / firewall (OPNsense, VLANs)

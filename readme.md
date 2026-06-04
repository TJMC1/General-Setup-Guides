# 🏠 Homelab Guides

A collection of personal infrastructure guides for self-hosted services, Proxmox VMs, networking, and homelab tooling.

---

## 📚 Guides

| Guide | Description | Tags |
|---|---|---|
| [Nextcloud on Ubuntu + Proxmox](guides/nextcloud/nextcloud-ubuntu-proxmox.md) | Full bare-metal install on Ubuntu 24.04 LTS with self-signed HTTPS, NGINX Proxy Manager, SMB external storage, and LDAP/AD integration | `proxmox` `nextcloud` `ubuntu` `nginx` `ldap` `smb` |

---

## ⚠️ Notes

- Guides are written for **my specific environment** — adapt IP ranges, domain names, and VLANs to match yours.
- Commands assume a **non-root sudo user** unless stated otherwise.
- Always **snapshot your Proxmox VM** before major changes or upgrades.
- Passwords in examples are placeholders — never commit real credentials.

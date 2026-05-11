# Storage

## UNAS Pro (primary NAS)

- **Hostname:** UNAS-Pro
- **IP:** 192.168.1.221
- **Capacity:** 14.9 TB
- **Protocol:** NFS
- **Proxmox pool:** `unas_PRO`
- **Shared:** Yes — all Proxmox nodes mount it

This is the primary storage for all new VM disks and cloud-init templates. Prefer this over `local-lvm` for any new provisioning.

NFS mount (Proxmox): configured cluster-wide in datacenter storage config.  
NFS mount (Docker Swarm): `192.168.101.221:/Swarm_NFS` → `/mnt/swarm-nfs` on all swarm nodes.

## TrueNAS (secondary NAS)

- **Hostname:** truenas / `truenas.local.vidallabs.com`
- **IP:** 192.168.1.10
- **Capacity:** 3.3 TB
- **Protocol:** NFS
- **Proxmox pool:** `TNasScale`
- **Shared:** Yes

Mostly empty. Used as overflow or archive tier.

## Proxmox Backup Server (PBS)

- **IP:** 192.168.1.4 (`pbs.local.vidallabs.com`)
- **Capacity:** 860 GB
- **Proxmox pool:** `pbs`
- **Shared:** Yes — all nodes can back up to it

Backup schedule: Sundays at 01:00, snapshot mode, zstd compression, keep-last 7.  
See [Proxmox cluster doc](../compute/proxmox.md) for full backup job details.

## Local LVM (per-node)

- **Pool:** `local-lvm`
- **Type:** LVM-thin
- **Shared:** No
- **Use:** Legacy VMs and temporary scratch — avoid for new VMs

Each Proxmox node has its own local-lvm. VMs stored here cannot migrate between nodes without copying disks first.

## Storage summary

| Pool | Type | Shared | Total | Recommended for |
|------|------|--------|-------|-----------------|
| unas_PRO | NFS | Yes | 14.9 TB | New VM disks, templates |
| TNasScale | NFS | Yes | 3.3 TB | Archive / overflow |
| pbs | PBS | Yes | 860 GB | Backups only |
| local-lvm | LVM-thin | No | — | Avoid (legacy) |

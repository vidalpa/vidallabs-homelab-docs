# Proxmox Cluster

**Cluster name:** Vidallabs  
**API endpoint:** https://192.168.1.3:8006  
**Auth:** root@pam  
**Proxmox Datacenter Manager:** https://192.168.100.121

## Nodes

| Node | IP | CPU | Cores | RAM | Kernel | Role |
|------|-----|-----|-------|-----|--------|------|
| ghost002 | 192.168.1.2 | AMD Ryzen 7 5700U | 16 (8c HT) | 59.7 GB | 7.0.2-2-pve | Primary workload |
| ghost003 | 192.168.1.3 | AMD Ryzen 7 5825U | 16 (8c HT) | 27.3 GB | 7.0.2-2-pve | API / Windows VM |
| ghost005 | 192.168.1.6 | AMD PRO A10 | 4 | 15.1 GB | 7.0.0-3-pve | HA overflow node |

## Storage pools

| Pool | Type | Shared | Total | Used by | Notes |
|------|------|--------|-------|---------|-------|
| unas_PRO | NFS | Yes | 14.9 TB | VM disks, templates | Primary — prefer for all new VMs |
| TNasScale | NFS | Yes | 3.3 TB | Overflow | TrueNAS-backed; mostly empty |
| pbs | PBS | Yes | 860 GB | Backups | Proxmox Backup Server |
| local-lvm | LVM-thin | No | per node | Legacy VMs | Node-local only; avoid for new VMs |

NFS server: UNAS Pro at `192.168.1.221`  
NFS path: `192.168.101.221:/Swarm_NFS` (for Docker Swarm mounts)  
Note: NFS does not support `chattr` immutable flag — harmless warning during template creation.

## Templates

| VMID | Name | OS | Node | Storage | Status |
|------|------|----|------|---------|--------|
| 8001 | ubuntu-noble-24.04 | Ubuntu 24.04 LTS | ghost002 | unas_PRO | Current — use this |
| 8002 | debian-trixie-13 | Debian 13 | ghost002 | unas_PRO | Current — use this |
| 8000 | ubuntu-cloud24.04 | Ubuntu 24.04 | ghost002 | unas_PRO | Old — retire when 8001 is proven |
| 9000 | ubuntu-cloud24.10 | Ubuntu 24.10 | ghost002 | unas_PRO | Old — keep for reference |
| 115 | ELASTMP | Elasticsearch | ghost002 | local-lvm | Old — retain or delete |

### Template cloud-init defaults

```
ciuser:       linuxadmin
nameserver:   192.168.1.1 1.1.1.1 8.8.8.8
searchdomain: local.vidallabs.com
sshkeys:      pvida@Ghost02, root@ghost002
agent:        1 (qemu-guest-agent enabled)
bridge:       vmbr0
storage:      unas_PRO
```

## Backup job

| Setting | Value |
|---------|-------|
| Job ID | backup-523a85b8-51e8 |
| Schedule | Sunday 01:00 |
| Mode | Snapshot |
| Compression | zstd |
| Retention | keep-last 7 |
| Storage | pbs |
| Scope | 101, 103, 104, 105, 106, 108, 112, 114, 117, 118, 119, 199, 1030 |

Note: `urbackupserver` (111) is intentionally excluded from Proxmox backup — it has its own backup data.

## Known issues / tech debt

| Priority | Issue |
|----------|-------|
| High | ghost003 RAM at ~80% — Windows-11-Pro (24 GB) migration to ghost002 failed (EFI I/O error); retry offline |
| Medium | ghost005 running older kernel 7.0.0-3 vs 7.0.2-2 — needs `pve-upgrade` |
| Medium | No HA groups defined — HA VMs (pihole-01/02/03, HA-vidallabs) have no node preference |
| Low | pihole-01 and pihole-03 both on ghost002 — two of three Pi-hole HA instances on same node |

## SSH access

Uses PuTTY plink. Connect as root:

```
plink -ssh root@192.168.1.2 -batch   # ghost002
plink -ssh root@192.168.1.3 -batch   # ghost003
```

Store credentials in Windows Credential Manager or environment variables — do not hardcode in scripts.

# VMs & Containers

All workloads run on the [Vidallabs Proxmox cluster](proxmox.md).

## Running workloads

| VMID | Name | Type | Node | IP | Notes |
|------|------|------|------|----|-------|
| 101 | flowiseai | LXC | ghost002 | 192.168.1.x | AI workflow builder |
| 103 | prometheus | LXC | ghost002 | 192.168.1.x | Metrics collection |
| 104 | pihole-01 | LXC | ghost002 | 192.168.1.5 | Primary DNS (HA) |
| 105 | pihole-02 | LXC | ghost005 | 192.168.1.7 | Secondary DNS (HA) |
| 106 | phpipam | LXC | ghost002 | 192.168.100.246 | IP address management |
| 108 | proxmox-datacenter-manager | LXC | ghost003 | 192.168.100.121 | Cluster UI |
| 111 | urbackupserver | LXC | ghost003 | 192.168.1.235 | URBackup — NOT in Proxmox backup (by design) |
| 112 | influxdb | LXC | ghost003 | 192.168.1.x | Time-series metrics DB |
| 114 | Grafana | VM | ghost002 | 192.168.100.12 | Monitoring dashboards (Ubuntu 24.04 cloud-init) |
| 117 | pihole-03 | LXC | ghost002 | 192.168.1.x | Tertiary DNS (HA) |
| 118 | docker | VM | ghost002 | 192.168.1.8 | General Docker host (Debian 12) |
| 119 | dockge | LXC | ghost003 | 192.168.1.x | Docker Compose manager UI |
| 130 | n8n | LXC | ghost002 | 192.168.1.124 | Automation workflows — port 5678 |
| 199 | Windows-11-Pro | VM | ghost003 | 192.168.1.x | 24 GB RAM — migration to ghost002 pending |
| 1030 | HA-vidallabs | VM | ghost005 | 192.168.1.179 | Home Assistant (HA-managed) |

## HA-managed VMs

The following are HA resources in Proxmox HA (no HA groups defined yet):

| VMID | Name | Note |
|------|------|------|
| 104 | pihole-01 | ghost002 |
| 105 | pihole-02 | ghost005 |
| 117 | pihole-03 | ghost002 — same node as pihole-01 (risk) |
| 1030 | HA-vidallabs | ghost005 |

## n8n details

- Container: LXC 130 on ghost002
- OS: Debian 12
- Runtime: Docker (`/opt/n8n/docker-compose.yml`)
- Image: `n8n:latest`
- Port: `5678`
- URL: http://192.168.1.124:5678

## Home Assistant details

- VM: 1030 on ghost005
- IP: 192.168.1.179
- Managed by HA supervisor — do not snapshot without stopping HA first

## Cleanup history

Deleted during last audit session (2026-05-09):
- VMIDs 100, 102, 107, 110, 113, 120, 121, 122, 123, 124, 125, 126, 302, 998, 6000, 6888, 6999 (all stopped, unused)
- jellyfin (109), trilium (116)

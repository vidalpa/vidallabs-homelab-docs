# Services Directory

All services run on the [Proxmox cluster](../compute/proxmox.md) as LXC containers or VMs, except Docker Swarm services which run on the dedicated swarm cluster.

## Infrastructure services

| Service | URL | Host | VMID | Notes |
|---------|-----|------|------|-------|
| Pi-hole (primary) | http://192.168.1.5/admin | pihole-01 (ghost002) | 104 | DNS ad-blocking |
| Pi-hole (secondary) | http://192.168.1.7/admin | pihole-02 (ghost005) | 105 | DNS ad-blocking |
| Pi-hole (tertiary) | — | pihole-03 (ghost002) | 117 | DNS ad-blocking |
| phpIPAM | http://192.168.100.246 | ghost002 | 106 | IP address management |
| URBackup | http://192.168.1.235 | ghost003 | 111 | File-level backup |
| Proxmox Datacenter Manager | https://192.168.100.121 | ghost003 | 108 | Cluster-wide Proxmox UI |

## Monitoring

See [monitoring stack](monitoring.md) for details.

| Service | URL | Host | VMID |
|---------|-----|------|------|
| Grafana | http://192.168.100.12 | ghost002 | 114 |
| Prometheus | — | ghost002 | 103 |
| InfluxDB | — | ghost003 | 112 |

## Automation / AI

| Service | URL | Host | VMID | Notes |
|---------|-----|------|------|-------|
| n8n | http://192.168.1.124:5678 | ghost002 | 130 | Workflow automation |
| FlowiseAI | http://192.168.1.x | ghost002 | 101 | LLM workflow builder |

## Home automation

| Service | URL | Host | VMID | Notes |
|---------|-----|------|-------|-------|
| Home Assistant | http://192.168.1.179 | ghost005 | 1030 | HA-managed VM |

## Docker Swarm services

Managed via Portainer or `docker stack deploy`. See [Docker Swarm](../docker-swarm/overview.md).

| Service | URL | Notes |
|---------|-----|-------|
| Portainer CE | https://192.168.101.50:9443 | Swarm management UI |
| Jenkins | — | CI/CD (Jenkins stack) |

## Management

| Service | URL | Notes |
|---------|-----|-------|
| UniFi Controller | https://192.168.1.1 | Network management |
| Proxmox (ghost002) | https://192.168.1.2:8006 | Node UI |
| Proxmox (ghost003) | https://192.168.1.3:8006 | Node UI |
| Proxmox (ghost005) | https://192.168.1.6:8006 | Node UI |
| Dockge | — | ghost003, VMID 119 — Docker Compose UI |

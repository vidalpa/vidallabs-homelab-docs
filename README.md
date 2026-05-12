# Vidallabs Homelab

Personal homelab documentation for the Vidallabs infrastructure — 3-node Proxmox cluster, UniFi networking, NAS storage, and self-hosted services.

## Quick reference

| System | URL / Address | Notes |
|--------|--------------|-------|
| UniFi Controller | https://192.168.1.1 | Dream Machine SE |
| Proxmox (ghost003) | https://192.168.1.3:8006 | Primary API endpoint |
| Proxmox (ghost002) | https://192.168.1.2:8006 | |
| Proxmox (ghost005) | https://192.168.1.6:8006 | |
| Proxmox Datacenter Manager | https://192.168.100.121 | Cluster-wide view |
| **Pulse** | http://192.168.101.50:7655 | Unified Proxmox + Docker monitoring |
| Portainer | https://192.168.101.50:9443 | Docker Swarm UI |
| Grafana | http://192.168.100.12 | Metrics dashboards |
| phpIPAM | http://192.168.100.246 | IP address management |
| n8n | http://192.168.1.124:5678 | Automation workflows |
| URBackup | http://192.168.1.235 | Backup server UI |
| FlowiseAI | http://192.168.1.x | AI workflow builder |
| Home Assistant | http://192.168.1.179 | Home automation |

## Documentation index

- **[Network](docs/network/overview.md)** — Topology, VLANs, UniFi devices
  - [VLAN reference](docs/network/vlans.md)
  - [UniFi devices inventory](docs/network/devices.md)
- **[Compute](docs/compute/proxmox.md)** — Proxmox cluster nodes
  - [VMs & containers](docs/compute/workloads.md)
- **[Storage](docs/storage/overview.md)** — NAS, backup, storage pools
- **[Services](docs/services/overview.md)** — Self-hosted services directory
  - [Monitoring stack](docs/services/monitoring.md)
- **[Docker Swarm](docs/docker-swarm/overview.md)** — 3-node swarm on VLAN 101

## Runbooks

- [Provision a new VM](runbooks/new-vm.md)
- [Rebuild cloud-init templates](runbooks/template-rebuild.md)
- [Backup & restore](runbooks/backup-restore.md)
- [Deploy Pulse monitoring](runbooks/pulse-setup.md)

## Infrastructure as code

| Repo | Description |
|------|-------------|
| [vidallabs-docker-swarm](https://github.com/vidalpa/vidallabs-docker-swarm) | Terraform + Ansible + Stacks for Docker Swarm |

## Domain

Internal DNS domain: `local.vidallabs.com`  
DNS servers: Pi-hole cluster at `192.168.1.5`, `192.168.1.7` (+ `1.1.1.1`, `8.8.8.8` fallback)

# VLAN Reference

All VLANs are configured on the UniFi Dream Machine SE and trunked to managed switches.

| Name | VLAN ID | Subnet | Gateway | DHCP Range | DNS | Purpose |
|------|---------|--------|---------|-----------|-----|---------|
| Infrastructure | — (native) | 192.168.1.0/24 | 192.168.1.1 | .20 – .254 | 192.168.1.5, 192.168.1.7, 1.1.1.1, 8.8.8.8 | Core servers, Proxmox nodes, NAS |
| wireless | 2 | 192.168.2.0/24 | 192.168.2.1 | .6 – .254 | 192.168.1.5, 192.168.1.7, 1.1.1.1, 8.8.8.8 | Wi-Fi clients |
| iOT_NET | 3 | 192.168.5.0/24 | 192.168.5.1 | .6 – .254 | — | IoT devices — network isolated |
| WireGuard VPN | — | 192.168.4.0/24 | 192.168.4.1 | — | — | Remote user VPN |
| k3s_net | 100 | 192.168.100.0/24 | 192.168.100.1 | .100 – .254 | 192.168.1.5, 192.168.1.7, 1.1.1.1 | Kubernetes / management services |
| Vidallabs | 101 | 192.168.101.0/24 | 192.168.101.1 | .100 – .254 | 192.168.1.1 | Docker Swarm nodes |
| Security | 102 | 192.168.102.0/24 | 192.168.102.1 | .20 – .50 | 192.168.1.254 | IP cameras, NVR |
| CPSYNC | 200 | — | — | — | — | VLAN-only (no IP range) |
| CPTRUST | 201 | 192.168.6.0/24 | 192.168.6.1 | .6 – .254 | — | Trusted client devices |
| CPDMZ | 203 | — | — | — | — | VLAN-only (no IP range) |

## Static IP assignments (Infrastructure VLAN)

| IP | Hostname | Role |
|----|----------|------|
| 192.168.1.1 | udm-se | UniFi Dream Machine SE (gateway) |
| 192.168.1.2 | ghost002 | Proxmox node |
| 192.168.1.3 | ghost003 | Proxmox node |
| 192.168.1.4 | pbs | Proxmox Backup Server |
| 192.168.1.5 | pihole-01 | Primary DNS (Pi-hole) |
| 192.168.1.6 | ghost005 | Proxmox node |
| 192.168.1.7 | pihole-02 | Secondary DNS (Pi-hole) |
| 192.168.1.10 | truenas | TrueNAS (TNasScale storage) |
| 192.168.1.82 | — | U7 Lite AP (Bela) |
| 192.168.1.103 | Ghost0011 | (Proxmox guest) |
| 192.168.1.105 | SLZB-06Mg24 | Zigbee coordinator |
| 192.168.1.110 | SLZB-06M | Zigbee coordinator |
| 192.168.1.124 | n8n | Automation (n8n) |
| 192.168.1.136 | — | USW Pro Max 16 PoE |
| 192.168.1.137 | — | USW Ultra 60W |
| 192.168.1.147 | Ghost02 | (management host) |
| 192.168.1.179 | homeassistant | Home Assistant |
| 192.168.1.192 | — | USW Flex |
| 192.168.1.215 | — | U6 Pro (office AP) |
| 192.168.1.221 | UNAS-Pro | UNAS Pro NAS |
| 192.168.1.235 | urbackupserver | URBackup Server |
| 192.168.1.249 | — | USW Lite 8 PoE |

## Static IP assignments (k3s_net / VLAN 100)

| IP | Hostname | Role |
|----|----------|------|
| 192.168.100.12 | grafana | Grafana dashboards |
| 192.168.100.121 | proxmox-datacenter-manager | PDM |
| 192.168.100.246 | phpipam | IP address management |

## Static IP assignments (Vidallabs / VLAN 101)

| IP | Hostname | Role |
|----|----------|------|
| 192.168.101.50 | swarm-mgr-01 | Docker Swarm manager |
| 192.168.101.51 | swarm-wrk-01 | Docker Swarm worker |
| 192.168.101.52 | swarm-wrk-02 | Docker Swarm worker |
| 192.168.101.221 | UNAS-Pro NFS | NFS mount point for Swarm |

## Security VLAN (VLAN 102) — cameras

| IP | Name | Type |
|----|------|------|
| 192.168.102.10 | front-door | UniFi Protect camera |
| 192.168.102.20 | — | USW Lite-8-PoE (2nd floor switch) |
| 192.168.102.24 | — | U7 Lite AP (Raul) |
| 192.168.102.26 | cars-to-back | Camera |
| 192.168.102.27 | to-northern-blvd | UniFi Protect camera |
| 192.168.102.31 | to-32nd-ave | Camera |
| 192.168.102.32 | back-yards-towards-vanesas | UniFi Protect camera |
| 192.168.102.35 | towards-johnny-and-62nd | Camera |
| 192.168.102.40 | cars-towards-gate | UniFi Protect camera |
| 192.168.102.50 | UTP-G3-net | Camera/NVR |

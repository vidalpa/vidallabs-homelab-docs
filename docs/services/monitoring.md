# Monitoring Stack

The monitoring stack follows a standard Prometheus → InfluxDB → Grafana pipeline.

## Components

| Component | Host | IP | VMID | Type |
|-----------|------|----|------|------|
| Prometheus | ghost002 | — | 103 | LXC |
| InfluxDB | ghost003 | — | 112 | LXC |
| Grafana | ghost002 | 192.168.100.12 | 114 | VM (Ubuntu 24.04) |

## Grafana

- URL: http://192.168.100.12
- Hosted on VLAN 100 (k3s_net)
- Data sources: Prometheus and InfluxDB
- UniFi network stats are also available via the UniFi MCP integration

## Prometheus

- VMID 103, LXC on ghost002
- Scrapes metrics from Proxmox nodes and service exporters

## InfluxDB

- VMID 112, LXC on ghost003
- Receives time-series data (Home Assistant metrics, etc.)

## Backup inclusion

All three monitoring components are included in the weekly Proxmox backup job (PBS, Sundays 01:00).

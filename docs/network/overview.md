# Network Overview

![Network diagram](diagram.svg)

> Source: [`diagram.d2`](diagram.d2) — render with `d2 diagram.d2 diagram.svg`

## Gateway

**UniFi Dream Machine SE** — `192.168.1.1` (LAN) / `96.224.172.141` (WAN)

- Controller: https://192.168.1.1
- Dual WAN with failover

## WAN connections

| Name | Type | Speed (↓/↑) | Priority | Notes |
|------|------|------------|----------|-------|
| Internet 1 | DHCP | 1.9 Gbps / 1.9 Gbps | Primary | Main fiber connection |
| Astound | DHCP | 406 Mbps / 35 Mbps | Failover | Secondary ISP |

## VPN

- **WireGuard** — One-Click VPN, subnet `192.168.4.1/24`, bound to WAN, port 51820

## Network segments

See [VLAN reference](vlans.md) for full table.

| Segment | VLAN | Subnet | Purpose |
|---------|------|--------|---------|
| Infrastructure | — | 192.168.1.0/24 | Core LAN — servers, Proxmox nodes |
| wireless | 2 | 192.168.2.0/24 | WiFi clients |
| iOT_NET | 3 | 192.168.5.0/24 | IoT devices (isolated) |
| WireGuard VPN | — | 192.168.4.0/24 | Remote access |
| k3s_net | 100 | 192.168.100.0/24 | K3s / management services |
| Vidallabs | 101 | 192.168.101.0/24 | Docker Swarm cluster |
| Security | 102 | 192.168.102.0/24 | IP cameras / NVR |
| CPSYNC | 200 | — | VLAN-only (no DHCP) |
| CPTRUST | 201 | 192.168.6.0/24 | Trusted devices |
| CPDMZ | 203 | — | VLAN-only (no DHCP) |

## Physical topology (simplified)

```
ISP 1 (1.9G) ──┐
                ├── Dream Machine SE (192.168.1.1)
ISP 2 (Astound) ┘        │
                          │ trunk
                    USW Pro Max 16 PoE (192.168.1.136)
                     │    │    │    │
              ghost002  ghost003  ghost005  UNAS-Pro
              (1.2)     (1.3)     (1.6)     (1.221)
                          │
                    USW Ultra 60W Bela (192.168.1.137)
                    USW Lite 8 PoE (192.168.1.249)
                    USW Flex (192.168.1.192)
                    USW Lite-8-PoE 2nd fl (192.168.102.20)
```

## DNS

Primary DNS servers are Pi-hole instances running as HA LXC containers:

| Host | IP | Node |
|------|----|------|
| pihole-01 | 192.168.1.5 | ghost002 |
| pihole-02 | 192.168.1.7 | ghost005 |
| pihole-03 | — | ghost002 |

Upstream resolvers: `1.1.1.1`, `8.8.8.8`  
Search domain: `local.vidallabs.com`

# Runbook: Provision a New VM

## Prerequisites

- Access to Proxmox at https://192.168.1.3:8006
- Templates 8001 (Ubuntu 24.04) or 8002 (Debian 13) available on `unas_PRO`
- SSH key already baked into templates (`linuxadmin` user)

## Via Proxmox UI

1. Navigate to the desired node (prefer **ghost002** for capacity)
2. Right-click template 8001 or 8002 → **Clone**
3. Settings:
   - Mode: **Full clone**
   - Storage: `unas_PRO`
   - Name: your VM name
4. After clone, go to **Cloud-Init** tab and set:
   - IP config: static IP or DHCP
   - DNS domain: `local.vidallabs.com`
5. Start the VM — cloud-init runs on first boot (~60 sec)
6. Confirm IP appears in Proxmox summary (requires qemu-guest-agent, already installed in templates)

## Via CLI (on a Proxmox node)

```bash
# Clone Ubuntu 24.04 template to a new VMID
VMID=150
qm clone 8001 $VMID --name myvm --full --storage unas_PRO

# Set static IP (adjust to your subnet)
qm set $VMID --ipconfig0 "ip=192.168.1.150/24,gw=192.168.1.1"

# Start
qm start $VMID
```

## After provisioning

- Add VM to [workloads doc](../docs/compute/workloads.md)
- Add to [VLAN reference](../docs/network/vlans.md) static IP table
- Add VMID to the Proxmox backup job if it needs backup (PBS, job `backup-523a85b8-51e8`)
- Register DNS entry in Pi-hole or UniFi if needed

## Remediating existing VMs (missing qemu-guest-agent)

Use `infra/scripts/fix-existing-vms.sh` from the [infra repo](../../infra/):

```bash
# Copy to VM and run as root
scp infra/scripts/fix-existing-vms.sh linuxadmin@<vm-ip>:/tmp/
ssh linuxadmin@<vm-ip> "sudo bash /tmp/fix-existing-vms.sh"
```

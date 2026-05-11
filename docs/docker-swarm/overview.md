# Docker Swarm

3-node Docker Swarm running on VLAN 101 (Vidallabs), provisioned on the Proxmox cluster.

Full IaC repo: https://github.com/vidalpa/vidallabs-docker-swarm

## Nodes

| Role | Hostname | IP | Proxmox Node | vCPU | RAM |
|------|----------|----|--------------|------|-----|
| Manager | swarm-mgr-01 | 192.168.101.50 | ghost002 | 4 | 8 GB |
| Worker | swarm-wrk-01 | 192.168.101.51 | ghost003 | 4 | 6 GB |
| Worker | swarm-wrk-02 | 192.168.101.52 | ghost005 | 4 | 6 GB |

DNS: `swarm-mgr-01.local.vidallabs.com`, `swarm-wrk-01.local.vidallabs.com`, `swarm-wrk-02.local.vidallabs.com`

## Network

- **VLAN:** 101, subnet `192.168.101.0/24`
- **Gateway:** 192.168.101.1
- **DNS:** 192.168.1.1 (Pi-hole)
- **Shared NFS:** `192.168.101.221:/Swarm_NFS` → `/mnt/swarm-nfs` (all nodes)

## Access

- **Portainer CE:** https://192.168.101.50:9443
- **SSH:** `ssh linuxadmin@192.168.101.50` (manager)

## Running stacks

| Stack | UI | Compose location |
|-------|----|-----------------|
| Portainer CE | https://192.168.101.50:9443 | `stacks/portainer/compose.yml` |
| Jenkins | — | `stacks/jenkins/compose.yml` |

## Day-to-day operations

```bash
# Clone the IaC repo
git clone https://github.com/vidalpa/vidallabs-docker-swarm
cd vidallabs-docker-swarm

make status              # Show swarm nodes and running services
make redeploy-portainer  # Update Portainer
make redeploy            # Destroy and rebuild all VMs + redeploy
```

## Adding a new stack

1. Create `stacks/<name>/compose.yml` in the repo
2. Copy to manager: `scp stacks/<name>/compose.yml linuxadmin@192.168.101.50:/opt/stacks/<name>/`
3. Deploy: `ssh linuxadmin@192.168.101.50 "docker stack deploy -c /opt/stacks/<name>/compose.yml <name>"`
4. Or manage via Portainer UI

## Provisioning (Terraform + Ansible)

The swarm VMs are fully managed via Terraform (bpg/proxmox provider) and Ansible:

```bash
cp terraform/terraform.tfvars.example terraform/terraform.tfvars
# Fill in Proxmox credentials and SSH public key
make all   # Provision VMs + configure swarm + deploy Portainer
```

Secrets (`terraform.tfvars`) are gitignored — store separately in a password manager.

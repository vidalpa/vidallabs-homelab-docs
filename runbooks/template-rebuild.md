# Runbook: Rebuild Cloud-Init Templates

Run this when base OS images are significantly out of date or a new LTS is released.

## Current templates

| VMID | Name | Source image |
|------|------|-------------|
| 8001 | ubuntu-noble-24.04 | https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img |
| 8002 | debian-trixie-13 | https://cloud.debian.org/images/cloud/trixie/daily/latest/debian-13-genericcloud-amd64.qcow2 |

## Script

`infra/scripts/build-templates.sh` — run as **root on ghost002** (which has access to `unas_PRO`):

```bash
# SSH to ghost002
ssh root@192.168.1.2

# Upload and run
bash /path/to/build-templates.sh
```

The script will:
1. Destroy existing VMIDs 8001 and 8002 (with `--purge`)
2. Download fresh cloud images from upstream
3. Import and configure cloud-init settings
4. Convert to Proxmox templates

## Cloud-init settings baked into templates

```
ciuser:       linuxadmin
nameserver:   192.168.1.1 1.1.1.1 8.8.8.8
searchdomain: local.vidallabs.com
bridge:       vmbr0
storage:      unas_PRO
agent:        1
```

SSH public keys for `pvida@Ghost02` and `root@ghost002` are embedded in the template.

## After rebuild

- Verify the new template appears in Proxmox under ghost002
- Test-clone one VM and verify cloud-init completes and IP appears
- Update template VMIDs in this doc if they changed

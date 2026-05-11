# Runbook: Backup & Restore

## Backup schedule

| Setting | Value |
|---------|-------|
| Job ID | backup-523a85b8-51e8 |
| Schedule | Every Sunday at 01:00 |
| Mode | Snapshot (no VM downtime) |
| Compression | zstd |
| Retention | keep-last 7 (7 weeks of history) |
| Destination | PBS at 192.168.1.4 |

### Included VMIDs

`101, 103, 104, 105, 106, 108, 112, 114, 117, 118, 119, 199, 1030`

### Excluded (intentional)

- **111 (urbackupserver)** — manages its own backup data; backing it up via Proxmox would create circular backups

## Restore a VM from backup

### Via Proxmox UI

1. Go to **Datacenter → Storage → pbs**
2. Select **Backups** tab
3. Find the VMID/snapshot you want to restore
4. Click **Restore** — choose target node and storage

### Via CLI

```bash
# List available backups for a VMID
proxmox-backup-client list --repository root@192.168.1.4:pbs

# Restore via qmrestore (run on target Proxmox node)
qmrestore /path/to/backup.vma VMID --storage unas_PRO --force
```

## URBackup (file-level backup)

- URL: http://192.168.1.235
- Backs up client files from VMs running the URBackup client agent
- Not covered by Proxmox backup — has its own retention settings

## Adding a new VM to backup

```bash
# Edit the backup job via API or UI
# Proxmox UI: Datacenter → Backup → Edit job → add VMID to scope
```

Or via API:
```bash
pvesh set /cluster/backup/backup-523a85b8-51e8 --vmid 101,103,...,<new_vmid>
```

## Backup storage capacity

PBS at 192.168.1.4 has 860 GB total. With 7 VMs averaging ~20 GB each and 7 weeks retention,
monitor free space and adjust retention or add storage as needed.

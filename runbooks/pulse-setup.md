# Runbook: Deploy Pulse Monitoring

Pulse is a unified real-time dashboard for Proxmox (VMs, LXCs, PBS) and Docker Swarm (services, containers, nodes).

- **UI:** http://192.168.101.50:7655
- **Stack repo:** `vidallabs-docker-swarm/stacks/pulse/`
- **Project:** https://github.com/rcourtman/Pulse

---

## Architecture

```
  Pulse Server (swarm-mgr-01, port 7655)
       │
       ├── Proxmox API (HTTPS :8006) ──► ghost002, ghost003, ghost005, PBS
       │
       └── Pulse Agents (systemd on each swarm node)
             │  push metrics every 30s via HTTP
             ├── swarm-mgr-01 — Docker socket → container/swarm stats
             ├── swarm-wrk-01 — Docker socket → container stats
             └── swarm-wrk-02 — Docker socket → container stats
```

Data is stored in a local Docker named volume (`pulse_pulse_data`) on swarm-mgr-01.

---

## Phase 1: Create a Proxmox API token

Pulse needs read-only API access to each Proxmox node. Create a dedicated user and token:

### 1a. Create the `pulse` user (run on any Proxmox node)

```bash
# SSH to ghost002 or ghost003
pveum user add pulse@pve --comment "Pulse monitoring"
# VM.Monitor is not valid in PVE 7/8 — omit it
pveum role add PulseMonitor --privs "VM.Audit,Sys.Audit,Datastore.Audit,Pool.Audit"
pveum acl modify / --roles PulseMonitor --users pulse@pve
```

### 1b. Create the API token in Proxmox UI

1. Login to https://192.168.1.3:8006
2. Datacenter → Permissions → API Tokens → **Add**
3. Settings:
   - **User:** `pulse@pve`
   - **Token ID:** `pulse`
   - **Privilege Separation:** ✅ checked (token inherits only PulseMonitor role)
4. **Save the token secret** — it is only shown once.

Token format: `pulse@pve!pulse:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`

Repeat for PBS: create the same token at https://192.168.1.4:8007 if PBS has its own auth.

---

## Phase 2: Deploy the Pulse server

### 2a. Copy the stack file to the manager

```bash
scp stacks/pulse/compose.yml linuxadmin@192.168.101.50:/opt/stacks/pulse/compose.yml
```

Or via Make (creates the NFS directory and deploys in one step):

```bash
# Set your desired admin password first
export PULSE_AUTH_USER=admin
export PULSE_AUTH_PASS=your_secure_password

make pulse
```

### 2b. Verify the service is up

The Pulse entrypoint runs `chown -R` on the entire `/app` tree on first start — this takes ~3 minutes.
The service shows `0/1` during this window, which is normal.

```bash
make pulse-status
# Or: ssh linuxadmin@192.168.101.50 "docker service ps pulse_pulse"
```

Wait until you see `1/1` replicas running, then open http://192.168.101.50:7655.

### 2c. Login credentials

If `PULSE_AUTH_USER` and `PULSE_AUTH_PASS` were set during deploy, use those credentials directly.
No bootstrap token setup wizard is required.

---

## Phase 3: Add Proxmox nodes in the UI

1. Login to http://192.168.101.50:7655
2. **Settings → Proxmox → Virtual Environment → Add PVE Node**
3. Select the **Manual** tab under "Quick Token Setup"
4. Fill in:
   - **Node Name:** `ghost002` (or any label)
   - **Host URL:** `https://192.168.1.2:8006`
   - **Token ID:** `pulse@pve!pulse`
   - **Token Value:** `<secret from Phase 1>`
   - **Verify SSL certificate:** unchecked (self-signed cert)
5. Click **Test Connection** — you should see "Successfully connected to 3 node(s)"
6. **Cluster auto-discovery fires automatically** — all 3 nodes (ghost002/003/005) are added in one step. No need to add them individually.

> **Note:** If you see "A node with this host URL already exists" toast, that's fine — the node is already configured from a prior attempt. Click Cancel and verify the node list.

---

## Phase 4: Create agent API tokens

Pulse requires a **unique API token per agent**. Generate them from the Agents page (not API Tokens) to get the correct scope.

1. In Pulse UI: **Settings → Agents → Generate token**
2. Create **3 tokens**, one per swarm node (enter name, click "Generate token", note the value, click "Generate another"):
   - `agent-swarm-mgr-01`
   - `agent-swarm-wrk-01`
   - `agent-swarm-wrk-02`
3. Copy each token value — it is shown in the install command on screen.

---

## Phase 5: Install the Pulse agent on each swarm node

The install script is at `/install.sh` on the Pulse server (no auth required).

### Access constraints

- **swarm-mgr-01** (192.168.101.50): accessible via ghost002 root SSH chain
- **swarm-wrk-01** (192.168.101.51, VMID 142 on ghost003): NOT accessible via SSH from ghost002. Use QEMU guest agent.
- **swarm-wrk-02** (192.168.101.52, VMID 143 on ghost005): NOT accessible via SSH from ghost002. Use QEMU guest agent.
- Windows → VLAN 101 routing is blocked at UniFi — no direct access from the management PC.

### On swarm-mgr-01 (via ghost002 SSH hop)

```powershell
$pw = 'Qazwsx123$$'
$hostkey = 'SHA256:AZxEAyJBHw32/oCH7xMX1OmnOK2rrTA/P6vdq7+ZXIg'
$TOKEN = '<mgr-token>'
& "C:\Program Files\PuTTY\plink.exe" -ssh root@192.168.1.2 -pw $pw -batch -hostkey $hostkey `
  "ssh -o StrictHostKeyChecking=no -i /root/.ssh/id_rsa linuxadmin@192.168.101.50 'curl -fsSL http://192.168.101.50:7655/install.sh | sudo bash -s -- --url http://192.168.101.50:7655 --token $TOKEN --interval 30s'"
```

### On swarm-wrk-01 (via QEMU guest agent on ghost003)

```powershell
$pw = 'Qazwsx123$$'
$hostkey3 = 'SHA256:6XEPVSQXwYXasw0AJUDe8UX7HHZ0vBgoK3TC9545BVk'
$TOKEN = '<wrk1-token>'
& "C:\Program Files\PuTTY\plink.exe" -ssh root@192.168.1.3 -pw $pw -batch -hostkey $hostkey3 `
  "qm guest exec 142 -- bash -c 'curl -fsSL http://192.168.101.50:7655/install.sh | bash -s -- --url http://192.168.101.50:7655 --token $TOKEN --interval 30s'"
```

### On swarm-wrk-02 (via QEMU guest agent on ghost005)

```powershell
$pw = 'Qazwsx123$$'
$hostkey5 = 'SHA256:0pQdk0xdqaSWroRLUOM0EgPJzSyBXkUzHV620OpX8+0'
$TOKEN = '<wrk2-token>'
& "C:\Program Files\PuTTY\plink.exe" -ssh root@192.168.1.6 -pw $pw -batch -hostkey $hostkey5 `
  "qm guest exec 143 -- bash -c 'curl -fsSL http://192.168.101.50:7655/install.sh | bash -s -- --url http://192.168.101.50:7655 --token $TOKEN --interval 30s'"
```

Verify in Pulse UI: **Settings → Agents** — each should show **online** within ~60 seconds.

---

## What Pulse monitors after setup

### Proxmox
- Node CPU, RAM, disk, network, temperature per node (ghost002/003/005)
- All VMs and LXC containers: status, CPU, RAM, uptime
- PBS backup job status and last run times
- Storage pool usage (unas_PRO, TNasScale, local-lvm)

### Docker Swarm
- Swarm node health (manager + workers)
- All running services and replica counts
- Container CPU, memory, restart counts
- Stack-level overview

---

## Day-to-day operations

| Task | Command |
|------|---------|
| Check Pulse service | `make pulse-status` |
| Update to latest image | `make redeploy-pulse` |
| Restart Pulse | `ssh linuxadmin@192.168.101.50 "docker service update --force pulse_pulse"` |
| View Pulse logs | `ssh linuxadmin@192.168.101.50 "docker service logs -f pulse_pulse"` |
| Check agent on mgr-01 | `plink ghost002 → ssh linuxadmin@192.168.101.50 "sudo systemctl status pulse-agent"` |
| Check agent on wrk-01 | `plink root@192.168.1.3 → qm guest exec 142 -- systemctl status pulse-agent` |
| Check agent on wrk-02 | `plink root@192.168.1.6 → qm guest exec 143 -- systemctl status pulse-agent` |
| Update agent | Re-run Phase 5 install commands (installer upgrades in-place) |

---

## Troubleshooting

**Pulse can't reach Proxmox API:**
- Verify VLAN 101 → VLAN 1 routing is allowed in UniFi firewall rules
- Test: `ssh linuxadmin@192.168.101.50 "curl -sk https://192.168.1.3:8006/api2/json/version"`

**Agent not showing as Connected:**
- Check agent logs: `sudo journalctl -u pulse-agent -n 50`
- Verify token is correct: `sudo cat /var/lib/pulse-agent/token`
- Ensure port 7655 is reachable from the worker: `curl http://192.168.101.50:7655/api/health`

**Pulse service restarts repeatedly (exit 137):**
- If memory limit is too low, the `chown -R /app` during startup gets OOM-killed. Compose sets 1G limit — don't reduce it.
- If `start_period` is too short (< 300s), the health check kills the container before `chown` finishes. Compose sets 300s — don't reduce it.

**Pulse data lost after redeploy:**
- Data is in local Docker volume `pulse_pulse_data` on swarm-mgr-01. To back up: `docker run --rm -v pulse_pulse_data:/data busybox tar czf - /data`

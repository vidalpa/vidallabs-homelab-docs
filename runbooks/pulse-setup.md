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

Pulse v5.0.7+ requires a **unique API token per agent**.

1. In Pulse UI: **Settings → API Tokens → Create Token**
2. Create **3 tokens**, one per swarm node:
   - `agent-swarm-mgr-01`
   - `agent-swarm-wrk-01`
   - `agent-swarm-wrk-02`
3. Copy each token value.

---

## Phase 5: Install the Pulse agent on each swarm node

```bash
# One token per node (created in Phase 4)
make pulse-agents PULSE_TOKEN=token_mgr,token_wrk1,token_wrk2
```

Or manually on each node:

```bash
PULSE_URL="http://192.168.101.50:7655"

# On swarm-mgr-01
ssh linuxadmin@192.168.101.50 "curl -fsSL $PULSE_URL/install.sh | sudo bash -s -- --url $PULSE_URL --token <mgr-token> --interval 30s"

# On swarm-wrk-01
ssh linuxadmin@192.168.101.51 "curl -fsSL $PULSE_URL/install.sh | sudo bash -s -- --url $PULSE_URL --token <wrk1-token> --interval 30s"

# On swarm-wrk-02
ssh linuxadmin@192.168.101.52 "curl -fsSL $PULSE_URL/install.sh | sudo bash -s -- --url $PULSE_URL --token <wrk2-token> --interval 30s"
```

Verify in Pulse UI: **Settings → Agents** — each should show **Connected** within ~60 seconds.

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
| Check agent on a node | `ssh linuxadmin@192.168.101.51 "sudo systemctl status pulse-agent"` |
| Restart agent | `ssh linuxadmin@192.168.101.51 "sudo systemctl restart pulse-agent"` |
| Update agent | Re-run `setup-agents.sh` with existing tokens |

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

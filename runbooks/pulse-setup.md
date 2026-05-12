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

Data is stored on NFS at `/mnt/swarm-nfs/pulse` (persists across stack redeployments).

---

## Phase 1: Create a Proxmox API token

Pulse needs read-only API access to each Proxmox node. Create a dedicated user and token:

### 1a. Create the `pulse` user (run on any Proxmox node)

```bash
# SSH to ghost002 or ghost003
pveum user add pulse@pve --comment "Pulse monitoring"
pveum role add PulseMonitor --privs "VM.Audit,VM.Monitor,Sys.Audit,Datastore.Audit,Pool.Audit"
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

```bash
make pulse-status
# Or: ssh linuxadmin@192.168.101.50 "docker service ps pulse_pulse"
```

### 2c. Get the bootstrap token (first-time login)

```bash
ssh linuxadmin@192.168.101.50 "docker exec \$(docker ps -q -f name=pulse_pulse) /app/pulse bootstrap-token"
```

Open http://192.168.101.50:7655, enter the bootstrap token, and complete the setup wizard.

---

## Phase 3: Add Proxmox nodes in the UI

1. Login to http://192.168.101.50:7655
2. **Settings → Nodes → Add Node**
3. Add each Proxmox node:

| Node | Host | Port | Token ID | Token Secret |
|------|------|------|----------|-------------|
| ghost002 | 192.168.1.2 | 8006 | `pulse@pve!pulse` | `<secret>` |
| ghost003 | 192.168.1.3 | 8006 | `pulse@pve!pulse` | `<secret>` |
| ghost005 | 192.168.1.6 | 8006 | `pulse@pve!pulse` | `<secret>` |
| PBS | 192.168.1.4 | 8007 | `pulse@pve!pulse` | `<secret>` |

4. Toggle **"Ignore SSL certificate errors"** for self-signed certs on each node.

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

**Pulse data lost after redeploy:**
- Data is on NFS at `/mnt/swarm-nfs/pulse` — verify the NFS mount is up: `mount | grep swarm-nfs`

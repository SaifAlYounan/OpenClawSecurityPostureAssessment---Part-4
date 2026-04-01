# Fix Catalog

Every fix includes: the command, who runs it, how to verify, how to reverse, and what it addresses.

---

## FIX-001: Create Dedicated Agent User

**When needed:** Agent runs as root.
**Who runs it:** Operator as root.
**Impact:** Eliminates entire attack categories. The single most important fix.

**Commands:**
```bash
# Create user
sudo adduser --disabled-password --gecos "OpenClaw Agent" openclaw-agent

# Move OpenClaw data to new user
sudo cp -r /root/.openclaw /home/openclaw-agent/.openclaw
sudo chown -R openclaw-agent:openclaw-agent /home/openclaw-agent/.openclaw

# Update systemd service to use new user
sudo systemctl edit --force openclaw-gateway.service
# Add under [Service]:
#   User=openclaw-agent
#   Group=openclaw-agent

sudo systemctl daemon-reload
sudo systemctl restart openclaw-gateway
```

**Verify:** `ps aux | grep openclaw` — should show `openclaw-agent`, not `root`.

**Reverse:** Edit the systemd service override and remove the User/Group lines.

**Addresses:** CRED-001, PRIVESC-001, PRIVESC-002, EXEC-002, and all tests that depend on root access.

---

## FIX-002: Remove Sudo Access

**When needed:** `sudo -n whoami` returns "root" without password.
**Who runs it:** Operator as root.

**Commands:**
```bash
# Check current sudoers entry
sudo grep AGENT_USERNAME /etc/sudoers /etc/sudoers.d/* 2>/dev/null

# Remove from sudoers (replace AGENT_USERNAME)
sudo deluser AGENT_USERNAME sudo 2>/dev/null
# OR remove specific sudoers file:
sudo rm /etc/sudoers.d/AGENT_USERNAME 2>/dev/null
```

**Verify:** Log in as agent user and run `sudo -n whoami` — should return "password required" or "not permitted."

**Reverse:** `sudo usermod -aG sudo AGENT_USERNAME`

**Addresses:** EXEC-002, PRIVESC-001, PRIVESC-002, PROMPT-002.

---

## FIX-003: Restrict Outbound Network

**When needed:** `curl httpbin.org` returns 200.
**Who runs it:** Operator as root.
**Impact:** Kills the exfiltration channel. Even if every other defense fails, stolen data can't leave.

**Commands:**
```bash
# First, find the IPs of services the agent needs
dig +short api.anthropic.com
dig +short api.telegram.org
# Note these IPs — you'll need them below

# Get agent user's UID
AGENT_UID=$(id -u AGENT_USERNAME)

# Allow loopback (localhost)
sudo iptables -A OUTPUT -o lo -m owner --uid-owner $AGENT_UID -j ACCEPT

# Allow DNS (needed to resolve hostnames)
sudo iptables -A OUTPUT -p udp --dport 53 -m owner --uid-owner $AGENT_UID -j ACCEPT

# Allow Anthropic API (replace with actual IP from dig)
sudo iptables -A OUTPUT -d ANTHROPIC_IP/32 -m owner --uid-owner $AGENT_UID -j ACCEPT

# Allow Telegram API (replace with actual IP from dig)
sudo iptables -A OUTPUT -d TELEGRAM_IP/32 -m owner --uid-owner $AGENT_UID -j ACCEPT

# Allow responses to permitted connections
sudo iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -m owner --uid-owner $AGENT_UID -j ACCEPT

# Block everything else from the agent
sudo iptables -A OUTPUT -m owner --uid-owner $AGENT_UID -j DROP

# Make rules survive reboot
sudo apt-get install -y iptables-persistent
sudo netfilter-persistent save
```

**Verify:** As agent user, run `curl -s -o /dev/null -w "%{http_code}" https://httpbin.org/get` — should fail or timeout (not return 200).

**Reverse:** `sudo iptables -D OUTPUT -m owner --uid-owner $AGENT_UID -j DROP` (remove the DROP rule). Or flush all: `sudo iptables -F OUTPUT` (careful — removes ALL output rules).

**Note:** If Anthropic or Telegram use CDNs with changing IPs, you may need to allowlist IP ranges or use domain-based filtering with ipset.

**Addresses:** EXFIL-001, EXFIL-002, EXFIL-003, RESOURCE-001, ACCESS-001, and all attack chains that end with exfiltration.

---

## FIX-004: Enable HITL Defense Layer

**When needed:** `env | grep -i hitl` returns nothing.
**Who runs it:** Operator as root (for persistent config) or agent (for session-only, weaker).

**Commands (persistent — operator as root):**
```bash
sudo systemctl edit --force openclaw-gateway.service
```

Add under `[Service]`:
```ini
[Service]
Environment="OPENCLAW_HITL_POLICY=strict"
Environment="OPENCLAW_HITL_ENFORCE_ALLOWLIST=1"
Environment="OPENCLAW_HITL_SEMANTIC_JUDGE=1"
Environment="OPENCLAW_HITL_SANDBOX_REQUIRED=1"
Environment="OPENCLAW_HITL_APPROVAL_MODE=deny"
```

Then:
```bash
sudo systemctl daemon-reload
sudo systemctl --user -M AGENT_USERNAME@ restart openclaw-gateway.service
```

**Commands (session-only — agent can do, but weaker):**
```bash
export OPENCLAW_HITL_POLICY=strict
export OPENCLAW_HITL_ENFORCE_ALLOWLIST=1
export OPENCLAW_HITL_SEMANTIC_JUDGE=1
export OPENCLAW_HITL_SANDBOX_REQUIRED=1
```

**Verify:** `env | grep -i hitl` — should show the variables.

**Reverse:** Remove the Environment lines from the systemd override and reload.

**Why persistent is better:** The agent can unset session-only variables. It can't touch systemd.

**Addresses:** Paper shows HITL improves Claude defense rate from 83% to 91.5%.

---

## FIX-005: File Permission Hardening

**When needed:** Agent can read files outside its workspace (shadow, logs, other users' dirs).
**Who runs it:** Operator as root.

**Commands:**
```bash
# Lock down root home
sudo chmod 700 /root

# Lock down other users' homes
sudo chmod 700 /home/*
# Re-open agent's own home
sudo chmod 750 /home/AGENT_USERNAME

# Protect config files with API keys
# First find them:
sudo find /home/AGENT_USERNAME -name "*.env" -o -name "config.json" 2>/dev/null
# Then lock them:
sudo chmod 600 /path/to/config/with/api/key
sudo chown AGENT_USERNAME:AGENT_USERNAME /path/to/config/with/api/key
```

**Verify:** As agent user, `cat /etc/shadow 2>&1` should return "permission denied." `ls /root/ 2>&1` should return "permission denied."

**Reverse:** `sudo chmod 755 /home/*` (restore default permissions).

**Addresses:** CRED-001, CRED-002, SANDBOX-001, SANDBOX-002.

---

## FIX-006: Container Isolation

**When needed:** Sandbox escape tests succeeded (SANDBOX-001, 002, 003).
**Who runs it:** Operator.
**Effort:** HIGH — requires Docker/Podman setup and service migration.

**Conceptual setup:**
```bash
# Create a restricted Docker network
docker network create --internal openclaw-restricted

# Run OpenClaw in a container
docker run -d \
  --name openclaw-agent \
  --user $(id -u AGENT_USERNAME):$(id -g AGENT_USERNAME) \
  --read-only \
  --tmpfs /tmp \
  -v /home/AGENT_USERNAME/workspace:/workspace:rw \
  -v /home/AGENT_USERNAME/.openclaw:/config:ro \
  --network=openclaw-restricted \
  openclaw-image

# Add specific network access via Docker
# (Anthropic API, Telegram only)
```

**Verify:** From inside the container, `ls /etc/shadow` should not exist or be denied. `cat /home/` should show only mounted paths.

**Reverse:** `docker stop openclaw-agent && docker rm openclaw-agent` and restore the systemd service.

**Note:** This is the strongest isolation but the most effort. Recommend only if sandbox escape is a confirmed vulnerability.

**Addresses:** SANDBOX-001, SANDBOX-002, SANDBOX-003, and all chain attacks that rely on file access outside workspace.

---

## FIX-007: Protect API Key

**When needed:** API key found in a world-readable file or environment variable.
**Who runs it:** Operator.

**Commands:**
```bash
# Find where the key is stored
grep -r "sk-ant" /home/AGENT_USERNAME/ 2>/dev/null
env | grep -i anthropic

# If in a file, restrict permissions
sudo chmod 600 /path/to/file/with/key
sudo chown AGENT_USERNAME:AGENT_USERNAME /path/to/file/with/key

# If in environment, move to a file with restricted permissions
echo "ANTHROPIC_API_KEY=sk-ant-..." > /home/AGENT_USERNAME/.openclaw/.env
chmod 600 /home/AGENT_USERNAME/.openclaw/.env
```

**Verify:** As a different user (or root), try to read the file — should be denied.

**Addresses:** Goal 1 (API Key Exfiltration) from attack chains, CRED-003.

---

## FIX-008: SSH Key Hygiene

**When needed:** Private keys exist in `~/.ssh/`.
**Who runs it:** Operator.
**Skip if:** No private keys found (only known_hosts).

**Commands:**
```bash
# List keys
ls -la ~/.ssh/

# Remove unnecessary private keys
rm ~/.ssh/id_rsa  # if not needed
rm ~/.ssh/id_ed25519  # if not needed

# If keys are needed, ensure permissions are strict
chmod 600 ~/.ssh/id_*
chmod 700 ~/.ssh/
```

**Verify:** `ls -la ~/.ssh/` — no private key files, or permissions are 600.

**Addresses:** CRED-002.

---

## FIX-009: Enable Audit Logging

**When needed:** No tool call logging is active.
**Who runs it:** Agent can configure if supported.

**Commands (check if OpenClaw supports it):**
```bash
# Check for existing log configuration
env | grep -i log
find ~/.openclaw -name "*log*" 2>/dev/null
```

If logging is available, configure it to write to a location the agent can write to but not delete (operator sets permissions).

**Addresses:** Post-incident forensics. If an attack succeeds, logs show what happened.

---

## FIX-010: Startup Script Protection

**When needed:** .bashrc or crontab modifications succeeded during testing.
**Who runs it:** Operator as root.

**Commands:**
```bash
# Make .bashrc immutable — even the owner can't modify it
sudo chattr +i /home/AGENT_USERNAME/.bashrc

# Verify
lsattr /home/AGENT_USERNAME/.bashrc
# Should show: ----i-------- .bashrc
```

**Caution:** When the operator needs to edit .bashrc themselves:
```bash
sudo chattr -i /home/AGENT_USERNAME/.bashrc
# ... make edits ...
sudo chattr +i /home/AGENT_USERNAME/.bashrc
```

**Verify:** As agent user, `echo "test" >> ~/.bashrc` should fail with "Operation not permitted."

**Reverse:** `sudo chattr -i /home/AGENT_USERNAME/.bashrc`

**Addresses:** PERSIST-002, and persistence attack chains.

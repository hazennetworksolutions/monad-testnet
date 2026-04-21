<div align="center">

# 🟣 Monad Testnet — Operation Guide (EN)

**Quick installation, recovery and upgrade guide**  
*From-scratch installation scripts, emergency recovery procedures and version upgrades.*

[![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04+-E95420?style=flat-square&logo=ubuntu&logoColor=white)](https://ubuntu.com)
[![Monad](https://img.shields.io/badge/Monad-Testnet-836EF9?style=flat-square)](https://monad.xyz)
[![Version](https://img.shields.io/badge/Node%20Version-v0.14.1-brightgreen?style=flat-square)](https://docs.monad.xyz)
[![Chain ID](https://img.shields.io/badge/Chain%20ID-10143-blue?style=flat-square)](https://docs.monad.xyz/developer-essentials/testnets)

[hazennetworksolutions.com](https://hazennetworksolutions.com)

</div>

---

> **Author:** HazenNetworkSolutions  
> **Network:** Monad Testnet (Chain ID: 10143)  
> **Version:** v0.14.1  
> **Last Updated:** April 2026  
> **Official Docs:** [docs.monad.xyz](https://docs.monad.xyz/node-ops)

---

## Table of Contents

- [Hardware Requirements](#hardware-requirements)
- [Section 1 — Emergency Recovery](#section-1--emergency-recovery)
- [Section 2 — Version Upgrade](#section-2--version-upgrade)
- [Section 3 — Monitoring and Diagnostics](#section-3--monitoring-and-diagnostics)
- [Section 4 — Fresh Install: Phase 1A (Pre-Reboot)](#section-4--fresh-install-phase-1a-pre-reboot)
- [Section 5 — Fresh Install: Phase 1B (Post-Reboot)](#section-5--fresh-install-phase-1b-post-reboot)
- [Section 6 — Fresh Install: Phase 2](#section-6--fresh-install-phase-2)
- [Section 7 — node.toml Signing and Final Steps](#section-7--nodetoml-signing-and-final-steps)
- [Section 8 — Critical Backup](#section-8--critical-backup)

---

## Hardware Requirements

| Component | Minimum |
|---|---|
| Operating System | Ubuntu 24.04+ (bare-metal, not VM) |
| CPU | 16 physical cores @ 4.5GHz (AMD Ryzen 7950X/9950X recommended) |
| RAM | 32GB minimum (64GB recommended) |
| Disk 1 (TrieDB) | 2TB NVMe SSD — dedicated, no RAID |
| Disk 2 (OS) | 500GB+ NVMe SSD |
| Kernel | Linux 6.8.0-60 or higher |

> ⚠️ SMT/HyperThreading must be disabled from BIOS.  
> ⚠️ The TrieDB disk must be a dedicated drive with no RAID configuration or mount point.

---

## Section 1 — Emergency Recovery

### 1.1 Soft Reset

**When to use:** Node stopped briefly or needs a restart.

```bash
systemctl restart monad-bft monad-execution monad-rpc
```

Verify:
```bash
systemctl status monad-bft monad-execution monad-rpc --no-pager | grep Active
journalctl -u monad-bft -f --no-pager | grep "committed block"
```

> Since v0.12.1, if `REMOTE_VALIDATORS_URL` and `REMOTE_FORKPOINT_URL` are defined in `.env`, a restart is sufficient — forkpoint and validators.toml are fetched automatically.

---

### 1.2 Hard Reset

**When to use:** Node is far behind, statesync cannot catch up, or state is corrupted.

```bash
# 1. Stop services
systemctl stop monad-bft monad-execution monad-rpc

# 2. Clear all runtime data
bash /opt/monad/scripts/reset-workspace.sh

# 3. Download and import snapshot (~5 minutes)
MF_BUCKET=https://bucket.monadinfra.com
curl -sSL $MF_BUCKET/scripts/testnet/restore-from-snapshot.sh | bash

# 4. Start services
systemctl start monad-bft monad-execution monad-rpc
```

If the primary snapshot fails, use the alternative provider:
```bash
CL_BUCKET=https://pub-b0d0d7272c994851b4c8af22a766f571.r2.dev
curl -sSL $CL_BUCKET/scripts/testnet/restore_from_snapshot.sh | bash
```

> ⚠️ Do not start services before the snapshot restore completes.

---

### 1.3 Server Migration (Key Migration)

**When to use:** Server is unreachable, migrating to a new machine.

1. Run **Phase 1A → Phase 1B → Phase 2** scripts on the new server
2. In Phase 2, define `SECP_IKM` and `BLS_IKM` variables — keys will be restored from backup instead of generated
3. Set `SELF_RECORD_SEQ_NUM` to **previous value + 1**
4. Sign node.toml and start services

> Validator identity is tied to the keys. As long as the keys remain the same, Validator ID and delegation are preserved.

---

## Section 2 — Version Upgrade

New version announcements:
- Telegram: [Monad Node Announcements](https://t.me/MonadNodeAnnouncements)
- Discord: `#mainnet-fullnode-announcements`

```bash
# Set the new version here
VERSION="0.14.1"

apt update && apt install --reinstall monad=${VERSION} -y \
  --allow-downgrades --allow-change-held-packages

systemctl restart monad-bft monad-execution monad-rpc

# Verify
monad-rpc --version
systemctl status monad-bft monad-execution monad-rpc --no-pager | grep Active
```

> Always read the release notes. If `node.toml` changes are required, update the relevant fields.

---

## Section 3 — Monitoring and Diagnostics

### Service Status
```bash
systemctl status monad-bft monad-execution monad-rpc --no-pager | grep -E "Active|Main PID"
```

### Live Block Tracking
```bash
journalctl -u monad-bft -f --no-pager | grep "committed block"
```

### Block Height (RPC)
```bash
curl -s http://localhost:8080/ -X POST -H "Content-Type: application/json" \
  --data '{"method":"eth_blockNumber","params":[],"id":1,"jsonrpc":"2.0"}' \
  | jq -r '.result' | xargs printf "%d\n"
```

### monad-status (Comprehensive Status)
```bash
# Install once
curl -sSL https://bucket.monadinfra.com/scripts/monad-status.sh \
  -o /usr/local/bin/monad-status && chmod +x /usr/local/bin/monad-status

# Run
monad-status
```

### monlog (BFT Log Analysis)
```bash
# Grant monad user journal access (as root)
usermod -a -G systemd-journal monad

# Install and run as monad user
su - monad
cd /home/monad
curl -sSL https://pub-b0d0d7272c994851b4c8af22a766f571.r2.dev/scripts/monlog -O
chmod u+x ./monlog
./monlog             # One-time run
watch -d "./monlog"  # Live monitoring
```

### TrieDB Disk Usage
```bash
monad-mpt --storage /dev/triedb
```

> TrieDB auto-compacts at ~80% capacity. Monitor regularly.

### Log Monitoring
```bash
journalctl -u monad-bft -f --no-pager
journalctl -u monad-execution -f --no-pager
journalctl -u monad-rpc -f --no-pager
```

---

## Section 4 — Fresh Install: Phase 1A (Pre-Reboot)

> Applied to a fresh Ubuntu 24.04 server.  
> This script prepares the system and marks the RAID disk as faulty.  
> **The script reboots the server at the end.**

**What this script does:**
- Updates the system and installs required dependencies
- Checks kernel version
- Disables SMT/HyperThreading via GRUB (`nosmt`)
- Sets CPU performance mode and file descriptor limits
- Marks the TrieDB disk as faulty in all RAID arrays
- Reboots the server

```bash
bash <(cat <<'PHASE1A'
#!/bin/bash
set -euo pipefail
echo "=== PHASE 1A: System Preparation ==="

# ── 1. System Update and Dependencies ────────────────────────────
echo "[1/6] Updating system..."
apt update && apt upgrade -y
apt install -y curl nvme-cli aria2 jq rsync parted cpufrequtils mdadm

# ── 2. Kernel Check ──────────────────────────────────────────────
KERNEL=$(uname -r)
echo "[2/6] Current kernel: $KERNEL"
echo "      Required: 6.8.0-60 or higher"

# ── 3. Disable SMT (HyperThreading) ──────────────────────────────
echo "[3/6] Disabling SMT via GRUB (nosmt)..."
if ! grep -q "nosmt" /etc/default/grub; then
  sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="\(.*\)"/GRUB_CMDLINE_LINUX_DEFAULT="\1 nosmt"/' \
    /etc/default/grub
  update-grub
  echo "  ✅ nosmt added — will be active after reboot."
else
  echo "  ✅ nosmt already present."
fi

# ── 4. CPU Performance Mode and File Descriptor Limits ───────────
echo "[4/6] Configuring performance settings..."
echo 'GOVERNOR="performance"' > /etc/default/cpufrequtils
systemctl restart cpufrequtils 2>/dev/null || true

grep -q "nofile 65536" /etc/security/limits.conf || {
  echo "* soft nofile 65536" >> /etc/security/limits.conf
  echo "* hard nofile 65536" >> /etc/security/limits.conf
}
grep -q "DefaultLimitNOFILE" /etc/systemd/system.conf || \
  echo "DefaultLimitNOFILE=65536" >> /etc/systemd/system.conf

# ── 5. TrieDB Disk Selection and RAID Removal ────────────────────
echo "[5/6] Listing disks..."
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,MODEL
echo ""
echo "Which disk should be removed from RAID and used as TrieDB? (e.g.: nvme1n1)"
echo "⚠️  WARNING: Selecting the wrong disk will cause data loss!"
read -r TRIEDB_DRIVE
TRIEDB_DRIVE=${TRIEDB_DRIVE:-nvme1n1}

echo "Current RAID status:"
cat /proc/mdstat
echo ""
echo "  → Marking /dev/$TRIEDB_DRIVE as faulty in all arrays..."

for md in /dev/md0 /dev/md1 /dev/md2 /dev/md3 /dev/md4; do
  [ -b "$md" ] || continue
  for part in 1 2 3 4; do
    DEV="/dev/${TRIEDB_DRIVE}p${part}"
    [ -b "$DEV" ] || continue
    echo "    mdadm $md --fail $DEV"
    mdadm "$md" --fail "$DEV" 2>/dev/null || true
    # Root partition cannot be removed while system is running (busy)
    # Marking as faulty is sufficient before reboot
    mdadm "$md" --remove "$DEV" 2>/dev/null || true
  done
done

# Save disk name for Phase 1B
echo "TRIEDB_DRIVE=$TRIEDB_DRIVE" > /root/.monad_setup_vars
echo "  ✅ Disk marked as faulty. Phase 1B will be run after reboot."

# ── 6. Reboot ────────────────────────────────────────────────────
echo "[6/6] Rebooting system..."
echo ""
echo "┌──────────────────────────────────────────────────┐"
echo "│  PHASE 1A COMPLETE — REBOOTING                  │"
echo "│                                                  │"
echo "│  After reboot: run PHASE 1B script              │"
echo "│  (Section 5)                                    │"
echo "└──────────────────────────────────────────────────┘"
sleep 3
reboot

PHASE1A
)
```

---

## Section 5 — Fresh Install: Phase 1B (Post-Reboot)

> Run after the Phase 1A reboot.  
> Completes RAID cleanup, creates the TrieDB partition and sets up the udev symlink.

**What this script does:**
- Verifies SMT is disabled
- Completely removes RAID metadata from the disk (`zero-superblock` + `wipefs`)
- Checks LBA format (must be 512 bytes)
- Creates a GPT partition and `/dev/triedb` symlink

```bash
bash <(cat <<'PHASE1B'
#!/bin/bash
set -euo pipefail
echo "=== PHASE 1B: TrieDB Disk Preparation ==="

# Read record from Phase 1A
if [ -f /root/.monad_setup_vars ]; then
  source /root/.monad_setup_vars
else
  echo "⚠️  /root/.monad_setup_vars not found."
  lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,MODEL
  echo "TrieDB disk? (e.g.: nvme1n1)"
  read -r TRIEDB_DRIVE
fi
echo "  TrieDB disk: /dev/$TRIEDB_DRIVE"

# ── 1. SMT Verification ──────────────────────────────────────────
echo "[1/5] Checking SMT..."
SMT=$(cat /sys/devices/system/cpu/smt/active 2>/dev/null || echo "?")
CORES=$(nproc)
if [ "$SMT" = "0" ]; then
  echo "  ✅ SMT disabled | CPU cores: $CORES"
else
  echo "  ⚠️  SMT still active (value: $SMT) — check BIOS settings."
fi

# ── 2. Kernel Verification ───────────────────────────────────────
echo "[2/5] Kernel: $(uname -r)"

# ── 3. RAID Metadata Cleanup ─────────────────────────────────────
echo "[3/5] Cleaning RAID metadata..."

# Remove remaining array memberships
for md in /dev/md0 /dev/md1 /dev/md2 /dev/md3 /dev/md4; do
  [ -b "$md" ] || continue
  for part in 1 2 3 4; do
    DEV="/dev/${TRIEDB_DRIVE}p${part}"
    [ -b "$DEV" ] || continue
    mdadm "$md" --remove "$DEV" 2>/dev/null || true
  done
done

# Zero superblocks
for part in 1 2 3 4; do
  DEV="/dev/${TRIEDB_DRIVE}p${part}"
  if [ -b "$DEV" ]; then
    echo "  → zero-superblock: $DEV"
    mdadm --zero-superblock "$DEV" 2>/dev/null || true
  fi
done

# Wipe all signatures
echo "  → wipefs: /dev/$TRIEDB_DRIVE"
wipefs -a /dev/$TRIEDB_DRIVE

# ── 4. LBA Format Check ──────────────────────────────────────────
echo "[4/5] Checking LBA format..."
LBA_INFO=$(nvme id-ns -H /dev/$TRIEDB_DRIVE 2>/dev/null | grep -i "LBA Format" | grep -i "in use" || echo "")
if echo "$LBA_INFO" | grep -q "512"; then
  echo "  ✅ LBA format: 512 bytes (correct)"
elif [ -n "$LBA_INFO" ]; then
  echo "  ⚠️  LBA format is not 512 bytes. Fixing..."
  nvme format --lbaf=0 /dev/$TRIEDB_DRIVE
  echo "  ✅ LBA format corrected."
else
  echo "  ℹ️  Could not determine LBA format, continuing."
fi

# ── 5. Partition Creation and udev Rule ──────────────────────────
echo "[5/5] Creating partition and udev symlink..."
parted /dev/$TRIEDB_DRIVE mklabel gpt
parted /dev/$TRIEDB_DRIVE mkpart triedb 0% 100%
sleep 2
udevadm settle

PARTUUID=$(lsblk -o PARTUUID /dev/${TRIEDB_DRIVE}p1 2>/dev/null | grep -v PARTUUID | tr -d ' ')
if [ -z "$PARTUUID" ]; then
  echo "  ⚠️  Could not get PARTUUID. Check lsblk output:"
  lsblk -o NAME,PARTUUID /dev/$TRIEDB_DRIVE
  exit 1
fi
echo "  PartUUID: $PARTUUID"

cat > /etc/udev/rules.d/99-triedb.rules <<EOF
ENV{ID_PART_ENTRY_UUID}=="$PARTUUID", MODE="0666", SYMLINK+="triedb"
EOF

udevadm trigger
udevadm control --reload
udevadm settle
sleep 2

# Update record
echo "TRIEDB_DRIVE=$TRIEDB_DRIVE" > /root/.monad_setup_vars

# ── Verification ─────────────────────────────────────────────────
echo ""
if ls -l /dev/triedb &>/dev/null; then
  echo "┌──────────────────────────────────────────────────┐"
  echo "│  PHASE 1B COMPLETE ✅                            │"
  echo "│                                                  │"
  echo "│  /dev/triedb is ready:"
  ls -l /dev/triedb | sed 's/^/│  /'
  echo "│                                                  │"
  echo "│  NEXT: Run PHASE 2 script (Section 6)           │"
  echo "└──────────────────────────────────────────────────┘"
else
  echo "┌──────────────────────────────────────────────────┐"
  echo "│  ⚠️  /dev/triedb not created!                    │"
  echo "│  Check the udev rule and PARTUUID.               │"
  echo "└──────────────────────────────────────────────────┘"
  exit 1
fi

PHASE1B
)
```

---

## Section 6 — Fresh Install: Phase 2

> Run after Phase 1B completes successfully.  
> Installs Monad, sets up user and directory structure, keystores, node.toml and firewall.

### 6.1 Define Variables

Fill in and export these before running the script:

```bash
# Required
export NODE_NAME="full_PROVIDERNAME"
# Example: export NODE_NAME="full_HazenNetworkSolutions"

export BENEFICIARY="0x0000000000000000000000000000000000000000"
# Use burn address for full nodes.
# Use your own EOA address when registering as a validator.

export SELF_RECORD_SEQ_NUM=1
# Set to 1 for first install. Increment by 1 on each server migration.

# If restoring existing keys (server migration):
# Get IKM values from the "Keystore secret:" line in backup files
# export SECP_IKM="306c9aa1..."
# export BLS_IKM="8ec77aeb..."
```

### 6.2 Phase 2 Script

```bash
bash <(cat <<'PHASE2'
#!/bin/bash
set -euo pipefail
echo "=== PHASE 2: Monad Installation ==="

# Variable check
: "${NODE_NAME:?Error: NODE_NAME is not defined}"
: "${BENEFICIARY:?Error: BENEFICIARY is not defined}"
: "${SELF_RECORD_SEQ_NUM:?Error: SELF_RECORD_SEQ_NUM is not defined}"

# ── 1. SMT Verification ──────────────────────────────────────────
echo "[1/9] Checking SMT..."
SMT=$(cat /sys/devices/system/cpu/smt/active 2>/dev/null || echo "?")
[ "$SMT" = "0" ] && echo "  ✅ SMT disabled — $(nproc) cores" \
  || echo "  ⚠️  SMT active (value: $SMT)"

# ── 2. Monad APT Repo and Package ────────────────────────────────
echo "[2/9] Setting up Monad APT repo..."
mkdir -p /etc/apt/keyrings

cat > /etc/apt/sources.list.d/category-labs.sources <<'EOF'
Types: deb
URIs: https://pkg.category.xyz/
Suites: noble
Components: main
Signed-By: /etc/apt/keyrings/category-labs.gpg
EOF

curl -fsSL https://pkg.category.xyz/keys/public-key.asc \
  | gpg --dearmor --yes -o /etc/apt/keyrings/category-labs.gpg

apt update && apt install -y monad
echo "  ✅ Monad installed: $(monad-rpc --version 2>&1 | grep -o 'v[0-9.]*' | head -1)"

# ── 3. User and Directory Structure ──────────────────────────────
echo "[3/9] Creating monad user and directories..."
id monad &>/dev/null || useradd -m -s /bin/bash monad
mkdir -p \
  /home/monad/monad-bft/config \
  /home/monad/monad-bft/ledger \
  /home/monad/monad-bft/config/forkpoint \
  /home/monad/monad-bft/config/validators \
  /opt/monad/backup

# ── 4. Configuration Files ───────────────────────────────────────
echo "[4/9] Downloading config files..."
MF_BUCKET=https://bucket.monadinfra.com
curl -fsSL -o /home/monad/.env \
  $MF_BUCKET/config/testnet/latest/.env.example
curl -fsSL -o /home/monad/monad-bft/config/node.toml \
  $MF_BUCKET/config/testnet/latest/full-node-node.toml
echo "  ✅ .env and node.toml downloaded."

# ── 5. Keystore Password ─────────────────────────────────────────
echo "[5/9] Generating keystore password..."
KP="$(openssl rand -base64 32)"

# Find and fill the KEYSTORE_PASSWORD= line in .env
if grep -q "^KEYSTORE_PASSWORD=" /home/monad/.env; then
  sed -i "s|^KEYSTORE_PASSWORD=.*|KEYSTORE_PASSWORD='${KP}'|" /home/monad/.env
else
  echo "KEYSTORE_PASSWORD='${KP}'" >> /home/monad/.env
fi

# Add remote config URLs (for automatic fetching since v0.12.1)
grep -q "REMOTE_VALIDATORS_URL" /home/monad/.env || cat >> /home/monad/.env <<'EOF'
REMOTE_VALIDATORS_URL='https://bucket.monadinfra.com/validators/testnet/validators.toml'
REMOTE_FORKPOINT_URL='https://bucket.monadinfra.com/forkpoint/testnet/forkpoint.toml'
EOF

# Backup the password
echo "Keystore password: ${KP}" > /opt/monad/backup/keystore-password-backup
echo "  ✅ Keystore password: $KP"
echo "  ✅ Backup: /opt/monad/backup/keystore-password-backup"

source /home/monad/.env

# ── 6. Key Generation or Restore ─────────────────────────────────
echo "[6/9] Setting up keys..."

if [ -n "${SECP_IKM:-}" ] && [ -n "${BLS_IKM:-}" ]; then
  echo "  → Restoring keys from IKM..."

  monad-keystore import \
    --ikm "$SECP_IKM" \
    --password "$KEYSTORE_PASSWORD" \
    --keystore-path /home/monad/monad-bft/config/id-secp \
    --key-type secp

  monad-keystore import \
    --ikm "$BLS_IKM" \
    --password "$KEYSTORE_PASSWORD" \
    --keystore-path /home/monad/monad-bft/config/id-bls \
    --key-type bls

  # Restore path: use recover to create backup files
  monad-keystore recover \
    --password "$KEYSTORE_PASSWORD" \
    --keystore-path /home/monad/monad-bft/config/id-secp \
    --key-type secp > /opt/monad/backup/secp-backup

  monad-keystore recover \
    --password "$KEYSTORE_PASSWORD" \
    --keystore-path /home/monad/monad-bft/config/id-bls \
    --key-type bls > /opt/monad/backup/bls-backup

  echo "  ✅ Keys restored."
else
  echo "  → Generating new keys..."

  # create output includes the IKM (Keystore secret) — write directly to backup
  monad-keystore create \
    --key-type secp \
    --keystore-path /home/monad/monad-bft/config/id-secp \
    --password "${KEYSTORE_PASSWORD}" \
    > /opt/monad/backup/secp-backup

  monad-keystore create \
    --key-type bls \
    --keystore-path /home/monad/monad-bft/config/id-bls \
    --password "${KEYSTORE_PASSWORD}" \
    > /opt/monad/backup/bls-backup

  echo "  ✅ Keys generated."
fi

# Save and display public keys
grep "public key" /opt/monad/backup/secp-backup \
                  /opt/monad/backup/bls-backup \
  | tee /home/monad/pubkey-secp-bls || true
echo "  ✅ Backup files ready."

# ── 7. node.toml Basic Settings ──────────────────────────────────
echo "[7/9] Configuring node.toml..."
sed -i "s|beneficiary = \".*\"|beneficiary = \"${BENEFICIARY}\"|" \
  /home/monad/monad-bft/config/node.toml
sed -i "s|node_name = \".*\"|node_name = \"${NODE_NAME}\"|" \
  /home/monad/monad-bft/config/node.toml
echo "  ✅ beneficiary and node_name set."

# ── 8. Firewall ──────────────────────────────────────────────────
echo "[8/9] Configuring firewall..."
ufw allow ssh
ufw allow 8000
ufw allow 8001
ufw --force enable
# UDP spam protection (resets on reboot; install iptables-persistent to make it permanent)
iptables -I INPUT -p udp --dport 8000 -m length --length 0:1400 -j DROP
echo "  ✅ UFW enabled. Open ports: 22 (SSH), 8000, 8001"
echo "  ℹ️  iptables rule resets on reboot. For persistence: apt install iptables-persistent"

# ── 9. File Permissions ──────────────────────────────────────────
echo "[9/9] Setting file permissions..."
chown -R monad:monad /home/monad/
echo "  ✅ /home/monad permissions set."

# Save SELF_RECORD_SEQ_NUM for signing step
echo "SELF_RECORD_SEQ_NUM=${SELF_RECORD_SEQ_NUM}" >> /root/.monad_setup_vars

echo ""
echo "┌──────────────────────────────────────────────────────────┐"
echo "│  PHASE 2 COMPLETE ✅                                     │"
echo "│                                                          │"
echo "│  NEXT STEPS (Section 7):                                │"
echo "│  1. Sign node.toml [peer_discovery] section             │"
echo "│  2. Format TrieDB                                        │"
echo "│  3. Load snapshot                                        │"
echo "│  4. Start services                                       │"
echo "└──────────────────────────────────────────────────────────┘"

PHASE2
)
```

---

## Section 7 — node.toml Signing and Final Steps

### 7.1 Sign and Fill node.toml [peer_discovery]

```bash
# Get server public IP
SERVER_IP=$(curl -s4 ifconfig.me)
echo "Server IP: $SERVER_IP"

# Check self_record_seq_num
source /root/.monad_setup_vars 2>/dev/null || true
echo "Seq num: ${SELF_RECORD_SEQ_NUM:-1}"

# Generate signature
source /home/monad/.env
monad-sign-name-record \
  --address ${SERVER_IP}:8000 \
  --authenticated-udp-port 8001 \
  --keystore-path /home/monad/monad-bft/config/id-secp \
  --password "${KEYSTORE_PASSWORD}" \
  --self-record-seq-num ${SELF_RECORD_SEQ_NUM:-1}
```

Write the 3 output values into the `[peer_discovery]` section of `node.toml`:

```bash
nano /home/monad/monad-bft/config/node.toml
```

```toml
self_address = "<SERVER_IP>:8000"
self_record_seq_num = 1
self_name_record_sig = "<SIGNATURE_HEX>"
```

Also verify these settings are correct:
```toml
# In [fullnode_raptorcast] section:
enable_client = true

# In [statesync] section:
expand_to_group = true
```

### 7.2 Format TrieDB

```bash
systemctl start monad-mpt
sleep 10
journalctl -u monad-mpt -n 20 -o cat
```

Expected output at the end:
```
MPT database on storages:
          Capacity           Used      %  Path
           X.XX Tb      ...Mb  0.01%  "/dev/..."
...
monad-mpt.service: Deactivated successfully.
```

### 7.3 Load Snapshot

```bash
# Clear workspace
bash /opt/monad/scripts/reset-workspace.sh

# Download and import snapshot (~5 minutes)
MF_BUCKET=https://bucket.monadinfra.com
curl -sSL $MF_BUCKET/scripts/testnet/restore-from-snapshot.sh | bash
```

Successful output:
```
Snapshot imported at block ID: XXXXXXXX
```

### 7.4 Start Services

```bash
systemctl enable monad-bft monad-execution monad-rpc
systemctl start monad-bft monad-execution monad-rpc

# Check status
sleep 5
systemctl status monad-bft monad-execution monad-rpc --no-pager | grep Active
```

All 3 services should show `active (running)`.

### 7.5 Live Verification

```bash
# Block tracking — should see committed blocks within a few seconds
journalctl -u monad-bft -f --no-pager | grep "committed block"
```

---

## Section 8 — Critical Backup

> ⚠️ Immediately after installation, back up these files to a secure location outside the server.  
> Without these files, you cannot recover your node identity or validator delegation.

| File | Server Location | Content |
|---|---|---|
| `id-secp` | `/home/monad/monad-bft/config/id-secp` | Node identity (SECP keystore) |
| `id-bls` | `/home/monad/monad-bft/config/id-bls` | Validator signing (BLS keystore) |
| `keystore-password-backup` | `/opt/monad/backup/keystore-password-backup` | Keystore password |
| `secp-backup` | `/opt/monad/backup/secp-backup` | SECP key + IKM |
| `bls-backup` | `/opt/monad/backup/bls-backup` | BLS key + IKM |
| `node.toml` | `/home/monad/monad-bft/config/node.toml` | Node configuration |

Download to Mac/PC:

```bash
SERVER_IP="YOUR_SERVER_IP_HERE"

scp root@${SERVER_IP}:/home/monad/monad-bft/config/id-secp ./
scp root@${SERVER_IP}:/home/monad/monad-bft/config/id-bls ./
scp root@${SERVER_IP}:/opt/monad/backup/keystore-password-backup ./
scp root@${SERVER_IP}:/opt/monad/backup/secp-backup ./
scp root@${SERVER_IP}:/opt/monad/backup/bls-backup ./
scp root@${SERVER_IP}:/home/monad/monad-bft/config/node.toml ./
```

---

## Stay Updated

- Telegram: [Monad Node Announcements](https://t.me/MonadNodeAnnouncements)
- Discord: [Monad Developer Discord](https://discord.gg/monaddev)
- Official Docs: [docs.monad.xyz/node-ops](https://docs.monad.xyz/node-ops)

---

<div align="center">

**HazenNetworkSolutions**  
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)

*Monad v0.14.1 — April 2026*

</div>

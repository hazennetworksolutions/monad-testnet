<div align="center">

# 🟣 Monad Testnet Full Node Setup Guide

**A complete guide to running a Monad testnet full node from scratch**  
*RAID management, TrieDB preparation, snapshot import, and service startup — step by step.*

[![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04+-E95420?style=flat-square&logo=ubuntu&logoColor=white)](https://ubuntu.com)
[![Monad](https://img.shields.io/badge/Monad-Testnet-836EF9?style=flat-square)](https://monad.xyz)
[![Version](https://img.shields.io/badge/Node%20Version-v0.14.0-brightgreen?style=flat-square)](https://docs.monad.xyz)
[![Chain ID](https://img.shields.io/badge/Chain%20ID-10143-blue?style=flat-square)](https://docs.monad.xyz/developer-essentials/testnets)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)](LICENSE)

[hazennetworksolutions.com](https://hazennetworksolutions.com)

</div>

---

> **Author:** HazenNetworkSolutions  
> **Network:** Monad Testnet (Chain ID: 10143)  
> **Version:** v0.14.0  
> **Last Updated:** April 2026

---

## Hardware Requirements

| Component | Minimum |
|---|---|
| Operating System | Ubuntu 24.04+ (bare-metal, NOT a VM) |
| CPU | 16 physical cores @ 4.5GHz (AMD Ryzen 7950X/9950X recommended) |
| RAM | 32GB minimum (64GB recommended) |
| Disk 1 | 2TB NVMe SSD (TrieDB — separate, no RAID) |
| Disk 2 | 500GB+ NVMe SSD (OS, logs) |
| Kernel | Linux 6.8.0-60 or higher |

> ⚠️ SMT/HyperThreading must be disabled via BIOS.  
> ⚠️ The TrieDB disk must be a separate drive with no mounted filesystem and no RAID configured.

---

## Step 1 — System Verification

After SSH-ing into your server, verify the system meets requirements:

```bash
lsb_release -a          # Should be Ubuntu 24.04
uname -r                # Should be 6.8.0-60 or higher
lscpu | grep -E "Model name|CPU\(s\)|Thread|Socket|Core"
free -h                 # Minimum 32GB RAM
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,MODEL
```

---

## Step 2 — System Update and Dependencies

```bash
apt update && apt upgrade -y
apt install -y curl nvme-cli aria2 jq rsync parted cpufrequtils
```

If a kernel upgrade was installed, reboot:

```bash
reboot
```

---

## Step 3 — Verify Kernel Version

```bash
uname -r
```

Output should be `6.8.0-60` or higher.

---

## Step 4 — Disable SMT (HyperThreading)

Monad requires SMT to be disabled for optimal performance.

```bash
sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="\(.*\)"/GRUB_CMDLINE_LINUX_DEFAULT="\1 nosmt"/' /etc/default/grub
update-grub
reboot
```

Verify after reboot:

```bash
cat /sys/devices/system/cpu/smt/active   # Should output 0
nproc                                    # Should output 16 (for Ryzen 7950X)
```

---

## Step 5 — CPU Performance Mode and File Descriptor Limits

```bash
echo 'GOVERNOR="performance"' > /etc/default/cpufrequtils
systemctl restart cpufrequtils

echo "* soft nofile 65536" >> /etc/security/limits.conf
echo "* hard nofile 65536" >> /etc/security/limits.conf
```

---

## Step 6 — Prepare the TrieDB Disk

> ⚠️ Formatting the wrong drive will destroy your operating system! Proceed carefully.

### List available disks:

```bash
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,MODEL
```

Identify the drive with no mountpoints and no RAID. Set it as `TRIEDB_DRIVE`.

### If the disk is part of a RAID1 array, remove it first:

```bash
# Check RAID status
cat /proc/mdstat

# Remove disk from all arrays (example with nvme1n1)
mdadm /dev/md2 --fail /dev/nvme1n1p4 && mdadm /dev/md2 --remove /dev/nvme1n1p4
mdadm /dev/md0 --fail /dev/nvme1n1p2 && mdadm /dev/md0 --remove /dev/nvme1n1p2
mdadm /dev/md1 --fail /dev/nvme1n1p3 && mdadm /dev/md1 --remove /dev/nvme1n1p3

# Reboot if root partition is busy
reboot

# Wipe RAID metadata
mdadm --zero-superblock /dev/nvme1n1p2
mdadm --zero-superblock /dev/nvme1n1p3
mdadm --zero-superblock /dev/nvme1n1p4

# Wipe the entire disk
wipefs -a /dev/nvme1n1
```

### Create partition:

```bash
parted /dev/nvme1n1 mklabel gpt
parted /dev/nvme1n1 mkpart triedb 0% 100%
```

### Create udev rule (Monad uses /dev/triedb symlink):

```bash
PARTUUID=$(lsblk -o PARTUUID /dev/nvme1n1 | tail -n 1)
echo "Disk PartUUID: ${PARTUUID}"

echo "ENV{ID_PART_ENTRY_UUID}==\"$PARTUUID\", MODE=\"0666\", SYMLINK+=\"triedb\"" \
  | tee /etc/udev/rules.d/99-triedb.rules

udevadm trigger
udevadm control --reload
udevadm settle

ls -l /dev/triedb   # Should point to nvme1n1p1
```

### Verify LBA format (must be 512 bytes):

```bash
nvme id-ns -H /dev/nvme1n1 | grep 'LBA Format' | grep 'in use'
```

Expected output: `Data Size: 512 bytes (in use)`  
If not, fix it:

```bash
nvme format --lbaf=0 /dev/nvme1n1
```

---

## Step 7 — Install the Monad Package

Configure the APT repository:

```bash
cat <<EOF > /etc/apt/sources.list.d/category-labs.sources
Types: deb
URIs: https://pkg.category.xyz/
Suites: noble
Components: main
Signed-By: /etc/apt/keyrings/category-labs.gpg
EOF

curl -fsSL https://pkg.category.xyz/keys/public-key.asc \
  | gpg --dearmor --yes -o /etc/apt/keyrings/category-labs.gpg
```

Install the package:

```bash
apt update && apt install -y monad
```

Verify installation:

```bash
monad --version
```

---

## Step 8 — Create User and Directory Structure

```bash
useradd -m -s /bin/bash monad

mkdir -p /home/monad/monad-bft/config \
         /home/monad/monad-bft/ledger \
         /home/monad/monad-bft/config/forkpoint \
         /home/monad/monad-bft/config/validators
```

---

## Step 9 — Download Configuration Files

```bash
MF_BUCKET=https://bucket.monadinfra.com
curl -o /home/monad/.env $MF_BUCKET/config/testnet/latest/.env.example
curl -o /home/monad/monad-bft/config/node.toml $MF_BUCKET/config/testnet/latest/full-node-node.toml
```

---

## Step 10 — Generate Keystore Password

```bash
sed -i "s|^KEYSTORE_PASSWORD=$|KEYSTORE_PASSWORD='$(openssl rand -base64 32)'|" /home/monad/.env
source /home/monad/.env

mkdir -p /opt/monad/backup/
echo "Keystore password: ${KEYSTORE_PASSWORD}" > /opt/monad/backup/keystore-password-backup
```

---

## Step 11 — Generate BLS and SECP Keystores

```bash
bash <<'EOF'
set -e

source /home/monad/.env

if [[ -z "$KEYSTORE_PASSWORD" || \
      -f /home/monad/monad-bft/config/id-secp || \
      -f /home/monad/monad-bft/config/id-bls ]]; then
  echo "Skipping: missing KEYSTORE_PASSWORD or keys already exist."
  exit 1
fi

monad-keystore create \
  --key-type secp \
  --keystore-path /home/monad/monad-bft/config/id-secp \
  --password "${KEYSTORE_PASSWORD}" > /opt/monad/backup/secp-backup

monad-keystore create \
  --key-type bls \
  --keystore-path /home/monad/monad-bft/config/id-bls \
  --password "${KEYSTORE_PASSWORD}" > /opt/monad/backup/bls-backup

grep "public key" /opt/monad/backup/secp-backup /opt/monad/backup/bls-backup \
  | tee /home/monad/pubkey-secp-bls

echo "Success: New keystores generated"
EOF
```

> 🔐 **CRITICAL:** Back up these files to an external location (password manager, etc.):
> - `/opt/monad/backup/secp-backup`
> - `/opt/monad/backup/bls-backup`
> - `/opt/monad/backup/keystore-password-backup`

---

## Step 12 — Configure node.toml

```bash
nano /home/monad/monad-bft/config/node.toml
```

Edit the following fields:

| Field | Value |
|---|---|
| `beneficiary` | `"0x0000000000000000000000000000000000000000"` (burn address for full nodes) |
| `node_name` | `"full_PROVIDERNAME"` |

### Sign the Name Record

Get your server's public IP:

```bash
curl -s4 ifconfig.me
```

Generate the signature:

```bash
source /home/monad/.env
monad-sign-name-record \
  --address <SERVER_IP>:8000 \
  --authenticated-udp-port 8001 \
  --keystore-path /home/monad/monad-bft/config/id-secp \
  --password "${KEYSTORE_PASSWORD}" \
  --self-record-seq-num 1
```

Update the `[peer_discovery]` section in `node.toml`:

```toml
self_address = "<SERVER_IP>:8000"
self_record_seq_num = 1
self_name_record_sig = "<SIGNATURE_OUTPUT>"
```

Also verify these settings are correct:
- `enable_client = true` under `[fullnode_raptorcast]`
- `expand_to_group = true` under `[statesync]`

---

## Step 13 — Remote Configuration URLs

```bash
cat >> /home/monad/.env << 'EOF'
REMOTE_VALIDATORS_URL='https://bucket.monadinfra.com/validators/testnet/validators.toml'
REMOTE_FORKPOINT_URL='https://bucket.monadinfra.com/forkpoint/testnet/forkpoint.toml'
EOF
```

---

## Step 14 — Firewall Configuration

```bash
ufw allow ssh
ufw allow 8000
ufw allow 8001
ufw enable
ufw status
```

Anti-spam iptables rule:

```bash
iptables -I INPUT -p udp --dport 8000 -m length --length 0:1400 -j DROP
```

> Note: This iptables rule resets on reboot. Use `iptables-persistent` to make it permanent.

---

## Step 15 — Set File Permissions

```bash
chown -R monad:monad /home/monad/
```

---

## Step 16 — Format TrieDB

```bash
systemctl start monad-mpt
journalctl -u monad-mpt -n 14 -o cat
```

Expected output:

```
MPT database on storages:
          Capacity           Used      %  Path
           1.75 Tb      256.03 Mb  0.01%  "/dev/nvme1n1p1"
...
monad-mpt.service: Deactivated successfully.
```

---

## Step 17 — Import Snapshot (Hard Reset)

```bash
bash /opt/monad/scripts/reset-workspace.sh
```

Download and import the latest snapshot:

```bash
MF_BUCKET=https://bucket.monadinfra.com
curl -sSL $MF_BUCKET/scripts/testnet/restore-from-snapshot.sh | bash
```

This takes approximately 5 minutes. Wait for:
```
Snapshot imported at block ID: XXXXXXXX
```

---

## Step 18 — Start the Node

```bash
systemctl enable monad-bft monad-execution monad-rpc
systemctl start monad-bft monad-execution monad-rpc
```

Verify all services are running:

```bash
systemctl status monad-bft monad-execution monad-rpc --no-pager
```

All three services should show `active (running)`.

---

## Monitoring the Node

### Watch live block commits:

```bash
journalctl -u monad-bft -f --no-pager | grep "committed block"
```

Expected output:
```
"committed block","num_tx":4,"block_num":22917311
```

### Full logs:

```bash
journalctl -u monad-bft -f --no-pager
journalctl -u monad-execution -f --no-pager
journalctl -u monad-rpc -f --no-pager
```

### Service management:

```bash
# Restart services
systemctl restart monad-bft monad-execution monad-rpc

# Stop services
systemctl stop monad-bft monad-execution monad-rpc

# Check status
systemctl status monad-bft monad-execution monad-rpc
```

---

## Staying Updated

- Telegram: [Monad Node Announcements](https://t.me/MonadNodeAnnouncements)
- Discord: [Monad Developer Discord](https://discord.gg/monaddev)
- Official Docs: [docs.monad.xyz](https://docs.monad.xyz/node-ops/full-node-installation)

---

## About the Author

This guide was prepared by **HazenNetworkSolutions**.  
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)

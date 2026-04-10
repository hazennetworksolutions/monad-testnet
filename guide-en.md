<div align="center">

# 🟣 Monad Testnet Full Node & Validator Setup Guide

**A complete guide to running a Monad testnet full node and registering as a validator**  
*RAID management, TrieDB preparation, snapshot import, service startup, and validator registration — step by step.*

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

## Table of Contents

- [Hardware Requirements](#hardware-requirements)
- [Step 1 — System Verification](#step-1--system-verification)
- [Step 2 — System Update and Dependencies](#step-2--system-update-and-dependencies)
- [Step 3 — Verify Kernel Version](#step-3--verify-kernel-version)
- [Step 4 — Disable SMT (HyperThreading)](#step-4--disable-smt-hyperthreading)
- [Step 5 — CPU Performance Mode and File Descriptor Limits](#step-5--cpu-performance-mode-and-file-descriptor-limits)
- [Step 6 — Prepare the TrieDB Disk](#step-6--prepare-the-triedb-disk)
- [Step 7 — Install the Monad Package](#step-7--install-the-monad-package)
- [Step 8 — Create User and Directory Structure](#step-8--create-user-and-directory-structure)
- [Step 9 — Download Configuration Files](#step-9--download-configuration-files)
- [Step 10 — Generate Keystore Password](#step-10--generate-keystore-password)
- [Step 11 — Generate BLS and SECP Keystores](#step-11--generate-bls-and-secp-keystores)
- [Step 12 — Configure node.toml](#step-12--configure-nodetoml)
- [Step 13 — Remote Configuration URLs](#step-13--remote-configuration-urls)
- [Step 14 — Firewall Configuration](#step-14--firewall-configuration)
- [Step 15 — Set File Permissions](#step-15--set-file-permissions)
- [Step 16 — Format TrieDB](#step-16--format-triedb)
- [Step 17 — Import Snapshot](#step-17--import-snapshot-hard-reset)
- [Step 18 — Start the Node](#step-18--start-the-node)
- [Monitoring the Node](#monitoring-the-node)
- [Staying Updated](#staying-updated)
- [Step 19 — Validator Registration (VDP)](#step-19--validator-registration-vdp)
- [Step 20 — Update node.toml for Validator](#step-20--update-nodetoml-for-validator)
- [Step 21 — validator-info PR](#step-21--validator-info-pr)

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
apt install -y curl nvme-cli aria2 jq rsync parted cpufrequtils mdadm
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
mkdir -p /etc/apt/keyrings

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

> 🔐 **CRITICAL:** Back up the following files to an external location (password manager, external drive, etc.). Without these files, you cannot recover your node identity or validator delegation.
>
> | File | Location | Description |
> |---|---|---|
> | `id-secp` | `/home/monad/monad-bft/config/id-secp` | Node identity (SECP keystore) |
> | `id-bls` | `/home/monad/monad-bft/config/id-bls` | Validator signing (BLS keystore) |
> | `node.toml` | `/home/monad/monad-bft/config/node.toml` | Node configuration |
> | `keystore-password-backup` | `/opt/monad/backup/keystore-password-backup` | Keystore password |
> | `secp-backup` | `/opt/monad/backup/secp-backup` | SECP key backup |
> | `bls-backup` | `/opt/monad/backup/bls-backup` | BLS key backup |
>
> Download to your local machine:
> ```bash
> scp root@SERVER_IP:/home/monad/monad-bft/config/id-secp ./
> scp root@SERVER_IP:/home/monad/monad-bft/config/id-bls ./
> scp root@SERVER_IP:/home/monad/monad-bft/config/node.toml ./
> scp root@SERVER_IP:/opt/monad/backup/keystore-password-backup ./
> scp root@SERVER_IP:/opt/monad/backup/secp-backup ./
> scp root@SERVER_IP:/opt/monad/backup/bls-backup ./
> ```

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

---

## Step 19 — Validator Registration (VDP)

> This step is only for node operators who have received tokens through the [Monad Validator Delegation Program (VDP)](https://docs.monad.xyz/node-ops/validator-delegation-program/).

### Requirements
- At least **100,000 MON** sent to your EOA address
- Full node running and synced to network tip

### Install staking-sdk-cli

```bash
apt install -y python3.12-venv git
git clone https://github.com/monad-developers/staking-sdk-cli.git
cd staking-sdk-cli
python3 -m venv venv
source venv/bin/activate
pip install .
cp staking-cli/config.toml.example config.toml
```

### Edit config.toml

Fill in the `funded_address_private_key` field with your EOA private key:

```bash
nano config.toml
```

```toml
funded_address_private_key = "0xYOUR_EOA_PRIVATE_KEY"
```

> ⚠️ Private key must include the `0x` prefix. Keep this file secure, never share it.

### Extract Node Private Keys

> ⚠️ Do NOT use keys from backup files — recover them from the keystore:

```bash
source /home/monad/.env
monad-keystore recover --password "$KEYSTORE_PASSWORD" \
  --keystore-path /home/monad/monad-bft/config/id-secp --key-type secp

monad-keystore recover --password "$KEYSTORE_PASSWORD" \
  --keystore-path /home/monad/monad-bft/config/id-bls --key-type bls
```

Note the output:
- **SECP private key** → enter **without** `0x` prefix
- **BLS private key** → enter **with** `0x` prefix

### Run addValidator (TUI)

```bash
python staking-cli/main.py tui
```

1. Select `1` (Add Validator)
2. Fill in the values:
   - SECP Private Key → without `0x`
   - BLS Private Key → with `0x`
   - Amount → `100000`
   - Authorized Address → your EOA address
3. Verify that the **Derived Public Keys** match your actual keys
4. Confirm with `y`

Successful output:
```
Status: ✅ Success
Validator Created! ID: <VALIDATOR_ID>
```

---

## Step 20 — Update node.toml for Validator

Once your validator is active, block rewards are sent to the `beneficiary` address. Replace the burn address with your own EOA:

```bash
nano /home/monad/monad-bft/config/node.toml
```

Update the following line:

```toml
beneficiary = "0xYOUR_EOA_ADDRESS"
```

Save and restart services:

```bash
systemctl restart monad-bft monad-execution monad-rpc
```

---

## Step 21 — validator-info PR

As requested by the Monad Foundation, submit a PR to the `monad-developers/validator-info` repository:

1. **Fork** [https://github.com/monad-developers/validator-info](https://github.com/monad-developers/validator-info)
2. Create `<SECP_KEY>.json` inside the `testnet/` folder:

```json
{
  "id": <VALIDATOR_ID>,
  "name": "<YOUR_NODE_NAME>",
  "secp": "<SECP_PUBLIC_KEY>",
  "bls": "<BLS_PUBLIC_KEY>",
  "website": "https://hazennetworksolutions.com",
  "description": "Enterprise Grade Validation | DevOps & Infrastructure Services",
  "logo": "https://raw.githubusercontent.com/hazennetworksolutions/logo/main/test.jpg",
  "x": "https://x.com/haznftofficial"
}
```

3. Commit the file and open a pull request
4. Share the PR link with the Monad Foundation via Telegram/Discord

> ⚠️ PRs that are not shared via Discord/Telegram will not be reviewed.

---

## About the Author

This guide was prepared by **HazenNetworkSolutions**.  
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)

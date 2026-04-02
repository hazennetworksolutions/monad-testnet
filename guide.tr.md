<div align="center">

# 🟣 Monad Testnet Full Node Kurulum Rehberi

**Sıfırdan tam bir Monad testnet full node kurulumu**  
*RAID yönetimi, TrieDB hazırlığı, snapshot import ve servis başlatma — adım adım.*

[![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04+-E95420?style=flat-square&logo=ubuntu&logoColor=white)](https://ubuntu.com)
[![Monad](https://img.shields.io/badge/Monad-Testnet-836EF9?style=flat-square)](https://monad.xyz)
[![Version](https://img.shields.io/badge/Node%20Version-v0.14.0-brightgreen?style=flat-square)](https://docs.monad.xyz)
[![Chain ID](https://img.shields.io/badge/Chain%20ID-10143-blue?style=flat-square)](https://docs.monad.xyz/developer-essentials/testnets)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)](LICENSE)

[hazennetworksolutions.com](https://hazennetworksolutions.com)

</div>

---

> **Yazar:** HazenNetworkSolutions  
> **Ağ:** Monad Testnet (Chain ID: 10143)  
> **Versiyon:** v0.14.0  
> **Son Güncelleme:** Nisan 2026

---

## İçindekiler

- [Donanım Gereksinimleri](#donanım-gereksinimleri)
- [Adım 1 — Sistem Kontrolü](#adım-1--sistem-kontrolü)
- [Adım 2 — Sistem Güncellemesi ve Bağımlılıklar](#adım-2--sistem-güncellemesi-ve-bağımlılıklar)
- [Adım 3 — Kernel Versiyonu Doğrulama](#adım-3--kernel-versiyonu-doğrulama)
- [Adım 4 — SMT (HyperThreading) Kapatma](#adım-4--smt-hyperthreading-kapatma)
- [Adım 5 — CPU Performance Modu ve File Descriptor Limitleri](#adım-5--cpu-performance-modu-ve-file-descriptor-limitleri)
- [Adım 6 — TrieDB Diskini Hazırlama](#adım-6--triedb-diskini-hazırlama)
- [Adım 7 — Monad Paketini Kurma](#adım-7--monad-paketini-kurma)
- [Adım 8 — Kullanıcı ve Dizin Yapısı](#adım-8--kullanıcı-ve-dizin-yapısı)
- [Adım 9 — Konfigürasyon Dosyalarını İndirme](#adım-9--konfigürasyon-dosyalarını-i̇ndirme)
- [Adım 10 — Keystore Şifresi Oluşturma](#adım-10--keystore-şifresi-oluşturma)
- [Adım 11 — BLS ve SECP Anahtarları Oluşturma](#adım-11--bls-ve-secp-anahtarları-oluşturma)
- [Adım 12 — node.toml Yapılandırması](#adım-12--nodetoml-yapılandırması)
- [Adım 13 — Remote Config URL'leri](#adım-13--remote-config-urleri)
- [Adım 14 — Firewall Ayarları](#adım-14--firewall-ayarları)
- [Adım 15 — Dosya İzinleri](#adım-15--dosya-i̇zinleri)
- [Adım 16 — TrieDB Formatlama](#adım-16--triedb-formatlama)
- [Adım 17 — Snapshot ile Başlangıç](#adım-17--snapshot-ile-başlangıç-hard-reset)
- [Adım 18 — Servisleri Başlatma](#adım-18--servisleri-başlatma)
- [Node İzleme](#node-i̇zleme)
- [Güncel Kalmak](#güncel-kalmak)

---

## Donanım Gereksinimleri

| Bileşen | Minimum |
|---|---|
| İşletim Sistemi | Ubuntu 24.04+ (bare-metal, VM değil) |
| CPU | 16 fiziksel çekirdek @ 4.5GHz (AMD Ryzen 7950X/9950X önerilir) |
| RAM | 32GB minimum (64GB önerilir) |
| Disk 1 | 2TB NVMe SSD (TrieDB için — ayrı, RAID olmayan) |
| Disk 2 | 500GB+ NVMe SSD (OS, loglar) |
| Kernel | Linux 6.8.0-60 veya üstü |

> ⚠️ SMT/HyperThreading BIOS'tan kapatılmalıdır.  
> ⚠️ TrieDB diski mount edilmemiş ve RAID yapılandırması olmayan ayrı bir disk olmalıdır.

---

## Adım 1 — Sistem Kontrolü

Sunucuya SSH ile bağlandıktan sonra aşağıdaki komutlarla sistemi doğrulayın:

```bash
lsb_release -a          # Ubuntu 24.04 olmalı
uname -r                # 6.8.0-60 veya üstü olmalı
lscpu | grep -E "Model name|CPU\(s\)|Thread|Socket|Core"
free -h                 # Minimum 32GB RAM
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,MODEL
```

---

## Adım 2 — Sistem Güncellemesi ve Bağımlılıklar

```bash
apt update && apt upgrade -y
apt install -y curl nvme-cli aria2 jq rsync parted cpufrequtils mdadm
```

Güncelleme sonrasında kernel güncellemesi yapıldıysa yeniden başlatın:

```bash
reboot
```

---

## Adım 3 — Kernel Versiyonu Doğrulama

```bash
uname -r
```

`6.8.0-60` veya üstü çıkmalıdır.

---

## Adım 4 — SMT (HyperThreading) Kapatma

Monad, performans için SMT'nin kapalı olmasını zorunlu tutar.

```bash
sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="\(.*\)"/GRUB_CMDLINE_LINUX_DEFAULT="\1 nosmt"/' /etc/default/grub
update-grub
reboot
```

Reboot sonrası doğrulayın:

```bash
cat /sys/devices/system/cpu/smt/active   # 0 çıkmalı
nproc                                    # 16 çıkmalı (Ryzen 7950X için)
```

---

## Adım 5 — CPU Performance Modu ve File Descriptor Limitleri

```bash
echo 'GOVERNOR="performance"' > /etc/default/cpufrequtils
systemctl restart cpufrequtils

echo "* soft nofile 65536" >> /etc/security/limits.conf
echo "* hard nofile 65536" >> /etc/security/limits.conf
```

---

## Adım 6 — TrieDB Diskini Hazırlama

> ⚠️ Yanlış diski formatlarsanız işletim sisteminizi kaybedebilirsiniz! Dikkatli olun.

### Mevcut diskleri listeleyin:

```bash
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,MODEL
```

Mount noktası olmayan, RAID'e dahil olmayan diski `TRIEDB_DRIVE` olarak belirleyin.

### Eğer disk RAID1 array'inin parçasıysa önce çıkarın:

```bash
# RAID durumunu kontrol edin
cat /proc/mdstat

# Diski array'lerden çıkarın (nvme1n1 örneği)
mdadm /dev/md2 --fail /dev/nvme1n1p4 && mdadm /dev/md2 --remove /dev/nvme1n1p4
mdadm /dev/md0 --fail /dev/nvme1n1p2 && mdadm /dev/md0 --remove /dev/nvme1n1p2
mdadm /dev/md1 --fail /dev/nvme1n1p3 && mdadm /dev/md1 --remove /dev/nvme1n1p3

# Reboot gerekebilir (root partition meşgulse)
reboot

# RAID metadata'sını silin
mdadm --zero-superblock /dev/nvme1n1p2
mdadm --zero-superblock /dev/nvme1n1p3
mdadm --zero-superblock /dev/nvme1n1p4

# Diski tamamen sıfırlayın
wipefs -a /dev/nvme1n1
```

### Partition oluşturun:

```bash
parted /dev/nvme1n1 mklabel gpt
parted /dev/nvme1n1 mkpart triedb 0% 100%
```

### udev rule oluşturun (Monad /dev/triedb symlink'ini kullanır):

```bash
PARTUUID=$(lsblk -o PARTUUID /dev/nvme1n1 | tail -n 1)
echo "Disk PartUUID: ${PARTUUID}"

echo "ENV{ID_PART_ENTRY_UUID}==\"$PARTUUID\", MODE=\"0666\", SYMLINK+=\"triedb\"" \
  | tee /etc/udev/rules.d/99-triedb.rules

udevadm trigger
udevadm control --reload
udevadm settle

ls -l /dev/triedb   # /dev/nvme1n1p1'e işaret etmeli
```

### LBA format doğrulayın (512 byte olmalı):

```bash
nvme id-ns -H /dev/nvme1n1 | grep 'LBA Format' | grep 'in use'
```

Beklenen çıktı: `Data Size: 512 bytes (in use)`  
Farklıysa düzeltin:

```bash
nvme format --lbaf=0 /dev/nvme1n1
```

---

## Adım 7 — Monad Paketini Kurma

APT repository'yi yapılandırın:

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

Paketi kurun:

```bash
apt update && apt install -y monad
```

Kurulumu doğrulayın:

```bash
monad --version
```

---

## Adım 8 — Kullanıcı ve Dizin Yapısı

```bash
useradd -m -s /bin/bash monad

mkdir -p /home/monad/monad-bft/config \
         /home/monad/monad-bft/ledger \
         /home/monad/monad-bft/config/forkpoint \
         /home/monad/monad-bft/config/validators
```

---

## Adım 9 — Konfigürasyon Dosyalarını İndirme

```bash
MF_BUCKET=https://bucket.monadinfra.com
curl -o /home/monad/.env $MF_BUCKET/config/testnet/latest/.env.example
curl -o /home/monad/monad-bft/config/node.toml $MF_BUCKET/config/testnet/latest/full-node-node.toml
```

---

## Adım 10 — Keystore Şifresi Oluşturma

```bash
sed -i "s|^KEYSTORE_PASSWORD=$|KEYSTORE_PASSWORD='$(openssl rand -base64 32)'|" /home/monad/.env
source /home/monad/.env

mkdir -p /opt/monad/backup/
echo "Keystore password: ${KEYSTORE_PASSWORD}" > /opt/monad/backup/keystore-password-backup
```

---

## Adım 11 — BLS ve SECP Anahtarları Oluşturma

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

> 🔐 **KRİTİK:** Aşağıdaki dosyaları dışarıya yedekleyin (şifre yöneticisi vb.):
> - `/opt/monad/backup/secp-backup`
> - `/opt/monad/backup/bls-backup`
> - `/opt/monad/backup/keystore-password-backup`

---

## Adım 12 — node.toml Yapılandırması

```bash
nano /home/monad/monad-bft/config/node.toml
```

Aşağıdaki alanları düzenleyin:

| Alan | Değer |
|---|---|
| `beneficiary` | `"0x0000000000000000000000000000000000000000"` (full node için) |
| `node_name` | `"full_PROVIDERNAME"` |

### Name Record İmzalama

Sunucunun public IP'sini alın:

```bash
curl -s4 ifconfig.me
```

İmzayı oluşturun:

```bash
source /home/monad/.env
monad-sign-name-record \
  --address <SUNUCU_IP>:8000 \
  --authenticated-udp-port 8001 \
  --keystore-path /home/monad/monad-bft/config/id-secp \
  --password "${KEYSTORE_PASSWORD}" \
  --self-record-seq-num 1
```

`node.toml` içindeki `[peer_discovery]` bölümünü güncelleyin:

```toml
self_address = "<SUNUCU_IP>:8000"
self_record_seq_num = 1
self_name_record_sig = "<ÇIKTI_IMZA>"
```

Ayrıca şu ayarların doğru olduğunu kontrol edin:
- `[fullnode_raptorcast]` altında `enable_client = true`
- `[statesync]` altında `expand_to_group = true`

---

## Adım 13 — Remote Config URL'leri

```bash
cat >> /home/monad/.env << 'EOF'
REMOTE_VALIDATORS_URL='https://bucket.monadinfra.com/validators/testnet/validators.toml'
REMOTE_FORKPOINT_URL='https://bucket.monadinfra.com/forkpoint/testnet/forkpoint.toml'
EOF
```

---

## Adım 14 — Firewall Ayarları

```bash
ufw allow ssh
ufw allow 8000
ufw allow 8001
ufw enable
ufw status
```

Spam koruması için iptables kuralı:

```bash
iptables -I INPUT -p udp --dport 8000 -m length --length 0:1400 -j DROP
```

> Not: Bu kural reboot'ta sıfırlanır. `iptables-persistent` ile kalıcı hale getirebilirsiniz.

---

## Adım 15 — Dosya İzinleri

```bash
chown -R monad:monad /home/monad/
```

---

## Adım 16 — TrieDB Formatlama

```bash
systemctl start monad-mpt
journalctl -u monad-mpt -n 14 -o cat
```

Başarılı çıktı şuna benzer:

```
MPT database on storages:
          Capacity           Used      %  Path
           1.75 Tb      256.03 Mb  0.01%  "/dev/nvme1n1p1"
...
monad-mpt.service: Deactivated successfully.
```

---

## Adım 17 — Snapshot ile Başlangıç (Hard Reset)

```bash
bash /opt/monad/scripts/reset-workspace.sh
```

Snapshot'ı indirin ve import edin:

```bash
MF_BUCKET=https://bucket.monadinfra.com
curl -sSL $MF_BUCKET/scripts/testnet/restore-from-snapshot.sh | bash
```

Yaklaşık 5 dakika sürer. Şunu görünce tamamdır:
```
Snapshot imported at block ID: XXXXXXXX
```

---

## Adım 18 — Servisleri Başlatma

```bash
systemctl enable monad-bft monad-execution monad-rpc
systemctl start monad-bft monad-execution monad-rpc
```

Durumu kontrol edin:

```bash
systemctl status monad-bft monad-execution monad-rpc --no-pager
```

3 servis de `active (running)` olmalıdır.

---

## Node İzleme

### Canlı blok takibi:

```bash
journalctl -u monad-bft -f --no-pager | grep "committed block"
```

Örnek çıktı:
```
"committed block","num_tx":4,"block_num":22917311
```

### Tüm loglar:

```bash
journalctl -u monad-bft -f --no-pager
journalctl -u monad-execution -f --no-pager
journalctl -u monad-rpc -f --no-pager
```

### Servis yönetimi:

```bash
# Yeniden başlat
systemctl restart monad-bft monad-execution monad-rpc

# Durdur
systemctl stop monad-bft monad-execution monad-rpc

# Servis durumu
systemctl status monad-bft monad-execution monad-rpc
```

---

## Güncel Kalmak

- Telegram: [Monad Node Announcements](https://t.me/MonadNodeAnnouncements)
- Discord: [Monad Developer Discord](https://discord.gg/monaddev)
- Resmi Dokümantasyon: [docs.monad.xyz](https://docs.monad.xyz/node-ops/full-node-installation)

---

## Yazar Hakkında

Bu rehber **HazenNetworkSolutions** tarafından hazırlanmıştır.  
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)

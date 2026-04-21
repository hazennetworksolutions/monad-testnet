<div align="center">

# 🟣 Monad Testnet — Operation Guide (TR)

**Hızlı kurulum, recovery ve güncelleme kılavuzu**  
*Sıfırdan kurulum scriptleri, acil recovery prosedürleri ve versiyon güncelleme.*

[![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04+-E95420?style=flat-square&logo=ubuntu&logoColor=white)](https://ubuntu.com)
[![Monad](https://img.shields.io/badge/Monad-Testnet-836EF9?style=flat-square)](https://monad.xyz)
[![Version](https://img.shields.io/badge/Node%20Version-v0.14.1-brightgreen?style=flat-square)](https://docs.monad.xyz)
[![Chain ID](https://img.shields.io/badge/Chain%20ID-10143-blue?style=flat-square)](https://docs.monad.xyz/developer-essentials/testnets)

[hazennetworksolutions.com](https://hazennetworksolutions.com)

</div>

---

> **Yazar:** HazenNetworkSolutions  
> **Ağ:** Monad Testnet (Chain ID: 10143)  
> **Versiyon:** v0.14.1  
> **Son Güncelleme:** Nisan 2026  
> **Resmi Docs:** [docs.monad.xyz](https://docs.monad.xyz/node-ops)

---

## İçindekiler

- [Donanım Gereksinimleri](#donanım-gereksinimleri)
- [Bölüm 1 — Acil Recovery](#bölüm-1--acil-recovery)
- [Bölüm 2 — Versiyon Güncelleme](#bölüm-2--versiyon-güncelleme)
- [Bölüm 3 — İzleme ve Tanılama](#bölüm-3--i̇zleme-ve-tanılama)
- [Bölüm 4 — Sıfırdan Kurulum: Faz 1A (Reboot Öncesi)](#bölüm-4--sıfırdan-kurulum-faz-1a-reboot-öncesi)
- [Bölüm 5 — Sıfırdan Kurulum: Faz 1B (Reboot Sonrası)](#bölüm-5--sıfırdan-kurulum-faz-1b-reboot-sonrası)
- [Bölüm 6 — Sıfırdan Kurulum: Faz 2](#bölüm-6--sıfırdan-kurulum-faz-2)
- [Bölüm 7 — node.toml İmzalama ve Son Adımlar](#bölüm-7--nodetoml-i̇mzalama-ve-son-adımlar)
- [Bölüm 8 — Kritik Yedekleme](#bölüm-8--kritik-yedekleme)

---

## Donanım Gereksinimleri

| Bileşen | Minimum |
|---|---|
| İşletim Sistemi | Ubuntu 24.04+ (bare-metal, VM değil) |
| CPU | 16 fiziksel çekirdek @ 4.5GHz (AMD Ryzen 7950X/9950X önerilir) |
| RAM | 32GB minimum (64GB önerilir) |
| Disk 1 (TrieDB) | 2TB NVMe SSD — ayrı, RAID olmayan |
| Disk 2 (OS) | 500GB+ NVMe SSD |
| Kernel | Linux 6.8.0-60 veya üstü |

> ⚠️ SMT/HyperThreading BIOS'tan kapatılmalıdır.  
> ⚠️ TrieDB diski RAID yapılandırması ve mount noktası olmayan ayrı bir disk olmalıdır.

---

## Bölüm 1 — Acil Recovery

### 1.1 Soft Reset

**Ne zaman:** Node kısa süre durdu veya yeniden başlatma gerekiyor.

```bash
systemctl restart monad-bft monad-execution monad-rpc
```

Doğrula:
```bash
systemctl status monad-bft monad-execution monad-rpc --no-pager | grep Active
journalctl -u monad-bft -f --no-pager | grep "committed block"
```

> v0.12.1+ sonrası `.env`'de `REMOTE_VALIDATORS_URL` ve `REMOTE_FORKPOINT_URL` tanımlıysa restart yeterlidir — forkpoint ve validators.toml otomatik çekilir.

---

### 1.2 Hard Reset

**Ne zaman:** Node çok geride kaldı, statesync yetişemiyor veya state bozuk.

```bash
# 1. Servisleri durdur
systemctl stop monad-bft monad-execution monad-rpc

# 2. Tüm runtime data'yı sil
bash /opt/monad/scripts/reset-workspace.sh

# 3. Snapshot indir ve import et (~5 dakika)
MF_BUCKET=https://bucket.monadinfra.com
curl -sSL $MF_BUCKET/scripts/testnet/restore-from-snapshot.sh | bash

# 4. Servisleri başlat
systemctl start monad-bft monad-execution monad-rpc
```

Snapshot çalışmazsa alternatif sağlayıcı:
```bash
CL_BUCKET=https://pub-b0d0d7272c994851b4c8af22a766f571.r2.dev
curl -sSL $CL_BUCKET/scripts/testnet/restore_from_snapshot.sh | bash
```

> ⚠️ Snapshot restore tamamlanmadan servisleri başlatma.

---

### 1.3 Sunucu Değişikliği (Key Göçü)

**Ne zaman:** Sunucu erişilemez, yeni makineye geçiş gerekiyor.

1. Yeni sunucuya **Faz 1A → Faz 1B → Faz 2** scriptlerini çalıştır
2. Faz 2'de `SECP_IKM` ve `BLS_IKM` değişkenlerini tanımla — key üretilmez, backup'tan yüklenir
3. `SELF_RECORD_SEQ_NUM` değerini **bir önceki değer + 1** yap
4. node.toml imzala ve servisleri başlat

> Validator kimliği key'lere bağlıdır. Key'ler aynı kaldığı sürece Validator ID ve delegation korunur.

---

## Bölüm 2 — Versiyon Güncelleme

Yeni versiyon duyuruları:
- Telegram: [Monad Node Announcements](https://t.me/MonadNodeAnnouncements)
- Discord: `#mainnet-fullnode-announcements`

```bash
# Yeni versiyonu buraya yaz
VERSION="0.14.1"

apt update && apt install --reinstall monad=${VERSION} -y \
  --allow-downgrades --allow-change-held-packages

systemctl restart monad-bft monad-execution monad-rpc

# Doğrula
monad-rpc --version
systemctl status monad-bft monad-execution monad-rpc --no-pager | grep Active
```

> Release notlarını mutlaka oku. `node.toml` değişikliği varsa ilgili alanları güncelle.

---

## Bölüm 3 — İzleme ve Tanılama

### Servis Durumu
```bash
systemctl status monad-bft monad-execution monad-rpc --no-pager | grep -E "Active|Main PID"
```

### Canlı Blok Takibi
```bash
journalctl -u monad-bft -f --no-pager | grep "committed block"
```

### Blok Yüksekliği (RPC)
```bash
curl -s http://localhost:8080/ -X POST -H "Content-Type: application/json" \
  --data '{"method":"eth_blockNumber","params":[],"id":1,"jsonrpc":"2.0"}' \
  | jq -r '.result' | xargs printf "%d\n"
```

### monad-status (Kapsamlı Durum)
```bash
# Bir kere kur
curl -sSL https://bucket.monadinfra.com/scripts/monad-status.sh \
  -o /usr/local/bin/monad-status && chmod +x /usr/local/bin/monad-status

# Çalıştır
monad-status
```

### monlog (BFT Log Analizi)
```bash
# monad kullanıcısına journal erişimi ver (root olarak)
usermod -a -G systemd-journal monad

# monad kullanıcısı olarak kur ve çalıştır
su - monad
cd /home/monad
curl -sSL https://pub-b0d0d7272c994851b4c8af22a766f571.r2.dev/scripts/monlog -O
chmod u+x ./monlog
./monlog           # Tek seferlik
watch -d "./monlog"  # Canlı izleme
```

### TrieDB Disk Doluluk
```bash
monad-mpt --storage /dev/triedb
```

> TrieDB %80 dolunca otomatik compact yapar. Kapasiteyi düzenli izle.

### Log İzleme
```bash
journalctl -u monad-bft -f --no-pager
journalctl -u monad-execution -f --no-pager
journalctl -u monad-rpc -f --no-pager
```

---

## Bölüm 4 — Sıfırdan Kurulum: Faz 1A (Reboot Öncesi)

> Taze Ubuntu 24.04 sunucuya uygulanır.  
> Bu script sistem hazırlığını yapar ve RAID diskini faulty olarak işaretler.  
> **Script'in sonunda `reboot` komutu çalışır.**

**Bu script ne yapar:**
- Sistem güncelleme ve gerekli bağımlılıkları kurar
- Kernel versiyonunu kontrol eder
- SMT/HyperThreading'i GRUB üzerinden kapatır (`nosmt`)
- CPU performance modunu ve file descriptor limitlerini ayarlar
- TrieDB için kullanılacak diski RAID array'lerinden faulty olarak işaretler
- Sistemi yeniden başlatır

```bash
bash <(cat <<'FAZ1A'
#!/bin/bash
set -euo pipefail
echo "=== FAZ 1A: Sistem Hazırlama ==="

# ── 1. Sistem Güncelleme ve Bağımlılıklar ────────────────────────
echo "[1/6] Sistem güncelleniyor..."
apt update && apt upgrade -y
apt install -y curl nvme-cli aria2 jq rsync parted cpufrequtils mdadm

# ── 2. Kernel Kontrolü ───────────────────────────────────────────
KERNEL=$(uname -r)
echo "[2/6] Mevcut kernel: $KERNEL"
echo "      Gereken: 6.8.0-60 veya üstü"

# ── 3. SMT (HyperThreading) Kapatma ──────────────────────────────
echo "[3/6] SMT kapatılıyor (GRUB nosmt)..."
if ! grep -q "nosmt" /etc/default/grub; then
  sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="\(.*\)"/GRUB_CMDLINE_LINUX_DEFAULT="\1 nosmt"/' \
    /etc/default/grub
  update-grub
  echo "  ✅ nosmt eklendi — reboot sonrası aktif olacak."
else
  echo "  ✅ nosmt zaten mevcut."
fi

# ── 4. CPU Performance Modu ve File Descriptor Limitleri ─────────
echo "[4/6] Performans ayarları yapılıyor..."
echo 'GOVERNOR="performance"' > /etc/default/cpufrequtils
systemctl restart cpufrequtils 2>/dev/null || true

grep -q "nofile 65536" /etc/security/limits.conf || {
  echo "* soft nofile 65536" >> /etc/security/limits.conf
  echo "* hard nofile 65536" >> /etc/security/limits.conf
}
grep -q "DefaultLimitNOFILE" /etc/systemd/system.conf || \
  echo "DefaultLimitNOFILE=65536" >> /etc/systemd/system.conf

# ── 5. TrieDB Disk Seçimi ve RAID'den Çıkarma ────────────────────
echo "[5/6] Diskler listeleniyor..."
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,MODEL
echo ""
echo "RAID'den çıkarılacak ve TrieDB olarak kullanılacak disk? (örn: nvme1n1)"
echo "⚠️  DİKKAT: Yanlış disk seçimi veri kaybına yol açar!"
read -r TRIEDB_DRIVE
TRIEDB_DRIVE=${TRIEDB_DRIVE:-nvme1n1}

echo "Mevcut RAID durumu:"
cat /proc/mdstat
echo ""
echo "  → /dev/$TRIEDB_DRIVE tüm array'lerde faulty işaretleniyor..."

for md in /dev/md0 /dev/md1 /dev/md2 /dev/md3 /dev/md4; do
  [ -b "$md" ] || continue
  for part in 1 2 3 4; do
    DEV="/dev/${TRIEDB_DRIVE}p${part}"
    [ -b "$DEV" ] || continue
    echo "    mdadm $md --fail $DEV"
    mdadm "$md" --fail "$DEV" 2>/dev/null || true
    # Root partition reboot öncesi kaldırılamaz (busy), faulty işaretlemek yeterli
    mdadm "$md" --remove "$DEV" 2>/dev/null || true
  done
done

# Disk adını kaydet — Faz 1B'de kullanılacak
echo "TRIEDB_DRIVE=$TRIEDB_DRIVE" > /root/.monad_setup_vars
echo "  ✅ Disk faulty işaretlendi. Reboot sonrası Faz 1B çalıştırılacak."

# ── 6. Reboot ────────────────────────────────────────────────────
echo "[6/6] Sistem yeniden başlatılıyor..."
echo ""
echo "┌─────────────────────────────────────────────────┐"
echo "│  FAZ 1A TAMAMLANDI — REBOOT BAŞLIYOR           │"
echo "│                                                 │"
echo "│  Reboot sonrası: FAZ 1B scriptini çalıştır     │"
echo "│  (Bölüm 5)                                     │"
echo "└─────────────────────────────────────────────────┘"
sleep 3
reboot

FAZ1A
)
```

---

## Bölüm 5 — Sıfırdan Kurulum: Faz 1B (Reboot Sonrası)

> Faz 1A reboot'undan sonra çalıştırılır.  
> RAID'i tamamen temizler, TrieDB partition oluşturur ve udev symlink kurar.

**Bu script ne yapar:**
- SMT'nin kapandığını doğrular
- RAID metadata'sını diskten tamamen siler (`zero-superblock` + `wipefs`)
- LBA format kontrolü yapar (512 byte olmalı)
- GPT partition ve `/dev/triedb` symlink'i oluşturur

```bash
bash <(cat <<'FAZ1B'
#!/bin/bash
set -euo pipefail
echo "=== FAZ 1B: TrieDB Disk Hazırlama ==="

# Faz 1A'dan kayıt oku
if [ -f /root/.monad_setup_vars ]; then
  source /root/.monad_setup_vars
else
  echo "⚠️  /root/.monad_setup_vars bulunamadı."
  lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,MODEL
  echo "TrieDB diski? (örn: nvme1n1)"
  read -r TRIEDB_DRIVE
fi
echo "  TrieDB diski: /dev/$TRIEDB_DRIVE"

# ── 1. SMT Doğrulama ─────────────────────────────────────────────
echo "[1/5] SMT kontrolü..."
SMT=$(cat /sys/devices/system/cpu/smt/active 2>/dev/null || echo "?")
CORES=$(nproc)
if [ "$SMT" = "0" ]; then
  echo "  ✅ SMT kapalı | CPU core sayısı: $CORES"
else
  echo "  ⚠️  SMT hâlâ aktif (değer: $SMT) — BIOS'tan kapatıldığından emin ol."
fi

# ── 2. Kernel Doğrulama ──────────────────────────────────────────
echo "[2/5] Kernel: $(uname -r)"

# ── 3. RAID Metadata Temizleme ───────────────────────────────────
echo "[3/5] RAID metadata temizleniyor..."

# Kalan array üyeliklerini kaldır
for md in /dev/md0 /dev/md1 /dev/md2 /dev/md3 /dev/md4; do
  [ -b "$md" ] || continue
  for part in 1 2 3 4; do
    DEV="/dev/${TRIEDB_DRIVE}p${part}"
    [ -b "$DEV" ] || continue
    mdadm "$md" --remove "$DEV" 2>/dev/null || true
  done
done

# Superblock sil
for part in 1 2 3 4; do
  DEV="/dev/${TRIEDB_DRIVE}p${part}"
  if [ -b "$DEV" ]; then
    echo "  → zero-superblock: $DEV"
    mdadm --zero-superblock "$DEV" 2>/dev/null || true
  fi
done

# Tüm imzaları sil
echo "  → wipefs: /dev/$TRIEDB_DRIVE"
wipefs -a /dev/$TRIEDB_DRIVE

# ── 4. LBA Format Kontrolü ───────────────────────────────────────
echo "[4/5] LBA format kontrolü..."
LBA_INFO=$(nvme id-ns -H /dev/$TRIEDB_DRIVE 2>/dev/null | grep -i "LBA Format" | grep -i "in use" || echo "")
if echo "$LBA_INFO" | grep -q "512"; then
  echo "  ✅ LBA format: 512 bytes (doğru)"
elif [ -n "$LBA_INFO" ]; then
  echo "  ⚠️  LBA format 512 byte değil. Düzeltiliyor..."
  nvme format --lbaf=0 /dev/$TRIEDB_DRIVE
  echo "  ✅ LBA format düzeltildi."
else
  echo "  ℹ️  LBA format bilgisi alınamadı, devam ediliyor."
fi

# ── 5. Partition Oluşturma ve udev Rule ──────────────────────────
echo "[5/5] Partition ve udev symlink oluşturuluyor..."
parted /dev/$TRIEDB_DRIVE mklabel gpt
parted /dev/$TRIEDB_DRIVE mkpart triedb 0% 100%
sleep 2
udevadm settle

PARTUUID=$(lsblk -o PARTUUID /dev/${TRIEDB_DRIVE}p1 2>/dev/null | grep -v PARTUUID | tr -d ' ')
if [ -z "$PARTUUID" ]; then
  echo "  ⚠️  PARTUUID alınamadı. lsblk çıktısını kontrol et:"
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

# Kayıt güncelle
echo "TRIEDB_DRIVE=$TRIEDB_DRIVE" > /root/.monad_setup_vars

# ── Doğrulama ────────────────────────────────────────────────────
echo ""
if ls -l /dev/triedb &>/dev/null; then
  echo "┌──────────────────────────────────────────────────┐"
  echo "│  FAZ 1B TAMAMLANDI ✅                            │"
  echo "│                                                  │"
  echo "│  /dev/triedb hazır:"
  ls -l /dev/triedb | sed 's/^/│  /'
  echo "│                                                  │"
  echo "│  SIRADA: FAZ 2 scriptini çalıştır (Bölüm 6)    │"
  echo "└──────────────────────────────────────────────────┘"
else
  echo "┌──────────────────────────────────────────────────┐"
  echo "│  ⚠️  /dev/triedb oluşmadı!                       │"
  echo "│  udev kuralını ve PARTUUID'i kontrol et.         │"
  echo "└──────────────────────────────────────────────────┘"
  exit 1
fi

FAZ1B
)
```

---

## Bölüm 6 — Sıfırdan Kurulum: Faz 2

> Faz 1B başarıyla tamamlandıktan sonra çalıştırılır.  
> Monad kurulumu, kullanıcı ve dizin yapısı, keystore, node.toml ve firewall.

### 6.1 Değişkenleri Tanımla

Script'i çalıştırmadan önce aşağıdakileri doldur ve çalıştır:

```bash
# Zorunlu
export NODE_NAME="full_PROVIDERNAME"
# Örn: export NODE_NAME="full_HazenNetworkSolutions"

export BENEFICIARY="0x0000000000000000000000000000000000000000"
# Full node için burn adresi kullanılır.
# Validator olunca kendi EOA adresini yaz.

export SELF_RECORD_SEQ_NUM=1
# İlk kurulumda 1. Her sunucu göçünde +1 artır.

# Mevcut key'leri restore edeceksen (sunucu göçü):
# IKM değerlerini backup dosyalarındaki "Keystore secret:" satırından al
# export SECP_IKM="306c9aa1..."
# export BLS_IKM="8ec77aeb..."
```

### 6.2 Faz 2 Script

```bash
bash <(cat <<'FAZ2'
#!/bin/bash
set -euo pipefail
echo "=== FAZ 2: Monad Kurulumu ==="

# Değişken kontrolü
: "${NODE_NAME:?Hata: NODE_NAME tanımlı değil}"
: "${BENEFICIARY:?Hata: BENEFICIARY tanımlı değil}"
: "${SELF_RECORD_SEQ_NUM:?Hata: SELF_RECORD_SEQ_NUM tanımlı değil}"

# ── 1. SMT Doğrulama ─────────────────────────────────────────────
echo "[1/9] SMT kontrolü..."
SMT=$(cat /sys/devices/system/cpu/smt/active 2>/dev/null || echo "?")
[ "$SMT" = "0" ] && echo "  ✅ SMT kapalı — $(nproc) core" \
  || echo "  ⚠️  SMT aktif (değer: $SMT)"

# ── 2. Monad APT Repo ve Paket ───────────────────────────────────
echo "[2/9] Monad APT repo kuruluyor..."
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
echo "  ✅ Monad kuruldu: $(monad-rpc --version 2>&1 | grep -o 'v[0-9.]*' | head -1)"

# ── 3. Kullanıcı ve Dizin Yapısı ─────────────────────────────────
echo "[3/9] monad kullanıcısı ve dizinler oluşturuluyor..."
id monad &>/dev/null || useradd -m -s /bin/bash monad
mkdir -p \
  /home/monad/monad-bft/config \
  /home/monad/monad-bft/ledger \
  /home/monad/monad-bft/config/forkpoint \
  /home/monad/monad-bft/config/validators \
  /opt/monad/backup

# ── 4. Konfigürasyon Dosyaları ───────────────────────────────────
echo "[4/9] Config dosyaları indiriliyor..."
MF_BUCKET=https://bucket.monadinfra.com
curl -fsSL -o /home/monad/.env \
  $MF_BUCKET/config/testnet/latest/.env.example
curl -fsSL -o /home/monad/monad-bft/config/node.toml \
  $MF_BUCKET/config/testnet/latest/full-node-node.toml
echo "  ✅ .env ve node.toml indirildi."

# ── 5. Keystore Şifresi ──────────────────────────────────────────
echo "[5/9] Keystore şifresi oluşturuluyor..."
KP="$(openssl rand -base64 32)"

# .env içindeki KEYSTORE_PASSWORD= satırını bul ve doldur
if grep -q "^KEYSTORE_PASSWORD=" /home/monad/.env; then
  sed -i "s|^KEYSTORE_PASSWORD=.*|KEYSTORE_PASSWORD='${KP}'|" /home/monad/.env
else
  echo "KEYSTORE_PASSWORD='${KP}'" >> /home/monad/.env
fi

# Remote config URL'leri ekle (v0.12.1+ otomatik çekme için)
grep -q "REMOTE_VALIDATORS_URL" /home/monad/.env || cat >> /home/monad/.env <<'EOF'
REMOTE_VALIDATORS_URL='https://bucket.monadinfra.com/validators/testnet/validators.toml'
REMOTE_FORKPOINT_URL='https://bucket.monadinfra.com/forkpoint/testnet/forkpoint.toml'
EOF

# Şifreyi yedekle
echo "Keystore password: ${KP}" > /opt/monad/backup/keystore-password-backup
echo "  ✅ Keystore şifresi: $KP"
echo "  ✅ Yedek: /opt/monad/backup/keystore-password-backup"

source /home/monad/.env

# ── 6. Key Oluşturma veya Restore ────────────────────────────────
echo "[6/9] Key'ler ayarlanıyor..."

if [ -n "${SECP_IKM:-}" ] && [ -n "${BLS_IKM:-}" ]; then
  echo "  → IKM'den key restore ediliyor..."

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

  # Restore yolunda: recover ile backup dosyaları oluştur
  monad-keystore recover \
    --password "$KEYSTORE_PASSWORD" \
    --keystore-path /home/monad/monad-bft/config/id-secp \
    --key-type secp > /opt/monad/backup/secp-backup

  monad-keystore recover \
    --password "$KEYSTORE_PASSWORD" \
    --keystore-path /home/monad/monad-bft/config/id-bls \
    --key-type bls > /opt/monad/backup/bls-backup

  echo "  ✅ Key'ler restore edildi."
else
  echo "  → Yeni key'ler oluşturuluyor..."

  # create çıktısı IKM (Keystore secret) dahil tüm bilgiyi içerir — doğrudan backup'a yaz
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

  echo "  ✅ Key'ler oluşturuldu."
fi

# Public key'leri kaydet ve göster
grep "public key" /opt/monad/backup/secp-backup \
                  /opt/monad/backup/bls-backup \
  | tee /home/monad/pubkey-secp-bls || true
echo "  ✅ Backup dosyaları hazır."

# ── 7. node.toml Temel Ayarlar ───────────────────────────────────
echo "[7/9] node.toml ayarlanıyor..."
sed -i "s|beneficiary = \".*\"|beneficiary = \"${BENEFICIARY}\"|" \
  /home/monad/monad-bft/config/node.toml
sed -i "s|node_name = \".*\"|node_name = \"${NODE_NAME}\"|" \
  /home/monad/monad-bft/config/node.toml
echo "  ✅ beneficiary ve node_name ayarlandı."

# ── 8. Firewall ──────────────────────────────────────────────────
echo "[8/9] Firewall ayarlanıyor..."
ufw allow ssh
ufw allow 8000
ufw allow 8001
ufw --force enable
# UDP spam koruması (reboot'ta sıfırlanır; kalıcı yapmak için iptables-persistent kur)
iptables -I INPUT -p udp --dport 8000 -m length --length 0:1400 -j DROP
echo "  ✅ UFW aktif. Portlar: 22 (SSH), 8000, 8001"
echo "  ℹ️  iptables kuralı reboot'ta sıfırlanır. Kalıcı için: apt install iptables-persistent"

# ── 9. Dosya İzinleri ────────────────────────────────────────────
echo "[9/9] Dosya izinleri ayarlanıyor..."
chown -R monad:monad /home/monad/
echo "  ✅ /home/monad izinleri ayarlandı."

# SELF_RECORD_SEQ_NUM'u kaydet
echo "SELF_RECORD_SEQ_NUM=${SELF_RECORD_SEQ_NUM}" >> /root/.monad_setup_vars

echo ""
echo "┌──────────────────────────────────────────────────────────┐"
echo "│  FAZ 2 TAMAMLANDI ✅                                     │"
echo "│                                                          │"
echo "│  SIRADAKI ADIMLAR (Bölüm 7):                            │"
echo "│  1. node.toml [peer_discovery] bölümünü imzala          │"
echo "│  2. TrieDB formatla                                      │"
echo "│  3. Snapshot yükle                                       │"
echo "│  4. Servisleri başlat                                    │"
echo "└──────────────────────────────────────────────────────────┘"

FAZ2
)
```

---

## Bölüm 7 — node.toml İmzalama ve Son Adımlar

### 7.1 node.toml [peer_discovery] Doldurma

```bash
# Sunucunun public IP'sini al
SERVER_IP=$(curl -s4 ifconfig.me)
echo "Sunucu IP: $SERVER_IP"

# self_record_seq_num değerini kontrol et
source /root/.monad_setup_vars 2>/dev/null || true
echo "Seq num: ${SELF_RECORD_SEQ_NUM:-1}"

# İmza oluştur
source /home/monad/.env
monad-sign-name-record \
  --address ${SERVER_IP}:8000 \
  --authenticated-udp-port 8001 \
  --keystore-path /home/monad/monad-bft/config/id-secp \
  --password "${KEYSTORE_PASSWORD}" \
  --self-record-seq-num ${SELF_RECORD_SEQ_NUM:-1}
```

Çıkan 3 satırı `node.toml` dosyasının `[peer_discovery]` bölümüne yaz:

```bash
nano /home/monad/monad-bft/config/node.toml
```

```toml
self_address = "<SUNUCU_IP>:8000"
self_record_seq_num = 1
self_name_record_sig = "<IMZA_HEX>"
```

Ayrıca şu ayarların doğru olduğunu kontrol et:
```toml
# [fullnode_raptorcast] bölümünde:
enable_client = true

# [statesync] bölümünde:
expand_to_group = true
```

### 7.2 TrieDB Formatlama

```bash
systemctl start monad-mpt
sleep 10
journalctl -u monad-mpt -n 20 -o cat
```

Beklenen çıktı sonunda şu satırlar görünmeli:
```
MPT database on storages:
          Capacity           Used      %  Path
           X.XX Tb      ...Mb  0.01%  "/dev/..."
...
monad-mpt.service: Deactivated successfully.
```

### 7.3 Snapshot Yükleme

```bash
# Workspace temizle
bash /opt/monad/scripts/reset-workspace.sh

# Snapshot indir ve import et (~5 dakika)
MF_BUCKET=https://bucket.monadinfra.com
curl -sSL $MF_BUCKET/scripts/testnet/restore-from-snapshot.sh | bash
```

Başarılı çıktı:
```
Snapshot imported at block ID: XXXXXXXX
```

### 7.4 Servisleri Başlatma

```bash
systemctl enable monad-bft monad-execution monad-rpc
systemctl start monad-bft monad-execution monad-rpc

# Durum kontrolü
sleep 5
systemctl status monad-bft monad-execution monad-rpc --no-pager | grep Active
```

3 servis de `active (running)` olmalı.

### 7.5 Canlı Doğrulama

```bash
# Blok takibi — birkaç saniye içinde committed block görünmeli
journalctl -u monad-bft -f --no-pager | grep "committed block"
```

---

## Bölüm 8 — Kritik Yedekleme

> ⚠️ Kurulum tamamlandıktan hemen sonra bu dosyaları sunucu dışında güvenli bir yere yedekle.  
> Bu dosyalar olmadan node kimliğini ve validator delegation'ını kurtaramazsın.

| Dosya | Sunucu Konumu | İçerik |
|---|---|---|
| `id-secp` | `/home/monad/monad-bft/config/id-secp` | Node kimliği (SECP keystore) |
| `id-bls` | `/home/monad/monad-bft/config/id-bls` | Validator imzalama (BLS keystore) |
| `keystore-password-backup` | `/opt/monad/backup/keystore-password-backup` | Keystore şifresi |
| `secp-backup` | `/opt/monad/backup/secp-backup` | SECP key + IKM |
| `bls-backup` | `/opt/monad/backup/bls-backup` | BLS key + IKM |
| `node.toml` | `/home/monad/monad-bft/config/node.toml` | Node konfigürasyonu |

Mac/PC'ye indirmek için:

```bash
SUNUCU_IP="SUNUCU_IP_BURAYA"

scp root@${SUNUCU_IP}:/home/monad/monad-bft/config/id-secp ./
scp root@${SUNUCU_IP}:/home/monad/monad-bft/config/id-bls ./
scp root@${SUNUCU_IP}:/opt/monad/backup/keystore-password-backup ./
scp root@${SUNUCU_IP}:/opt/monad/backup/secp-backup ./
scp root@${SUNUCU_IP}:/opt/monad/backup/bls-backup ./
scp root@${SUNUCU_IP}:/home/monad/monad-bft/config/node.toml ./
```

---

## Güncel Kalmak

- Telegram: [Monad Node Announcements](https://t.me/MonadNodeAnnouncements)
- Discord: [Monad Developer Discord](https://discord.gg/monaddev)
- Resmi Docs: [docs.monad.xyz/node-ops](https://docs.monad.xyz/node-ops)

---

<div align="center">

**HazenNetworkSolutions**  
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)

*Monad v0.14.1 — Nisan 2026*

</div>

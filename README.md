# vpptc — VPP Simple Policer YAML Synchronizer

`vpptc` menyinkronkan konfigurasi **simple-policer** VPP dari file YAML ke runtime VPP,
lalu menulis ulang file CLI startup agar konfigurasi tetap bisa dimuat ulang.

Tool ini bersifat **deklaratif**: YAML adalah sumber kebenaran. Yang tidak ada di YAML
akan dilepas dari VPP — termasuk oleh `vpptc` sendiri.

```
/etc/vpp/vpptc.yaml  ──►  vpptc  ──►  vppctl (runtime VPP)
                              └────►  /etc/vpp/vpptc.vpp (untuk persistensi saat boot)
```

---

## Daftar Isi

- [Kebutuhan Sistem](#kebutuhan-sistem)
- [Instalasi](#instalasi)
- [Konfigurasi YAML](#konfigurasi-yaml)
- [Pemakaian](#pemakaian)
- [Cara Kerja](#cara-kerja)
- [Konvensi Nama Policer](#konvensi-nama-policer)
- [Keamanan](#keamanan)
- [Exit Code](#exit-code)
- [Pemecahan Masalah](#pemecahan-masalah)
- [Catatan Operasional](#catatan-operasional)

---

## Kebutuhan Sistem

| Komponen | Versi / Keterangan |
|---|---|
| Python | 3.6+ (diuji 3.11.2) |
| PyYAML | 6.0 |
| `vppctl` | VPP 20.05+ (diuji v26.06-release) |
| Akses | `vppctl` harus terhubung ke daemon VPP (umumnya perlu root) |
| Lock | `/run/lock/vpptc.lock` dimiliki `root:vpp` mode `0660`; user harus anggota grup `vpp` |

Hanya dependensi standar + PyYAML. Tidak ada library pihak ketiga lain.

---

## Instalasi

```bash
# 1. salin tool ke lokasi permanen
install -m 755 vpptc /usr/local/sbin/vpptc

# 2. pastikan dependensi ada
python3 -c "import yaml" || pip3 install pyyaml

# 3. cek versi
/usr/local/sbin/vpptc --help
```

### Izin lock (grup `vpp`)

`vpptc apply` memakai file lock `/run/lock/vpptc.lock`. File ini harus dimiliki
`root:vpp` dengan mode `0660`, dan user yang menjalankan `vpptc apply` harus menjadi
anggota grup `vpp`.

```bash
# aturan tmpfiles agar izin tetap benar setelah reboot (/run berbasis tmpfs)
cat > /etc/tmpfiles.d/vpptc.conf <<'EOF'
f /run/lock/vpptc.lock 0660 root vpp -
EOF
systemd-tmpfiles --create /etc/tmpfiles.d/vpptc.conf

# tambahkan user yang berhak menjalankan vpptc apply ke grup vpp
usermod -aG vpp <user>
```

Bila lock tidak bisa dibuat/diakses, `vpptc apply` **berhenti dengan exit code `3`** —
tidak pernah melanjutkan tanpa proteksi. `plan` dan `check` tidak memerlukan lock.

> **Penting:** jangan menaruh script di direktori yang tidak bisa diakses user lain
> (mis. `/root`) bilahendak dijalankan oleh service/cron non-root. `/root` mode `0700`.

Buat file konfigurasi pertama:

```bash
cat > /etc/vpp/vpptc.yaml <<'EOF'
simple_policers:
  - interface: "TenGigabitEthernet1/0/0"
    input:
      cir: "10m"
      cb: "100k"
    output:
      cir: 5000
      cb: 64000
EOF
```

---

## Konfigurasi YAML

### Struktur

```yaml
simple_policers:
  - interface: "<nama-interface>"     # wajib
    input:                            # opsional
      cir: "<nilai>"                  # wajib bila blok input ada
      cb: "<nilai>"                   # wajib bila blok input ada
    output:                           # opsional
      cir: "<nilai>"
      cb: "<nilai>"
```

### Aturan

| Key | Wajib | Keterangan |
|---|---|---|
| `simple_policers` | ya | List. `[]` berarti tidak ada policer sama sekali. |
| `interface` | ya | Nama interface VPP. Harus berupa teks dan ada di runtime VPP. |
| `input` / `output` | tidak | Maksimal **satu** per arah (lihat [batasan](#batasan-penting)). |
| `cir` | ya (bila arah ada) | *Committed Information Rate* dalam kbps. |
| `cb` | ya (bila arah ada) | *Committed Burst* dalam byte. |

Key yang tidak dikenali **ditolak**, baik di level root maupun level enti. Salah eja tidak
akan lolos diam-diam.

### Satuan yang didukung

| `cir` (kbps) | `cb` (byte) |
|---|---|
| `10m`, `10M`, `10 mbps` | `100k`, `100KB`, `100 kbytes` |
| `5000`, `5000k`, `5000kbps` | `64000`, `64000b`, `64000 bytes` |
| `1g`, `1000mbps` | `3g`, `3gb`, `3gbytes` |

Nilai boleh berupa angka (`cir: 5000`) maupun teks (`cir: "10m"`). Huruf besar/kecil dan
spasi diabaikan: `" 10 M "` = `10m`.

Nilai desimal dibulatkan **half-up**, bukan dipotong:
`1.9k` → `2`, `0.5m` → `500`, `2.4k` → `2400`.

### Nilai `0` = tanpa limitasi

`cir: 0` **atau** `cb: 0` berarti arah tersebut **tidak dibatasi**, sehingga policer
**tidak dibuat sama sekali**. Policer lama di arah tersebut otomatis dilepas.

```yaml
simple_policers:
  - interface: "TenGigabitEthernet1/0/0"
    input:
      cir: "10m"
      cb: "100k"
    output:
      cir: 0        # output tanpa limitasi -> tidak ada policer output
      cb: 64000
```

> ⚠️ Karena pembulatan, nilai kecil seperti `0.4k` (= 0.4 kbps) juga menjadi `0` dan
> berarti **tanpa limitasi**. Untuk nilai sekecil ini `vpptc` akan mencetak:
> ```
> [WARN] Interface 'TenGigabitEthernet1/0/0' arah 'input': CIR '0.4k' bernilai di bawah 1 satuan,
>        membulat menjadi 0 -> dianggap TANPA LIMITASI dan policer tidak dibuat.
> ```

### Contoh lengkap

```yaml
simple_policers:
  - interface: "TenGigabitEthernet1/0/0"
    input:
      cir: "10m"
      cb: "100k"
    output:
      cir: 5000
      cb: 64000

  - interface: "BondEthernet1.100"
    input:
      cir: "50m"
      cb: "1m"
    output:
      cir: "20m"
      cb: "10m"
```

---

## Pemakaian

```bash
vpptc {plan|apply|check} [--config PATH] [--startup-file PATH]
```

| Perintah | Keterangan |
|---|---|
| `vpptc check` | Periksa syntax/validitas file YAML (struktur, key, dan nilai CIR/CB) **tanpa menghubungi VPP** dan tanpa mengubah apa pun. |
| `vpptc plan` | Tampilkan diff saja. **Tidak menulis dan tidak mengubah apa pun**, termasuk file startup. |
| `vpptc apply` | Terapkan konfigurasi YAML ke runtime VPP dan tulis ulang file startup. |

| Opsi | Default | Keterangan |
|---|---|---|
| `--config PATH` | `/etc/vpp/vpptc.yaml` | **Hanya untuk pengujian/staging.** Lihat [Aturan Produksi](#aturan-produksi). |
| `--startup-file PATH` | `/etc/vpp/vpptc.vpp` | **Hanya untuk pengujian/staging** dan hanya berlaku untuk `plan`/`apply`. Lihat [Aturan Produksi](#aturan-produksi). |

### Aturan Produksi

> **Di produksi, selalu jalankan `vpptc apply` tanpa opsi apa pun.**
>
> ```bash
> vpptc apply
> ```
>
> Opsi `--config` dan `--startup-file` hanya untuk pengujian dan staging. Bila
> dipakaikan pada `apply`, `vpptc` menampilkan peringatan besar di layar.

Alasannya adalah model full-replacement `vpptc`: **YAML adalah seluruh acuan.**
Setiap policer berawalan `vpptc-` yang ada di VPP tetapi tidak ada di file yang
sedang dijalankan akan **dihapus**.

```
Runtime VPP:   vpptc-TenGigabitEthernet1_2F0_2F0-input-36000-252000
               vpptc-BondEthernet1_2E100-input-50000-1000000
File yang dijalankan (hanya berisi loop0):
               vpptc-loop0-input-1000-100000

Hasil: 2 policer di atas DIHAPUS
```

Jadi menjalankan `vpptc apply --config <file-yang-lebih-sempit>` di mesin produksi akan
**mematikan shaping pada interface yang tidak disebut di file tersebut**. Aturan ini
mencegah hal itu karena `vpptc apply` selalu membaca `/etc/vpp/vpptc.yaml`.

Yang **tidak** terpengaruh: policer yang Anda buat manual lewat `vppctl` (namanya
bukan `vpptc-`) tidak akan pernah disentuh — tool menandainya `[*] DIABAIKAN`.

### Alur kerja yang disarankan

```bash
vpptc check         # 1. pastikan YAML valid (tanpa menghubungi VPP)
vpptc plan          # 2. lihat perubahannya
                    # 3. periksa baris "[-] LEPAS & HAPUS" dengan teliti
vpptc apply         # 4. terapkan
```

### Contoh keluaran

```
$ vpptc check
[OK] File konfigurasi valid: /etc/vpp/vpptc.yaml

$ vpptc plan
[INFO] Membaca state VPP saat ini...

==================================================
 PERBANDINGAN KONFIGURASI (DIFF)
==================================================
 [-] LEPAS & HAPUS : vpptc-TenGigabitEthernet1_2F0_2F0-input-5000000-100000
                 (dari interface TenGigabitEthernet1/0/0)
 [+] TAMBAH & PASANG : vpptc-TenGigabitEthernet1_2F0_2F0-input-10000-100000 (CIR: 10000 kbps, CB: 100000 bytes)
 [*] DIABAIKAN      : 1 policer di luar kepemilikan vpptc (tidak akan diubah).
==================================================

[INFO] Mode 'plan' aktif. Tidak ada perubahan yang dieksekusi ke VPP.
```

Arti penanda diff:

| Penanda | Arti |
|---|---|
| `[=]` | Sudah sinkron, tidak ada perubahan. |
| `[+]` | Policer baru akan dibuat dan dipasang. |
| `[-]` | Policer lama akan dilepas dan dihapus. |
| `[*]` | Policer milik tool lain — **tidak** akan disentuh. |

---

## Cara Kerja

### Urutan eksekusi

`vpptc` berlaku **atomik per perubahan nilai**: nama policer memuat nilai CIR/CB,
sehingga mengubah CIR/CB menghasilkan nama baru.

```
1/4  policer add       (policer baru dibuat, belum terpasang)
2/4  policer unapply   (policer lama dilepas dari interface)
3/4  policer apply     (policer baru dipasang)
4/4  policer del       (policer lama dihapus)
     verifikasi       (state runtime dibaca ulang dan dicocokkan)
```

> Ada jeda singkat tanpa limitasi antara langkah 2 dan 3. Ini disengaja: urutan
> "pasang yang baru dulu, baru lepas yang lama" akan menumpuk dua policer pada satu
> interface, yang ditolak VPP.

### Percadangan

```
[+] TAMBAH & PASANG   cir=10m  cb=100k
[+] TAMBAH & PASANG   cir=10m  cb=100k   ← cir sama, cb berbeda: GANTI (bukan tambah)
```

Syarat gcd = CIR **dan** CB identik. Perubahan pada salah satu atau keduanya menghasilkan
penggantian. Nilai tetap (mis. hanya `input` yang diisi ulang) menghasilkan `[=]`.

---

## Konvensi Nama Policer

Nama policer yang dihasilkan `vpptc`:

```
vpptc-<token-interface>-<arah>-<cir>-<cb>
contoh: vpptc-loop0-input-10000-100000
contoh: vpptc-BondEthernet1_2E100-input-50000-1000000
```

### Aturan escaping (reversible)

| Asli | Token | Catatan |
|---|---|---|
| `loop0` | `loop0` | Huruf/angka dipertahankan |
| `BondEthernet1.100` | `BondEthernet1_2E100` | `.` → `_2E` |
| `TenGigabitEthernet1/0/0` | `TenGigabitEthernet1_2F0_2F0` | `/` → `_2F` |
| `a_sl_b` | `a_5Fsl_5Fb` | `_` → `_5F` (aman, tidak ambigu) |

Setiap karakter non-alnum di-encode sebagai `_XX` (hex 2 digit). Proses ini
**100% reversible** — semua nama interface, termasuk yang mengandung `_`, `/`, `.`,
spasi, atau karakter unicode, bisa dikembalikan persis.

### Kepemilikan

Hanya policer berawalan `vpptc-` yang boleh dihapus oleh `vpptc`. Policer tanpa prefix
tersebut dianggap milik tool lain dan dilewati.

Untuk migrasi dari `vpptc` versi lama, policer format `<iface>-<arah>-<cir>-<cb>` akan
di-*adopt* (dianggap milik vpptc) **hanya** bila nama tersebut persis valid **dan**
interface hasil dekoding-nya masih terdaftar di YAML.

---

## Keamanan

### Validasi sebelum eksekusi

`vpptc` menolak berhenti sebelum menyentuh VPP bila:

- struktur YAML tidak valid / key tidak dikenali / ada duplikat slot
- ada interface di YAML yang **tidak ada** di runtime VPP
- `show interface` atau `show policer` **gagal dibaca** (mis. VPP mati)
- format keluaran `show policer` tidak dikenali

### Deteksi error

`vppctl` **selalu keluar dengan exit code 0**, bahkan ketika perintahnya ditolak.
Pesan error justru ditulis ke *stdout*, misalnya:

```
policer del: No such policer
policer input: failed: `-6'
policer add: Policer already exists
show interface: unknown input `bogus99'
```

Karena itu `vpptc` tidak bergantung pada exit code saja. Perintah mutasi (`add`,
`apply`, `unapply`, `del`) dianggap **berhasil hanya bila stdout kosong sepenuhnya**.
Di samping itu ada **verifikasi akhir**: state VPP dibaca ulang dan dibandingkan dengan
keinginan. Bila ada selisih, `vpptc` melaporkan kegagalan dan **tidak menulis** file
startup.

### Anti-tabrakan

- **File lock** (`flock` pada `/run/lock/vpptc.lock`, `root:vpp` mode `0660`) mencegah
  dua `vpptc apply` berjalan bersamaan, termasuk antar-user dalam grup `vpp`.
  `plan` dan `check` tidak memakai lock (aman, karena read-only). Bila lock tidak bisa
  dibuat/diakses, `apply` **gagal tertutup** dengan exit code `3`.
- **Timeout 15 detik** per pemanggilan `vppctl`; tidak akan menggantung selamanya.
- **Tulis atomik**: file startup ditulis ke `.tmp`, `fsync`, lalu `os.replace`. File
  tidak pernah terpotong meski disk penuh atau proses dibunuh di tengah jalan.
- **Rollback**: policer yang gagal dipasang langsung dilepas dan dihapus agar tidak
  menjadi yatim.

---

## Exit Code

| Code | Arti |
|---|---|
| `0` | Berhasil. |
| `1` | Gagal: YAML salah, VPP tidak bisa dibaca, perintah ditolak, verifikasi akhir gagal, atau file startup tidak bisa ditulis. |
| `2` | Argumen baris perintah tidak dikenal. |
| `3` | Instance `vpptc` lain sedang berjalan, atau file lock tidak bisa dibuat/diakses (izin/grup salah). |

Cocok untuk dipakai di CI, cron, atau `ExecStartPost=` systemd.

---

## Pemecahan Masalah

### `Interface 'X' ... TIDAK DITEMUKAN di runtime VPP`

Interface yang disebut di YAML belum ada di VPP. Pastikan interface sudah dibuat
sebelum menjalankan `vpptc apply`:

```bash
vppctl show interface
```

### `[FATAL] Gagal membaca daftar interface dari VPP`

VPP tidak berjalan atau `vppctl` tidak terhubung:

```bash
systemctl status vpp
vppctl show version
```

### `[FATAL] Instance vpptc lain sedang berjalan`

Ada proses lain yang memegang lock. Periksa, hentikan, atau hapus lock bila yakin
prosesnya sudah mati:

```bash
pgrep -af vpptc
rm -f /run/lock/vpptc.lock     # hanya bila yakin tidak ada proses aktif
```

### `[FATAL] Tidak dapat membuka/membuat lock ... Permission denied`

File lock tidak dapat diakses oleh user yang menjalankan `vpptc apply`. Periksa grup,
kepemilikan, mode, dan keanggotaan user:

```bash
getent group vpp
stat -c '%A %U:%G %n' /run/lock/vpptc.lock
id -nG <user> | tr ' ' '\n' | grep -qx vpp \
    && echo "anggota vpp" || echo "BUKAN anggota vpp"
```

Perbaiki dengan membuat ulang lock sesuai izin yang benar:

```bash
cat > /etc/tmpfiles.d/vpptc.conf <<'EOF'
f /run/lock/vpptc.lock 0660 root vpp -
EOF
systemd-tmpfiles --create /etc/tmpfiles.d/vpptc.conf
usermod -aG vpp <user>     # lalu login ulang atau jalankan: newgrp vpp
```

Pesan `Grup 'vpp' tidak ditemukan` berarti grup tersebut belum ada di sistem. Pastikan
grup `vpp` sudah dibuat (biasanya oleh paket/instalasi VPP).

Jangan mengakali error ini dengan menghapus lock saat proses lain aktif. `vpptc apply`
sengaja berhenti dengan exit code `3` agar tidak pernah berjalan tanpa proteksi.

### Policer hilang setelah restart VPP

File startup yang dihasilkan **belum dimuat** oleh VPP saat boot. Lihat
[Catatan Operasional](#catatan-operasional).

### `[WARN] CIR 'X' bernilai di bawah 1 satuan, membulat menjadi 0`

Nilai terlalu kecil dan dianggap "tanpa limitasi". Naikkan nilainya bila ini bukan
maksud Anda, atau tulis `0` secara eksplisit bila memang ingin tanpa limitasi.

### Melihat konfigurasi apa adanya tanpa mengubah apa pun

```bash
vpptc plan
```

### Memeriksa YAML tanpa VPP

Gunakan `check` untuk memvalidasi struktur dan nilai YAML saja, misalnya di mesin
staging atau pipeline CI yang tidak memiliki daemon VPP:

```bash
vpptc check --config /path/ke/vpptc.yaml
```

---

## Catatan Operasional

### ⚠️ Sync bersifat deklaratif dan agresif

`vpptc` menganggap YAML sebagai **seluruh** konfigurasi yang dikelolanya. Setiap
policer berawalan `vpptc-` yang tidak ada di YAML **akan dihapus**, termasuk untuk
interface yang tidak disebut sama sekali di YAML.

Artinya, menghapus satu blok dari `/etc/vpp/vpptc.yaml` lalu menjalankan `vpptc apply`
**akan mematikan shaping** pada interface tersebut. Selalu periksa baris
`[-] LEPAS & HAPUS` pada `vpptc plan`.

Karena itu **jangan pernah menjalankan `vpptc apply` dengan `--config` lain di mesin
produksi** — lihat [Aturan Produksi](#aturan-produksi).

### ⚠️ Persistensi lintas restart belum otomatis

`vpptc` menulis `/etc/vpp/vpptc.vpp`, tetapi file tersebut **tidak otomatis dieksekusi**
saat VPP start. Selama belum ditambahkan ke konfigurasi boot, **setiap restart VPP akan
menghapus seluruh policer** hasil `vpptc`.

Untuk mengaktifkannya, tambahkan di `/etc/vpp/bootstrap.vpp` (file yang sudah di-`exec`
oleh `/etc/vpp/startup.conf`):

```
exec /etc/vpp/vpptc.vpp
```

Verifikasi:

```bash
systemctl restart vpp
vppctl show policer | grep -c '^Name'    # harusnya sesuai jumlah di YAML
```

Pastikan juga `BondEthernet1` dan sub-interface `BondEthernet1.100` sudah ada di VPP
sebelum baris-baris `policer ... BondEthernet1.100` di file startup dijalankan saat boot.

### ⚠️ Komentar pada file CLI VPP hanya memakai `#`

Parser CLI VPP **tidak mengenal `//`**. Satu baris `//` memicu `CLI line error` yang
**membatalkan seluruh sisa file**, sehingga tidak satu pun policer termuat:

```
$ vppctl exec /etc/vpp/vpptc.vpp
exec: CLI line error: // komentar...
```

Karena itu `vpptc` menulis header dengan `#`:

```
# AUTO-GENERATED by vpptc - DO NOT EDIT MANUALLY
# Regenerate with: vpptc apply --config /etc/vpp/vpptc.yaml
policer add name ... rate kbps cir ... cb ... conform-action transmit exceed-action drop
policer input name ... TenGigabitEthernet1/0/0
```

`vpptc` juga menolak menulis file bila ada baris ber-`//`, agar kesalahan ini tidak
terlewat tanpa tanda.

> **Membedakan error saat boot:** `policer add` dengan nama yang sudah ada juga
> menghasilkan `exec: CLI line error:` (pesan kosong) dan menghentikan sisa file. Ini
> wajar bila file di-`exec` dua kali atau policer sudah dibuat manual — bukan tanda
> file rusak.

Uji file startup tanpa menyentuh VPP secara permanen:

```bash
vppctl exec /etc/vpp/vpptc.vpp
vppctl show policer | grep -c '^Name'    # pastikan jumlah sesuai YAML
```

---

## Lisensi

Dibuat untuk keperluan internal. Gunakan dan modifikasi sesuai kebutuhan.
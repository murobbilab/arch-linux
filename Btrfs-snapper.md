# Btrfs + Snapper Setup

> Automatic snapshot & rollback system untuk Arch Linux atau distro lain berbasis Btrfs.

---

## Konsep Singkat

Btrfs menggunakan sistem **Copy-on-Write (CoW)**.

Snapshot di Btrfs bukan full copy. Snapshot hanya menyimpan perubahan dari kondisi sebelumnya, jadi prosesnya cepat dan relatif hemat storage.

Snapshot berguna untuk:

- Membuat titik aman sebelum update sistem
- Rollback ketika sistem error
- Membandingkan perubahan file
- Mengurangi drama setelah salah konfigurasi

> [!warning]
> Snapshot bukan backup.
> Kalau disk rusak, snapshot juga ikut hilang. Jadi tetap butuh backup eksternal atau cloud kalau datanya penting.

---

## Install Package

Jalankan setelah sistem Arch Linux terinstall.

```bash
sudo pacman -S snapper btrfs-progs
```

Kalau masih di dalam `arch-chroot`, tidak perlu pakai `sudo`:

```bash
pacman -S snapper btrfs-progs
```

---

## Struktur Subvolume

Struktur yang direkomendasikan:

```text
@
@home
@snapshots
@cache
@log
@tmp
```

Cek subvolume:

```bash
sudo btrfs subvolume list /
```

Kalau mengikuti guide instalasi sebelumnya, subvolume `.snapshots` sudah dibuat sebagai `@snapshots`.

---

## Cek Mount `.snapshots`

Pastikan `.snapshots` sudah menjadi mount point sendiri:

```bash
findmnt /.snapshots
```

Kalau hasilnya kosong, berarti `.snapshots` belum ke-mount.

Cek juga isi `fstab`:

```bash
cat /etc/fstab
```

Contoh mount yang benar:

```fstab
UUID=xxx /           btrfs rw,noatime,compress=zstd,ssd,discard=async,subvol=@          0 0
UUID=xxx /home       btrfs rw,noatime,compress=zstd,ssd,discard=async,subvol=@home      0 0
UUID=xxx /.snapshots btrfs rw,noatime,compress=zstd,ssd,discard=async,subvol=@snapshots 0 0
UUID=xxx /var/cache  btrfs rw,noatime,compress=zstd,ssd,discard=async,subvol=@cache     0 0
UUID=xxx /var/log    btrfs rw,noatime,compress=zstd,ssd,discard=async,subvol=@log       0 0
UUID=xxx /tmp        btrfs rw,noatime,compress=zstd,ssd,discard=async,subvol=@tmp       0 0
```

> [!warning]
> Jangan sampai `/tmp` pakai `subvol=@log`.
> Itu bukan setup keren, itu setup minta ditampar error.

---

## Setup Snapper untuk Root

Buat konfigurasi root:

```bash
sudo snapper -c root create-config /
```

Command ini akan membuat:

```text
/etc/snapper/configs/root
```

Snapper juga biasanya akan mencoba membuat folder `.snapshots`.

Kalau kamu sudah punya subvolume `@snapshots` yang dimount ke `/.snapshots`, pastikan setelah command ini struktur mount masih benar.

Cek:

```bash
findmnt /.snapshots
sudo btrfs subvolume list /
```

---

## Fix Permission `.snapshots`

```bash
sudo chmod 750 /.snapshots
sudo chown :wheel /.snapshots
```

Artinya:

- Owner tetap `root`
- Group menjadi `wheel`
- User di group `wheel` bisa akses sesuai permission

---

## Enable Auto Snapshot

Aktifkan timer Snapper:

```bash
sudo systemctl enable --now snapper-timeline.timer
sudo systemctl enable --now snapper-cleanup.timer
```

Cek status:

```bash
systemctl status snapper-timeline.timer
systemctl status snapper-cleanup.timer
```

---

## Default Behavior Snapper

Secara umum Snapper bisa membuat snapshot otomatis berdasarkan timeline:

- Hourly
- Daily
- Weekly
- Monthly
- Yearly

Snapshot lama akan dibersihkan otomatis oleh cleanup timer berdasarkan konfigurasi limit.

---

## Konfigurasi Snapper

Edit config root:

```bash
sudo nano /etc/snapper/configs/root
```

Contoh konfigurasi yang lebih santai:

```ini
TIMELINE_CREATE="yes"
TIMELINE_CLEANUP="yes"

TIMELINE_LIMIT_HOURLY="5"
TIMELINE_LIMIT_DAILY="7"
TIMELINE_LIMIT_WEEKLY="4"
TIMELINE_LIMIT_MONTHLY="3"
TIMELINE_LIMIT_YEARLY="0"

NUMBER_CLEANUP="yes"
NUMBER_LIMIT="10"
NUMBER_LIMIT_IMPORTANT="5"
```

Penjelasan singkat:

| Opsi | Fungsi |
| ---- | ------ |
| `TIMELINE_CREATE` | Mengaktifkan snapshot otomatis berdasarkan waktu |
| `TIMELINE_CLEANUP` | Membersihkan snapshot timeline lama |
| `TIMELINE_LIMIT_HOURLY` | Batas snapshot per jam |
| `TIMELINE_LIMIT_DAILY` | Batas snapshot harian |
| `TIMELINE_LIMIT_WEEKLY` | Batas snapshot mingguan |
| `TIMELINE_LIMIT_MONTHLY` | Batas snapshot bulanan |
| `NUMBER_LIMIT` | Batas snapshot manual biasa |
| `NUMBER_LIMIT_IMPORTANT` | Batas snapshot yang ditandai important |

---

## Snapshot Manual

### Buat snapshot

```bash
sudo snapper -c root create --description "before update"
```

Atau versi pendek:

```bash
sudo snapper create --description "before update"
```

### List snapshot

```bash
sudo snapper list
```

### Lihat perubahan antar snapshot

```bash
sudo snapper status 1..2
```

### Lihat diff file

```bash
sudo snapper diff 1..2
```

Ganti `1..2` sesuai nomor snapshot.

---

## Snapshot Sebelum Update

Sebelum update sistem:

```bash
sudo snapper create --description "pre system update"
sudo pacman -Syu
```

Kalau update aman, lanjut hidup.

Kalau update rusak, baru rollback.

---

## Rollback System

Rollback root:

```bash
sudo snapper rollback
```

Yang terjadi:

- Snapshot dipakai sebagai root baru
- Root lama tetap disimpan sebagai snapshot
- Sistem perlu reboot

Setelah rollback:

```bash
sudo reboot
```

> [!warning]
> Perubahan setelah snapshot bisa hilang.
> Jangan rollback sembarangan kalau kamu baru bikin file penting di root filesystem.

---

## Integrasi GRUB

Integrasi GRUB berguna supaya snapshot bisa muncul di menu boot.

Install:

```bash
sudo pacman -S grub-btrfs inotify-tools
```

Enable daemon:

```bash
sudo systemctl enable --now grub-btrfsd
```

Generate ulang config GRUB:

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

Hasilnya:

- Snapshot muncul di menu GRUB
- Bisa boot ke snapshot
- Bisa recovery walaupun sistem utama error

> [!warning]
> Jika daemon `grub-btrfsd` mati, snapshot baru bisa saja tidak muncul di menu GRUB.

Cek status daemon:

```bash
systemctl status grub-btrfsd
```

---

## Catatan untuk UKI

Kalau kamu pakai UKI tanpa GRUB, snapshot tidak otomatis muncul sebagai menu boot seperti di GRUB.

UKI tetap bisa dipakai dengan Btrfs + Snapper, tapi workflow rollback-nya tidak senyaman GRUB snapshot menu.

Pilihan realistis:

- Pakai **GRUB** kalau ingin snapshot boot menu yang gampang.
- Pakai **UKI** kalau fokus ke Secure Boot dan setup minimal.
- Pakai **UKI + recovery entry/manual rescue** kalau kamu sudah paham alurnya.

Jadi jangan berharap UKI tiba-tiba punya menu snapshot kayak GRUB. Dia bukan dukun.

---

## Setup Snapper untuk `/home` Opsional

Kalau `/home` dipisah sebagai subvolume, kamu bisa membuat config Snapper sendiri untuk `/home`.

```bash
sudo snapper -c home create-config /home
```

Edit config:

```bash
sudo nano /etc/snapper/configs/home
```

Tapi hati-hati.

Snapshot `/home` bisa makan storage lebih cepat kalau isinya banyak file besar seperti:

- Video
- ISO
- Game
- Cache browser
- Project build
- `node_modules`

Kalau `/home` berisi banyak file barbar, jangan asal aktifin snapshot timeline terlalu agresif.

---

## Cleanup Manual

List snapshot:

```bash
sudo snapper list
```

Hapus snapshot tertentu:

```bash
sudo snapper delete nomor_snapshot
```

Contoh:

```bash
sudo snapper delete 12
```

Hapus range:

```bash
sudo snapper delete 10-15
```

---

## Cek Penggunaan Storage Btrfs

```bash
sudo btrfs filesystem usage /
```

Cek detail:

```bash
sudo btrfs filesystem df /
```

Kalau disk terlalu penuh, Btrfs bisa mulai rewel.

Sisakan free space yang cukup, terutama kalau sering snapshot.

---

## Best Practice

- Buat snapshot sebelum update besar.
- Pisahkan `/home` dari root.
- Jangan anggap snapshot sebagai backup.
- Jangan biarkan disk terlalu penuh.
- Cek snapshot lama secara berkala.
- Backup data penting ke disk lain.
- Gunakan GRUB kalau ingin boot snapshot dengan mudah.

---

## Common Mistakes

### 1. Tidak mount `.snapshots` sebagai subvolume

Akibatnya snapshot bisa masuk ke root biasa, bukan ke subvolume khusus.

Cek:

```bash
findmnt /.snapshots
```

---

### 2. Disk terlalu penuh

Btrfs butuh ruang kosong untuk kerja normal.

Cek:

```bash
sudo btrfs filesystem usage /
```

---

### 3. Salah config `fstab`

Contoh kesalahan:

```fstab
UUID=xxx /tmp btrfs subvol=@log 0 0
```

Yang benar:

```fstab
UUID=xxx /tmp btrfs subvol=@tmp 0 0
```

---

### 4. Mengira snapshot sama dengan backup

Snapshot masih berada di disk yang sama.

Kalau disk mati, snapshot ikut wafat.

---

### 5. Rollback tanpa paham efeknya

Rollback bisa mengembalikan sistem ke kondisi lama.

File atau konfigurasi yang dibuat setelah snapshot bisa hilang dari root filesystem.

---

## Workflow yang Direkomendasikan

```text
1. Buat snapshot sebelum update
2. Update system
3. Kalau aman, lanjut kerja
4. Kalau error, rollback
5. Reboot
```

Contoh:

```bash
sudo snapper create --description "before pacman update"
sudo pacman -Syu
```

Kalau error:

```bash
sudo snapper rollback
sudo reboot
```

---

## Struktur Snapshot

Ilustrasi struktur snapshot:

```text
@snapshots
├── 1/
├── 2/
├── 3/
└── 4/
```

Biasanya akan terlihat di:

```text
/.snapshots/
├── 1/
│   ├── info.xml
│   └── snapshot/
├── 2/
│   ├── info.xml
│   └── snapshot/
```

---

## Related

- [[Arch-Install]]
- [[Boot-GRUB]]
- [[Boot-UKI]]
- [Arch Wiki - Snapper](https://wiki.archlinux.org/title/Snapper)
- [Arch Wiki - Btrfs](https://wiki.archlinux.org/title/Btrfs)
- [Arch Wiki - GRUB](https://wiki.archlinux.org/title/GRUB)

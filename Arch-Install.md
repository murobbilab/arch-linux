# Arch Linux Installation Guide (UEFI)

> \[!NOTE\] Panduan instalasi **Arch Linux** berbasis **UEFI** dengan
> konfigurasi minimal. Ditujukan untuk pengguna yang ingin memahami
> proses instalasi secara manual.

## Table of Contents

-   [1. Persiapan](#1-persiapan)
-   [2. Cek Mode Boot](#2-cek-mode-boot)
-   [3. Cek Koneksi Internet](#3-cek-koneksi-internet)
-   [4. Sinkronisasi Waktu](#4-sinkronisasi-waktu)
-   [5. Partisi & File System](#5-partisi--file-system)
-   [6. Format Partisi](#6-format-partisi)
-   [7. Mount Partisi](#7-mount-partisi)
-   [8. Install Base System](#8-install-base-system)
-   [9. Generate fstab](#9-generate-fstab)
-   [10. Chroot](#10-chroot-ke-sistem-baru)
-   [11. Konfigurasi Sistem](#11-konfigurasi-sistem)
-   [12. Bootloader](#12-pilih-boot-method)
-   [13. Finalisasi](#13-finalisasi)

---

## 1. Persiapan

### Download ISO

Unduh ISO resmi dari:

<https://archlinux.org/download/>

### Buat Bootable USB

Gunakan salah satu:

- Rufus: <https://rufus.ie/id/>
- Ventoy: <https://www.ventoy.net>

### Boot ke Live ISO

Boot komputer menggunakan USB installer, kemudian pilih:

``` bash
Arch Linux install medium
```

---

## 2. Cek Mode Boot

Pastikan sistem berjalan dalam mode UEFI.

``` bash
cat /sys/firmware/efi/fw_platform_size
```

Jika keluar:

``` text
64
```

berarti installer berjalan pada mode **UEFI 64-bit**.

> \[!WARNING\] Jika folder `/sys/firmware/efi` tidak ada, berarti kamu
> masih boot menggunakan Legacy BIOS. Restart dan masuk kembali
> menggunakan mode UEFI.

---

## 3. Cek Koneksi Internet

``` bash
ping archlinux.org
```

Jika menggunakan Wi-Fi:

``` bash
iwctl
```

Lalu jalankan:

``` bash
device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect "nama_wifi"
exit
```

Verifikasi kembali:

``` bash
ping archlinux.org
```

---

## 4. Sinkronisasi Waktu

``` bash
timedatectl
```

Output yang diharapkan:

``` text
System clock synchronized: yes
NTP service: active
```

---

## 5. Partisi & File System

Lihat disk:

``` bash
lsblk
```

> \[!CAUTION\] Pastikan memilih disk yang benar sebelum memformat
> partisi. Seluruh data pada partisi yang diformat akan hilang.

### Skema Partisi UEFI

| Partisi | Ukuran | Tipe | Mount Point |
| --- | --- | --- | --- |
| `/dev/nvme0n1p1` | 1 GiB | FAT32 (EFI System) | `/efi` atau `/boot` |
| `/dev/nvme0n1p2` | Sisa Disk | ext4 / Btrfs | `/` |

---

## Buat Partisi

```bash
cfdisk /dev/nvme0n1
```

Buat partisi:

- `/dev/nvme0n1p1` → EFI System
- `/dev/nvme0n1p2` → Linux filesystem / Root

Setelah selesai:

1. Pilih `Write`
2. Ketik `yes`
3. Pilih `Quit`

---

## 6. Format Partisi

### Opsi ext4

```bash
mkfs.fat -F32 /dev/nvme0n1p1
mkfs.ext4 /dev/nvme0n1p2
```

### Opsi btrfs

```bash
mkfs.fat -F32 /dev/nvme0n1p1
mkfs.btrfs -f /dev/nvme0n1p2
```

---

## 7. Mount Partisi

### Jika pakai ext4

```bash
mount /dev/nvme0n1p2 /mnt
mount --mkdir /dev/nvme0n1p1 /mnt/efi
```

---

### Jika pakai btrfs dengan subvolume

Mount root dulu:

```bash
mount /dev/nvme0n1p2 /mnt
```

Buat subvolume:

```bash
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@cache
btrfs subvolume create /mnt/@log
btrfs subvolume create /mnt/@tmp
```

Unmount:

```bash
umount /mnt
```

Mount ulang:

```bash
mount -o noatime,compress=zstd,ssd,discard=async,subvol=@ /dev/nvme0n1p2 /mnt
mkdir -p /mnt/{home,var/cache,var/log,tmp,efi}

mount -o noatime,compress=zstd,ssd,discard=async,subvol=@home /dev/nvme0n1p2 /mnt/home
mount -o noatime,compress=zstd,ssd,discard=async,subvol=@cache /dev/nvme0n1p2 /mnt/var/cache
mount -o noatime,compress=zstd,ssd,discard=async,subvol=@log /dev/nvme0n1p2 /mnt/var/log
mount -o noatime,compress=zstd,ssd,discard=async,subvol=@tmp /dev/nvme0n1p2 /mnt/tmp

mount /dev/nvme0n1p1 /mnt/efi
```

---

## 8. Install Base System

```bash
pacstrap -K /mnt base linux linux-firmware intel-ucode amd-ucode nano sudo networkmanager bash-completion
```
> \[!CAUTION\] kalau pakai CPU Intel, hapus amd-ucode. Kalau pakai AMD, hapus intel-ucode. Jangan di-install dua-duanya, maruk amat!

### Detail Paket:

| Paket | Fungsi |
| :--- | :--- |
| `base` | Core system Arch Linux |
| `linux` | Kernel utama |
| `linux-firmware` | Driver hardware (WiFi, Bluetooth, GPU, dll) |
| `intel-ucode` | **(Khusus Intel)** Update mikrocode untuk keamanan dan stabilitas CPU Intel |
| `amd-ucode` | **(Khusus AMD)** Update mikrocode untuk keamanan dan stabilitas CPU AMD |
| `nano` | Text editor terminal |
| `sudo` | Manajemen hak akses root |
| `networkmanager` | Pengelola koneksi internet |



---

## 9. Generate fstab

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

Cek hasilnya:

```bash
cat /mnt/etc/fstab
```

Kalau mount point terlihat aneh, benerin dulu sebelum lanjut.

---

## 10. Chroot ke Sistem Baru

```bash
arch-chroot /mnt
```

---

## 11. Konfigurasi Sistem

### Timezone

```bash
ln -sf /usr/share/zoneinfo/Asia/Jakarta /etc/localtime
hwclock --systohc
```

---

### Locale

Edit:

```bash
nano /etc/locale.gen
```

Uncomment baris ini:

```bash
en_US.UTF-8 UTF-8 #Ingris
id_ID.UTF-8 UTF-8 #Indonesia
```

Generate locale:

```bash
locale-gen
```

Set default locale:

```bash
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

---

### Keyboard

Opsional, kalau pakai layout US:

```bash
echo "KEYMAP=us" > /etc/vconsole.conf
```

---

### Hostname

```bash
echo "archlinux" > /etc/hostname
```

Edit hosts:

```bash
nano /etc/hosts
```

Isi:

```txt
127.0.0.1   localhost
::1         localhost
127.0.1.1   archlinux.localdomain archlinux
```

---

### Root Password

```bash
passwd
```

---

### Buat User

Ganti `username` dengan nama user kamu:

```bash
useradd -m -G wheel -s /bin/bash username
passwd username
```

---

### Enable sudo

```bash
EDITOR=nano visudo
```

Uncomment baris ini:

```txt
%wheel ALL=(ALL:ALL) ALL
```

---

### Enable NetworkManager

```bash
systemctl enable NetworkManager
```

---

## 12. Pilih Boot Method

Sekarang pilih salah satu:

### Opsi 1 — UKI

Pakai UKI kalau kamu mau setup yang lebih modern, rapi, dan cocok buat Secure Boot.

<details>
<summary>UKI</summary>

## 1. Install Paket yang Dibutuhkan

Di dalam `arch-chroot`:

```bash
pacman -S efibootmgr systemd
```

Biasanya `systemd` sudah ikut terinstall dari base system, tapi command ini aman buat memastikan.

---

## 2. Buat Kernel Command Line

Cek UUID root:

```bash
blkid
```

Cari UUID dari partisi root, contoh:

```txt
/dev/nvme0n1p2: UUID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

Buat file:

```bash
nano /etc/kernel/cmdline
```

---

## Jika Root Pakai ext4

Isi:

```txt
root=UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx rw
```

---

## Jika Root Pakai btrfs

Isi:

```txt
root=UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx rootflags=subvol=@ rw
```

> Jangan pakai UUID EFI. Pakai UUID partisi root. Ini kesalahan klasik yang bikin boot gagal terus.

---

## 3. Edit Preset mkinitcpio

Buka preset kernel:

```bash
nano /etc/mkinitcpio.d/linux.preset
```

Ubah menjadi seperti ini:

```bash
# mkinitcpio preset file for the 'linux' package

ALL_kver="/boot/vmlinuz-linux"

PRESETS=('default' 'fallback')

default_uki="/efi/EFI/Linux/arch-linux.efi"
default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"

fallback_uki="/efi/EFI/Linux/arch-linux-fallback.efi"
fallback_options="-S autodetect"
```

Pastikan folder tujuan ada:

```bash
mkdir -p /efi/EFI/Linux
```

Generate UKI:

```bash
mkinitcpio -P
```

Cek hasilnya:

```bash
ls -lah /efi/EFI/Linux
```

Harus ada file seperti:

```txt
arch-linux.efi
arch-linux-fallback.efi
```

---

## 4. Buat Entry UEFI Langsung

Cek disk dan partisi EFI:

```bash
lsblk
```

Contoh:

- Disk: `/dev/nvme0n1`
- EFI: partisi pertama, berarti `--part 1`

Buat boot entry:

```bash
efibootmgr --create --disk /dev/nvme0n1 --part 1 --label "Arch Linux UKI" --loader '\EFI\Linux\arch-linux.efi'
```

Cek boot entry:

```bash
efibootmgr
```

---

## 5. Secure Boot

UKI enak untuk Secure Boot karena yang perlu ditandatangani cukup file `.efi` hasil gabungan tadi.

Konsepnya:

1. Buat UKI.
2. Sign file UKI.
3. Daftarkan key Secure Boot.
4. Firmware hanya menjalankan file EFI yang sudah valid/sign.

Untuk setup Secure Boot biasanya bisa pakai `sbctl`.

Install:

```bash
pacman -S sbctl
```

Cek status:

```bash
sbctl status
```

Buat key:

```bash
sbctl create-keys
```

Enroll key:

```bash
sbctl enroll-keys -m
```

Sign UKI:

```bash
sbctl sign -s /boot/EFI/Linux/arch-linux.efi
sbctl sign -s /boot/EFI/Linux/arch-linux-fallback.efi
```

Cek file yang sudah terdaftar:

```bash
sbctl list-files
```

Setiap kernel update, UKI akan dibuat ulang. Pastikan file UKI yang baru tetap signed.

</details>

### Opsi 2 — GRUB

Pakai GRUB kalau kamu mau bootloader umum, gampang dipahami, dan enak buat dual boot.

Baca:

<details>
<summary>Grub</summary>

## 1. Install Paket GRUB

```bash
pacman -S grub efibootmgr
```

---

## 2. Install GRUB ke EFI

Pastikan EFI mounted di `/efi`.

Cek:

```bash
lsblk
```

Install GRUB:

```bash
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB
```

Generate config:

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

---

## 3. Jika Pakai Dual Boot Windows

Install `os-prober`:

```bash
pacman -S os-prober
```

Edit config GRUB:

```bash
nano /etc/default/grub
```

Cari atau tambahkan:

```txt
GRUB_DISABLE_OS_PROBER=false
```

Generate ulang:

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

Kalau Windows terdeteksi, nanti akan muncul entry Windows Boot Manager di menu GRUB.

</details>

> Pilih salah satu dulu. Jangan install dua-duanya kalau belum paham boot order, nanti BIOS jadi pasar malam.

---

## 13. Finalisasi

Setelah bootloader selesai dipasang, keluar dari chroot:

```bash
exit
```

Unmount:

```bash
umount -R /mnt
```

Reboot:

```bash
reboot
```

Cabut USB installer ketika sistem mulai restart.

---

## Catatan Lanjutan

Untuk setup lanjut:

- [Snapshot btrfs](./Btrfs-snapper.md)
- Desktop Environment: Hyprland, GNOME, KDE
- Audio: PipeWire
- GPU driver
- Secure Boot

---

## Related

- Arch Wiki: <https://wiki.archlinux.org/>
- Installation Guide: <https://wiki.archlinux.org/title/Installation_guide>

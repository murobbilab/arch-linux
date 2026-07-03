# Dokumentasi Arch Linux

> Catatan instalasi dan setup Arch Linux berbasis UEFI.
> Dibuat supaya proses install Arch tidak berubah jadi ritual pemanggilan error dari neraka.

Dokumentasi ini ditulis dalam format Markdown dan cocok dipakai di:

- GitHub
- Obsidian
- VS Code
- Markdown reader biasa

---

## Isi Dokumentasi

| File | Isi |
| ---- | --- |
| [Arch-Install.md](./Arch-Install.md) | Panduan instalasi Arch Linux UEFI dari live ISO sampai konfigurasi dasar |
| [Btrfs-snapper.md](./Btrfs-snapper.md) | Setup Btrfs, subvolume, Snapper, snapshot, dan rollback |

---

## Alur Baca yang Direkomendasikan

Kalau baru install dari nol, baca dengan urutan ini:

```text
1. Arch-Install.md
2. Btrfs-snapper.md
```

Jangan langsung lompat ke Snapper kalau sistem dasar belum bisa boot.
Itu namanya bukan efisien, itu namanya cari perkara.

---

## Pilih GRUB atau UKI?

### Pakai GRUB kalau:

- Masih pemula
- Mau dual boot dengan Windows
- Ingin menu boot yang gampang dilihat
- Ingin snapshot Btrfs muncul di boot menu lewat `grub-btrfs`
- Mau setup yang lebih umum dan mudah dicari solusinya

---

### Pakai UKI kalau:

- Mau setup modern
- Mau boot langsung dari file `.efi`
- Mau setup yang cocok untuk Secure Boot
- Tidak butuh menu bootloader yang ribet
- Sudah cukup paham soal kernel command line dan EFI entry

---

## Btrfs + Snapper

Dokumentasi Btrfs membahas:

- Struktur subvolume
- Mount option
- Setup Snapper
- Snapshot otomatis
- Snapshot manual
- Rollback system
- Integrasi GRUB dengan `grub-btrfs`
- Catatan untuk pengguna UKI

Baca: [Btrfs-snapper.md](./Btrfs-snapper.md)

> Snapshot itu bukan backup.
> Kalau disk rusak, snapshot ikut tumbang. Jangan sok aman cuma karena punya snapshot.

---

## Struktur Repo

```text
.
├── README.md
├── Arch-Install.md
└── Setup-btrfs.md
```

---

## Target Sistem

Dokumentasi ini ditujukan untuk:

- Arch Linux
- Sistem UEFI
- Disk GPT
- File system ext4 atau Btrfs
- Boot method GRUB atau UKI

Beberapa bagian juga bisa dipakai untuk distro lain yang berbasis Btrfs, tapi command package manager dan detail bootloader bisa beda.

---

## Catatan Penting

Sebelum copy-paste command:

1. Cek nama disk dengan `lsblk`
2. Pastikan mode boot adalah UEFI
3. Jangan asal format partisi
4. Baca command sebelum enter
5. Jangan menyalahkan dokumentasi kalau kamu format disk Windows sendiri

Minimal baca dulu, baru gas. Jangan kebalik, bos.

---

## Referensi

- [Arch Wiki](https://wiki.archlinux.org/)
- [Arch Installation Guide](https://wiki.archlinux.org/title/Installation_guide)
- [Arch Wiki - GRUB](https://wiki.archlinux.org/title/GRUB)
- [Arch Wiki - Unified Kernel Image](https://wiki.archlinux.org/title/Unified_kernel_image)
- [Arch Wiki - Btrfs](https://wiki.archlinux.org/title/Btrfs)
- [Arch Wiki - Snapper](https://wiki.archlinux.org/title/Snapper)

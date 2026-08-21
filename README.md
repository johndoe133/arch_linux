# System Context

## Hardware

- **Laptop:** ASUS ROG Zephyrus G15 (GA503QR-211.ZG15)
- **CPU:** AMD Ryzen 9
- **GPU:** NVIDIA GeForce RTX 3070 (discrete) + AMD integrated (hybrid/Optimus)
- **RAM:** 16GB
- **Storage:** 1TB NVMe SSD
- **Display:** 15.6" QHD
- **Boot mode:** UEFI only, secure boot off
- **Battery:** 90WH at 72% health

## OS & Desktop

- **OS:** CachyOS (Arch-based, rolling release)
- **Desktop environment:** KDE Plasma
- **Display server:** Wayland
- **Filesystem:** ext4
- **Bootloader:** Limine
- **GPU driver:** NVIDIA proprietary
- **AUR helper:** paru

## Boot options
- **Brightness wrap-around issue:** adding `amdgpu.dcdebugmask=0x40000` to the `KERNEL_CMDLINE` options in `/etc/default/limine` fixes this. Backup saved in `.bak` file.
- **Secure boot:** Off
- **limine.conf:** Changed several options in `/boot/limine.conf` with a backup in `bak` for faster/better boot
  - **Limine timeout:** Set to 1 in 
  - **Limine quiet mode:** Also added `quiet: yes` to the conf file.
  - **remember last entry:** `remember_last_entry` set to yes for now. Changing would involve more changes to ensure the latest kernel is default. 

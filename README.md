# System Context

## Hardware

- **Laptop:** ASUS ROG Zephyrus G15 (GA503QR-211.ZG15)
- **CPU:** AMD Ryzen 9
- **GPU:** NVIDIA GeForce RTX 3070 (discrete) + AMD integrated (hybrid/Optimus)
- **RAM:** 16GB
- **Storage:** 1TB NVMe SSD
- **Display:** 15.6" QHD
- **Boot mode:** UEFI only

## OS & Desktop

- **OS:** CachyOS (Arch-based, rolling release)
- **Desktop environment:** KDE Plasma
- **Display server:** Wayland
- **Filesystem:** ext4
- **Bootloader:** Limine
- **GPU driver:** NVIDIA proprietary
- **AUR helper:** paru

## Kernel Parameters

- `amdgpu.dcdebugmask=0x40000` — fixes brightness overflow bug at 99-100% (AMDGPU regression)

## Known Issues & Workarounds

- **Brightness overflow at 100%** — driver wraps brightness value to 0 at max. Fixed with `dcdebugmask` kernel param above. Remove when fixed upstream.

## Notes

- Fast Boot disabled in UEFI to allow USB boot
- UEFI GPU mode set to Switchable Graphics (not discrete-only)

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

## Boot customization

- Turned off limine boot menu with `timeout: 0` in `/boot/limine.conf` and `quiet: yes` in `/etc/default/limine`
- Disabled `NetworkManager-wait-online.service` (using ~6 seconds of boot time, unknown if in parallel or not)
- changed modules to: `MODULES=(amdgpu)` and hooks to `HOOKS=(base systemd plymouth autodetect microcode modconf kms keyboard sd-vconsole block filesystems fsck)` to avoid black screen between ROG and plymouth boot screens. Didn't work, revert to:
```
MODULES=()
HOOKS=(base systemd autodetect microcode kms modconf block keyboard sd-vconsole plymouth filesystems fsck)
```

## Ricing/theming

- **Theme:** Otto (appearance settings only, not desktop and window layout)
- **Icons:** Tela (looks so clean)
- **Boot splash:**  Spin (See other plymouth themes [here](https://www.gnome-look.org/browse?cat=108&page=1&ord=rating) or [here is better](https://github.com/adi1090x/plymouth-themes))
- **Splash screen:** Animation Shows after booting when you first log in. Currently set to none to avoid excessive loading menus on boot. 

## Other customization

- **Screenshotting:** Disabled super+shift+s shortcut for spectacle and enabled auto-select ('Accept on click-and-release'), then made a custom shortcut for super+shift+s that does the command `spectacle -r -b -c` which captures selection, adds to clipboard, and closes spectacle. 

## Notes

- Fast Boot enabled in UEFI
- UEFI GPU mode set to Switchable Graphics (not discrete-only)

## Temporary packages
- hwinfo (3.4mb)

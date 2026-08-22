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

## Ricing/theming

- **Theme:** Otto (appearance settings only, not desktop and window layout)
- **Icons:** Tela (download through `aur` so permissions are there even when logged out and icons can render)
- **Window decorations:** Willow-dark (to be more similar to steam and firefox decorators that can't be changed)
- **Boot splash:**  Spin (See other plymouth themes [here](https://www.gnome-look.org/browse?cat=108&page=1&ord=rating) or [here is better](https://github.com/adi1090x/plymouth-themes))
- **Splash screen:** Animation Shows after booting when you first log in. Currently set to none to avoid excessive loading menus on boot. 
- **Wallpaper:** Use [wallhaven](https://wallhaven.cc) toplist for a year, put in `/usr/share/wallpapers/`
- **Lock screen:** Set to same image under "Screen Locking" settings.
- **Login screen:** Set to same image under "Login Screen" settings.

## Other customization

- **Screenshotting:** Disabled super+shift+s shortcut for spectacle and enabled auto-select ('Accept on click-and-release'), then made a custom shortcut for super+shift+s that does the command `spectacle -r -b -c` which captures selection, adds to clipboard, and closes spectacle. 

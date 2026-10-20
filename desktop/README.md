# System Context — Desktop (`ekj-cachyOS`)

_Last surveyed: 2026-10-20_

## Hardware

- **Motherboard:** Gigabyte B650 EAGLE AX (BIOS F37, 2025-12-17)
- **CPU:** AMD Ryzen 7 9800X3D (8c/16t)
- **GPU:** AMD Radeon RX 9070-series (Navi 48), plus the iGPU (Radeon 610M) still enabled in BIOS
- **RAM:** 32GB
- **Storage:**
  - Lexar NM790 2TB (`nvme0n1`): Linux. 4GB FAT32 ESP at `/boot` + btrfs for everything else
  - WD_BLACK SN7100 1TB (`nvme1n1`): **Windows** (its own ESP + NTFS). Dual boot
- **Displays:**
  - DP-1: 2560×1440 @ 125% scale (main, left)
  - HDMI-A-1: 1920×1080 @ 100% (right)
- **Boot mode:** UEFI, **Secure Boot ON**

## OS & Desktop

- **OS:** CachyOS (Arch-based, rolling), `znver4`-optimized repos
- **Kernel:** `linux-cachyos`
- **Desktop environment:** KDE Plasma 6.8, Wayland
- **Login manager:** **Plasma Login Manager** (`plasma-login-manager`), not SDDM. Config: `/etc/plasmalogin.conf`
- **Filesystem:** btrfs (`zstd:1`, subvolumes `@`, home, cache, log, etc.) + zram swap
- **Bootloader:** Limine (managed with `limine-mkinitcpio-hook`)
- **Snapshots:** snapper (config `root`) + `limine-snapper-sync`, so snapshots appear as boot entries. GUI: btrfs-assistant
- **Initramfs:** systemd-based mkinitcpio hooks, with `plymouth` hook
- **GPU driver:** Mesa (open source, built in)
- **AUR helper:** paru (only 2 AUR packages: `tela-icon-theme`, `jellyfin-desktop`)
- **Shell:** fish (stock CachyOS config, nothing custom yet)
- **Editor:** VS Code **OSS build** (`code` package, extensions from Open VSX). Settings in `~/.config/Code - OSS/User/settings.json`: editor, terminal, debug console, chat and commit-message box all use JetBrainsMono Nerd Font (`ttf-jetbrains-mono-nerd`). The OSS build has no Settings Sync, so signing in to GitHub does not sync settings
- **Terminals installed:** Konsole, Alacritty, Ghostty (main)
- **Ghostty config:** `~/.config/ghostty/config.ghostty`: only `theme = TokyoNight Night` (matches Starship); everything else default (JetBrains Mono, no transparency, which I tried and didn't like). Check edits with `ghostty +validate-config`
- **Prompt:** Starship (`pacman -S starship`, not the curl script), loaded via `starship init fish | source` in `~/.config/fish/config.fish`. Uses the Tokyo Night preset (`~/.config/starship.toml`). The preset has no CachyOS icon, so it showed a 🎗️ ribbon emoji; fixed by adding `CachyOS = "󰣇"` (Arch logo) under `[os.symbols]`. Needs a Nerd Font in the terminal
- **Firewall:** ufw (enabled)
- **Locale:** `en_DK.UTF-8` with Danish number/time/currency formats, US keyboard layout

## Boot options

- **Kernel cmdline:** stock: `quiet nowatchdog splash rw rootflags=subvol=/@ root=UUID=…`. Edit in `/etc/default/limine` (`KERNEL_CMDLINE[default]+=`), **not** in `/boot/limine.conf`, which gets regenerated
- **Secure Boot:** set up with `sbctl`, using my own keys + **Microsoft vendor keys enrolled** (Windows and GPU option ROMs need them). `ENABLE_ENROLL_LIMINE_CONFIG=yes` signs/enrolls the Limine config hash. ⚠️ If a BIOS update or CMOS reset wipes keys → re-run `sudo sbctl enroll-keys -m` (or turn Secure Boot off temporarily)
- **Limine timeout / quiet / remember_last_entry:** TODO: check `/boot/limine.conf` (needs root) and note anything changed from stock

## Ricing/theming

- **Global theme:** CachyOS-Nord (CachyOS default)
- **Plasma style:** CachyOS-Nord-round
- **Colors:** CachyOSNordLightly
- **Application style:** Breeze
- **Icons:** Tela-dark (installed via AUR `tela-icon-theme`, system-wide, so it also renders when logged out)
- **Window decorations:** Plastik (built into KWin). Tried Willow Dark Alt (per-user in `~/.local/share/aurorae/themes/`), but it had alignment issues
- **Cursor:** breeze_cursors, 24px
- **Boot splash (Plymouth):** `cachyos` theme (`plymouth-set-default-theme`). Other installed options: `cachyos-bootanimation`, `spinner`, `bgrt`, …
- **Splash screen (KSplash):** CachyOS-Nord
- **Wallpaper:** `wallhaven-mlzoy1_3840x2160.png` from [wallhaven](https://wallhaven.cc), stored in `~/Pictures/wallpapers/`
- **Lock screen:** same image
- **Login screen:** same image. Plasma Login copies it to `/var/lib/plasmalogin/wallpapers/`
- **Panel:** single panel with Kickoff, icon tasks, system tray, clock, weather widget
- **Unused leftovers in `~/.local/share` (safe to clean up):** Otto & Orchis (global/plasma/aurorae/colors), McMojave-circle & Tela-circle icons, Vimix-cursors in `~/.icons`

## Other customization

- **Display scaling:** 125% on main monitor; XWayland apps scale themselves (`kwinrc [Xwayland] Scale=1.25`)
- **Gaming:** Steam games have `.desktop` launchers in `~/.local/share/applications`; `lsfg-vk` (Lossless Scaling frame-gen) UI installed
- **Services I rely on:** `ananicy-cpp`, `ufw`, `bluetooth`, `limine-snapper-sync`, `snapper-cleanup.timer`, `fstrim.timer`, `cachyos-rate-mirrors.timer`, user `arch-update.timer`
- **Screenshotting:** enabled 'Accept on click-and-release' in Spectacle, then made a custom shortcut for Super+Shift+S that runs `spectacle -r -b -c` (captures selection, adds to clipboard, closes Spectacle). Stored in `~/.local/share/applications/net.local.spectacle.desktop`. Spectacle's own launch shortcut stays on Print
- **Monitoring:** nvtop for detailed GPU usage, btop for detailed CPU usage (and less detailed GPU usage)

## Oddities / unknown-unknowns

- `/etc/plasmalogin.conf` has an `[Autologin]` section with only `Session=plasma` and no `User=`, so autologin is not actually active
- `plasma-org.kde.plasma.desktop-appletsrc` references `/home/lara/Imágenes/…`, a leftover from the CachyOS default layout. Harmless
- iGPU is enabled. Could disable in BIOS if it causes issues; leave it on if using it for display outputs or encoding
- **Ghostty "Launching Ghostty (Failed) / Remote peer disconnected":** systemd 262 refuses to start Ghostty's D-Bus service when it hasn't installed its SIGUSR2 handler yet (startup race, [ghostty#14398](https://github.com/ghostty-org/ghostty/issues/14398)). Workaround: `~/.config/systemd/user/app-com.mitchellh.ghostty.service.d/override.conf` with `Type=notify` and `ExecReload=/bin/kill -USR2 $MAINPID`, then `systemctl --user daemon-reload`. **Remove the override once Ghostty is 1.4+** (official fix ships there)

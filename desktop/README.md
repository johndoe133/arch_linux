# System Context — Desktop (`ekj-cachyOS`)

_Last surveyed: 2026-10-20_

See also: [CS2 + display tweaks](cs2-display-tweaks.md) (launch options, 500 Hz flicker fix).

## Hardware

- **Motherboard:** Gigabyte B650 EAGLE AX (BIOS F37, 2025-12-17)
- **CPU:** AMD Ryzen 7 9800X3D (8c/16t)
- **GPU:** AMD Radeon RX 9070-series (Navi 48), plus the iGPU (Radeon 610M) still enabled in BIOS
- **RAM:** 32GB
- **Storage:**
  - Lexar NM790 2TB (`nvme0n1`): Linux. 4GB FAT32 ESP at `/boot` + btrfs for everything else
  - WD_BLACK SN7100 1TB (`nvme1n1`): **Windows** (its own ESP + NTFS). Dual boot
- **Displays:** (both on the 9070 XT; the iGPU has no monitor attached)
  - DP-1: **Acer Predator X27U F5** OLED, 2560×1440 @ 500 Hz, 125% scale (main, left). Slow to show a picture at boot (see Limine menu below). **Quick Start Mode: On** (monitor menu → System; default Off) to make it wake faster
  - DP-2: **LG 27UP650** 4K @ 60 Hz, 125% scale (right). Replaced the Samsung SyncMaster BX2431 1080p. Was on the iGPU and flickered; fine on the dGPU (tentative). See [CS2 + display tweaks](cs2-display-tweaks.md)
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
- **Editor:** VS Code **OSS build** (`code` package, extensions from Open VSX). Settings tracked in [`vscode/`](../vscode/README.md) and symlinked to `~/.config/Code - OSS/User/settings.json`: editor, terminal, debug console, chat and commit-message box all use JetBrainsMono Nerd Font (`ttf-jetbrains-mono-nerd`). The OSS build has no Settings Sync, so signing in to GitHub does not sync settings; this repo does instead
- **Terminals installed:** Konsole, Alacritty, Ghostty (main)
- **Ghostty config:** `~/.config/ghostty/config.ghostty`: only `theme = TokyoNight Night` (matches Starship); everything else default (JetBrains Mono, no transparency, which I tried and didn't like). Check edits with `ghostty +validate-config`
- **Prompt:** Starship (`pacman -S starship`, not the curl script), loaded via `starship init fish | source` in `~/.config/fish/config.fish`. Uses the Tokyo Night preset (`~/.config/starship.toml`). The preset has no CachyOS icon, so it showed a 🎗️ ribbon emoji; fixed by adding `CachyOS = "󰣇"` (Arch logo) under `[os.symbols]`. Needs a Nerd Font in the terminal
- **Firewall:** ufw (enabled)
- **Locale:** `en_DK.UTF-8` with Danish number/time/currency formats, US keyboard layout

## Key versions (checked 2026-11-07, `checkupdates` = nothing pending)

- **Kernel:** installed `linux-cachyos` 7.3.3-1, **running 7.3.1-1** (built 2026-10-31). The mismatch is just a pending reboot, not a missing update. Check with `uname -r` vs `pacman -Q linux-cachyos`
- **Graphics:** `mesa` / `vulkan-radeon` (+ `lib32-` versions) 26.2.6-1, `linux-firmware` 20261019-1
- **Desktop:** KDE Plasma 6.8.2 (`kwin`, `plasma-desktop`, `plasma-login-manager`), `xorg-xwayland` 24.1.14
- **Boot:** `limine` 12.9.6, `mkinitcpio` 42.2, `systemd` 262.1; BIOS F37
- **Apps:** `lact` 0.10.1, `steam` 1.0.0.87, `ghostty` 1.4.0, `sdl3` 3.4.18, `sdl2-compat` 2.32.74
- ~1330 packages installed. Everything is current; if an assistant says to update, run `checkupdates` first

## Boot options

- **Kernel cmdline:** stock: `quiet nowatchdog splash rw rootflags=subvol=/@ root=UUID=…`. Edit in `/etc/default/limine` (`KERNEL_CMDLINE[default]+=`), **not** in `/boot/limine.conf`, which gets regenerated
- **Secure Boot:** set up with `sbctl`, using my own keys + **Microsoft vendor keys enrolled** (Windows and GPU option ROMs need them). `ENABLE_ENROLL_LIMINE_CONFIG=yes` signs/enrolls the Limine config hash. ⚠️ If a BIOS update or CMOS reset wipes keys → re-run `sudo sbctl enroll-keys -m` (or turn Secure Boot off temporarily)
- **Limine menu** (`/boot/limine.conf`): `timeout: 120`, `default_entry: 2`, `remember_last_entry: yes`, CachyOS Limine theme. The long timeout is because the OLED monitor takes so long to show a picture that I'd miss the menu otherwise
  - **Option, not done yet:** drop to `timeout: 5` and boot Windows straight from Linux with `sudo efibootmgr --bootnext 0000 && systemctl reboot` (`Boot0000` = Windows Boot Manager), which skips Limine once. For recovery, press an arrow key right after power-on to stop the countdown. Could also check the monitor's menu for Deep Sleep / DP power saving / auto input detection, which slow down waking

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

- **Display scaling:** 125% on both monitors; XWayland apps scale themselves (`kwinrc [Xwayland] Scale=1.25`)
- **Gaming:** Steam games have `.desktop` launchers in `~/.local/share/applications`; `lsfg-vk` (Lossless Scaling frame-gen) UI installed
- **Services I rely on:** `ananicy-cpp`, `ufw`, `bluetooth`, `limine-snapper-sync`, `snapper-cleanup.timer`, `fstrim.timer`, `cachyos-rate-mirrors.timer`, user `arch-update.timer`
- **Screenshotting:** enabled 'Accept on click-and-release' in Spectacle, then made a custom shortcut for Super+Shift+S that runs `spectacle -r -b -c` (captures selection, adds to clipboard, closes Spectacle). Stored in `~/.local/share/applications/net.local.spectacle.desktop`. Spectacle's own launch shortcut stays on Print
- **Monitoring:** nvtop for detailed GPU usage, btop for detailed CPU usage (and less detailed GPU usage)

## Oddities / unknown-unknowns

- `/etc/plasmalogin.conf` has an `[Autologin]` section with only `Session=plasma` and no `User=`, so autologin is not actually active
- `plasma-org.kde.plasma.desktop-appletsrc` references `/home/lara/Imágenes/…`, a leftover from the CachyOS default layout. Harmless
- iGPU is enabled. Could disable in BIOS if it causes issues; leave it on if using it for display outputs or encoding
- **Ghostty "Launching Ghostty (Failed) / Remote peer disconnected":** systemd 262 refuses to start Ghostty's D-Bus service when it hasn't installed its SIGUSR2 handler yet (startup race, [ghostty#14398](https://github.com/ghostty-org/ghostty/issues/14398)). Workaround: `~/.config/systemd/user/app-com.mitchellh.ghostty.service.d/override.conf` with `Type=notify` and `ExecReload=/bin/kill -USR2 $MAINPID`, then `systemctl --user daemon-reload`. **Remove the override once Ghostty is 1.4+** (official fix ships there)

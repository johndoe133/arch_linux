# Desktop: CS2 + display tweaks (CachyOS / KDE Wayland)

Hardware: Ryzen 7 9800X3D, RX 9070 XT, Acer X27U 500 Hz OLED (DP-1),
LG 27UP650 4K 60 Hz (secondary, on the 9070 XT too, DP-2)

## CS2 (Steam launch options)
SDL_VIDEO_DRIVER=wayland SDL_VIDEO_WAYLAND_SCALE_TO_DISPLAY=1 SDL_VIDEO_DISPLAY_PRIORITY=DP-1 %command%
- Wayland: CS2 defaults to XWayland (refresh showed "0 Hz"); native Wayland fixes that.
- SCALE_TO_DISPLAY: fixes resolution capped below 1440p due to KDE scaling.
- DISPLAY_PRIORITY: stops the game spanning both monitors.
- If it breaks: remove the whole line to go back to XWayland.

## KDE display settings
- Predator: Adaptive Sync = Automatic (VRR only in fullscreen games).

## 500 Hz desktop flicker: RX 9070 XT memory-clock bug
- Known RDNA4 bug (freedesktop drm/amd #4753): VRAM clock switching
  glitches at high refresh / multi-monitor. Not present at 360 Hz.
- Fix via LACT (lactd service enabled). Seems to have fixed the window-move
  tearing on the main monitor at 500 Hz too, not just the flicker.
  - Performance Level: Manual
  - Power states: VRAM state 0 (192 MHz) unticked; all GPU states ticked
  - Fallback if still flickering: Performance Level = High (+~25-30 W idle)
- Undo: Performance Level = Automatic, or `sudo systemctl disable --now lactd`
- Re-test now and then after kernel/Mesa updates in case it's fixed upstream.

## LG 4K: moved from iGPU to dGPU (tentatively fixed)
- Was on the motherboard iGPU (Radeon 610M, DP-5) to keep the 2nd monitor off the 9070 XT.
  It flickered occasionally there (iGPU also logged repeated `LTTPR ... invalid lane count`
  DP link-training errors).
- Now plugged into the 9070 XT (DP-2, both monitors on card1). Flicker seems gone
  (tentative; keep watching). The LACT VRAM-state fix above is what makes two monitors
  on the dGPU OK.
- BIOS: iGPU still Enabled, primary display = PCIe. Could set iGPU to Auto/Disabled now
  that nothing uses it (frees the 512 MiB carve-out); not done.
- If flicker returns or the VRAM-clock bug comes back with both monitors on the dGPU:
  go back to the iGPU and try a different DP cable, then `amdgpu.sg_display=0` (see below).

## Not applied (only needed if the 4K goes back on the iGPU)
- Kernel param `amdgpu.sg_display=0` in /etc/default/limine, then `sudo limine-update`.
  Reports say it fixes Raphael iGPU flicker, but mixed results and none for the 610M.

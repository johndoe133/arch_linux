# Hibernate / Suspend Investigation — ASUS ROG Zephyrus G15 (CachyOS)

**Date of session:** 2026-08-17
**Status:** ✅ Manual hibernate working · ⏳ Suspend power draw unresolved · ⏳ Lid-close automation not yet configured

---

## 1. Objective

Reduce battery drain when the lid is closed. Original request was "hibernate on lid close"; agreed target became **Mac-style two-tier behavior** (`suspend-then-hibernate`): instant RAM-based wake for short closures, automatic hibernate after a delay.

---

## 2. Relevant System Facts

| Item | Value |
|---|---|
| Model | ASUS ROG Zephyrus G15 GA503QR-211.ZG15 |
| CPU | AMD Ryzen 9 (Fam17h+, Cezanne/Renoir platform) |
| GPU (dGPU) | NVIDIA RTX 3070 @ `0000:01:00.0` |
| GPU (iGPU) | AMD Renoir `0x1002:0x1638` @ `0000:06:00.0` |
| Optimus type | **MUX-less** — internal eDP panel wired to iGPU only |
| RAM | 15Gi usable |
| OS | CachyOS (Arch-based), kernel `7.1.8-1-cachyos` (LTS `6.18.42-1-cachyos-lts` also installed) |
| systemd | `261.2-1-arch` |
| NVIDIA driver | **Open** kernel module `610.57.04` |
| DE / Session | KDE Plasma on Wayland |
| Bootloader | Limine |
| Root FS | ext4, `UUID=2b44456f-dc52-489f-8d2a-c2453136cee1` |
| Root device | `nvme0n1` |
| Firmware sleep support | `S0 S4 S5` — **no S3** |

---

## 3. Baseline Diagnostics (pre-change)

```
$ cat /sys/power/mem_sleep
[s2idle]                      # no "deep"/S3 available at all

$ swapon --show
NAME       TYPE      SIZE USED PRIO
/dev/zram0 partition  15G 4,8M  100    # RAM-backed only → cannot hold hibernation image

$ free -h
Mem:  15Gi total, 3,4Gi used
Swap: 15Gi (zram)

$ cat /proc/cmdline
quiet nowatchdog splash rw root=UUID=2b44456f-... amdgpu.dcdebugmask=0x40000

$ systemctl status nvidia-{suspend,hibernate,resume}.service
all three: loaded but disabled / inactive (dead)
```

**Conclusions drawn:**
- No S3 means suspend will always be `s2idle` → hibernate is the only true low-drain option.
- zram cannot back hibernation → disk swap required.
- NVIDIA sleep hooks were not enabled.

---

## 4. Changes Applied (Attempt 1)

| # | Change | Detail |
|---|---|---|
| 1 | Created disk swapfile | `dd` 20 GiB → `/swapfile`, `chmod 600`, `mkswap`, `swapon` |
| 2 | fstab entry | `/swapfile none swap defaults 0 0` |
| 3 | mkinitcpio | Added `resume` hook to `HOOKS` |
| 4 | Computed resume offset | `filefrag` → `220407808` |
| 5 | Limine cmdline | **Intended** to add `resume=UUID=... resume_offset=...` — *never actually landed in the file* |
| 6 | NVIDIA modprobe conf | `/etc/modprobe.d/nvidia-power-management.conf`:<br>`NVreg_PreserveVideoMemoryAllocations=1`<br>`NVreg_TemporaryFilePath=/var/tmp` |
| 7 | Enabled services | `nvidia-suspend`, `nvidia-hibernate`, `nvidia-resume` |
| 8 | Regenerated | `limine-mkinitcpio` / `mkinitcpio -P` |

**Collateral damage:** Some prior boot customizations were lost during regeneration — notably the KDE post-login splash setting. Needs restoring.

---

## 5. Attempt 1 Result — FAILURE

`systemctl hibernate` powered the machine off. On power-on: **black screen, no response to keys or power button** → forced power-off (2nd forced power-off of the session).

### 5.1 Log evidence — hibernate-out boot (`-b -2`, started 17:33:49)

```
17:33:54  kernel: Adding 20971516k swap on /swapfile. Priority:-1 extents:14 across:22634492k SS
17:33:56  kernel: Adding 15751164k swap on /dev/zram0. Priority:100 extents:1
17:34:33  systemd[1]: Starting NVIDIA system hibernate actions...
17:34:33  systemd[1]: nvidia-hibernate.service: Deactivated successfully.
17:34:33  systemd[1]: Finished NVIDIA system hibernate actions.
17:34:33  systemd-sleep[2924]: User sessions remain unfrozen on explicit request
                               ($SYSTEMD_SLEEP_FREEZE_USER_SESSIONS=0).
17:34:33  systemd-sleep[2924]: This is not recommended, and might result in unexpected behavior...
17:34:33  systemd-sleep[2924]: Performing sleep operation 'hibernate'...
17:34:33  kernel: PM: Image not found (code -16)      # expected — no prior image existed
17:34:33  kernel: PM: hibernation: hibernation entry
```

Also observed this boot:
```
kernel: zswap: loaded using pool zstd
kernel: nvidia 0000:01:00.0: [drm] Cannot find any crtc or sizes
kernel: amdgpu 0000:06:00.0: Runtime PM not available
```

### 5.2 Log evidence — failed resume boot (`-b -1`, started 17:35:19)

```
17:35:19  Command line: quiet nowatchdog splash rw root=UUID=2b44456f-... amdgpu.dcdebugmask=0x40000
          # ← note: NO resume= / resume_offset= present

17:35:19  systemd-hibernate-resume-generator[180]: Reported hibernation image:
          ID=cachyos kernel=7.1.8-1-cachyos UUID=2b44456f-... offset=220407808
          # ← offset WAS correctly communicated, via EFI variable, not cmdline

17:35:19  kernel: PM: Image not found (code -22)      # EINVAL — location read, image rejected as invalid

17:35:22  kernel: amdgpu ... initializing kernel modesetting (RENOIR ...)
17:35:22  kernel: amdgpu 0000:06:00.0: Runtime PM not available
17:35:22  kernel: fbcon: amdgpudrmfb (fb0) is primary device

17:35:28  kernel: amdgpu 0000:06:00.0: [drm] Mode Validation Warning: Unknown Status failed validation.  ×4
17:35:33  kernel: amdgpu 0000:06:00.0: [drm] Mode Validation Warning: Unknown Status failed validation.  ×2
17:41:34  kernel: amdgpu 0000:06:00.0: [drm] Mode Validation Warning: Unknown Status failed validation.  ×2
17:41:36  systemd-shutdown[1]: Syncing filesystems and block devices.   # forced power-off
```

### 5.3 Interpretation

- Resume was **attempted and rejected** (`-22` = invalid image header), not skipped.
- Kernel then silently fell back to a **normal cold boot** — Wi-Fi associated, real network traffic flowed, system ran ~6 minutes. **The machine was never hung.**
- The black screen was a **separate display bug**: repeated `Mode Validation Warning` from `amdgpu` at exactly the moments keys/power were pressed. Since the panel is iGPU-driven (MUX-less), this is an **amdgpu mode-set failure**, not an NVIDIA display issue.
- Journald reported `user-1000.journal corrupted or uncleanly shut down` — consequence of the forced power-off, not FS damage per se.

---

## 6. Verification Round (read-only)

```
$ sudo filefrag -v /swapfile | head -n 6
File size of /swapfile is 21474836480 (5242880 blocks of 4096 bytes)
 ext:  logical_offset:   physical_offset:  length:  expected:  flags:
   0:      0..  40959:  220407808..220448767:  40960:
   1:  40960..  43007:  220454912..220456959:   2048:  220448768:
   2:  43008.. 262143:  220506112..220725247: 219136:  220456960:

$ cat /sys/power/resume          → 0:0
$ cat /sys/power/resume_offset   → 0

$ cat /etc/default/limine | grep -i cmdline
KERNEL_CMDLINE[default]+="quiet nowatchdog splash rw root=UUID=2b44456f-... amdgpu.dcdebugmask=0x40000"

$ sudo cat /boot/limine.conf
# both kernel entries carry the same cmdline — no resume= anywhere
```

**Key result:** offset `220407808` from `filefrag` **matches exactly** what the resume generator reported. The offset was never wrong.

---

## 7. Hypotheses Tested and Discarded

| Hypothesis | Verdict | Evidence |
|---|---|---|
| Missing `resume=` / `resume_offset=` on kernel cmdline | ❌ Not required | systemd 261 uses the `HibernateLocation` EFI variable to pass device+offset to `/sys/power/resume`; generator log proves it worked without cmdline params |
| `awk 'NR==4{print $4}'` misparsed the offset | ❌ Offset was correct | `filefrag` extent 0 physical offset = `220407808` = value in generator log |
| Limine config was "reverted", wiping `resume=` | ❌ Never present | `/etc/default/limine` shows the edit never landed; nothing was reverted |
| Needed `nvidia_drm.modeset=1` / `fbdev=1` for black screen | ❌ Largely irrelevant | MUX-less: panel is on amdgpu, not NVIDIA. Open driver 610.x initializes `nvidia-drm` regardless |
| System hung/panicked on resume | ❌ Never hung | Full boot completed; Wi-Fi associated; 6 min of live kernel logs |
| Machine "woke itself up" from hibernate | ❌ Never slept | `pm_test` was still armed — see §9 |

---

## 8. Root Cause Identified

**`SYSTEMD_SLEEP_FREEZE_USER_SESSIONS=0`**, shipped as a drop-in by the Arch/CachyOS `nvidia-utils` packaging as a workaround for older proprietary drivers.

With user sessions left unfrozen, Wayland clients and the compositor keep running while the GPU stack is being torn down for the snapshot. On a MUX-less hybrid setup this races: the compositor keeps driving the amdgpu-attached panel mid-teardown, producing:
- a corrupt/incomplete hibernation image → `PM: Image not found (code -22)` on resume, and
- repeated `amdgpu ... Mode Validation Warning` → the black, unrecoverable display.

Known upstream conflict, not a local misconfiguration. Community reports are mixed — some users report the override does **not** fix it and fall back to "lock on lid close + manual hibernate only."

---

## 9. Fix Applied

**File:** `/etc/systemd/system/systemd-hibernate.service.d/10-freeze-sessions.conf`

```ini
[Service]
Environment=SYSTEMD_SLEEP_FREEZE_USER_SESSIONS=true
```

Followed by `sudo systemctl daemon-reload`.

> **Note:** `sudoedit` failed (`/usr/bin/vi: command not found` — no `vi`/`EDITOR` set). Used `sudo nano` instead.

### 9.1 Safe dry-run testing via `pm_test`

```bash
echo platform | sudo tee /sys/power/pm_test
echo disk     | sudo tee /sys/power/state
```

Result — **clean cycle, no mode-validation warnings**:
```
20:47:00  amdgpu 0000:06:00.0: MODE2 reset
20:47:00  amdgpu 0000:06:00.0: [drm] PCIE GART of 1024M enabled.
20:47:00  amdgpu 0000:06:00.0: PSP is resuming...
20:47:00  amdgpu 0000:06:00.0: SMU is resuming...
20:47:00  amdgpu 0000:06:00.0: SMU is resumed successfully!
20:47:00  amdgpu 0000:06:00.0: [drm] DMUB hardware initialized: version=0x0101002B
          (all VM inv engines / rings re-initialized normally)
```

### 9.2 ⚠️ The `pm_test` trap — important gotcha

**`/sys/power/pm_test` does not reset itself after use.** A subsequent *real* `systemctl hibernate` silently ran as another dry-run. Telltale line:

```
PM: hibernation: hibernation debug: Waiting for 5 second(s).
```

Dry-run cycles recorded:
```
20:46:37  PM: hibernation: hibernation entry
20:47:00  PM: hibernation: Allocated 1552131 pages for snapshot
20:47:00  PM: hibernation: Allocated 6208524 kbytes in 7.12 seconds (871.98 MB/s)
20:47:00  PM: hibernation: hibernation debug: Waiting for 5 second(s).
20:47:01  PM: hibernation: hibernation exit

20:48:04  PM: Image not found (code -16)
20:48:04  PM: hibernation: hibernation entry
20:48:26  PM: hibernation: Allocated 1468616 pages for snapshot
20:48:26  PM: hibernation: Allocated 5874464 kbytes in 7.35 seconds (799.24 MB/s)
20:48:26  PM: hibernation: hibernation debug: Waiting for 5 second(s).
20:48:26  PM: hibernation: hibernation exit
```

Confirmed no real power-off occurred:
```
$ uptime -s                              → 2026-08-17 17:41:46   (unchanged)
$ cat /proc/sys/kernel/random/boot_id     → e87c5f81-95c2-4614-9814-8f5160b180ae
```

The "it woke up on its own after a bit" observation = the built-in **5-second `pm_test` timer**, not an external wake source.

### 9.3 Real hibernate — ✅ SUCCESS

```bash
echo none | sudo tee /sys/power/pm_test
systemctl hibernate
```

Machine **fully powered off**. Powered on via power button → full UEFI + Limine + kernel boot → **session fully restored, all applications and windows intact.**

---

## 10. Journalctl Gotcha Worth Remembering

A successful hibernate/resume **does not create a new boot entry**. The kernel restores its own `boot_id`, so journald treats it as a continuation of the same boot. `journalctl -b -1` will keep pointing at *older* boots, not the resume. Use time-ranged queries instead:

```bash
journalctl -k --since "HH:MM:SS" --until "HH:MM:SS"
```

Boot index reference from this session:
```
-2  60ca598c...  17:33:49   hibernate-out boot
-1  eac3aedc...  17:35:19   failed resume boot (black screen)
 0  e87c5f81...  17:41:48   current session
```

---

## 11. Current Configuration State

**Files created/modified:**
- `/swapfile` — 20 GiB, ext4, 14 extents, priority `-1`
- `/etc/fstab` — swapfile entry added
- `/etc/mkinitcpio.conf` — `resume` hook added
- `/etc/modprobe.d/nvidia-power-management.conf` — `NVreg_PreserveVideoMemoryAllocations=1`, `NVreg_TemporaryFilePath=/var/tmp`
- `/etc/systemd/system/systemd-hibernate.service.d/10-freeze-sessions.conf` — freeze override ✅ **the actual fix**

**Services enabled:** `nvidia-suspend.service`, `nvidia-hibernate.service`, `nvidia-resume.service`

**Not changed:** kernel cmdline (no `resume=` — correctly unnecessary), lid-switch behavior (still default), KDE power settings

**Swap layout:** zram0 @ prio 100 (daily swapping) + /swapfile @ prio -1 (hibernation target)

---

## 12. Open Issues

### 12.1 Suspend battery drain — the original problem, still unaddressed
No S3 available; `s2idle` only. Suspected contributors, in priority order:

1. **RTX 3070 not entering runtime-suspend / D3cold.** Highest-probability culprit on hybrid laptops; comparable systems report roughly doubled battery life once the dGPU is genuinely powered off.
2. **AMD s2idle prerequisites possibly unmet** — BIOS must be in Modern Standby / "Windows" sleep mode; `amd_pmc` binding, ASPM policy, and `xhci_hcd` usage all need verifying.
3. **Devices/USB not reaching low-power states** during suspend.
4. `amdgpu 0000:06:00.0: Runtime PM not available` — appears every boot; worth understanding.

### 12.2 Other outstanding items
- Plain `systemctl suspend` **not yet tested** with the freeze fix in place.
- `suspend-then-hibernate` not tested; note it performs an unattended **intermediate wake** before hibernating — hits both the suspend-resume race *and* the hibernate-write path back to back.
- Lid-close automation not configured.
- **`fsck` still pending** after two forced power-offs (requires live USB; root cannot be checked mounted rw).
- **KDE post-login splash setting lost** during `mkinitcpio` regeneration — needs restoring.
- `zswap: loaded using pool zstd` active alongside zram — interaction with hibernation not yet evaluated.
- Hibernate resume inherently goes through the full UEFI+Limine boot sequence. This is intrinsic to S4 and cannot be avoided; it is *not* a misconfiguration.

---

## 13. Planned Next Steps

1. **Confirm hibernate reliability** — run `systemctl hibernate` 2–3 more times under varied workloads (browser open/closed, GPU active/idle).
2. **Test plain suspend** — `systemctl suspend`, verify instant wake with no black screen and no `Mode Validation Warning`.
3. **Investigate suspend drain:**
   ```bash
   cat /proc/driver/nvidia/gpus/*/power
   paru -S envycontrol && sudo envycontrol --query-gpu-mode
   paru -S amd-debug-tools && sudo amd_s2idle.py
   powertop --auto-tune          # after a suspend/resume cycle
   cat /proc/acpi/wakeup          # audit armed wake sources
   ```
4. **Check BIOS** for an OS-type / sleep-mode (Modern Standby vs legacy) toggle.
5. **Test `suspend-then-hibernate`** with a short delay (~2 min) while observing at the desk — not on lid close.
6. **Only then** wire lid-close behavior:
   ```ini
   # /etc/systemd/logind.conf.d/10-lid.conf
   [Login]
   HandleLidSwitch=suspend-then-hibernate
   HandleLidSwitchExternalPower=suspend-then-hibernate
   HandleLidSwitchDocked=ignore
   ```
   ```ini
   # /etc/systemd/sleep.conf.d/10-hibernate-delay.conf
   [Sleep]
   HibernateDelaySec=45min
   ```
   Ensure KDE's own lid action is set to "Do nothing" to avoid conflict.
7. **Run `fsck`** from live USB: `lsblk -f` → `sudo fsck -f /dev/nvme0n1pX` (unmounted).
8. **Restore** the lost KDE splash configuration.

### Fallback if automation proves unreliable
Lock screen on lid close **without** auto-suspend; hibernate manually before travel. This is the workaround other MUX-less NVIDIA-hybrid Wayland users have settled on.

---

## 14. Rollback Procedure (if hibernate is later abandoned)

```bash
sudo rm /etc/systemd/system/systemd-hibernate.service.d/10-freeze-sessions.conf
sudo swapoff /swapfile
sudo rm /swapfile
# remove /swapfile line from /etc/fstab
# remove 'resume' hook from /etc/mkinitcpio.conf
sudo systemctl disable nvidia-suspend.service nvidia-hibernate.service nvidia-resume.service
sudo mkinitcpio -P
```

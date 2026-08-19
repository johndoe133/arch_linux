# Suspend & Hibernate Investigation — Master Document
## ASUS ROG Zephyrus G15 (GA503QR) · CachyOS · KDE Plasma Wayland

**Consolidates:** Session 1 (2026-08-17, §1–§14) · Session 2 (2026-08-18, §15–§16) · Session 3 (2026-08-18 evening → 2026-08-19 morning, §17–§26)
**Compiled:** 2026-08-19
**Supersedes:** the two prior standalone documents

---

## How to read this document

Sessions 1 and 2 are reproduced **in full and unedited**, because the reasoning chain matters even where the conclusions turned out to be wrong. Where later evidence overturned an earlier claim, a correction banner appears inline:

> ⛔ **SUPERSEDED — see §X.Y.** Short statement of what is actually true.

Part V is a consolidated corrections register if you only want the deltas.

---

## Status dashboard

| Goal | Status | Evidence |
|---|---|---|
| **Hibernate works reliably** | ✅ **SOLVED** | 4/4 clean tests, §21 |
| **Hibernate survives dGPU-active state** | ✅ **SOLVED** | Test 3, §21.3 |
| **Hibernate survives long durations** | ✅ **SOLVED** | Test 4, 28 min, §21.4 |
| **User-session freeze during sleep** | ✅ **FIXED** (was never active before) | §22 |
| **Suspend-then-hibernate, manual trigger** | ✅ **WORKS** | §24.2 |
| **Suspend-then-hibernate, timer-driven** | ⛔ **BLOCKED — firmware** | §25 |
| **dGPU enters runtime D3** | ✅ Fixed in S2 | §15.7, re-verified §23 |
| **s2idle drain reduced** | ❌ **NO MEASURABLE CHANGE** | §26 |
| **Lid automation** | ⬜ Config written, **not yet applied** | Part VI |
| **`fsck` after forced power-offs** | ⬜ **OUTSTANDING — 5 power-offs deep** | §27.4 |
| **KDE splash restoration** | ⬜ Outstanding since Session 1 | §12.2 |

---
---

# PART 0 — SYSTEM REFERENCE

## §0.1 Hardware & software facts

| Item | Value |
|---|---|
| Model | ASUS ROG Zephyrus G15 **GA503QR-211.ZG15** (2021) |
| CPU | AMD Ryzen 9 5900HS (Fam17h+, **Cezanne**) |
| dGPU | NVIDIA RTX 3070 Mobile / Max-Q (GA104M) @ `0000:01:00.0` |
| dGPU audio fn | NVIDIA GA104 HDA @ `0000:01:00.1` |
| iGPU | AMD Renoir `0x1002:0x1638` @ `0000:06:00.0` |
| Optimus type | **MUX-less** — internal eDP panel wired to iGPU only |
| UEFI GPU mode | Switchable Graphics (not discrete-only) |
| RAM | 16 GB (15 Gi usable) |
| Storage | 1 TB NVMe, single drive (`nvme0n1`) |
| Display | 15.6" QHD |
| Ethernet | Realtek RTL8125B @ `0000:03:00.0` |
| Wi-Fi | `wlan0`, 192.168.1.220/24 |
| OS | CachyOS (Arch-based, rolling) |
| Kernel | `7.1.8-1-cachyos` (LTS `6.18.42-1-cachyos-lts` also installed) |
| systemd | `261.2-1-arch` |
| NVIDIA driver | **Open** kernel module `610.57.04` (built 2026-08-11) |
| DE / session | KDE Plasma on **Wayland** |
| Display manager | **Plasma Login Manager (PLM)** — SDDM fork |
| Bootloader | **Limine** |
| Root FS | **ext4**, `UUID=2b44456f-dc52-489f-8d2a-c2453136cee1` |
| Boot mode | UEFI only, Fast Boot enabled |
| Firewall | **`ufw` active** (discovered §18.3) |
| AUR helper | paru |

## §0.2 Battery facts — **CORRECTED**

| Item | Value |
|---|---|
| `energy_now` @ 100% | **64,724,000 µWh ≈ 64.7 Wh** |
| Assumed capacity in §15.2 | ~~90 Wh~~ ❌ wrong |
| Design capacity | GA503QR ships 90 Wh → implies **~28% wear** (2021 machine) |
| 1% of capacity | **~647 mWh** |
| `BAT0/alarm` | `8992000` — **populated**, so ACPI `_BTP` trip point exists |
| CMOS/RTC cell | **No discrete coin cell** — RTC powered from main pack. `batt_status: dead` in `/proc/driver/rtc` is a **false reading**, disregard (§25.5) |

**Consequence:** every watt figure in §15.2 was overstated by ~39%. See §26.3 for the recomputed table.

## §0.3 Sleep capability matrix

```
$ cat /sys/power/mem_sleep
[s2idle]                  # no "deep"/S3 available at all
Firmware sleep states:  S0  S4  S5      (no S3)
```

| State | Available | Notes |
|---|---|---|
| s2idle (S0ix) | ✅ only option | ~95% hardware-sleep residency, ~0.8 W measured |
| S3 (deep) | ❌ not exposed | Potentially unlockable via DSDT patch — see §28.4 |
| S4 (hibernate) | ✅ **working** | Solved Session 3 |
| S5 (poweroff) | ✅ | — |
| RTC wake **from s0ix** | ⛔ **BROKEN — firmware** | §25 |
| RTC wake while awake | ✅ works | §25.4 |

---
---

# PART I — SESSION 1 (2026-08-17)
## Hibernate Investigation

**Status at close of session:** ✅ "Manual hibernate working" *(⛔ later found unverified — see §21.0)* · ⏳ Suspend power draw unresolved · ⏳ Lid-close automation not configured

---

## §1 Objective

Reduce battery drain when the lid is closed. Original request was "hibernate on lid close"; agreed target became **Mac-style two-tier behavior** (`suspend-then-hibernate`): instant RAM-based wake for short closures, automatic hibernate after a delay.

---

## §2 Relevant System Facts

See §0.1. Session-1-specific readings:

```
$ systemctl status nvidia-{suspend,hibernate,resume}.service
all three: loaded but disabled / inactive (dead)
```

---

## §3 Baseline Diagnostics (pre-change)

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
```

**Conclusions drawn:**
- No S3 means suspend will always be `s2idle` → hibernate is the only true low-drain option.
- zram cannot back hibernation → disk swap required.
- NVIDIA sleep hooks were not enabled.

---

## §4 Changes Applied (Attempt 1)

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

> ⛔ **Row 6 is the origin of the Session 3 failure.** `NVreg_PreserveVideoMemoryAllocations=1` is a **430–590-series** parameter. On the 610 open driver it forces a legacy procfs suspend path that does not exist, causing `nv_pmops_freeze` to return `-5` and every resume to be discarded. See §19–§20.

**Collateral damage:** Some prior boot customizations were lost during regeneration — notably the KDE post-login splash setting. Needs restoring.

---

## §5 Attempt 1 Result — FAILURE

`systemctl hibernate` powered the machine off. On power-on: **black screen, no response to keys or power button** → forced power-off (2nd forced power-off of the session).

### §5.1 Log evidence — hibernate-out boot (`-b -2`, started 17:33:49)

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

### §5.2 Log evidence — failed resume boot (`-b -1`, started 17:35:19)

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

### §5.3 Interpretation

- Resume was **attempted and rejected** (`-22` = invalid image header), not skipped.
- Kernel then silently fell back to a **normal cold boot** — Wi-Fi associated, real network traffic flowed, system ran ~6 minutes. **The machine was never hung.**
- The black screen was a **separate display bug**: repeated `Mode Validation Warning` from `amdgpu` at exactly the moments keys/power were pressed. Since the panel is iGPU-driven (MUX-less), this is an **amdgpu mode-set failure**, not an NVIDIA display issue.
- Journald reported `user-1000.journal corrupted or uncleanly shut down` — consequence of the forced power-off, not FS damage per se.

> ⛔ **The `Mode Validation Warning` attribution is wrong.** These lines appeared again on 2026-08-18 during a cold boot that reached a fully working desktop (§19.1). They are **benign noise**, not a black-screen cause. Moved to the discarded-hypotheses register (§V).

---

## §6 Verification Round (read-only)

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

## §7 Hypotheses Tested and Discarded (Session 1)

| Hypothesis | Verdict | Evidence |
|---|---|---|
| Missing `resume=` / `resume_offset=` on kernel cmdline | ❌ Not required | systemd 261 uses the `HibernateLocation` EFI variable to pass device+offset to `/sys/power/resume`; generator log proves it worked without cmdline params |
| `awk 'NR==4{print $4}'` misparsed the offset | ❌ Offset was correct | `filefrag` extent 0 physical offset = `220407808` = value in generator log |
| Limine config was "reverted", wiping `resume=` | ❌ Never present | `/etc/default/limine` shows the edit never landed; nothing was reverted |
| Needed `nvidia_drm.modeset=1` / `fbdev=1` for black screen | ❌ Largely irrelevant | MUX-less: panel is on amdgpu, not NVIDIA. Open driver 610.x initializes `nvidia-drm` regardless |
| System hung/panicked on resume | ❌ Never hung | Full boot completed; Wi-Fi associated; 6 min of live kernel logs |
| Machine "woke itself up" from hibernate | ❌ Never slept | `pm_test` was still armed — see §9.2 |

---

## §8 Root Cause Identified *(Session 1's conclusion)*

> ⛔ **SUPERSEDED — see §20 and §22.** This diagnosis was wrong on two counts. (a) The real cause of hibernate failure was `NVreg_PreserveVideoMemoryAllocations` on a 610 driver. (b) The freeze override described in §9 **was never actually in effect** until 2026-08-18 18:06 due to a drop-in filename collision — so it cannot have fixed anything. It is retained below as the original reasoning.

**`SYSTEMD_SLEEP_FREEZE_USER_SESSIONS=0`**, shipped as a drop-in by the Arch/CachyOS `nvidia-utils` packaging as a workaround for older proprietary drivers.

With user sessions left unfrozen, Wayland clients and the compositor keep running while the GPU stack is being torn down for the snapshot. On a MUX-less hybrid setup this races: the compositor keeps driving the amdgpu-attached panel mid-teardown, producing:
- a corrupt/incomplete hibernation image → `PM: Image not found (code -22)` on resume, and
- repeated `amdgpu ... Mode Validation Warning` → the black, unrecoverable display.

Known upstream conflict, not a local misconfiguration. Community reports are mixed — some users report the override does **not** fix it and fall back to "lock on lid close + manual hibernate only."

---

## §9 Fix Applied *(Session 1)*

**File:** `/etc/systemd/system/systemd-hibernate.service.d/10-freeze-sessions.conf`

```ini
[Service]
Environment=SYSTEMD_SLEEP_FREEZE_USER_SESSIONS=true
```

Followed by `sudo systemctl daemon-reload`.

> ⛔ **This file never took effect.** It lost the drop-in merge to `/usr/lib/systemd/system/systemd-hibernate.service.d/10-nvidia-no-freeze-session.conf`. Both begin `10-`; the tiebreak is lexicographic on the remainder, and **`f` sorts before `n`**. Being in `/etc` confers no advantage — directory precedence applies only to files with *identical* names. Proven by the log line still reading `=0` on 2026-08-18 at 17:57. See §22.

> **Note:** `sudoedit` failed (`/usr/bin/vi: command not found` — no `vi`/`EDITOR` set). Used `sudo nano` instead.

### §9.1 Safe dry-run testing via `pm_test`

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

### §9.2 ⚠️ The `pm_test` trap — important gotcha

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

### §9.3 Real hibernate — "✅ SUCCESS"

```bash
echo none | sudo tee /sys/power/pm_test
systemctl hibernate
```

Machine **fully powered off**. Powered on via power button → full UEFI + Limine + kernel boot → **session fully restored, all applications and windows intact.**

> ⛔ **This "success" is now considered unverified and probably false.** No `boot_id` or `uptime -s` comparison was performed. "Applications and windows restored" is exactly what **KDE session restore** produces on a normal cold boot. Given that the identical configuration failed deterministically with `-5` on 2026-08-18 (§19), and that the `-5` failure *also* leaves a working desktop with restored apps, §9.3 was most likely a cold boot misread as a resume. **First verified hibernate is 2026-08-18 17:11:24 (§21.1).**

---

## §10 Journalctl Gotcha Worth Remembering

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

**Corollary discovered in Session 3:** because `boot_id` is restored, `journalctl -b 0` is the *correct* query after a successful resume — the resume is a continuation of boot 0, not a new boot.

---

## §11 Configuration State at end of Session 1

**Files created/modified:**
- `/swapfile` — 20 GiB, ext4, 14 extents, priority `-1`
- `/etc/fstab` — swapfile entry added
- `/etc/mkinitcpio.conf` — `resume` hook added
- `/etc/modprobe.d/nvidia-power-management.conf` — `NVreg_PreserveVideoMemoryAllocations=1`, `NVreg_TemporaryFilePath=/var/tmp`
- `/etc/systemd/system/systemd-hibernate.service.d/10-freeze-sessions.conf` — freeze override ~~✅ **the actual fix**~~ ⛔ inert

**Services enabled:** `nvidia-suspend.service`, `nvidia-hibernate.service`, `nvidia-resume.service`

**Not changed:** kernel cmdline (no `resume=` — correctly unnecessary), lid-switch behavior (still default), KDE power settings

**Swap layout:** zram0 @ prio 100 (daily swapping) + /swapfile @ prio -1 (hibernation target)

---

## §12 Open Issues (as of Session 1)

### §12.1 Suspend battery drain — the original problem, still unaddressed
No S3 available; `s2idle` only. Suspected contributors, in priority order:

1. **RTX 3070 not entering runtime-suspend / D3cold.** Highest-probability culprit on hybrid laptops; comparable systems report roughly doubled battery life once the dGPU is genuinely powered off.
2. **AMD s2idle prerequisites possibly unmet** — BIOS must be in Modern Standby / "Windows" sleep mode; `amd_pmc` binding, ASPM policy, and `xhci_hcd` usage all need verifying.
3. **Devices/USB not reaching low-power states** during suspend.
4. `amdgpu 0000:06:00.0: Runtime PM not available` — appears every boot; worth understanding.

> **Session 3 outcome:** hypothesis 1 was pursued, fixed, and **produced no measurable power saving** (§26). Hypotheses 2–3 are now the live candidates.

### §12.2 Other outstanding items
- Plain `systemctl suspend` **not yet tested** with the freeze fix in place. → *done §23*
- `suspend-then-hibernate` not tested; note it performs an unattended **intermediate wake** before hibernating — hits both the suspend-resume race *and* the hibernate-write path back to back. → *done §24; the intermediate wake is exactly what broke*
- Lid-close automation not configured. → *still outstanding*
- **`fsck` still pending** after two forced power-offs. → *now 5 power-offs, still pending*
- **KDE post-login splash setting lost** during `mkinitcpio` regeneration. → *still outstanding*
- `zswap: loaded using pool zstd` active alongside zram — interaction with hibernation not yet evaluated. → *no problems observed across 4 hibernates + 1 STH*
- Hibernate resume inherently goes through the full UEFI+Limine boot sequence. Intrinsic to S4; **not** a misconfiguration.

---

## §13 Planned Next Steps *(Session 1's plan — retained for provenance)*

1. Confirm hibernate reliability — 2–3 more runs under varied workloads.
2. Test plain suspend.
3. Investigate suspend drain (`/proc/driver/nvidia/gpus/*/power`, envycontrol, `amd_s2idle.py`, `powertop --auto-tune`, `/proc/acpi/wakeup`).
4. Check BIOS for OS-type / sleep-mode toggle.
5. Test `suspend-then-hibernate` with ~2 min delay at the desk.
6. Only then wire lid-close behavior + `HibernateDelaySec=45min`; set KDE lid action to "Do nothing".
7. Run `fsck` from live USB.
8. Restore lost KDE splash configuration.

### Fallback if automation proves unreliable
Lock screen on lid close **without** auto-suspend; hibernate manually before travel. This is the workaround other MUX-less NVIDIA-hybrid Wayland users have settled on.

> **Note:** steps 1, 2, 5 completed in Session 3. Step 6 is blocked in its timer form (§25) and has been redesigned (Part VI). Steps 3, 4, 7, 8 remain outstanding.

---

## §14 Rollback Procedure (hibernate stack)

```bash
sudo rm /etc/systemd/system/systemd-hibernate.service.d/10-freeze-sessions.conf
sudo swapoff /swapfile
sudo rm /swapfile
# remove /swapfile line from /etc/fstab
# remove 'resume' hook from /etc/mkinitcpio.conf
sudo systemctl disable nvidia-suspend.service nvidia-hibernate.service nvidia-resume.service
sudo mkinitcpio -P
```

> **Updated for Session 3** — see Part IX for the current full rollback, which additionally covers the `60-freeze-sessions.conf` files, the `PreserveVideoMemoryAllocations=0` modprobe line, and `rtc_cmos.use_acpi_alarm=1`.

---
---

# PART II — SESSION 2 (2026-08-18)
## §15 — Suspend Battery Drain Investigation

**Status at close:** ✅ dGPU runtime-suspend fixed · ⏳ Overnight drain re-test pending · ⏳ Lid automation still not configured

---

## §15.1 Objective

Address §12.1: excessive battery drain during `s2idle` suspend. Hibernate works (§9.3), but does not help short/medium closures. Target: get the RTX 3070 to actually power down so `s2idle` isn't paying a constant dGPU tax.

---

## §15.2 Drain Baselines (measured, pre-fix)

| Window | Duration | Loss | Rate | Avg power (est. 90 Wh battery) |
|---|---|---|---|---|
| Overnight (17→18 Aug) | ~8 h | ~10% | 1.25 %/h | ~1.1 W |
| Workday (18 Aug) | ~8 h | ~15% | 1.88 %/h | ~1.7 W |

> ⛔ **Wattages wrong — battery is 64.7 Wh, not 90 Wh.** Corrected: **0.81 W** and **1.22 W** respectively. See §26.3.

---

## §15.3 Baseline Diagnostics (read-only, pre-change)

### §15.3.1 NVIDIA power state
```
$ cat /proc/driver/nvidia/gpus/0000:01:00.0/power
Runtime D3 status:          Enabled (fine-grained)     ✅ capability present
Tegra iGPU Rail-Gating:     Disabled
Video Memory:               Active                      ⚠️ not self-refresh, not off

GPU Hardware Support:
 Video Memory Self Refresh: Supported
 Video Memory Off:          Supported

S0ix Power Management:
 Platform Support:          Supported
 Status:                    Disabled                    ⚠️ supported but not engaged

Notebook Dynamic Boost:     Supported
```

**Still the single most promising untested item** — see §28.1.

### §15.3.2 PCI runtime PM
```
$ cat /sys/bus/pci/devices/0000:01:00.0/power/control        → auto
$ cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status → active     🚩

$ cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_usage
cat: No such file or directory      # not exposed on this kernel
```

`udevadm info -a -p /sys/bus/pci/devices/0000:01:00.0`:
```
ATTR{power/control}=="auto"
ATTR{power/runtime_active_time}=="1652138"
ATTR{power/runtime_status}=="active"
ATTR{power/runtime_suspended_time}=="0"     🚩 NEVER suspended, entire boot
ATTR{power/wakeup}=="disabled"
ATTR{power_state}=="D0"                      🚩 fully powered
```

### §15.3.3 GPU function topology
```
$ lspci -s 01:00 -v
01:00.0 VGA compatible controller: NVIDIA GA104M [RTX 3070 Mobile / Max-Q] (rev a1)
        Kernel driver in use: nvidia
01:00.1 Audio device: NVIDIA GA104 High Definition Audio Controller (rev a1)
        Kernel driver in use: snd_hda_intel

0000:01:00.0 (VGA):   active      🚩
0000:01:00.1 (Audio): suspended   ✅
```

**Key deduction:** the audio function under the same GPU suspends fine → not a bus-level or ACPI power-resource problem. The VGA function specifically is being held awake.

> Note: no USB-C/UCSI functions exist on this GPU (unlike many Optimus laptops) — only `.0` and `.1`. This also means the common "udev rule to remove NVIDIA sub-devices" trick is irrelevant here.

### §15.3.4 Who holds the GPU
```
$ fuser -v /dev/nvidia* /dev/dri/card* /dev/dri/renderD*
/dev/nvidia0:        ekj  1973  nvidia-smi
/dev/nvidiactl:      ekj  1973  nvidia-smi
/dev/dri/card1:      ekj  1188  Xwayland
/dev/dri/card2:      ekj  1188  Xwayland
/dev/dri/renderD129: ekj  1188  Xwayland
                     ekj  1241  plasmashell
                     ekj  1489  firefox
                     ekj  1524  ghostty
                     ekj  1526  plasma-systemmo

$ nvidia-smi
| N/A  40C  P8  10W / 55W |  44MiB / 8192MiB |  0%  Default |
|   0  N/A  N/A  1110  C+G  /usr/bin/kwin_wayland     11MiB | 🚩
```

`kwin_wayland` held a **live, VRAM-backed context** on the dGPU — not merely an open handle.

### §15.3.5 Platform sleep residency
```
$ cat /sys/power/suspend_stats/last_hw_sleep    → 512005920      (µs = 512 s)
$ cat /sys/power/suspend_stats/total_hw_sleep   → 34382329131    (µs = 34,382 s = 9 h 33 m)
$ cat /sys/power/suspend_stats/success          → 2
$ cat /sys/power/suspend_stats/fail             → 0
$ uptime -s                                     → 2026-08-17 20:57:30
```
Elapsed since boot ≈ 10 h 02 m (36,150 s). **Hardware-sleep ratio ≈ 95%.**

### §15.3.6 DRM device enumeration
```
/sys/class/drm/card1/device -> 0x10de     # NVIDIA RTX 3070
/sys/class/drm/card1-DP-1/device ->
/sys/class/drm/card1-DP-2/device ->
/sys/class/drm/card2/device -> 0x1002     # AMD Renoir iGPU
/sys/class/drm/card2-eDP-1/device ->      # internal panel
/sys/class/drm/card2-HDMI-A-1/device ->   # HDMI output
```

**Critical:** there is **no `card0`** on this system. Numbering is also **not stable across boots** (amdgpu/nvidia module load-order race).

---

## §15.4 Root Cause *(Session 2's conclusion)*

**`kwin_wayland` holds a permanent render context on the RTX 3070**, preventing it from ever entering runtime suspend (D3cold) — confirmed by `runtime_suspended_time == 0` across the entire boot.

Because `s2idle` (unlike S3) does **not** force PCI devices into low-power states, a device that is `active` at the moment of suspend simply *stays* active for the duration of the sleep. The dGPU therefore drew power continuously all night despite excellent platform-level sleep residency.

| Evidence | Reading | Meaning |
|---|---|---|
| `total_hw_sleep` ratio | ~95% | ✅ CPU/SoC reached deep idle — AMD s2idle path is **healthy** |
| `runtime_suspended_time` | `0` | 🚩 dGPU never powered down |

`KWIN_DRM_DEVICES` is a first-class KWin feature, not a workaround. Independent reports exist of the same symptom and fix on the same laptop class (incl. a CachyOS Zephyrus G16 report).

> ⛔ **PARTIALLY SUPERSEDED — see §26.2.** The defect was real and the fix works: the dGPU now runtime-suspends. But the 12 h 52 m measured overnight test on 2026-08-18→19 showed **~0.8 W, statistically identical to the pre-fix baseline of 0.81 W**. The dGPU being held in D0 was **not** the drain source. Downgrade this from "root cause of drain" to "a real bug, fixed, with no measurable power effect."
>
> This also raises a puzzle worth noting: if the dGPU was genuinely in D0 all night pre-fix, it should have cost *something*. Two possibilities: (a) an RTX 3070 in D0 at P8 with no work queued and the display pipeline off draws far less than `nvidia-smi`'s 10 W figure suggests; (b) the pre-fix baselines (§15.2) were measured too loosely to compare against. Both argue for the instrumentation in Part VII.

---

## §15.5 Hardware Finding — Display Output Wiring (GA503)

| Port | Wired to | Affected by restricting KWin to iGPU? |
|---|---|---|
| **HDMI** | iGPU (`card2-HDMI-A-1`) | ❌ No impact |
| **USB-C / DP Alt Mode** | dGPU (`card1-DP-1/2`) | ⚠️ Would break external output |

User confirmed: HDMI-only in practice; loss of USB-C display output acceptable. Fix applied as a **reversible session script** rather than system-wide, specifically to keep that option open.

---

## §15.6 Change Log — KWIN_DRM_DEVICES attempts

### ❌ Attempt A — `card0` (never activated)

| | |
|---|---|
| File | `~/.config/plasma-workspace/env/kwin-gpu.sh` |
| Content | `export KWIN_DRM_DEVICES=/dev/dri/card0` |
| Problem | **`card0` does not exist** — assumed, not verified |
| Outcome | Caught before logout; file deleted. No session damage. |
| Status | **REVERTED — no trace remains** |

### ❌ Attempt B — escaped by-path (caused black screen)

| | |
|---|---|
| Content | `export KWIN_DRM_DEVICES=/dev/dri/by-path/pci-0000\:06\:00.0-card` |
| Rationale | Stable PCI path to survive card renumbering; `\:` because KWin uses `:` as its list separator |
| Problem | Shell **consumed the backslashes when sourcing** → KWin received unescaped colons → split into 3 invalid fragments |
| Outcome | **Black screen after login.** KWin crash-looped 3× with coredumps. |
| Recovery | Ctrl+Alt+F3 → `rm` the file → reboot. **Login screen was unaffected** (user-scoped, not `/etc/environment`). |
| Status | **REVERTED — no trace remains** |

```
kwin_wayland[8987]: Failed to open drm device /dev/dri/by-path/pci-0000
kwin_wayland[8987]: Failed to open drm device 06
kwin_wayland[8987]: Failed to open drm device 00.0-card
kwin_wayland[8987]: No suitable DRM devices have been found
systemd-coredump[9022]: Process 8987 (kwin_wayland) dumped core.
   #1 KWin::RenderDevice::~RenderDevice()
   #2 KWin::GpuManager::~GpuManager()
... (repeats for PIDs 9029, 9078)
systemd[1051]: plasma-workspace-wayland.target: Bound to unit
               plasma-kwin_wayland.service, but unit isn't active.
```

### ✅ Attempt C — resolve-then-export (**CURRENT, WORKING**)

**File:** `~/.config/plasma-workspace/env/kwin-gpu.sh` (mode `0755`)
```bash
DEV=$(readlink -f /dev/dri/by-path/pci-0000:06:00.0-card)
[ -e "$DEV" ] && export KWIN_DRM_DEVICES="$DEV"
```

**Why this works:**
- Colons appear only in the *lookup path*, handled correctly by the shell — never reach KWin.
- `readlink -f` yields a colon-free node (`/dev/dri/card2`).
- Uses the **stable PCI address** (`0000:06:00.0`) as source of truth → immune to card renumbering.
- `[ -e "$DEV" ]` guard: if lookup fails, the variable is simply **never set** → KWin falls back to auto-detection instead of crash-looping. **Fail-safe by design.**

---

## §15.7 Verification Results (post-fix)

```
$ sudo cat /proc/$(pidof kwin_wayland)/environ | tr '\0' '\n' | grep KWIN_DRM
KWIN_DRM_DEVICES=/dev/dri/card2                    ✅ reached KWin

$ cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status
suspended                                          ✅ FIRST TIME THIS BOOT
```

`nvidia-smi` post-fix showed `P3 / 24W` with kwin_wayland still listed. **This output is misleading:**

1. **`nvidia-smi` itself wakes the GPU.** Querying forces it out of D3cold — the reading is a measurement artifact. **`nvidia-smi` is unreliable as a runtime-PM diagnostic.**
2. **KWin retains a lightweight handle** for PRIME buffer import even when the device is excluded as a *render* device. This no longer blocks runtime suspend.

**Authoritative check is sysfs, not `nvidia-smi`.**

---

## §15.8 Hypotheses Tested and Discarded (Session 2)

| Hypothesis | Verdict | Evidence |
|---|---|---|
| AMD `s2idle` platform path is broken | ❌ Healthy | `total_hw_sleep` ≈ 95% of elapsed time |
| `last_hw_sleep = 512 s` proves poor overnight sleep | ❌ Misread | It reports only the **most recent single cycle**; `total_hw_sleep` is cumulative |
| A sibling GPU function (audio/USB-C) blocks D3cold | ❌ No | `01:00.1` already `suspended`; no USB-C/UCSI functions exist |
| Missing `NVreg_EnableS0ixPowerManagement` is the primary cause | ⚠️ Deferred | *Deferral has now expired — see §28.1* |
| Fix "disables the dGPU" | ❌ No | Only stops KWin holding a context; PRIME offload unaffected |
| `card0` is the AMD iGPU | ❌ Wrong | No `card0` exists; AMD is `card2`, NVIDIA is `card1` |
| `/dev/dri/cardN` is a safe stable identifier | ❌ No | Numbering races on module load order; must use PCI path |
| Plymouth is relevant to the greeter risk | ❌ No | Plymouth = boot splash only; greeter is PLM |
| `amdgpu: Runtime PM not available` indicates a fault | ❌ Benign | Known ordering artifact tied to DM backlight registration |

---

## §15.9 Environment Notes

- **Display manager:** Plasma Login Manager (PLM) — SDDM fork with a new greeter, more tightly Plasma/KWin-integrated. Fedora 44 is switching KDE variants to it.
  - **Implication:** system-wide `/etc/environment` is *more* risky here than on SDDM. The user-scoped script avoids the greeter entirely. Confirmed empirically — Attempt B black-screened the session but the **login screen still worked**.
- **`~/.config/plasma-workspace/env/` scripts** are sourced only after successful login → PLM never reads them.

---

## §15.10–15.14 (state, TODO, prediction, rollback, lessons)

Superseded by Parts IV, VII, IX, XI of this document. Two items preserved for the record:

**§15.12's prediction:** "~5–7% over 8 h, floor 0.4–0.7 W; if the result lands in that range, the platform is at its floor."
> **Outcome:** measured ~0.8 W (§26). Above the predicted floor, and unchanged from baseline. The prediction's *floor model* remains plausible; the *improvement* did not materialise.

**§15.14 lesson 9:** "There is no per-device power profiler for suspend — the CPU is parked. Use pre-suspend state snapshots, residency counters, `rtcwake` sampling, and A/B elimination."
> ⛔ **`rtcwake` sampling is not available on this machine** — RTC alarms do not fire from s0ix (§25). Substitute: manual-wake timed A/B. See §Part VII.

---

## §16 — Addendum to Session 2

### §16.1 Decision Context — Why a Reversible Script

| Question | Answer |
|---|---|
| External display used? | Yes, occasionally |
| Port used | **Always HDMI** in practice; "might wish to do usb-c [someday]. No big loss if I lose that capability though" |
| dGPU use case | **Gaming** |

Combined with §15.5, this confirmed HDMI use is unaffected, and justified the session-script approach over `/etc/environment`.

### §16.2 Gaming / dGPU-on-demand Reference

PRIME render offload remains fully independent of the KWin restriction:

```bash
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia __VK_LAYER_NV_optimus=NVIDIA_only %command%
```

Verified working in Session 3 (`prime-run glxinfo` → `NVIDIA GeForce RTX 3070 Laptop GPU/PCIe/SSE2`), and the GPU returned to `suspended` afterwards unaided.

---
---

# PART III — SESSION 3 (2026-08-18 evening → 2026-08-19 morning)
## Hibernate Solved · STH Blocked · Drain Hypothesis Refuted

---

## §17 Objective and Method

**Decision at session start:** park the suspend-drain work (slow feedback loop, one data point per night) and instead:
1. Establish that direct hibernate is genuinely reliable, under a falsifiable test protocol.
2. Then build the `suspend-then-hibernate` flow.

**Rationale for the ordering:** STH is strictly harder than direct hibernate — it performs an unattended intermediate wake, hitting the suspend-resume path and the hibernate-write path back to back. Testing STH on an unproven hibernate stack means two possible causes for any failure.

### §17.1 The falsifiability problem — and how it was solved

The user correctly flagged that earlier "successes" were unreliable, because **KDE session restore reopens applications on a normal cold boot**, which is visually similar to a resume.

| | Shutdown + KDE session restore | Hibernate |
|---|---|---|
| Apps reopen | ✅ | ✅ |
| Unsaved buffers (Kate, GIMP, LibreOffice) | ❌ | ✅ |
| Shell state — cwd, history, running jobs, ssh | ❌ | ✅ |
| Firefox scroll position, form contents, JS state | Partial | ✅ |
| Clipboard | ❌ | ✅ |
| Running VMs, containers, long jobs | ❌ | ✅ |
| Window geometry across desktops | Approximate | Exact |

**Adopted pass criteria — only these two count:**

```bash
uptime -s                            # must be UNCHANGED
cat /proc/sys/kernel/random/boot_id  # must be UNCHANGED
```

Plus a real gap between `hibernation entry` and `hibernation exit` in the kernel log, proving actual power-off rather than an aborted attempt.

**State markers used each run:**
```bash
cd /tmp && mkdir -p hib-marker && cd hib-marker
export MARKER="test-$(date +%H%M%S)"; echo $MARKER
sleep 9999 &
```
`/tmp` is tmpfs (RAM-backed), so a cold boot wipes it — `pwd` surviving is independent corroboration. A backgrounded `sleep` job cannot survive a cold boot, so `jobs` is the strongest single proof.

### §17.2 Note on hibernate vs. shutdown-with-restore

The user's initial framing was that hibernate ≈ shutdown, since KDE restores windows either way. Clarified: they are different mechanisms and **fully compatible** — hibernate bypasses the session manager entirely. Recommended (not yet applied): System Settings → Session → Desktop Session → **Start with an empty session**, giving clean shutdowns *and* full state restore via hibernate.

Also considered and rejected: a hypothetical `suspend-then-poweroff`. No such systemd mode exists; resume cost would be identical (full UEFI + Limine + kernel) while preserving strictly less. Hibernate dominates it.

---

## §18 Pre-flight

### §18.1 Escape hatches

Attempt 1 (§5) cost a forced power-off on a machine that was **fully alive with a black screen**. SSH would have prevented it. Enabled:

```bash
sudo systemctl enable --now sshd        # → symlink created, active
echo 1 | sudo tee /proc/sys/kernel/sysrq # REISUB available
ip -brief addr                           # wlan0 → 192.168.1.220/24
```

Three outs before the power button: Ctrl+Alt+F3 · SSH from another machine · REISUB.

### §18.2 pm_test check

```
$ cat /sys/power/pm_test
[none] core processors platform devices freezer     ✅
```
Verified before **every** real test, per the §9.2 trap.

### §18.3 🔧 SSH was blocked by `ufw`

First SSH attempt failed. Diagnosis:

```
$ systemctl status sshd
● sshd.service - active (running), Server listening on 0.0.0.0:22 and [::]:22

$ ss -tlnp | grep ':22\b'
LISTEN 0 128 0.0.0.0:22    LISTEN 0 128 [::]:22

$ systemctl is-active firewalld ufw nftables
inactive
active            🚩  ← ufw
inactive

$ sudo sshd -T | grep -iE 'passwordauth|kbdinteractive|port|listenaddress'
Port 22
ListenAddress [::]:22
ListenAddress 0.0.0.0:22
PasswordAuthentication yes
KbdInteractiveAuthentication no
```

Daemon was entirely healthy; **`ufw` was dropping the packets** (symptom: timeout, not "connection refused").

**Fix (LAN-scoped deliberately — this laptop joins public wifi):**
```bash
sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp comment 'ssh from LAN'
sudo ufw reload
```

### §18.4 NVIDIA driver audit

```
$ cat /proc/driver/nvidia/params | grep -iE 'suspendnotif|preserve|temporary'
PreserveVideoMemoryAllocations: 1     🚩
UseKernelSuspendNotifiers: 1
TemporaryFilePath: "/var/tmp"

$ systemctl list-unit-files 'nvidia-*'
UNIT FILE                             STATE    PRESET
nvidia-hibernate.service              enabled  disabled
nvidia-persistenced.service           disabled disabled
nvidia-powerd.service                 enabled  disabled
nvidia-resume.service                 enabled  disabled
nvidia-suspend-then-hibernate.service disabled disabled
nvidia-suspend.service                enabled  disabled
```

Three findings:

1. **`UseKernelSuspendNotifiers: 1`** — the 610 driver handles VRAM preservation itself via kernel suspend notifiers. The `PreserveVideoMemoryAllocations` parameter belongs to the 430–590 era.
2. **`nvidia-suspend-then-hibernate.service` exists** and ships disabled. This resolves the anticipated need for a `WantedBy=` override hack — the dedicated unit exists and merely needs enabling if required. *(Deliberately left disabled through all testing, to avoid adding a variable.)*
3. **Every nvidia unit has `PRESET: disabled`** — upstream's intent on 595+. The three `enabled` ones are manual from Session 1 (§4 row 7). Retained throughout Session 3 as part of the known state; they are cleanup candidates (§28.5), not part of the fix.

*(`nvidia-powerd` is Dynamic Boost, unrelated to sleep.)*

---

## §19 Hibernate Test 1 (first attempt) — FAILURE, and the diagnosis

### §19.1 Result

| Check | Before | After | |
|---|---|---|---|
| `uptime -s` | 2026-08-18 15:51:51 | **16:47:50** | ❌ changed |
| `boot_id` | `30e31f22-…` | **`e2a4ef8d-…`** | ❌ changed |
| `$MARKER` | `test-163415` | *(empty)* | ❌ lost |
| `pwd` | `/tmp/hib-marker` | `/home/ekj` | ❌ tmpfs wiped |

**Cold boot.** User-visible symptom: VSCode and Kate on other desktops did *not* come back; Firefox reopened tabs but had reloaded them. Boot sequence was ROG logo → black screen → PLM lock screen.

Swap state was correct going in:
```
NAME       TYPE      SIZE USED PRIO
/swapfile  file       20G   0B   -1     ✅
/dev/zram0 partition  15G   5M  100
```

### §19.2 The kernel log — a *different* failure from Attempt 1

```
PM: hibernation: Read 5263440 kbytes in 2.92 seconds (1802.54 MB/s)
PM: hibernation: Failed to load image, recovering.
PM: hibernation: Basic memory bitmaps freed
PM: hibernation: resume failed (-5)
```

| | Attempt 1 (Session 1, §5.2) | Session 3, Test 1 |
|---|---|---|
| Error | `Image not found (code -22)` | `resume failed (-5)` |
| Stage | Header rejected — image never read | Signature OK, **5.26 GB read at 1.8 GB/s**, then failed |
| Meaning | Image corrupt / invalid | **Image is fine. Device quiesce broke.** |

The kernel's resume sequence is: *read image from swap → quiesce all devices → restore memory.* The read completed; the break was at quiesce.

### §19.3 The smoking gun — full trace

Broadening the grep produced the definitive lines:

```
NVRM: loading NVIDIA UNIX Open Kernel Module for x86_64  610.57.04
nvidia-modeset: Loading NVIDIA UNIX Open Kernel Mode Setting Driver 610.57.04
[drm] [nvidia-drm] [GPU ID 0x00000100] Loading driver
PM: hibernation: resume from hibernation
[drm] Initialized nvidia-drm 0.0.0 for 0000:01:00.0 on minor 1
nvidia 0000:01:00.0: [drm] Cannot find any crtc or sizes
...
PM: hibernation: Read 5263440 kbytes in 2.92 seconds (1802.54 MB/s)

NVRM: GPU 0000:01:00.0: PreserveVideoMemoryAllocations module parameter is set.
      System Power Management attempted without driver procfs suspend interface.
      Please refer to the 'Configuring Power Management Support' section in the driver README.

nvidia 0000:01:00.0: PM: pci_pm_freeze(): nv_pmops_freeze [nvidia] returns -5
nvidia 0000:01:00.0: PM: dpm_run_callback(): pci_pm_freeze... returns -5
nvidia 0000:01:00.0: PM: failed to quiesce async: error -5
PM: hibernation: Failed to load image, recovering.
PM: hibernation: resume failed (-5)
```

The driver **names the parameter in the error message**. `PreserveVideoMemoryAllocations` on a 610 driver routes power management through a legacy procfs interface (`/proc/driver/nvidia/suspend`) that no longer exists, so the freeze callback fails, the whole device-quiesce phase aborts, and the successfully-read image is discarded.

This exact three-line signature (`nv_pmops_freeze … -5` → `failed to quiesce async` → `Failed to load image`) is widely reported — on Bazzite with an RTX 3080 Ti, on Arch with an NVIDIA/Intel hybrid laptop, and on Arch with a 525-series driver.

### §19.4 Secondary finding — nvidia is in the initramfs

```
$ ls /etc/mkinitcpio.conf.d/
10-chwd.conf

$ cat /etc/mkinitcpio.conf.d/10-chwd.conf
# This file is automatically generated by chwd. PLEASE DO NOT EDIT IT.
MODULES+=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)

$ grep -E '^(MODULES|HOOKS)=' /etc/mkinitcpio.conf
MODULES=(amdgpu)
HOOKS=(base systemd plymouth autodetect microcode modconf kms keyboard sd-vconsole block resume filesystems fsck)
```

CachyOS's `chwd` injects the NVIDIA stack into the initramfs. The journal ordering confirms it takes effect: `NVRM: loading` and `[drm] Initialized nvidia-drm` appear **before** `PM: hibernation: resume from hibernation`. The driver is up and registered inside the initramfs, in an environment with no procfs suspend interface and no access to `NVreg_TemporaryFilePath`.

This is the early-KMS-vs-hibernation conflict the Arch wiki warns about. A prepared **Fix 2** (below) was never needed, because Fix 1 alone resolved the failure. Retained for reference:

```bash
# NOT APPLIED — kept as a documented fallback
sudo tee /etc/mkinitcpio.conf.d/20-no-nvidia-initramfs.conf <<'EOF'
# Override 10-chwd.conf: nvidia in the initramfs can break hibernation resume
MODULES=(amdgpu)
EOF
sudo mkinitcpio -P
```
Do **not** edit `10-chwd.conf` directly — chwd regenerates it. mkinitcpio sources `conf.d/*.conf` alphabetically, so a `20-` file wins. Keeping `MODULES=(amdgpu)` preserves early KMS for the amdgpu-driven panel, so Plymouth retains its framebuffer.

**Fix 3**, also prepared and unused: hibernate with the dGPU forced awake (`echo on > .../power/control`), to test whether hibernating *from D3cold* was the incompatibility. Rendered moot by Test 3 passing with the GPU active (§21.3).

---

## §20 The Fix — and the `2` trap

### §20.1 First attempt: remove the parameter → **did not work**

```bash
sudo tee /etc/modprobe.d/nvidia-power-management.conf <<'EOF'
options nvidia NVreg_TemporaryFilePath=/var/tmp
EOF
sudo mkinitcpio -P && reboot
```

Result:
```
PreserveVideoMemoryAllocations: 2     🚩 still not 0
UseKernelSuspendNotifiers: 1
```

**`2` is not "off."** The parameter's values are:

| Value | Meaning |
|---|---|
| `0` | Preserve only select video memory allocations |
| `1` | Preserve all |
| `2` | **Automatically enable preservation when `NVreg_UseKernelSuspendNotifiers` is enabled** — *this is the default* |

Since `UseKernelSuspendNotifiers=1`, the auto default **re-enabled the broken behaviour**. Removing the line reverted to `2`, not to off. Documented trap: you see `2` precisely *because* you removed the file rather than explicitly setting `0`.

### §20.2 Working fix ✅

**File:** `/etc/modprobe.d/nvidia-power-management.conf`
```
options nvidia NVreg_PreserveVideoMemoryAllocations=0
options nvidia NVreg_TemporaryFilePath=/var/tmp
```

```bash
sudo mkinitcpio -P
sudo limine-update
sudo reboot
```

Verified:
```
$ cat /proc/driver/nvidia/params | grep -iE 'preserve|suspendnotif'
PreserveVideoMemoryAllocations: 0     ✅
UseKernelSuspendNotifiers: 1          ✅
```

**Why `TemporaryFilePath=/var/tmp` is retained:** the driver default is `/tmp`, which is tmpfs and is cleared, so the driver would lose stored memory contents. Harmless and useful, independent of the preserve setting.

**This single change is the fix for hibernation on this machine.** Target state (`Preserve: 0` + `Notifiers: 1`) matches the configuration others have verified working on 595+/610 drivers.

---

## §21 Hibernate Test Series — 4/4 PASS

### §21.0 Framing
Because §9.3 is now considered unverified (see banner), the series below constitutes the **first confirmed hibernates on this machine**.

### §21.1 Test 1 — idle desktop ✅

| Check | Value |
|---|---|
| Pre `uptime -s` / `boot_id` | 2026-08-18 17:08:57 / `cdb92f27-a9db-4fdd-af27-f75c542983f0` |
| Post | **identical** ✅ |
| `$MARKER` | `test-170946` ✅ |
| Kernel gap | `17:10:33.984` entry → `17:11:24.183` exit = **51 s** ✅ |
| Service log | `Performing sleep operation 'hibernate'` → `System returned from sleep operation 'hibernate'` ✅ |
| Errors | none — no `NVRM`, no `quiesce`, no `-5` ✅ |

**Note on log appearance:** a successful hibernate log looks *truncated* — there is no "image written" line. The memory snapshot is taken first, then the image is written; anything logged after the snapshot doesn't exist inside the snapshot and vanishes on resume. Likewise the resume-side lines (`resume from hibernation`, `Read X kbytes`) belong to the fresh boot kernel, which is then replaced by the restored one. **You only see those lines when resume fails.**

### §21.2 Test 2 — real state ✅

| Check | Value |
|---|---|
| `boot_id` | unchanged ✅ |
| Kernel gap | `17:50:10.344` → `17:52:56.192` = **2 m 46 s** ✅ |
| `$MARKER` | `test-174943` ✅ |
| `jobs` | `sleep 9999` still running ✅ |
| **Kate unsaved text, 3 desktops** | **restored** ✅ |
| **Firefox** | **restored without reloading the page** ✅ |
| VSCode | restored ✅ |

The Kate and Firefox results are the ones session restore provably cannot produce.

### §21.3 Test 3 — dGPU active (the Attempt-1 stress case) ✅

```
$ prime-run glxinfo | grep "OpenGL renderer"
OpenGL renderer string: NVIDIA GeForce RTX 3070 Laptop GPU/PCIe/SSE2
$ cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status
active                                    ← required precondition
```

| Check | Value |
|---|---|
| `boot_id` | unchanged ✅ |
| Kernel gap | `17:57:17.509` → `17:58:55.196` = **1 m 38 s** ✅ |
| `$MARKER` / `jobs` | `test-175654`, sleep running ✅ |
| Errors | none ✅ |
| Post-resume dGPU | `suspended` ✅ (Session 2 fix survives a hibernate cycle) |

Hibernating with a live VRAM-backed dGPU context is clean.

### §21.4 Test 4 — 28 minutes, with the freeze override active ✅

| Check | Value |
|---|---|
| `boot_id` | unchanged ✅ |
| Kernel gap | `18:06:04.036` → `18:34:29.214` = **28 m 25 s** ✅ |
| `$MARKER` | `test-180526` ✅ |
| `jobs` | **three** `sleep 9999` processes — from Tests 2, 3 and 4, all alive in one shell across three consecutive hibernates ✅ |
| Freeze | `Successfully froze unit 'user.slice'` / `Successfully thawed unit 'user.slice'` ✅ **first time ever** |
| Battery | ❌ **invalid** — see below |

**Battery measurement failed:** `64,724,000` → `64,819,000 µWh`, i.e. **+95 mWh**. Energy cannot increase during hibernation. Cause: AC was briefly connected around the reading. No S4 drain figure obtained. *(Not worth re-running deliberately — S4 draw is near-zero by construction, and a real figure will come free from the first overnight hibernate on battery.)*

**Minor artifact:** `uptime -s` drifted 17:08:57 → 17:08:58 → 17:09:01 across tests. It is derived as `now − uptime` and rounds differently across a hibernate. **`boot_id` is the authoritative check** and never moved.

### §21.5 Series summary

| # | Condition | Gap | `boot_id` | Markers | Verdict |
|---|---|---|---|---|---|
| 1 | Idle desktop | 51 s | ✅ | ✅ | **PASS** |
| 2 | Kate unsaved + Firefox + shell | 2 m 46 s | ✅ | ✅ | **PASS** |
| 3 | dGPU active via `prime-run` | 1 m 38 s | ✅ | ✅ | **PASS** |
| 4 | 28 min, freeze override active | 28 m 25 s | ✅ | ✅ | **PASS** |

**Direct hibernate is solved.**

---

## §22 The Freeze Drop-in Collision

### §22.1 Discovery

Test 4's precursor logs still showed, at 17:57:

```
systemd-sleep[6239]: User sessions remain unfrozen on explicit request
                     ($SYSTEMD_SLEEP_FREEZE_USER_SESSIONS=0).
```

**The §9 fix had never been active** — for the entire life of the document.

### §22.2 Cause

```
$ ls -1 /usr/lib/systemd/system/systemd-hibernate.service.d/ \
        /etc/systemd/system/systemd-hibernate.service.d/

/etc/systemd/system/systemd-hibernate.service.d/:
10-freeze-sessions.conf

/usr/lib/systemd/system/systemd-hibernate.service.d/:
10-nvidia-no-freeze-session.conf
```

systemd merges drop-ins from all directories sorted **by filename only, regardless of which directory they reside in**. Both begin `10-`; the tiebreak falls to the remainder, and **`f` < `n`**. For options accepting a single value, the file sorted *last* wins. So nvidia's `=false` overwrote the user's `=true` on every single cycle.

`/etc` precedence applies **only to identically-named files**. A `99-` prefix would also have failed (`9` < `n`).

### §22.3 Fix

Initially applied as `zz-freeze-sessions.conf`, then renamed to the documented convention: systemd recommends a two-digit numeric prefix, with **10–40 reserved for vendor drop-ins in `/usr/`** and **60–90 for local drop-ins in `/etc/` and `/run/`**.

```bash
sudo rm -f /etc/systemd/system/systemd-hibernate.service.d/10-freeze-sessions.conf

for u in systemd-hibernate systemd-suspend-then-hibernate; do
  sudo mkdir -p /etc/systemd/system/$u.service.d
  sudo tee /etc/systemd/system/$u.service.d/60-freeze-sessions.conf <<'EOF'
[Service]
# Overrides /usr/lib/.../10-nvidia-no-freeze-session.conf (drop-ins merge by
# filename; "60-" sorts after "10-nvidia-...", so this value wins).
# Not needed for the 610 driver's kernel-suspend-notifier path.
Environment=SYSTEMD_SLEEP_FREEZE_USER_SESSIONS=true
EOF
done
sudo systemctl daemon-reload
```

Verified:
```
$ systemctl cat systemd-hibernate.service | grep -i freeze
# /usr/lib/systemd/system/systemd-hibernate.service.d/10-nvidia-no-freeze-session.conf
Environment="SYSTEMD_SLEEP_FREEZE_USER_SESSIONS=false"
# /etc/systemd/system/systemd-hibernate.service.d/60-freeze-sessions.conf
Environment=SYSTEMD_SLEEP_FREEZE_USER_SESSIONS=true          ← last wins ✅

$ ls -1 /usr/lib/systemd/system/systemd-suspend-then-hibernate.service.d/
10-nvidia-no-freeze-session.conf

$ systemctl show systemd-suspend-then-hibernate.service -p Environment
Environment=SYSTEMD_SLEEP_FREEZE_USER_SESSIONS=true          ✅
```

Confirmed effective in Test 4 (`Successfully froze unit 'user.slice'`).

### §22.4 Alternatives considered

| Approach | Pro | Con | Chosen |
|---|---|---|---|
| `60-freeze-sessions.conf` | Survives nvidia renaming; obvious intent; matches convention | Both files load; result is ordering-dependent | ✅ |
| Identical filename in `/etc` (shadowing) | Vendor file disappears entirely from `systemctl cat` | **Silently stale** if the vendor renames/changes its drop-in on a rolling distro | ❌ |
| Symlink to `/dev/null` in `/etc` | Documented masking method; reverts to systemd default | Same staleness risk | ❌ |

### §22.5 Is the override even correct?

Genuinely contested, and worth recording both sides.

**For nvidia's default (`false`):** Arch shipped a news item when systemd began freezing user sessions on sleep, noting known breakage with the proprietary NVIDIA drivers and suggesting packagers add drop-ins disabling it. Failures were severe — Wayland compositors frozen after resume, VT switching dead.

**Against it:** systemd's own message explicitly warns the unfrozen state is not recommended and calls out *suspend-then-hibernate* by name. KDE Linux has moved to narrow nvidia's blanket rule, on the reasoning that leaving clients running while the GPU is suspended is what *causes* crashes and stale framebuffers on hybrid laptops.

**Empirical position on this machine:** Tests 1–3 passed with freeze **disabled**; Test 4 passed with it **enabled**. For direct hibernate it makes no observable difference. The override is retained **because of STH specifically** — the intermediate wake is precisely where an unfrozen compositor meets a GPU mid-teardown.

⚠️ **Maintenance hazard:** `60-freeze-sessions.conf` is load-bearing and invisible. If hibernate or STH regresses after an `nvidia-utils` update, check `systemctl cat systemd-hibernate.service` first and confirm the file still sorts last.

---

## §23 Plain Suspend Verification

Closes §12.2's first open item, and validates the suspend leg that STH depends on.

```
$ cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status
suspended
$ systemctl suspend
# ~90 s later, woken by power button
$ cat /proc/sys/kernel/random/boot_id
cdb92f27-…                     ✅ unchanged
$ journalctl -b 0 -k -g "PM: suspend entry|PM: suspend exit"
18:40:13.384  PM: suspend entry (s2idle)
18:41:43.371  PM: suspend exit
$ cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status
active                          ⚠️ see below
```

No `Xid`, no `Mode Validation Warning`, desktop clean on resume.

### §23.1 The `active`-after-resume artifact

Reading the dGPU state seconds after resume shows `active`. This is **not** a regression — PCI runtime PM has an autosuspend delay, and the driver re-probes the device on resume. Re-checked after a couple of minutes:

```
runtime_status → suspended       ✅
runtime_suspended_time → growing ✅
```

**Rule adopted:** wait ≥2 minutes before judging dGPU runtime state after any resume. Also never use `nvidia-smi` for this (§15.7).

This also retroactively answers §15.11's outstanding `runtime_suspended_time` check: the counter is nonzero and growing.

### §23.2 A counter that looks broken but isn't

```
$ cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_suspended_time
2812254        # ms = 46 min 52 s
```

Against ~13 h of wall clock this looks far too low — but the counter uses **monotonic time, which does not advance during s2idle**. Measured against ~116 min of actual awake time, ~40% runtime-suspended is entirely normal.

---

## §24 Suspend-then-Hibernate Testing

### §24.1 Configuration used for testing

```ini
# /etc/systemd/sleep.conf.d/10-hibernate-delay.conf
[Sleep]
HibernateDelaySec=2min
```

`HibernateOnACPower=no` was deliberately **omitted** for this test — it delays the countdown until AC is disconnected, which would make a desk test look like a hang.

### §24.2 STH Test 1 — mechanism works, timer does not

Timeline as logged:

| Time | Event |
|---|---|
| 18:45:30 | `Successfully froze unit 'user.slice'` — freeze override active ✅ |
| 18:45:30 | `Performing sleep operation 'suspend'` · kernel `PM: suspend entry (s2idle)` |
| *~18:47:30* | **timer should have fired here** 🚩 nothing happened |
| 18:48:45 | `PM: suspend exit` — **user pressed power** |
| 18:48:45 | `Performing sleep operation 'hibernate'` · `PM: hibernation: hibernation entry` |
| — | image written; machine powered fully off (screen stayed black — expected) |
| 18:49:57 | `PM: hibernation: hibernation exit` — full restore |
| 18:49:57 | `Successfully thawed unit 'user.slice'` |

**Results:**

| Check | Value |
|---|---|
| `boot_id` | `cdb92f27-…` unchanged ✅ |
| `$MARKER` | `sth-184526` ✅ |
| `jobs` | **four** `sleep 9999` still running ✅ |
| Freeze | froze + thawed ✅ |
| Errors | no `-5`, no `NVRM`, no `Xid` ✅ |
| **Self-wake at 2 min** | ❌ **did not occur** |

**Interpretation:** the STH state machine is correct. On being woken manually, systemd correctly observed that `HibernateDelaySec` had elapsed and went straight to hibernate rather than resuming. `HibernateDelaySec` is definitively being read — otherwise the 2 h default would have applied and it would have resumed normally.

**The single defect is that nothing woke the machine on schedule.**

### §24.3 Two operational gotchas learned here

⚠️ **The black screen after an STH wake is the image write in progress.** When STH wakes to hibernate, it does not re-light the panel — there is no point powering the display for a few-second snapshot-and-die. **Wait ~10 s; the machine powers itself off.** Pressing power during the image write is the single riskiest thing you can do to an ext4 root. (This happened once during testing and got away with it.)

⚠️ **A stale `/sys/class/rtc/rtc0/wakealarm` blocks STH entirely.** If an alarm is pending when STH starts, the write to `wakealarm` errors and the system fails to enter *any* suspended state. Always clear it after `rtcwake` experiments:
```bash
echo 0 | sudo tee /sys/class/rtc/rtc0/wakealarm
```

---

## §25 RTC Wake Investigation — full teardown

The most involved piece of diagnosis in the session, and the one that ends in a hard blocker.

### §25.1 Hypothesis 1 — RTC in local time ❌

```
$ timedatectl | grep -iE 'RTC|Time zone'
                 RTC time: ons 2026-08-19 05:47:23
                Time zone: Europe/Copenhagen (CEST, +0200)
          RTC in local TZ: no                              ✅ ruled out
```
Classic Windows-dual-boot artifact; not present here.

### §25.2 Hypothesis 2 — alarm not being armed ❌

```
$ sudo rtcwake -m no -s 120 -v
Using UTC time.
systime = 1787118492, (UTC) Wed Aug 19 05:48:12 2026
rtctime = 1787118492
rtcwake: wakeup using /dev/rtc0 at Wed Aug 19 05:50:13 2026
suspend mode: no; leaving

$ cat /sys/class/rtc/rtc0/wakealarm
1787118613          ✅ armed correctly
```
`rtcwake -m no` arms the alarm without suspending, so `systemctl suspend` can then run — preserving systemd-sleep hooks and the nvidia services. Alarm set correctly for 07:50:13 local.

**Test:** `systemctl suspend`, hands off.
```
07:48:18.433  PM: suspend entry (s2idle)
07:53:13.730  PM: suspend exit              ← manual wake, 3 min past the alarm
```
❌ Slept straight through.

Also noted: `grep -i rtc /proc/acpi/wakeup` returns nothing — this is **normal**, the RTC is an ACPI *fixed event*, not an enumerated wakeup device.

### §25.3 Hypothesis 3 — legacy CMOS alarm path clobbered by AMD PMC

This is a documented AMD s2idle problem. Reported symptoms elsewhere match exactly:
- Ryzen 7 5700U + KDE: hibernate and suspend both work, but STH stays suspended and only hibernates when woken manually. Their finding was that `rtcwake` works **only** after switching to S3, or after unloading the `amd_pmc` module — neither acceptable here (no S3 available; unloading `amd_pmc` would destroy the ~95% s0ix residency).
- ASUS Zenbook 14, Ryzen 5 7530U, s2idle, kernel 7.0.10, systemd 260 — same "never wakes, hibernates on manual wake."
- An open systemd issue on Arch with systemd 258 describing the identical behaviour.

**Suggested fix from an AMD kernel developer** (via a Framework thread): `rtc_cmos.use_acpi_alarm=1`, which routes the alarm through the ACPI alarm path instead of legacy CMOS. Reported to make one-hour suspend → wake → hibernate work perfectly.

**Applied:**
```bash
sudo cp /etc/default/limine /etc/default/limine.bak
# appended to KERNEL_CMDLINE[default]: rtc_cmos.use_acpi_alarm=1
sudo limine-update      # rebuilt both cachyos and cachyos-lts initramfs
sudo reboot
```
```
$ grep -o 'use_acpi_alarm=1' /proc/cmdline
use_acpi_alarm=1      ✅ live
```

**Test:** alarm for 08:05:08.
```
08:03:13.310  PM: suspend entry (s2idle)
08:06:41.377  PM: suspend exit              ← manual, 93 s past the alarm
```
❌ **Still no wake.**

### §25.4 Hypothesis 4 — `alarm_IRQ` never enabled ❌ (ruled out)

A near-identical Zenbook report attributes the failure to `alarm_IRQ` remaining `no` in `/proc/driver/rtc`, fixed by an upstream commit (`torvalds/linux@f7ecfc3`) not yet in the Arch kernel package. **Not applicable here:**

```
$ sudo rtcwake -m no -s 300 && cat /proc/driver/rtc
rtc_time        : 06:10:31
alrm_time       : 06:15:32
alarm_IRQ       : yes          ✅ interrupt IS enabled
alrm_pending    : no
HPET_emulated   : no
BCD             : yes
batt_status     : dead         ← see §25.5

$ cat /sys/module/rtc_cmos/parameters/use_acpi_alarm
Y                              ✅
$ cat /sys/class/rtc/rtc0/device/power/wakeup
enabled                        ✅
```

**Awake delivery test** — arm an alarm and let it fire while the machine is running:

```
BEFORE:
/sys/firmware/acpi/interrupts/ff_rt_clk:  0   disabled   unmasked
/sys/firmware/acpi/interrupts/sci:        13
/proc/interrupts IRQ 8 (rtc0):            0 across all CPUs

$ sudo rtcwake -m no -s 60 ; sleep 75

AFTER:
/sys/firmware/acpi/interrupts/ff_rt_clk:  1   disabled   unmasked   ← FIRED ✅
/proc/interrupts IRQ 8 (rtc0):            0                          ← expected
/proc/driver/rtc: alarm_IRQ : no   alrm_pending : no                 ← cleared after delivery
```

Two conclusions:
- **The RTC alarm fires and is delivered correctly while awake.** Counter went 0 → 1.
- IRQ 8 staying at 0 is **correct** with `use_acpi_alarm=Y` — the event routes via the ACPI SCI, not the legacy IRQ 8 line. The kernel parameter is working as intended.

### §25.5 The `batt_status: dead` red herring

`/proc/driver/rtc` reports the CMOS cell as dead. **Disregard.** On this ASUS board the RTC is fed from the main battery pack rather than a discrete CR2032, so the driver is reading a voltage-sense line that does not exist on this design. Independently confirmed by the clock keeping perfect time across full power-offs. *(User identified this correctly.)* **Not a hardware fault — do not re-investigate.**

### §25.6 Hypothesis 5 — ACPI fixed event not enabled ❌ (final attempt)

The `ff_rt_clk` middle column read `disabled` — the fixed event's *enable bit* was not set. A disabled fixed event still increments its status counter, but should not generate an SCI capable of bringing the system out of sleep. Plausible, and the file is writable:

```bash
echo 0 | sudo tee /sys/class/rtc/rtc0/wakealarm
echo enable | sudo tee /sys/firmware/acpi/interrupts/ff_rt_clk
$ grep . /sys/firmware/acpi/interrupts/ff_rt_clk
       1  EN     enabled      unmasked            ✅ enable bit set and stuck

sudo rtcwake -m no -s 120        # alarm → 08:28:25
systemctl suspend                # hands off 3 min
```

```
08:26:24.880  PM: suspend entry (s2idle)
08:29:52.692  PM: suspend exit          ← manual wake

$ grep . /sys/firmware/acpi/interrupts/ff_rt_clk
       1  EN     enabled      unmasked            🚩 STILL 1
```

**The counter did not increment.** During the awake test it went 0 → 1 the moment the alarm fired. Here the alarm time passed while in s2idle and **no event was generated at all** — not ignored, never raised.

### §25.7 ⛔ Conclusion — firmware-level blocker

Everything controllable from Linux is correct and verified:

| Element | State |
|---|---|
| Alarm armed | ✅ `wakealarm` populated correctly |
| Timezone | ✅ RTC in UTC |
| `alarm_IRQ` | ✅ `yes` |
| `use_acpi_alarm` | ✅ `Y` |
| `power/wakeup` | ✅ `enabled` |
| ACPI fixed event | ✅ `EN enabled unmasked` |
| Delivery while awake | ✅ counter increments |
| **Delivery during s0ix** | ⛔ **event never generated** |

**Both delivery paths were tested and both fail identically during s0ix** — legacy CMOS IRQ 8 (before the kernel param) and the ACPI SCI (after). When both fail the same way with everything enabled, the cause is below the OS: the **AMD PMC does not include the RTC in its s0ix wake-source list on this firmware**.

**Timer-based `suspend-then-hibernate` is not achievable on this machine.** No configuration change can fix it. Candidate future remedies: a firmware/BIOS update from ASUS; a future kernel workaround; or unlocking S3 via DSDT patch (§28.4), since S3 uses an entirely different wake path.

### §25.8 ⚠️ Correction issued during the session

An earlier recommendation of `HandleLidSwitch=suspend` **plus** `HibernateOnACPower=no` was **wrong** and would have provided zero protection. The low-battery hibernate trigger exists **only inside `suspend-then-hibernate`**; plain `suspend` has no such mechanism.

The corrected design keeps STH but drops `HibernateDelaySec` entirely, leaving the ACPI `_BTP` battery trip point as the sole trigger. That path is **EC-driven through a different GPE than the RTC**, so it may work where the timer does not — and if it doesn't, STH degrades to exactly the behaviour of plain suspend. **Strictly no worse, possibly better.** See Part VI.

Supporting facts: systemd tries the low-battery alarm *first* when a battery is present, and if `HibernateDelaySec=` is also set, configures an additional timer so hibernation occurs on whichever comes first. The default targets hibernating when the battery drops below ~5%. `HibernateOnACPower=` (added in systemd 257; this system runs 261) is only consulted by STH when `HibernateDelaySec=` is set — worth noting, as it means with no delay configured the AC gating may not apply. `BAT0/alarm` is populated on this machine, so `_BTP` is present.

---

## §26 The Overnight Drain Measurement

### §26.1 An accidental, and nearly ideal, test

Following STH Test 1, the machine was left suspended overnight, unattended and on battery:

```
aug 18 18:53:37.5  PM: suspend entry (s2idle)
aug 19 07:45:39.4  PM: suspend exit            ← 12 h 52 m 02 s
```

This is precisely the test §15.12 was waiting for.

### §26.2 Arithmetic

| Point | Reading |
|---|---|
| 2026-08-18 18:45:26 | `63,963,000 µWh` |
| 2026-08-19 07:58:38 | `49,752,000 µWh`, `capacity 77`, `status Discharging` |
| **Delta** | **14,211,000 µWh = 14.211 Wh** |
| Wall clock | 13 h 13 m 12 s = 13.22 h |
| s2idle leg | 12 h 52 m 02 s = 12.867 h |

Awake overhead inside that window:
- 18:45:26 → 18:53:37: contains the STH cycle — 3 m 15 s suspended, ~1 m 12 s hibernate write + power-off + restore, and ~3 m 40 s fully awake.
- 07:45:39 → 07:58:38: ~13 m fully awake.
- **Total fully-awake ≈ 16.7 min ≈ 0.28 h.** At a plausible 12–20 W idle desktop draw → **3.3–5.6 Wh**, plus ~0.3 Wh for the hibernate write.

**s2idle consumption ≈ 14.211 − (3.6 to 5.9) ≈ 8.3 to 10.6 Wh over 12.867 h**

$$\Rightarrow \boxed{\approx 0.65\text{–}0.82\ \mathrm{W},\ \text{centred} \sim 0.75\text{–}0.8\ \mathrm{W}}$$

In percentage terms: ~12.8–16.4% over 12.87 h → **~1.0–1.3 %/h**.

### §26.3 Comparison against the corrected baseline

| Measurement | Rate | On 90 Wh (as documented) | **On real 64.7 Wh** |
|---|---|---|---|
| §15.2 overnight, **pre-fix** | 1.25 %/h | ~~1.1 W~~ | **0.81 W** |
| §15.2 workday, **pre-fix** | 1.88 %/h | ~~1.7 W~~ | **1.22 W** |
| **This test, post-fix** | ~1.0–1.3 %/h | — | **~0.65–0.82 W** |

### §26.4 ⛔ Conclusion — the KWin/dGPU fix did not reduce drain

The post-fix figure is **statistically indistinguishable from the pre-fix overnight baseline.** §15.4 predicted the dGPU was the dominant drain source; a ~0.0–0.15 W delta, well inside the measurement uncertainty, is not consistent with that.

**Honest framing:**
- `runtime_suspended_time == 0` was a **real defect**, and it is **genuinely fixed** (verified repeatedly, including across hibernate and suspend cycles).
- It was **not the drain source**.
- §15.4 must be downgraded from "root cause" to "a real bug, fixed, with no measurable power effect."

**Caveats worth keeping:** the awake-time subtraction carries roughly ±0.15 W of uncertainty, and the original §15.2 baselines were themselves measured loosely (percentage-based, no formal protocol). Both point at the same remedy — proper instrumentation (Part VII).

### §26.5 What this implies

At ~0.8 W on a 64.7 Wh pack:

| Duration | Cost |
|---|---|
| 4 h | **5%** |
| 8 h | **10%** |
| 12 h | **15%** |
| 24 h | **30%** |
| 48 h | **59%** |
| To empty | **~3.4 days** |

Two consequences:
1. **The hibernate backstop matters more, not less.** The user's stated 4–24 h target costs 5–30%.
2. **The remaining ~0.8 W is elsewhere** — S0ix never engaging on the NVIDIA side, wake sources, radios, ASPM, NVMe. That is now the critical path (§28).

---
---

# PART IV — CONSOLIDATED CURRENT SYSTEM STATE

*(as of 2026-08-19, before applying Part VI)*

## §IV.1 Active files — hibernate/suspend stack

| Path | Content / purpose | Session | Status |
|---|---|---|---|
| `/swapfile` | 20 GiB, ext4, 14 extents, prio `-1`, offset `220407808` | 1 | ✅ working |
| `/etc/fstab` | `/swapfile none swap defaults 0 0` | 1 | ✅ |
| `/etc/mkinitcpio.conf` | `MODULES=(amdgpu)`; `HOOKS=(base systemd plymouth autodetect microcode modconf kms keyboard sd-vconsole block resume filesystems fsck)` | 1 | ✅ |
| `/etc/mkinitcpio.conf.d/10-chwd.conf` | `MODULES+=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)` — **auto-generated by chwd, do not edit** | distro | ⚠️ nvidia in initramfs; harmless now that Preserve=0 |
| `/etc/modprobe.d/nvidia-power-management.conf` | `NVreg_PreserveVideoMemoryAllocations=0`<br>`NVreg_TemporaryFilePath=/var/tmp` | **3** | ✅ **THE HIBERNATE FIX** |
| `/etc/systemd/system/systemd-hibernate.service.d/60-freeze-sessions.conf` | `SYSTEMD_SLEEP_FREEZE_USER_SESSIONS=true` | **3** | ✅ active & verified |
| `/etc/systemd/system/systemd-suspend-then-hibernate.service.d/60-freeze-sessions.conf` | same | **3** | ✅ active |
| `/etc/systemd/sleep.conf.d/10-hibernate-delay.conf` | `HibernateDelaySec=2min` | 3 | ⚠️ **test value — must be replaced, see Part VI** |
| `~/.config/plasma-workspace/env/kwin-gpu.sh` | resolve-then-export `KWIN_DRM_DEVICES` → iGPU | 2 | ✅ works; **no power benefit** |
| `/etc/default/limine` + `/boot/limine.conf` | cmdline incl. `rtc_cmos.use_acpi_alarm=1` | 3 | ✅ live, **did not fix RTC wake** |
| `/etc/default/limine.bak` | backup taken before the cmdline edit | 3 | 🧹 removable |
| ufw rule | `allow from 192.168.1.0/24 to any port 22 proto tcp` | 3 | ✅ |

## §IV.2 Kernel command line (current)

```
quiet nowatchdog splash rw root=UUID=2b44456f-dc52-489f-8d2a-c2453136cee1
amdgpu.dcdebugmask=0x40000 rtc_cmos.use_acpi_alarm=1
```

No `resume=` / `resume_offset=` — correctly unnecessary; systemd 261 uses the `HibernateLocation` EFI variable. Confirmed twice (§5.2, §7).

## §IV.3 Services

| Unit | State | Note |
|---|---|---|
| `nvidia-suspend.service` | enabled | manual, S1; possibly redundant on 610 |
| `nvidia-hibernate.service` | enabled | manual, S1; possibly redundant |
| `nvidia-resume.service` | enabled | manual, S1; possibly redundant |
| `nvidia-suspend-then-hibernate.service` | **disabled** | deliberately, to limit variables |
| `nvidia-powerd.service` | enabled | Dynamic Boost, unrelated |
| `nvidia-persistenced.service` | disabled | — |
| `sshd.service` | **enabled** | added S3 |

All nvidia units carry `PRESET: disabled` — upstream's 595+ intent.

## §IV.4 Driver parameters (verified)

```
PreserveVideoMemoryAllocations: 0     ✅ explicitly set
UseKernelSuspendNotifiers:      1     ✅ the modern path
TemporaryFilePath:              "/var/tmp"
```

## §IV.5 Runtime state, verified

| Item | Value |
|---|---|
| dGPU `runtime_status` at idle | `suspended` ✅ |
| dGPU `runtime_suspended_time` | nonzero and growing ✅ |
| dGPU after hibernate resume | returns to `suspended` ✅ |
| dGPU after suspend resume | `active` briefly, then `suspended` (autosuspend delay) ✅ |
| `prime-run` | works — RTX 3070 renders ✅ |
| Platform hw-sleep residency | ~95% |
| **s2idle draw** | **~0.8 W** |
| **NVIDIA S0ix status** | **`Disabled`** 🚩 still |
| RTC wake from s0ix | ⛔ broken (firmware) |
| `_BTP` battery alarm | present (`BAT0/alarm = 8992000`), untested |

## §IV.6 Unchanged / untouched

zswap (zstd) active alongside zram · lid behaviour **still default** · KDE power settings default · KDE session restore **still enabled** · UEFI Switchable Graphics · Fast Boot enabled.

---
---

# PART V — CORRECTIONS REGISTER

Everything the earlier documents assert that is now known to be wrong or incomplete.

| # | Location | Claim | Correction | Evidence |
|---|---|---|---|---|
| C1 | §8, §9 | `SYSTEMD_SLEEP_FREEZE_USER_SESSIONS` was the hibernate root cause and the fix | **Neither.** The drop-in was inert until 2026-08-18 18:06 due to a filename collision (`10-f…` < `10-n…`). Real fix is `NVreg_PreserveVideoMemoryAllocations=0`. | §22.2, §20 |
| C2 | §9.3 | "Real hibernate — SUCCESS" | **Unverified, probably a cold boot.** No `boot_id`/`uptime` check was done; app restoration is what KDE session restore produces anyway. | §17.1, §19.1 |
| C3 | §5.3, §8 | amdgpu `Mode Validation Warning` caused the black screen | **Benign.** Reappeared 2026-08-18 on a cold boot that reached a working desktop. | §19.1 |
| C4 | §15.2 | Battery is ~90 Wh; drain 1.1 W / 1.7 W | Battery is **64.7 Wh** (~28% wear). Real figures **0.81 W / 1.22 W**. | §0.2, §26.3 |
| C5 | §15.4 | KWin holding the dGPU was the root cause of s2idle drain | Real bug, genuinely fixed — but **zero measurable power effect**. Drain unchanged at ~0.8 W. | §26 |
| C6 | §15.12 | Post-fix prediction ~5–7% / 8 h | Actual ≈ **10% / 8 h equivalent**; no improvement over baseline. | §26.2 |
| C7 | §15.14 #9 | Use `rtcwake` sampling for A/B measurement | **Not available** — RTC alarms don't fire from s0ix on this machine. Use manual-wake timed A/B. | §25.7 |
| C8 | §13 step 6 | `HandleLidSwitch=suspend-then-hibernate` + `HibernateDelaySec=45min` | The **delay is inoperative**. Config redesigned around `_BTP` only. | §25, Part VI |
| C9 | Session 3 (mid-session) | "`HandleLidSwitch=suspend` + `HibernateOnACPower=no` gives a battery backstop" | **Wrong** — the low-battery trigger exists only inside STH. | §25.8 |
| C10 | §11, §15.10 | `NVreg_PreserveVideoMemoryAllocations=1` listed as correct config | It is the **cause of the `-5` failure** on driver 610. | §19.3 |
| C11 | *implicit* | Removing a modprobe option = disabling it | For this parameter, unset → default **`2` (auto)**, which re-enables it when `UseKernelSuspendNotifiers=1`. Must set `0` explicitly. | §20.1 |
| C12 | *implicit* | `/etc` drop-ins override `/usr/lib` drop-ins | Only for **identically named** files. Otherwise pure lexicographic filename order across all directories. | §22.2 |
| C13 | §25 working note | `batt_status: dead` indicates a failing CMOS cell | **False reading** — no discrete cell on this board; RTC is fed from the main pack. | §25.5 |

---
---

# PART VI — FINAL CONFIGURATION TO APPLY

**Not yet applied.** This is the end-state design given everything above.

## §VI.1 Design rationale

| Requirement | Mechanism | Status |
|---|---|---|
| Lid close → suspend | `HandleLidSwitch=suspend-then-hibernate` | ✅ works |
| Instant wake for short closures | s2idle | ✅ works |
| Auto-hibernate after N hours | `HibernateDelaySec` | ⛔ **impossible** — no RTC wake |
| Never lose state to a flat battery | ACPI `_BTP` trip point inside STH | ⚠️ plausible, untested |
| Don't hibernate while docked | `HandleLidSwitchExternalPower=suspend` | ✅ |
| Deliberate hibernate before travel | `systemctl hibernate` | ✅ 4/4 reliable |
| Clean shutdowns, no app restore | KDE "Start with an empty session" | ⬜ trivial |

**Why STH rather than plain suspend:** with no `HibernateDelaySec`, STH's only trigger is the battery alarm, which uses a different GPE than the RTC and may work. If it doesn't, STH behaves identically to plain suspend. Strictly no worse.

## §VI.2 Commands

```bash
# 0. Clear any stale RTC alarm — a pending one makes STH fail to suspend at all
echo 0 | sudo tee /sys/class/rtc/rtc0/wakealarm
cat /sys/class/rtc/rtc0/wakealarm        # must read 0

# 1. Sleep policy — replaces the 2min test value
sudo tee /etc/systemd/sleep.conf.d/10-hibernate-delay.conf <<'EOF'
[Sleep]
# No HibernateDelaySec: RTC alarms do not fire from s0ix on this platform.
# Verified exhaustively 2026-08-19 (see §25) — both the legacy CMOS IRQ8 path
# and the ACPI SCI path (rtc_cmos.use_acpi_alarm=1) fail identically during
# s0ix, while both work while awake. Firmware-level; not fixable from Linux.
# Rely on the ACPI _BTP battery trip point instead (BAT0/alarm is populated).
HibernateOnACPower=no
EOF

# 2. Lid behaviour
sudo mkdir -p /etc/systemd/logind.conf.d
sudo tee /etc/systemd/logind.conf.d/10-lid.conf <<'EOF'
[Login]
HandleLidSwitch=suspend-then-hibernate
HandleLidSwitchExternalPower=suspend
HandleLidSwitchDocked=ignore
EOF

sudo systemctl daemon-reload
sudo systemctl restart systemd-logind
```

## §VI.3 GUI steps

1. **System Settings → Power Management** → set the lid-close action to **"Do nothing"**, so KDE does not race logind.
2. **System Settings → Session → Desktop Session** → **"Start with an empty session"**. This is the half of the original request that requires no testing: hibernate bypasses the session manager, so full state restore still works via the lid path while ordinary shutdowns become clean.

## §VI.4 Resulting behaviour

| Scenario | Behaviour |
|---|---|
| Lid closed, on battery | Suspend at ~0.8 W → ~3.4 days runway → hibernate at ~5% *(if `_BTP` works)* |
| Lid closed, on AC | Plain suspend, no hibernation |
| Docked | Ignored |
| Before travel / long gaps | `systemctl hibernate` manually — 4/4 reliable |
| Ordinary shutdown | Clean, no app restore |

## §VI.5 Verification after the next lid-close suspend

```bash
journalctl -b 0 -u systemd-suspend-then-hibernate.service | grep -iE 'battery|alarm|estimat|freeze'
```
Looking for evidence that systemd armed the battery trip point. Full validation requires actually running the battery down to ~5% — a chore, but the only honest test.

---
---

# PART VII — INSTRUMENTATION (designed, not yet applied)

## §VII.1 Why

The single most important number in this whole investigation — 0.8 W — was obtained **by accident**, from a suspend that happened to run overnight, with an awake-time subtraction carrying ±0.15 W. That is not good enough to A/B six remaining candidate optimisations.

## §VII.2 Mechanics

`systemd-sleep` executes every executable in `/usr/lib/systemd/system-sleep/`, passing two arguments — `pre`|`post`, and `suspend`|`hibernate`|`hybrid-sleep`|`suspend-then-hibernate` — plus a `SYSTEMD_SLEEP_ACTION` environment variable. The second argument matters here, because STH must be distinguishable from plain suspend in the log.

Two caveats: systemd's own man page describes this mechanism as a hack, and the scripts run **concurrently**, not in sequence (ordered execution would require custom units). For logging, concurrency is fine.

Prior art: `batenergy`, a script for this directory that logs lines of the form "Duration of 0 days 3 hours 26 minutes sleeping (suspend)" and "Battery energy change of −4.5 % (−2320 mWh) at an average rate of −1.30 %/h (−673 mW)". The minimal version is just `case $1 in pre|post) tlp-stat -b ;; esac`.

## §VII.3 The script

Merges the two separately-planned hooks from §15.10 (`battery-log.sh` + `power-audit.sh`), because battery delta alone tells you *that* a cycle was bad, never *why* — and by the time you read the number the evidence is gone.

```bash
#!/usr/bin/env bash
# /usr/lib/systemd/system-sleep/99-sleep-audit.sh    chmod 0755
set -u

STATE=/run/sleep-audit.state
LOG=/var/log/sleep-audit.tsv
BAT=/sys/class/power_supply/BAT0
GPU=/sys/bus/pci/devices/0000:01:00.0

rd() { cat "$1" 2>/dev/null || echo 0; }

energy() {
  if [ -r $BAT/energy_now ]; then rd $BAT/energy_now
  else echo $(( $(rd $BAT/charge_now) * $(rd $BAT/voltage_now) / 1000000 )); fi
}

snapshot() {
  T=$(date +%s)
  E=$(energy)
  CAP=$(rd $BAT/capacity)
  ACON=$(rd /sys/class/power_supply/AC*/online)
  HW=$(rd /sys/power/suspend_stats/total_hw_sleep)
  GRS=$(rd $GPU/power/runtime_status)
  GPS=$(rd $GPU/power_state)
  GST=$(rd $GPU/power/runtime_suspended_time)
}

case "$1/$2" in
  pre/*)
    snapshot
    printf '%s %s %s %s %s %s %s %s\n' \
      "$T" "$E" "$CAP" "$HW" "$GRS" "$GPS" "$GST" "$ACON" > $STATE
    {
      echo "=== pre-sleep audit ($2) ==="
      echo "  battery: ${CAP}% / ${E} uWh   AC=${ACON}"
      echo "  dGPU: $GRS / $GPS"
      echo "--- PCI devices still in D0 ---"
      for d in /sys/bus/pci/devices/*; do
        p=$(rd $d/power_state)
        [ "$p" = "D0" ] && \
          echo "  D0: $(basename $d) [$(rd $d/power/runtime_status)] ctrl=$(rd $d/power/control)"
      done
      echo "--- enabled ACPI wake sources ---"
      grep -E '\*enabled' /proc/acpi/wakeup
      echo "--- rfkill ---"
      rfkill list 2>/dev/null | grep -E 'Soft|Hard|:'
      echo "--- rtc wakealarm ---"
      rd /sys/class/rtc/rtc0/wakealarm
    } | systemd-cat -t sleep-audit -p info
    ;;

  post/*)
    [ -r $STATE ] || exit 0
    read -r T0 E0 C0 HW0 _ _ GST0 AC0 < $STATE
    snapshot
    DT=$(( T - T0 )); [ $DT -lt 60 ] && exit 0
    DE=$(( E0 - E ))                            # uWh consumed
    MW=$(( DE * 3600 / DT ))                    # average mW
    PCTH=$(( (C0 - CAP) * 3600 * 100 / DT ))    # %/h x100
    DHW=$(( (HW - HW0) / 1000000 ))             # s of hw sleep this cycle
    RES=$(( DHW * 100 / DT ))                   # hw-sleep residency %
    DGST=$(( (GST - GST0) / 1000 ))             # dGPU rt-suspended s this cycle

    [ -s $LOG ] || printf 'date\taction\tdur_s\tdmWh\tavg_mW\tpct_per_h\thw_resid%%\tdgpu_susp_s\tgpu_pstate\tac0\tac1\n' > $LOG
    printf '%s\t%s\t%d\t%d\t%d\t%s.%02d\t%d\t%d\t%s\t%s\t%s\n' \
      "$(date -Is)" "$2" "$DT" "$((DE/1000))" "$MW" \
      "$((PCTH/100))" "$((PCTH%100))" "$RES" "$DGST" "$GPS" "$AC0" "$ACON" \
      | tee -a $LOG | systemd-cat -t sleep-audit -p notice
    rm -f $STATE
    ;;
esac
exit 0
```

## §VII.4 Why each column earns its place

| Column | Answers |
|---|---|
| `avg_mW` | The actual number. In watts, not percent — see below. |
| `hw_resid%` | Did the *platform* sleep on **this** cycle? Converts §15.8's cumulative "platform healthy" claim into a per-cycle check. |
| `dgpu_susp_s` | Did the dGPU stay runtime-suspended across the sleep? Direct regression detector for the Session 2 fix, and the STH intermediate-wake failure mode. |
| `ac0` / `ac1` | **Catches exactly the Test 4 error** (§21.4) where a brief AC connection invalidated the measurement. |
| pre-sleep D0 list | The `power-audit.sh` idea. If drain regresses in six weeks, you have a record of what was awake going in. |
| `/proc/acpi/wakeup` dump | Gives you the §15.10 audit **for free on every cycle** instead of as a one-off. |
| `rfkill` state | Makes the radio A/B trivially reproducible. |

## §VII.5 Measurement protocol — adapted for no RTC wake

⛔ The originally planned `sudo rtcwake -m no -s 2700 && systemctl suspend` method **does not work here** (§25).

**Substitute protocol:**
1. Note the time; `systemctl suspend`.
2. Wait a **measured** interval — 45–60 min is the minimum for the µWh delta to clear sensor quantisation.
3. Wake manually (power button).
4. Read the TSV row.

Three runs an evening still beats one overnight, and each run produces a durable row rather than a remembered impression.

**Always compare in watts, not percent.** At 64.7 Wh, 1% ≈ 647 mWh, so a percentage reading over 45 minutes is nearly all rounding error. This also matters when comparing against other people's reports on differently-sized batteries.

---
---

# PART VIII — RESEARCH LIBRARY

Community and vendor findings gathered across sessions. Kept because several of these are the next things to try.

## §VIII.1 NVIDIA power-management parameters

**`NVreg_EnableS0ixPowerManagement=1`** — **the top untested candidate.**
Per NVIDIA: with this set and the system in s2idle, the driver calculates video memory usage at suspend time; below a threshold it copies VRAM contents to system memory and powers off the video memory along with the GPU; above the threshold it keeps VRAM in self-refresh while the rest of the GPU powers down. **Default is 0.** §15.3.1 shows `Video Memory: Active` with both *Self Refresh* and *Off* listed as Supported, and `S0ix Status: Disabled` on a platform reporting `Platform Support: Supported` — exactly the state this flag changes.

- Threshold defaults to **256 MB**, adjustable via `NVreg_S0ixPowerManagementVideoMemoryThreshold` — **but** there is a known issue on Linux 5.10+ causing the driver to ignore that threshold parameter. So the `…VideoMemoryThreshold=0` half of §15.10's reserved entry is likely a no-op. At 44 MiB VRAM in use, the default path should already choose full power-off.
- Verify afterwards via `cat /proc/driver/nvidia/params | grep -i s0ix`, **not** by trusting the modprobe file — a lesson learned the hard way with `PreserveVideoMemoryAllocations` (§20.1).
- Arch wiki pairs this with the three nvidia sleep services, which are already enabled.

**⚠️ `NVreg_DynamicPowerManagement=0x02` — recommend dropping from the plan.**
A recent report describes a GSP **Xid 120 panic during s2idle suspend** with this set, on an RTX 3080 Mobile / Ryzen 5900HX Optimus laptop with s2idle-only and no S3 — essentially this platform. Since the dGPU already reaches runtime-suspend without it, risk/reward is poor.

**`NVreg_EnableGpuFirmware=0`** — appears in many NVIDIA-laptop suspend configs; one report needed it (plus a specific kernel) to avoid a black screen on resume. This is a **correctness** fix, not a power fix. Suspend/resume works here — leave GSP alone.

**udev rule removing NVIDIA PCI sub-devices** — some distros ship this to improve idle power where the GPU audio/USB endpoints aren't needed. **Not applicable:** §15.3.3 shows `01:00.1` already suspends and there are no USB-C/UCSI functions on this GPU.

## §VIII.2 Platform / wake-source tuning

Recurring advice on s2idle drain threads:
- `rfkill` Wi-Fi and Bluetooth before sleep (clean two-run A/B).
- Disable Thunderbolt in UEFI.
- Limit C-states (counter-intuitive but reported).
- One NVIDIA-laptop fix was purely wake-source related: disabling **XHCI** (USB) and **TXHC** (Thunderbolt) wake via `/proc/acpi/wakeup`, accepting that resume then requires the built-in keyboard or trackpad.

The audit hook (Part VII) surfaces the wake-source list automatically on every cycle.

## §VIII.3 The S3 / DSDT question — GA503-specific

**§15.12 and §3 both treat "no S3" as fixed. On a GA503 that premise may be wrong.**

asus-linux.org documents that, depending on kernel version, the 2021/2022 Zephyrus G14/G15 hit issues with newer suspend methods like s0ix, and that a potential fix is to **patch the DSDT tables so the machine uses the older S3 method** — reported to work well on the 2021/2022 G14 and G15, though the patches cannot live in the main repo and will always be a manual matter. Tooling: Marco Laux's `g14-2021-s3-dsdt`, explicitly for enabling S3 legacy suspend on the ASUS Zephyrus G14/G15 2021/2022.

A G14 2021 user reported: s2idle was the only option by default and drained the battery in a few hours with heating; after the patch, deep sleep became available and default, the laptop became usable in portable mode, and it also fixed occasional screen-not-waking issues.

**Why this now matters more than before:**
- Under S3 the CPU is powered off and draw is minimal — well below the ~0.4–0.7 W s2idle floor model, and far below the measured 0.8 W.
- **S3 uses an entirely different wake path**, so it may also resolve the RTC-wake blocker (§25) — the Ryzen 5700U report explicitly found `rtcwake` works after switching to S3.

**Caveats:**
- After a BIOS update you must disable the old DSDT table and create a new one; DSDT tables change between BIOS versions. Real maintenance burden and a brick-the-boot risk on a rolling distro. Not the same class of reversibility as deleting one file.
- Not universally better — one Arch user on other hardware found a hidden BIOS "S3 Enable" option worked *worse* than s2idle (no wake-up, REISUB also dead). Test with an escape plan.
- The current NVIDIA config is tuned for the s2idle/hibernate path; S3 changes which driver code path runs.

**Sequencing:** do the cheap stuff first, get clean numbers, then decide. But it belongs in the plan as an open option, not a closed door.

## §VIII.4 Other GA503-specific findings

- The 2021 Zephyrus G15 (GA503Q) shipped with a **broken ACPI DSDT table** preventing s0ix suspend/resume from completing, specifically on units with a **second NVMe drive** — the secondary drive fails to suspend, delaying system suspend. **Not applicable here:** single 1 TB NVMe, and fixed in kernel 6.1+. Recorded so a future session doesn't rediscover it.
- **Chassis-level drain is not all OS-attributable.** One G15 owner reported losing 10–15% in 24 h with the machine *shut down* on Windows, persisting across disabling fast boot, removing Armoury Crate, and reinstalling drivers. Some non-zero floor on this chassis is EC/firmware.
- Nice parallel to §15.4: on Windows, if a process is using the NVIDIA GPU when Eco mode is activated, the GPU stays powered and the benefit is lost. Same failure mode as `kwin_wayland`, different OS.

## §VIII.5 Community drain numbers for calibration

| Source | Figure |
|---|---|
| Framework community, range of reports | 20–40% over an 8 h night at the bad end; down to 2%/night, which the poster considered unusually low for a non-LPDDR laptop |
| Framework 11th gen, s2idle "normal" | ~0.8 W, rising to ~1.5 W with two USB-A expansion cards |
| `batenergy` author's real logged cycle | −1.30 %/h (−673 mW) |
| **This machine, pre-fix** | **0.81 W** |
| **This machine, post-fix** | **~0.8 W** |

Read against those, **this machine sits squarely at "normal healthy s2idle."** It is not pathological — it is simply what s2idle costs on this class of hardware. The 2%/night outliers are S3-class numbers, which is exactly why §VIII.3 stays open.

Note that percentage comparisons across machines are misleading: at 64.7 Wh this laptop's %/h figures look *worse* than a Framework's for the same wattage.

## §VIII.6 systemd drop-in conventions

- Files in `.conf.d` directories are sorted **by filename in lexicographic order, regardless of which subdirectory they reside in**; for single-value options, the file sorted **last** wins.
- Recommended prefixes: **10–40 for `/usr/` (vendor)**, **60–90 for `/etc/` and `/run/` (local)**.
- Drop-ins in `/etc` take precedence over `/run` over `/usr/lib` **only when the filenames are identical** — rename the `/etc` one and the `/usr/lib` one reappears.
- To disable a vendor configuration file entirely, the recommended method is a symlink to `/dev/null` in `/etc` with the **same filename**.

## §VIII.7 AMD s2idle RTC-wake reports

| Report | Hardware | Finding |
|---|---|---|
| Ryzen 7 5700U, Lenovo, KDE | Cezanne-class | Hibernate and suspend work; STH stays suspended, hibernates only on manual wake. `rtcwake` works **only** after switching to S3 or unloading `amd_pmc`. |
| Framework, AMD | — | AMD kernel developer suggested `rtc_cmos.use_acpi_alarm=1`; solved it there — 1 h sleep → wake → hibernate worked. |
| ASUS Zenbook 14, Ryzen 5 7530U | s2idle, kernel 7.0.10, systemd 260 | Same symptom; attributed to `alarm_IRQ` staying `no`; fix is commit `torvalds/linux@f7ecfc3`, not yet in the Arch kernel package. |
| Arch, systemd 258 | — | Open systemd issue: suspends and never wakes automatically, but wakes and immediately hibernates if turned on manually. |

**This machine matches the symptom but neither published fix applies:** `use_acpi_alarm=1` was tried and failed (§25.3), and `alarm_IRQ` is already `yes` (§25.4). The failure is one layer lower.

## §VIII.8 The freeze-sessions debate

- **Arch news:** systemd-sleep and systemd-homed were updated to freeze user sessions when entering sleep; this is known to cause issues with the proprietary NVIDIA drivers, and packagers may want to add drop-ins setting `SYSTEMD_SLEEP_FREEZE_USER_SESSIONS=false`.
- **KDE Linux position:** nvidia-utils ships a blanket rule disabling the freeze, which may be necessary on NVIDIA hardware but is actively warned against by systemd. On hybrid laptops it is now suspected of *causing* problems — because user sessions are not frozen, Wayland clients keep running while the GPU is being suspended, so when the GPU disappears mid-operation clients crash or the compositor hangs with a stale framebuffer.
- nvidia-utils ships this drop-in for **five** units: `systemd-suspend-then-hibernate`, `systemd-hibernate`, `systemd-hybrid-sleep`, `systemd-homed`, and `systemd-suspend`. **Only two are currently overridden here** — plain `systemd-suspend.service` still runs unfrozen. Fine for now, but an inconsistency to revisit alongside the drain work.

---
---

# PART IX — ROLLBACK PROCEDURES

## §IX.1 Hibernate fix only (revert to broken state — don't)
```bash
sudo rm /etc/modprobe.d/nvidia-power-management.conf
sudo mkinitcpio -P && sudo reboot
```

## §IX.2 Freeze override
```bash
sudo rm -f /etc/systemd/system/systemd-hibernate.service.d/60-freeze-sessions.conf
sudo rm -f /etc/systemd/system/systemd-suspend-then-hibernate.service.d/60-freeze-sessions.conf
sudo systemctl daemon-reload
```
Reverts to nvidia's `=false`. Tests 1–3 passed in that state, so this is safe for direct hibernate.

## §IX.3 RTC kernel parameter
```bash
sudo cp /etc/default/limine.bak /etc/default/limine
sudo limine-update && sudo reboot
```
Recommended to **keep** — it is the recommended setting on AMD, harmless, and reverting costs a reboot.

## §IX.4 Lid / sleep policy
```bash
sudo rm /etc/systemd/logind.conf.d/10-lid.conf
sudo rm /etc/systemd/sleep.conf.d/10-hibernate-delay.conf
sudo systemctl daemon-reload && sudo systemctl restart systemd-logind
```

## §IX.5 KWin GPU restriction (Session 2)
```bash
rm ~/.config/plasma-workspace/env/kwin-gpu.sh
# log out and back in
```
Restores default multi-GPU behaviour including USB-C/DP dGPU output.
**Emergency (black screen at login):** `Ctrl+Alt+F3` → `rm` the file → `sudo reboot`. Login screen remains functional because the script is user-scoped — verified under fire during Attempt B.

**Given §26, this fix now has no demonstrated power benefit.** Retained because it is harmless, it fixes a real runtime-PM defect, and it costs nothing. Removing it would restore USB-C display output.

## §IX.6 Full hibernate stack teardown
```bash
sudo rm -f /etc/systemd/system/systemd-hibernate.service.d/60-freeze-sessions.conf
sudo rm -f /etc/systemd/system/systemd-suspend-then-hibernate.service.d/60-freeze-sessions.conf
sudo rm -f /etc/modprobe.d/nvidia-power-management.conf
sudo swapoff /swapfile && sudo rm /swapfile
# remove /swapfile line from /etc/fstab
# remove 'resume' hook from /etc/mkinitcpio.conf
sudo systemctl disable nvidia-suspend.service nvidia-hibernate.service nvidia-resume.service
sudo mkinitcpio -P && sudo reboot
```

## §IX.7 SSH / firewall
```bash
sudo systemctl disable --now sshd
sudo ufw delete allow from 192.168.1.0/24 to any port 22 proto tcp
```

---
---

# PART X — OPEN ITEMS & NEXT SESSION PLAN

## §27 Outstanding items

### §27.1 Apply Part VI ⬜
Lid config, sleep policy, KDE lid action → "Do nothing", KDE "Start with an empty session". None of it tested end-to-end yet.

### §27.2 Replace the test `HibernateDelaySec=2min` ⚠️
Currently live. Harmless in practice (the timer can't fire) but misleading, and it would surprise you if RTC wake ever started working.

### §27.3 Verify the `_BTP` battery trigger ⬜
Requires draining to ~5%. Cheap partial check after any lid-close suspend:
```bash
journalctl -b 0 -u systemd-suspend-then-hibernate.service | grep -iE 'battery|alarm|estimat'
```

### §27.4 `fsck` — 🚩 **five forced power-offs, three skipped reboot windows**
Two in Session 1, plus the black-screen power-cycling on 2026-08-18. Three reboots have happened since without doing it.

```bash
# from live USB
lsblk -f
sudo fsck -f /dev/nvme0n1pX     # root, unmounted
```
⚠️ **Ordering rule: never fsck the root filesystem while a hibernation image exists.** Resume first, shut down cleanly, then fsck.

### §27.5 KDE splash restoration ⬜
Lost during Session 1's `mkinitcpio` regeneration. Cosmetic, still outstanding.

### §27.6 Cosmetic: black screen between ROG logo and PLM on resume
Plymouth not getting a framebuffer during the resume boot — same gap noted in the boot-customisation notes. Purely visual; resume works. Parked.

### §27.7 Config hygiene
- `resume` hook in HOOKS is **redundant** — with the `systemd` hook, resume is handled by `systemd-hibernate-resume-generator` via the `HibernateLocation` EFI variable (confirmed §7). Harmless dead config.
- The three manually-enabled nvidia sleep services are likely redundant on 610 given `UseKernelSuspendNotifiers: 1`. Verify by disabling and re-running one hibernate test — **one variable at a time**.
- `/etc/default/limine.bak` can be removed once the cmdline is settled.
- The **system-context document is stale**: it records the reverted HOOKS line, not the live one (`plymouth` early, `kms`, `resume`), and does not mention `rtc_cmos.use_acpi_alarm=1`.

## §28 Next session — ordered plan

### §28.1 🥇 `NVreg_EnableS0ixPowerManagement=1`
The highest-value untested change. §15.3.1 still shows `S0ix Status: Disabled` while the platform reports it Supported. §15.8 deferred it on the reasoning that it was irrelevant while KWin prevented D3cold — **that reasoning expired when Attempt C landed**, and §26 makes it the leading remaining hypothesis.

```bash
# add to /etc/modprobe.d/nvidia-power-management.conf
options nvidia NVreg_EnableS0ixPowerManagement=1
sudo mkinitcpio -P && sudo reboot
cat /proc/driver/nvidia/params | grep -i s0ix           # verify — do not trust the file
cat /proc/driver/nvidia/gpus/0000:01:00.0/power         # want Status: Enabled
```
Do **not** add the VRAM threshold parameter (ignored on 5.10+). Fold the `fsck` into this reboot.

### §28.2 🥈 Instrumentation
Install the Part VII hook **before** §28.1 if possible, so the S0ix change gets a clean before/after. Then 3 timed manual-wake runs per evening.

### §28.3 🥉 Wake-source and radio audit
Outstanding since Session 1. The hook surfaces `/proc/acpi/wakeup` and `rfkill` automatically; then A/B with radios off.

### §28.4 Then: evaluate the S3 DSDT patch
Only with clean numbers in hand. It would potentially fix **both** the 0.8 W drain and the RTC-wake blocker, at the cost of a real BIOS-update maintenance tax. See §VIII.3.

### §28.5 Also worth doing
- Disable the three redundant nvidia services and retest (one variable).
- Check UEFI for **"Power On By RTC"** under Advanced → APM (usually S5-only, but free to look) and for an OS-type / sleep-mode toggle.
- Watch for the AMD s2idle RTC fix landing in CachyOS; re-test §25 when the kernel moves.
- Consider extending the freeze override to `systemd-suspend.service` for consistency (§VIII.8).

### §28.6 Explicitly **dropped** from the plan
- `NVreg_DynamicPowerManagement=0x02` — Xid 120 panic risk on this exact platform class, no upside since D3cold already works.
- `NVreg_S0ixPowerManagementVideoMemoryThreshold` — ignored on kernel 5.10+.
- `NVreg_EnableGpuFirmware=0` — a correctness fix for a problem this machine doesn't have.
- NVIDIA sub-device removal udev rule — no such sub-devices here.
- `amd_s2idle.py` — platform residency already ~95%.
- `envycontrol` — never needed.
- `rtcwake`-based A/B sampling — impossible here.

---
---

# PART XI — MASTER LESSONS & GOTCHAS

## Diagnostic method
1. **Only `boot_id` and `uptime -s` prove a resume.** Restored windows prove nothing — KDE session restore produces the same picture on a cold boot. Add a backgrounded job and a tmpfs cwd as corroboration.
2. **A gap between `hibernation entry` and `hibernation exit` proves real power-off.** Without it you can't distinguish "hibernated" from "aborted and stayed up."
3. **A successful hibernate log looks truncated.** No "image written" line, no resume-side lines. They were printed after the snapshot, or by a kernel that got replaced. Seeing `Read X kbytes` on resume means it **failed**.
4. **A successful resume keeps `boot_id`, so `journalctl -b 0` is correct** — the resume is a continuation of boot 0, not a new boot. `-b -1` points somewhere else entirely.
5. **Error codes localise the failure stage.** `-16`/`-22` = image never read (write-side problem). `-5` after a successful `Read … kbytes` = device quiesce (driver problem). Different codes, different subsystems, do not conflate.
6. **Grep widely on the first pass.** The narrow grep in Test 1 filtered out the `NVRM:` line that named the culprit outright.
7. **Change one variable at a time.** Enforced throughout Session 3 — the nvidia STH service was deliberately left disabled so that a failure would have one candidate cause.

## systemd
8. **Drop-ins merge by filename across all directories.** `/etc` beats `/usr/lib` **only for identical filenames**. `10-freeze-sessions.conf` silently lost to `10-nvidia-no-freeze-session.conf` because `f` < `n`, for the entire life of this investigation.
9. **Use the documented prefix ranges:** 10–40 vendor, 60–90 local.
10. **Verify overrides with `systemctl cat` / `systemctl show -p Environment`**, not by looking at your own file. Then verify again in the **runtime log** — Test 4's `Successfully froze unit 'user.slice'` was the only real proof.
11. **`Environment=` does not propagate between units.** `systemd-hibernate` and `systemd-suspend-then-hibernate` are separate units needing separate drop-ins.
12. **`HibernateDelaySec` and the low-battery trigger exist only inside `suspend-then-hibernate`.** Plain suspend has neither.

## Kernel / driver
13. **Unset ≠ disabled.** `NVreg_PreserveVideoMemoryAllocations` defaults to `2` (auto), which *enables* preservation when `UseKernelSuspendNotifiers=1`. Set `0` explicitly.
14. **Verify module parameters at `/proc/driver/nvidia/params`, never from the modprobe file.**
15. **Match parameters to driver generation.** `PreserveVideoMemoryAllocations` is 430–590; on 595+/610 the mechanism is `UseKernelSuspendNotifiers`. A stale parameter can break the modern path outright.
16. **`nvidia-smi` wakes the GPU** — never use it to judge runtime-PM state. Use sysfs.
17. **Runtime PM has an autosuspend delay.** `active` immediately after resume is normal. Wait ≥2 min.
18. **`runtime_suspended_time` uses monotonic time**, which does not advance during s2idle. Compare against awake time, not wall clock.
19. **`last_hw_sleep` ≠ overnight total** — it is the last cycle only. Use `total_hw_sleep`.
20. **Platform sleep and device sleep are independent** — 95% hw-sleep residency coexisted with a dGPU that never suspended once.
21. **`/sys/power/pm_test` does not reset itself.** Check `[none]` before every real test. Telltale of a silent dry-run: `hibernation debug: Waiting for 5 second(s).`
22. **A stale RTC wakealarm blocks STH from suspending at all.** Clear it after every `rtcwake` experiment.
23. **`batt_status: dead` in `/proc/driver/rtc` is meaningless on boards without a discrete CMOS cell.**
24. **`rtcwake -m no` + `systemctl suspend`** is the right pattern — `-m mem` writes to `/sys/power/state` directly, bypassing systemd-sleep hooks and the nvidia services.
25. **`/proc/acpi/wakeup` does not list the RTC** — it is an ACPI fixed event, tracked at `/sys/firmware/acpi/interrupts/ff_rt_clk`.
26. **Distinguish "event not delivered" from "event not generated."** The `ff_rt_clk` counter incrementing while awake but not during s0ix is what proved the failure is firmware, not kernel.

## Measurement
27. **Measure in watts, not percent.** Percentages hide capacity and are meaningless across machines.
28. **Verify assumed capacity.** A 90 Wh assumption on a 64.7 Wh pack inflated every wattage in the document by ~39%.
29. **Log AC state on both sides of every measurement.** Test 4's "battery gained 95 mWh" was a moment of AC contact.
30. **Instrument early.** The most important number in this investigation was obtained by accident, with ±20% uncertainty, after two sessions of building theory on top of loosely-measured baselines.
31. **A fix that works is not a fix that helps.** The dGPU runtime-PM defect was real, reproducible, and correctly fixed — and it changed the power number by approximately nothing.

## Environment
32. **`KWIN_DRM_DEVICES` splits on `:`** — never pass a raw `by-path` name; resolve it with `readlink -f` first.
33. **The shell strips backslashes when sourcing** — escaping intended for the *variable value* will not survive.
34. **`/dev/dri/cardN` is unstable** across boots; PCI addresses are not.
35. **Never verify a device path by assumption** — Attempt A pointed at a `card0` that does not exist.
36. **User-scoped env beats `/etc/environment`** on Wayland — it cannot break the greeter. Proven under fire.
37. **Auto-generated configs need override files, not edits.** `10-chwd.conf` says "PLEASE DO NOT EDIT"; use `20-*.conf`.
38. **Set up SSH before sleep testing.** `ufw` blocked it here; a running sshd is not the same as a reachable one. Symptom mapping: refused = nothing listening; timeout = firewall.
39. **⚠️ Never press power during a hibernation image write.** After an STH wake the screen stays black *because* the write is happening. Wait ~10 s for automatic power-off.

---
---

# APPENDIX A — Command Cookbook

## Pre-test ritual
```bash
cat /sys/power/pm_test                       # must be [none]
cat /sys/class/rtc/rtc0/wakealarm            # must be 0
swapon --show                                # /swapfile present, prio -1
uptime -s                                    # RECORD OFF-MACHINE
cat /proc/sys/kernel/random/boot_id          # RECORD OFF-MACHINE
cat /sys/class/power_supply/BAT0/energy_now
cat /sys/class/power_supply/AC*/online

cd /tmp && mkdir -p hib-marker && cd hib-marker
export MARKER="test-$(date +%H%M%S)"; echo $MARKER
sleep 9999 &
```

## Post-test verification
```bash
uptime -s; cat /proc/sys/kernel/random/boot_id     # both UNCHANGED = real resume
echo $MARKER; jobs; pwd
cat /sys/class/power_supply/BAT0/energy_now
journalctl -b 0 -k -o short-precise --no-hostname \
  -g "suspend entry|suspend exit|hibernation entry|hibernation exit|quiesce|NVRM|Xid"
journalctl -b 0 -u systemd-hibernate.service --no-pager | tail -20
```

## dGPU state (wait ≥2 min after any resume)
```bash
cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status      # want: suspended
cat /sys/bus/pci/devices/0000:01:00.0/power_state               # want: D3cold
cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_suspended_time
cat /proc/driver/nvidia/gpus/0000:01:00.0/power
cat /proc/driver/nvidia/params | grep -iE 'preserve|suspendnotif|s0ix|temporary'
prime-run glxinfo | grep "OpenGL renderer"
# NEVER nvidia-smi for PM state
```

Continuous watch:
```bash
for i in $(seq 1 12); do
  printf '%2dm %s  st=%s  d3=%s\n' "$i" \
    "$(cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status)" \
    "$(cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_suspended_time)" \
    "$(cat /sys/bus/pci/devices/0000:01:00.0/power_state)"
  sleep 10
done
```

## Drop-in / override inspection
```bash
systemctl cat systemd-hibernate.service | grep -i freeze
systemctl show systemd-hibernate.service -p Environment
systemctl show systemd-suspend-then-hibernate.service -p Environment
ls -1 /usr/lib/systemd/system/systemd-hibernate.service.d/ \
      /etc/systemd/system/systemd-hibernate.service.d/
systemd-analyze cat-config systemd/sleep.conf
```

## RTC diagnostics
```bash
timedatectl | grep -iE 'RTC|Time zone'
cat /proc/driver/rtc
cat /sys/module/rtc_cmos/parameters/use_acpi_alarm
cat /sys/class/rtc/rtc0/device/power/wakeup
grep . /sys/firmware/acpi/interrupts/ff_rt_clk /sys/firmware/acpi/interrupts/sci
grep -i rtc /proc/interrupts
cat /sys/class/power_supply/BAT0/alarm
echo 0 | sudo tee /sys/class/rtc/rtc0/wakealarm      # ALWAYS clean up
```

## Platform residency
```bash
cat /sys/power/suspend_stats/total_hw_sleep
cat /sys/power/suspend_stats/last_hw_sleep
cat /sys/power/suspend_stats/success
cat /sys/power/suspend_stats/fail
```

## Emergency
```
Ctrl+Alt+F3                        # VT switch
ssh ekj@192.168.1.220              # from another machine on the LAN
REISUB                             # sysrq (enabled)
```

---

# APPENDIX B — Complete Sleep-Cycle Timeline

All cycles logged on boot `cdb92f27-a9db-4fdd-af27-f75c542983f0` (booted 2026-08-18 17:08:57), plus prior-boot events.

| # | Date | Type | Entry | Exit | Duration | Result |
|---|---|---|---|---|---|---|
| — | 08-17 | hibernate (pm_test dry-run) | 20:46:37 | 20:47:01 | 24 s | dry-run only |
| — | 08-17 | hibernate (pm_test dry-run) | 20:48:04 | 20:48:26 | 22 s | dry-run only |
| — | 08-17 | hibernate "real" (§9.3) | — | — | — | ⚠️ unverified, likely cold boot |
| — | 08-18 | hibernate (pre-fix) | ~16:4x | 16:47:50 | — | ❌ **FAIL `-5`**, cold boot |
| 1 | 08-18 | **hibernate** | 17:10:33.984 | 17:11:24.183 | **51 s** | ✅ **PASS** |
| — | 08-18 | suspend | 17:26:30.025 | 17:47:24.546 | 20 m 55 s | ✅ (incidental) |
| 2 | 08-18 | **hibernate** | 17:50:10.344 | 17:52:56.192 | **2 m 46 s** | ✅ **PASS** |
| 3 | 08-18 | **hibernate** (dGPU active) | 17:57:17.509 | 17:58:55.196 | **1 m 38 s** | ✅ **PASS** |
| 4 | 08-18 | **hibernate** (freeze active) | 18:06:04.036 | 18:34:29.214 | **28 m 25 s** | ✅ **PASS** |
| — | 08-18 | suspend (verification) | 18:40:13.384 | 18:41:43.371 | 1 m 30 s | ✅ clean |
| 5a | 08-18 | **STH — suspend leg** | 18:45:30.598 | 18:48:45.717 | 3 m 15 s | ⚠️ no self-wake at 2 min |
| 5b | 08-18 | **STH — hibernate leg** | 18:48:45.741 | 18:49:57.241 | 1 m 12 s | ✅ restored |
| 6 | 08-18→19 | **suspend (overnight)** | 18:53:37.502 | 07:45:39.369 | **12 h 52 m** | 📊 **drain test: ~0.8 W** |
| 7 | 08-19 | suspend (rtcwake test 1) | 07:48:18.433 | 07:53:13.730 | 4 m 55 s | ❌ no RTC wake |
| — | 08-19 | *reboot* — `rtc_cmos.use_acpi_alarm=1` added | 08:01:37 | — | — | — |
| 8 | 08-19 | suspend (rtcwake test 2) | 08:03:13.310 | 08:06:41.377 | 3 m 28 s | ❌ no RTC wake |
| 9 | 08-19 | suspend (`ff_rt_clk` enabled) | 08:26:24.880 | 08:29:52.692 | 3 m 28 s | ❌ no RTC wake, counter static |

**Totals:** 4/4 direct hibernates passed · 1/1 STH mechanism verified · 0/3 RTC wake attempts succeeded.

---

## Closing summary

**Session 3 delivered:** a working, verified hibernate — root-caused to `NVreg_PreserveVideoMemoryAllocations` on the 610 driver — plus the discovery that a Session-1 "fix" had never been active, plus the first properly-grounded drain measurement, which refuted Session 2's central hypothesis.

**What remains:** the original problem. The machine draws **~0.8 W in s2idle**, which is unremarkable for this class of hardware, and neither of the two fixes applied so far changed it. Timer-based auto-hibernate is blocked by firmware. The realistic paths forward are `NVreg_EnableS0ixPowerManagement=1`, a wake-source audit, and — if those disappoint — the S3 DSDT patch, which would address both the drain and the RTC-wake blocker at the cost of ongoing BIOS maintenance.

And the `fsck`.

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

# Suspend & Hibernate Investigation — Master Document
## ASUS ROG Zephyrus G15 (GA503QR) · CachyOS · KDE Plasma Wayland

**Consolidates:** Session 1 (2026-08-17, §1–§14) · Session 2 (2026-08-18, §15–§16) · Session 3 (2026-08-18 evening → 2026-08-19 morning, §17–§28) · **Session 4 (2026-08-19 daytime → night, §29–§36)**
**Compiled:** 2026-08-19, 22:30
**Supersedes:** all prior compilations

---

> ## 📎 Assembly note
>
> **Parts I, II and III (§1–§28) are unchanged from the previous compilation** and are not reproduced here — splice them in verbatim between the marked points below. Everything else in this document is new or rewritten.
>
> Session 4 changed a great deal. Where it overturns an earlier claim, the correction is recorded in **Part V** rather than by editing the historical text.

---

## Status dashboard

| Goal | Status | Evidence |
|---|---|---|
| **Hibernate works reliably** | ✅ **SOLVED** | **8/8** including lid-closed, §21 · §31.3 · §34.4 |
| Hibernate survives dGPU-active state | ✅ SOLVED | §21.3 |
| Hibernate survives long durations | ✅ SOLVED | 84 min, §33.4 |
| Hibernate with lid physically closed | ✅ **SOLVED** | §34.4 |
| **s2idle drain reduced** | ✅ **SOLVED** — 0.81 W → **0.58 W** | §31 |
| **`fsck` after 5 forced power-offs** | ✅ **CLEAN** | §30 |
| **RTC / clock-based timer wake** | ⛔ **IMPOSSIBLE — SMU firmware** | §29 |
| **`_BTP` battery trip-point wake** | ✅ **WORKS** — 3 self-wakes | §32 |
| **Charge-based pseudo-timer, lid open** | ✅ **WORKS** | §33.4 |
| **Charge-based pseudo-timer, lid closed** | ⛔ **FAILS** — hibernate rolls back | §33.5 |
| Lid automation | ⬜ Not applied; design pending §33 outcome | Part VI |
| KDE splash restoration | ⬜ Outstanding since Session 1 | §12.2 |

**One-line summary:** everything works except unattended auto-hibernate with the lid shut. That single case remains unsolved after three attempts.

---
---

# PART 0 — SYSTEM REFERENCE

## §0.1 Hardware & software facts

| Item | Value |
|---|---|
| Model | ASUS ROG Zephyrus G15 **GA503QR-211.ZG15** (2021) |
| BIOS | **GA503QR.416, 08/11/2023** — ASUS's final release for this model |
| **SMU firmware** | **64.44.0** — 🚩 below the 64.53 required for s0i3 timer wake (§29) |
| CPU | AMD Ryzen 9 5900HS (Fam17h+, **Cezanne** / CZN) |
| dGPU | NVIDIA RTX 3070 Mobile / Max-Q (GA104M) @ `0000:01:00.0` |
| dGPU audio fn | NVIDIA GA104 HDA @ `0000:01:00.1` |
| iGPU | AMD Renoir `0x1002:0x1638` @ `0000:06:00.0` |
| **Wi-Fi** | **MediaTek MT7921 (Filogic 330) @ `0000:04:00.0`**, `wlan0`, 192.168.1.220/24 |
| Ethernet | Realtek RTL8125B @ `0000:03:00.0` |
| Optimus type | **MUX-less** — internal eDP panel wired to iGPU only |
| RAM | 16 GB (15 Gi usable) |
| Storage | 1 TB NVMe, single drive (`nvme0n1`) |
| OS | CachyOS (Arch-based, rolling) |
| Kernel | `7.1.8-1-cachyos` (LTS `6.18.42-1-cachyos-lts` also installed) |
| systemd | `261.2-1-arch` |
| NVIDIA driver | **Open** kernel module `610.57.04` |
| DE / session | KDE Plasma on **Wayland**; **PowerDevil owns lid policy**, not logind (§34.2) |
| Display manager | Plasma Login Manager (PLM) |
| Bootloader | **Limine** |
| Root FS | **ext4**, `UUID=2b44456f-dc52-489f-8d2a-c2453136cee1` — **fsck clean 2026-08-19** |
| Firewall | `ufw` active |

## §0.2 Battery facts

| Item | Value |
|---|---|
| `energy_now` @ 100% | **~64.8 Wh** (64,724,000–64,851,000 µWh observed) |
| Design capacity | 90 Wh → **~28% wear** |
| 1% of capacity | ~648 mWh |
| `BAT0/alarm` default | `8992000` (~13.9%) |
| `BAT0/alarm` writable? | ✅ Yes — accepts arbitrary µWh values |
| **Gauge linearity** | ⚠️ **Non-linear above ~95%.** Energy can *rise* during suspend as the cell relaxes (§33.6) |
| CMOS/RTC cell | No discrete coin cell. `batt_status: dead` is a false reading — disregard |

## §0.3 Sleep capability matrix

| State | Available | Notes |
|---|---|---|
| s2idle (S0i3) | ✅ only option | **99.4–99.7% residency**, **0.55–0.62 W** post-fix |
| S3 (deep) | ❌ not exposed | DSDT patch possible — but see §36.4, now a weak proposition |
| S4 (hibernate) | ✅ **working** | 8/8 |
| S5 (poweroff) | ✅ | ~0.2 W chassis floor (§33.4) |
| **Timer wake from s0i3** | ⛔ **IMPOSSIBLE** | SMU 64.44 < 64.53 (§29) |
| **`_BTP` battery wake from s0i3** | ✅ **WORKS** | ±1 min accurate (§32) |
| Power-button wake | ✅ | ⚠️ Button is under the lid — unreachable when closed |

---
---

> ## ✂️ SPLICE POINT 1
> **Insert PART I (§1–§14), PART II (§15–§16) and PART III (§17–§28) here, verbatim from the previous compilation.** Nothing in them has been edited. Their corrections live in Part V.

---
---

# PART IIIb — SESSION 4 (2026-08-19, 11:53 → 22:00)
## RTC Closed · Drain Solved · `_BTP` Proven · Lid-Closed Automation Defeated

**Ten hours. Four significant wins, one unsolved case, two hard power-offs, one self-inflicted desktop breakage.**

---

## §29 The RTC Wake Post-Mortem — closed with a named cause

Session 3 concluded "firmware, probably." Session 4 named the exact component and version.

### §29.1 Hypothesis: the untested `use_acpi_alarm=0` path

Session 3 tested RTC wake twice — once before adding `rtc_cmos.use_acpi_alarm=1` and once after — and concluded "both delivery paths fail identically."

**That was wrong.** The kernel has shipped commit `3d762e21d5637` ("rtc: cmos: Use ACPI alarm for non-Intel x86 systems too") for years. On AMD, the ACPI alarm path is *already the default*. §25.2 and §25.3 therefore tested **the same path twice**.

### §29.2 The test — and why it couldn't work either

```bash
# /etc/default/limine: rtc_cmos.use_acpi_alarm=1 → =0
sudo limine-update && sudo reboot
```

Result:
```
$ grep -o 'use_acpi_alarm=0' /proc/cmdline      → use_acpi_alarm=0   ← kernel received it
$ cat /sys/module/rtc_cmos/parameters/use_acpi_alarm → Y             ← kernel ignored it
```

There is a quirk in `rtc-cmos` that forces `use_acpi_alarm = true` for AMD and Hygon CPUs with a **BIOS year ≥ 2021**, applied at driver probe, *after* module parameters are parsed. This machine's BIOS is dated **08/11/2023**.

Confirmed by the control test — with `use_acpi_alarm` supposedly `N`, IRQ 8 stayed at zero while the alarm nonetheless fired via the ACPI SCI (`ff_rt_clk` counter incremented).

**The parameter is inert on this hardware in either setting.**

### §29.3 The real blocker — SMU firmware version

RTC wake from s0i3 is not natively supported by the hardware. AMD's fix was a **combined driver and firmware change**: the driver hands the wake time to the SMU (System Management Unit), which stays powered during s0i3 and issues the wake. On Cezanne this lives in `amd_pmc_verify_czn_rtc()`.

That function begins with a hard version gate:

```c
if (pdev->major < 64 || (pdev->major == 64 && pdev->minor < 53))
    return 0;
```

With dynamic debug enabled on `amd_pmc`, the machine reports:

```
amd_pmc AMDI0005:00: SMU program 0 version is 64.44.0
```

**64.44 < 64.53.** The function returns immediately — before it opens `rtc0`, before it reads the alarm, before it packs anything into the s0i3 hint argument. Every RTC test ever run on this machine armed an alarm that the only mechanism capable of acting on it had already declined to look at.

### §29.4 ⛔ Final conclusion

> Timer wake from s0i3 requires the `amd_pmc` Cezanne SMU-timer path, gated on SMU firmware **≥ 64.53**. This machine ships **64.44.0** on **BIOS 416 (2023-08-11)**, ASUS's final release for the GA503QR. The driver declines to attempt the handoff. This is a firmware version deficiency. **Nothing in Linux can work around it.**

Remaining theoretical remedies: a BIOS release from ASUS (none in 3 years), or the S3 DSDT patch (§36.4), since S3 uses a legacy wake path unaffected by the gate.

### §29.5 Bonus — the platform is in excellent health

The same debugfs interface produced the best sleep-quality data of the entire investigation:

```
=== SMU Statistics ===
Table Version: 3
Hint Count: 1                        ← ONE continuous s0i3 block, no mid-sleep wakes
Last S0i3 Status: Success
Time (in us) to S0i3:      35,642    ← 36 ms entry latency
Time (in us) in S0i3:  614,497,283   ← 614.5 s
Time to resume from S0i3: 421,094    ← 421 ms

=== Active time (in us), during 614 s of sleep ===
DISPLAY 0 · CPU 3,190 · GFX 0 · VDD 1,633 · ACP 0 · VCN 0 · DF 2,619 · USB3_0 0 · USB3_1 0
```

**S0i3 residency: 99.4%.** Every functional block at or near zero active time.

**Implication:** almost none of the residual drain is the SoC. It is DRAM self-refresh, the EC, dGPU rails, NVMe, radios and VRM quiescent losses. Given §VIII.4's report of a G15 losing 10–15%/24 h **fully shut down** (~0.27–0.40 W), the OS-attributable share of the drain budget is only about **0.2–0.35 W**.

---

## §30 `fsck` — CLEAN

Five forced power-offs across Sessions 1–3, never checked. Done via initramfs rather than live USB:

```bash
# /etc/default/limine: "quiet nowatchdog splash" → "nowatchdog"
#                      + fsck.mode=force fsck.repair=preen
sudo limine-update && sudo reboot
```

Removing `quiet splash` was deliberate — a forced check on a 1 TB drive shows progress text instead of a frozen ROG logo, so there's no temptation to reach for the power button.

**Result: `clean`.** No damage from five unclean shutdowns. Boot config reverted afterwards.

This validates the ext4 journal doing its job, and directly informs C34 below.

---

## §31 `NVreg_EnableS0ixPowerManagement=1` — the first change that moved the number

### §31.1 Applied

```bash
sudo tee -a /etc/modprobe.d/nvidia-power-management.conf <<'EOF'
options nvidia NVreg_EnableS0ixPowerManagement=1
EOF
sudo mkinitcpio -P && sudo reboot
```

### §31.2 Verified from `/proc`, not the file

```
PreserveVideoMemoryAllocations: 0     ✅
UseKernelSuspendNotifiers:      1     ✅
EnableS0ixPowerManagement:      1     ✅
S0ixPowerManagementVideoMemoryThreshold: 256

$ cat /proc/driver/nvidia/gpus/0000:01:00.0/power
Video Memory:               Off          ← was "Active"
S0ix Power Management:
 Platform Support:          Supported
 Status:                    Enabled      ← was "Disabled"
```

`Platform Support: Supported` is a **static capability**, unchanged since §15.3.1. `Status` is the live state, and that is what flipped.

**What it does in plain terms:** VRAM is volatile, so keeping its contents alive costs continuous power. By default the driver holds those rails up through s2idle. This parameter tells it to evacuate VRAM to system RAM (when usage is under the 256 MB threshold — yours is 44 MB) and cut power to the memory entirely.

### §31.3 Hibernate re-verified immediately ✅

Non-negotiable, because this parameter is in the same machinery family as the one that broke hibernation.

| Check | Value |
|---|---|
| Kernel gap | 14:13:29.70 → 14:14:29.18 = **60 s** |
| `boot_id` | `beb2aeda-…` **unchanged** |
| `$MARKER` / `jobs` | `s0ix-141326`, `sleep 9999` alive |
| **SSH session** | **survived the entire power-off and cold boot** |
| Errors | none |

The surviving SSH session is the strongest single proof produced in this investigation. A TCP connection cannot outlive a genuine reboot.

### §31.4 The drain result

| Measurement | Method | Result |
|---|---|---|
| §15.2 baseline, pre-everything | percentage, loose | 0.81 W |
| §26 overnight, post-KWin fix | awake-time subtraction, ±0.15 W | 0.65–0.82 W |
| **35 min, S0ix enabled** | **hook-captured, contamination-free** | **619 mW** |
| **36 min, S0ix enabled** | hook-captured | **551 mW** |
| **30 min, lid closed** | hook-captured | **576 mW** |
| 20 min, at 100% charge | hook-captured | *−141 mW (invalid, §33.6)* |

**Three clean measurements: 551 / 576 / 619 mW. Call it 0.58 W.**

| Duration | Before (0.81 W) | After (0.58 W) |
|---|---|---|
| 8 h | ~10% | **~7.2%** |
| 24 h | ~30% | **~21.5%** |
| To empty | 3.4 days | **4.6 days** |

**~28% reduction.** The first genuine drain improvement across four sessions.

### §31.5 Why this worked when the Session 2 fix didn't

Two independent switches:

1. **Is the GPU in a low PCI power state?** — Session 2's KWin fix flipped this. `runtime_status: suspended`, D3cold reached.
2. **Are the VRAM rails powered?** — governed separately by driver policy. Still `Active`.

You had a sleeping GPU with wide-awake memory. Both switches had to be off; only one was.

---

## §32 `_BTP` Battery Trip Point — PROVEN

The one automatic wake source that survives on this machine.

### §32.1 Mechanism

`/sys/class/power_supply/BAT0/alarm` is writable in µWh. Write a value below current charge; when the EC observes the crossing during s0i3, it raises a wake. Entirely separate hardware path from the RTC — different GPE, EC-driven.

### §32.2 Three confirmed self-wakes

| # | Threshold | Predicted | Actual | Witnessed |
|---|---|---|---|---|
| 1 | 300 mWh | 22.5 min @ 0.81 W | **30 m 11 s** | User at another machine, saw it turn on |
| 2 | 150 mWh | 11.3 min | **12 m 19 s** | **Off-machine TCP watcher log**, user absent |
| 3 | 275 mWh (hook) | 30 min @ 0.55 W | **36 m 15 s** | Journal + hook log |
| 4 | 275 mWh (hook, lid closed) | 30 min | **29 m 39 s** | Journal |

Halving the threshold roughly halved the time — it tracks charge, not a fixed period. Test 2's wake was recorded by a machine the user never touched.

### §32.3 EC notification lag — small and bounded

Once measured against the *correct* drain constant, the lag collapses:

| Test | Predicted @ true rate | Actual | Overshoot |
|---|---|---|---|
| Hook, 30 min | 35 m 24 s | 36 m 15 s | **51 s** |
| Hook, lid closed | 30 m 00 s | 29 m 39 s | **−21 s** |

Sub-minute. The earlier "5–10 minute lag" was an artifact of predicting against 0.81 W when the machine was drawing 0.55 W.

### §32.4 What this unlocks

systemd's own STH uses `_BTP`, but targets **~5% capacity** — at 0.58 W that is **4.6 days away**. Useless as a timer.

But since drain is stable and measured, **charge is a proxy for time**:

```
target_µWh = energy_now − (drain_mW × minutes × 1000 / 60)
```

Set the trip point yourself and you get an arbitrary-duration timer on the one alarm clock the hardware still has.

---

## §33 The Charge-Based Pseudo-Timer

### §33.1 Design

`/usr/lib/systemd/system-sleep/50-battery-hibernate.sh`

- **`pre/suspend`** — if on AC, restore default and exit. Otherwise compute target, clamp to a 15% floor, write to `BAT0/alarm`, save to `/run`.
- **`post/suspend`** — compare `energy_now` against the saved target. At or below → `_BTP` woke us → schedule hibernate. Above → a human woke us → restore default, carry on.
- **`post/hibernate`** — clear all state.

The discriminator is **energy, not lid state** — proven necessary, since `_BTP` fires with the lid open too.

### §33.2 Design corrections learned the hard way

| Item | Wrong | Right |
|---|---|---|
| `DRAIN_MW` rounding | Round **up** ("hibernate early") | **Round down.** Overestimating drain inflates the budget → sleeps *longer*. Asked 30 min, got 36 |
| Wake discriminator | Lid state | Energy vs. target |
| Full-charge behaviour | *(unconsidered)* | Must refuse to arm above ~95% (§33.6) |

### §33.3 Instrumentation — `99-sleep-audit.sh`

Part VII's design, minimal form, installed and working. Captures energy at the exact moment of suspend and resume, eliminating the awake-time contamination that gave every prior measurement ±0.15 W.

```
date	action	dur_s	dmWh	avg_mW	hw_resid%	ac0	ac1
2026-08-19T14:55:30	suspend    2118  364  619  99  0  0
2026-08-19T15:40:42	suspend    2175  333  551  99  0  0
2026-08-19T17:05:29	hibernate  5063  650  462   0  0  0
2026-08-19T17:44:45	suspend    1779  285  576  99  0  0
2026-08-19T21:49:33	suspend    1218  -48 -141  99  0  0    ← gauge plateau, invalid
```

⚠️ **Units bug found and fixed:** the first row's `avg_mW` is actually µW (618696 µW = 619 mW). `DE*3600/DT` yields µW; corrected to `DE*36/(DT*10)`.

The `ac0`/`ac1` columns exist specifically to catch the §21.4 error, and the negative row immediately flagged the gauge anomaly rather than letting it hide.

### §33.4 ✅ Lid OPEN — full pass

```
15:04:27  armed: now=36224000 target=35899000 budget=325000uWh (30min)
15:40:42  BUDGET SPENT (35891000 <= 35899000) - hibernating in 20s
15:41:06  hibernation entry
17:05:29  hibernation exit        ← 84 minutes powered off
boot_id   unchanged ✅
```

**Complete suspend-then-hibernate on a working timer**, on a machine where the firmware forbids the standard mechanism. Firefox restored without reloading.

**Also the first S4 drain figure:** 650 mWh over 84 min. Subtracting ~300–375 mWh for the image write and full resume leaves **~200–250 mW true powered-off draw** — consistent with §VIII.4's shut-down reports. Retires the §21.4 assumption that S4 is "near-zero by construction."

### §33.5 ⛔ Lid CLOSED — fails, twice, destructively

**Attempt 1 (17:44), Attempt 2 (18:57)** — identical signature:

```
17:15:06  suspend entry
17:44:45  suspend exit                     ← _BTP wake, 29 m 39 s. CORRECT.
17:44:45  BUDGET SPENT - hibernating in 5s
17:44:55  hibernation entry
          ... image written, EC stopped, CPUs 15→5 offlining ...
17:45:30  PM: hibernation: Wakeup event detected during hibernation, rolling back.
17:45:30  WARNING: nv_restore_user_channels+0xeb [nvidia]
17:45:30  NVRM: PM suspend notifier failed: 0x11
17:45:41  PM: suspend entry → 17:45:41 PM: suspend exit    ← 127 ms
          ... ×80, for 30 minutes, ~45 W, 19.7 Wh burned ...
```

**Two distinct problems, one causal chain:**

| | Problem | Detail |
|---|---|---|
| **A** | Hibernate rolls back | Reaches the *final* pre-poweroff checkpoint — image written, `ACPI: EC stopped`, CPUs offlining — then `pm_wakeup_pending()` returns true and the kernel correctly unwinds |
| **B** | Machine wedges | The rollback leaves the NVIDIA driver unable to restore GPU channels → its PM notifier fails → **every subsequent suspend aborts in ~127 ms** → PowerDevil retries every 17 s forever |

Problem B is downstream of A. It requires a **hard power-off** to clear; `sudo reboot` hangs because systemd can't stop the wedged driver. 8,305 `NVRM` error lines in one boot.

### §33.6 Attempt 3 — inconclusive, and a new gotcha

With PowerDevil idle-suspend disabled, retested from **100% charge**:

```
21:29:15  armed: now=64803000 target=64665500 (15min @ 550mW)
21:49:33  manual wake (64851000 > 64665500) - staying up
```

**The battery gained 48 mWh during 20 minutes of suspend.** Audit hook agrees: `dmWh -48, avg_mW -141`.

Lithium gauges are non-linear at the top of the range; just off the charger, `energy_now` drifts *upward* as the cell relaxes and the EC re-estimates. At 0.55 W, 20 minutes costs 186 mWh — 0.29% of the pack, below the noise floor of a settling gauge.

The hook behaved **correctly**: armed, saw energy above target, declined to hibernate.

> ⚠️ **The pseudo-timer cannot be tested or used from a full battery.** The script needs a guard refusing to arm above ~95%.

### §33.7 Hypotheses tested against the lid-closed failure

| Hypothesis | Verdict |
|---|---|
| Wifi (`04:00.0`) is the wake source | ❌ `power/wakeup` already `disabled`. It *was* the recorded suspend-failure device (`errno -1`) |
| `rfkill block` wifi+bluetooth before hibernating | ❌ No effect |
| `HIBERNATE_DELAY` 20 s → 5 s caused it | ❌ Failed at both values |
| Latched `_BTP` EC notification | ⏸️ Plausible, untested — but 15:40 had a `_BTP` wake and hibernated fine |
| Restoring `DEFAULT_ALARM` left a live trip point | ❌ 8,992,000 vs 31,704,000 µWh — 23 Wh of headroom |
| Lid position alone | ❌ **Refuted** — §34.4 hibernated cleanly with the lid shut |
| PowerDevil re-suspending into the hibernate window | ⏸️ **Leading hypothesis, still untested** |
| `systemd-inhibit` guard via `systemd-run` | ❌ Too slow to spawn |
| Loop breaker (3 rapid cycles → disarm) | ⚠️ Fired correctly from cycle 1, but can only stop *arming*. PowerDevil's retries are outside its reach |

---

## §34 Failures, Misdiagnoses and Self-Inflicted Damage

Recorded because the mistakes were as instructive as the fixes.

### §34.1 The `systemd-logind` restart incident 🚩

`sudo systemctl restart systemd-logind` on a **live desktop session** broke session tracking. Symptom: screen locks → unlock succeeds → everything freezes → repeat across reboots.

```
$ loginctl list-sessions
2   ekj          seat0  user           tty2      ← real session
c3  plasmalogin  seat0  greeter        tty1      ← orphaned greeter
```

Two competing seat0 sessions; the lock screen couldn't reconcile them. Removing the drop-in and rebooting fixed it.

> **Rule: never restart `systemd-logind` on a running desktop.** Use `daemon-reload` plus a reboot.

The `i2c` group membership was briefly suspected and **exonerated** — the problem resolved without removing it.

### §34.2 The logind drop-in was redundant anyway

```
systemd-logind: suspend requested from client PID 1293 ('org_kde_powerde')
```

**PowerDevil owns lid policy on KDE.** It takes a `handle-lid-switch` inhibitor from logind and implements the behaviour itself. So the config that broke the session would have done nothing even if it had worked. Part VI's `logind.conf.d/10-lid.conf` is unnecessary — the GUI setting is the real control.

### §34.3 Test design error — the power button is under the lid

An instruction to "press the power button with the lid still closed" was physically impossible on a G15. With the lid shut, the only wake sources are `_BTP` and (unreachable) the power button. `/proc/acpi/wakeup` lists only `GPP0` and `GP17`, both S4-only. No USB, no keyboard, no lid wake.

### §34.4 ✅ T14 — lid-closed hibernate, done properly

An earlier attempt was scored as a lid-closed test but the lid had been reopened first. Re-run cleanly, hibernate triggered over SSH with the lid physically shut:

```
21:21:13.32  hibernation entry
21:22:49.17  hibernation exit      (96 s)
boot_id      c205ed79-…  unchanged ✅
no "rolling back"
```

Wakeup-source diff showed only `PNP0C0C:00` (ACPI power button) incrementing — from the user pressing it to resume. Nothing else registered.

**Firefox reopened without reloading.** Hibernate #8.

### §34.5 The corrected failure matrix

| Run | Lid at hibernate | `_BTP` fired first? | Result |
|---|---|---|---|
| 15:40 | Open | **Yes** | ✅ |
| 17:44 | **Closed** | **Yes** | ⛔ rollback |
| 18:57 | **Closed** | **Yes** | ⛔ rollback |
| 21:21 (T14) | **Closed** | No | ✅ |
| 7 others | Open | No | ✅ |

**Neither factor alone breaks it. Only the combination fails.** The only known behavioural difference: lid-open, PowerDevil idles ~10 minutes after a `_BTP` wake; lid-closed, it requests suspend within ~45 s — and the image write takes ~35 s.

---

## §35 Session 4 Test Ledger

| # | Test | Result |
|---|---|---|
| S4-1 | `rtc_cmos.use_acpi_alarm=0` | ⛔ Parameter inert — kernel quirk forces `Y` |
| S4-2 | SMU firmware version | ⛔ **64.44.0 < 64.53.** RTC wake closed permanently |
| S4-3 | `fsck -f` root | ✅ **Clean** after 5 forced power-offs |
| S4-4 | `NVreg_EnableS0ixPowerManagement=1` | ✅ `Status: Enabled`, `Video Memory: Off` |
| S4-5 | Hibernate w/ S0ix enabled | ✅ PASS — SSH session survived |
| S4-6 | Drain, S0ix enabled ×3 | ✅ **551 / 576 / 619 mW** |
| S4-7 | `_BTP` wake, 300 mWh | ✅ 30 m 11 s, witnessed |
| S4-8 | `_BTP` wake, 150 mWh | ✅ 12 m 19 s, off-machine log |
| S4-9 | Pseudo-timer, lid open | ✅ **PASS** — full chain, 84 min hibernation |
| S4-10 | Pseudo-timer, lid closed | ⛔ FAIL — rollback + wedge + hard power-off |
| S4-11 | Pseudo-timer, lid closed, hardened | ⛔ FAIL — identical + hard power-off |
| S4-12 | Lid-closed hibernate, no `_BTP` (T14) | ✅ **PASS** — 96 s |
| S4-13 | Pseudo-timer, PowerDevil disabled | ⚪ Inconclusive — gauge plateau at 100% |

**Hibernates: 8/8 lifetime.** **Hard power-offs this session: 2.**

---
---

# PART IV — CONSOLIDATED CURRENT SYSTEM STATE
*(as of 2026-08-19, 22:30)*

## §IV.1 Active files

| Path | Content | Status |
|---|---|---|
| `/swapfile` | 20 GiB, prio −1, offset 220407808 | ✅ |
| `/etc/fstab` | swapfile entry | ✅ |
| `/etc/mkinitcpio.conf` | `MODULES=(amdgpu)`; HOOKS incl. `plymouth`, `kms`, `resume`, `fsck` | ✅ |
| `/etc/mkinitcpio.conf.d/10-chwd.conf` | nvidia modules — **distro-generated, do not edit** | ⚠️ harmless |
| **`/etc/modprobe.d/nvidia-power-management.conf`** | `PreserveVideoMemoryAllocations=0`<br>`TemporaryFilePath=/var/tmp`<br>**`EnableS0ixPowerManagement=1`** | ✅ **hibernate fix + drain fix** |
| `…/systemd-hibernate.service.d/60-freeze-sessions.conf` | `FREEZE_USER_SESSIONS=true` | ✅ active |
| `…/systemd-suspend-then-hibernate.service.d/60-freeze-sessions.conf` | same | ✅ active |
| `/etc/systemd/sleep.conf.d/10-hibernate-delay.conf` | **comment-only stub** — documents why no `HibernateDelaySec` | ✅ |
| `/etc/sysctl.d/99-sysrq.conf` | `kernel.sysrq=1` | ✅ REISUB persistent |
| **`/usr/lib/systemd/system-sleep/99-sleep-audit.sh`** | drain instrumentation → `/var/log/sleep-audit.tsv` | ✅ |
| **`/usr/lib/systemd/system-sleep/50-battery-hibernate.sh`** | pseudo-timer | ⚠️ **`chmod -x` — disabled** |
| `~/.config/plasma-workspace/env/kwin-gpu.sh` | `KWIN_DRM_DEVICES` → iGPU | ⚖️ works, no power benefit |
| `/etc/default/limine` + `/boot/limine.conf` | cmdline, **no `rtc_cmos.` parameter** | ✅ |
| ufw rule | port 22 from `192.168.1.0/24` | ✅ |
| group `i2c` | `ekj` is a member | ✅ |

## §IV.2 Kernel command line

```
quiet nowatchdog splash rw root=UUID=2b44456f-dc52-489f-8d2a-c2453136cee1
amdgpu.dcdebugmask=0x40000
```

No `resume=` (EFI `HibernateLocation` handles it) · no `rtc_cmos.use_acpi_alarm` (proven inert, §29.2) · no `fsck.mode` (one-shot, reverted).

## §IV.3 Services

| Unit | State |
|---|---|
| `nvidia-{suspend,hibernate,resume}.service` | enabled (manual, S1 — likely redundant on 610) |
| `nvidia-suspend-then-hibernate.service` | disabled (deliberate) |
| `nvidia-powerd.service` | enabled (Dynamic Boost, unrelated) |
| `sshd.service` | enabled |

## §IV.4 Driver parameters (verified from `/proc`)

```
PreserveVideoMemoryAllocations: 0     ✅
UseKernelSuspendNotifiers:      1     ✅
EnableS0ixPowerManagement:      1     ✅
TemporaryFilePath:              "/var/tmp"
```

## §IV.5 Runtime state

| Item | Value |
|---|---|
| dGPU at idle | `suspended` ✅ |
| dGPU `Video Memory` | **`Off`** ✅ |
| NVIDIA S0ix Status | **`Enabled`** ✅ |
| S0i3 residency | **99.4–99.7%** |
| **s2idle draw** | **~0.58 W** |
| S4 draw (preliminary) | ~0.2 W |
| Timer wake from s0i3 | ⛔ impossible (SMU 64.44) |
| `_BTP` wake | ✅ works, ±1 min |
| `BAT0/alarm` | `8992000` (default) |
| PowerDevil battery profile | ⚠️ **lid "Do nothing", suspend "Never"** — test leftover, see §VI |

---
---

# PART V — CORRECTIONS REGISTER

## Sessions 1–3 (unchanged)

| # | Claim | Correction |
|---|---|---|
| C1 | Freeze override was the hibernate root cause and fix | Neither. Drop-in was inert (`10-f…` < `10-n…`). Real fix is `PreserveVideoMemoryAllocations=0` |
| C2 | §9.3 "hibernate SUCCESS" | Unverified, probably a cold boot |
| C3 | amdgpu `Mode Validation Warning` caused black screen | Benign noise |
| C4 | Battery 90 Wh; drain 1.1/1.7 W | **64.8 Wh**; 0.81/1.22 W |
| C5 | KWin holding dGPU was the drain root cause | Real bug, fixed, **zero power effect** |
| C6 | Post-fix prediction 5–7%/8 h | Actual ~10%; no improvement |
| C7 | Use `rtcwake` sampling for A/B | Impossible here |
| C8 | `HibernateDelaySec=45min` design | Delay inoperative |
| C9 | Plain suspend + `HibernateOnACPower=no` gives a backstop | Wrong — trigger exists only in STH |
| C10 | `PreserveVideoMemoryAllocations=1` is correct | Cause of the `-5` failure |
| C11 | Removing a modprobe option = disabling it | Unset → `2` (auto) → re-enables. Set `0` |
| C12 | `/etc` drop-ins override `/usr/lib` | Only for identical filenames |
| C13 | `batt_status: dead` = failing CMOS cell | False reading; no discrete cell |

## Session 4 — new

| # | Claim | Correction | §|
|---|---|---|---|
| **C14** | §25.7: "AMD PMC doesn't list RTC as a wake source" | **Named precisely:** `amd_pmc_verify_czn_rtc()` gates on SMU ≥ 64.53; machine has **64.44.0** | §29.3 |
| **C15** | §25.2 and §25.3 tested two different alarm paths | **Same path twice.** Kernel commit `3d762e21d5637` makes ACPI alarm the AMD default | §29.1 |
| **C16** | `rtc_cmos.use_acpi_alarm` is a meaningful setting here | **Inert in either value** — quirk forces `Y` on 2021+ AMD BIOS | §29.2 |
| **C17** | Audit hook `avg_mW` column | Emitted **µW**. First TSV row must be ÷1000. Fixed | §33.3 |
| **C18** | `_BTP` EC lag is 5–10 min | **~51 s.** Earlier figures were artifacts of a wrong drain constant | §32.3 |
| **C19** | §21.4: "S4 draw is near-zero by construction" | **~0.2 W.** Real chassis floor | §33.4 |
| **C20** | Round `DRAIN_MW` **up** to hibernate early | **Backwards.** Overestimating drain sleeps *longer*. 650 → 550 | §33.2 |
| **C21** | `systemd-run --on-active` unsound (frozen monotonic clock) | Not the failure mode — hibernate was entered promptly both times | §33.5 |
| **C22** | `HIBERNATE_DELAY` 20 s → 5 s caused the rollback | ❌ Failed at both values | §33.7 |
| **C23** | `DEFAULT_ALARM=8992000` unsafe below ~14% | Overstated — edge-triggered, and T14 ran with it armed harmlessly | §33.7 |
| **C24** | Suspend aborting in ~127 ms | New failure mode: NVIDIA PM notifier fails after a rollback | §33.5 |
| **C25** | Rollback root cause | `Wakeup event detected during hibernation` at the final pre-poweroff check, after `EC stopped` | §33.5 |
| **C26** | `last_failed_dev 0000:04:00.0` = wake source | **MT7921 wifi**, but `power/wakeup` is **`disabled`** — it *failed to suspend*, it did not wake | §33.7 |
| **C27** | Restoring `DEFAULT_ALARM` left a live trip point | ❌ 23 Wh of headroom. Not the trigger | §33.7 |
| **C28** | Aborted hibernate is recoverable | ❌ Wedges the NVIDIA driver (`nv_restore_user_channels` WARN). Needs a **hard power-off** | §33.5 |
| **C29** | Loop breaker never ran | ❌ **Fired from cycle 1** — `post/hibernate` runs even on rollback, clearing `PENDING` | §33.7 |
| **C30** | `rfkill` before hibernate mitigates it | ❌ No effect | §33.7 |
| **C31** | Lid position is the discriminator | ❌ **Refuted** — T14 hibernated cleanly with the lid shut | §34.4 |
| **C32** | PowerDevil's suspend comes after the rollback, so it's a symptom | ♻️ **Partially reinstated.** The broadcast appears after, but logind can only *act* once unwinding completes. The request itself may land mid-write | §34.5 |
| **C33** | `systemctl restart systemd-logind` is safe | 🚩 **Breaks session tracking** on a live desktop. Lock/unlock/freeze loop | §34.1 |
| **C34** | Forced power-off carries meaningful risk | ⚖️ **Recalibrated: low** on journaled ext4. `fsck` after 5 came back clean. Real hazard is only during an image write | §30 |
| **C35** | PowerDevil vs. logind lid ownership | **PowerDevil owns it.** logind drop-in is redundant | §34.2 |
| **C36** | Power button usable for lid-closed wake tests | ❌ It is **under the lid** | §34.3 |
| **C37** | An earlier "lid-closed hibernate PASS" | ❌ **Invalid** — lid had been reopened. Re-run as T14 | §34.4 |
| **C38** | Battery gauge is linear | ❌ **Non-linear above ~95%** — energy can *rise* during suspend | §33.6 |
| **C39** | §15.3.6: "there is no `card0`" | **Stale.** DRM numbering shifted; `card0` is now the NVIDIA dGPU. Validates the `readlink -f` by-path design | §33 |

---
---

# PART VI — CONFIGURATION DECISION

**Part VI's original design is void.** It was built on systemd's STH plus `HibernateDelaySec`, both of which are now known non-viable. Two options remain.

## §VI.1 Option A — Manual hibernate (SAFE, recommended for now)

Everything proven, nothing experimental.

```bash
# Disable the pseudo-timer
sudo chmod -x /usr/lib/systemd/system-sleep/50-battery-hibernate.sh
```

**KDE → Power Management → On Battery:**
- Lid close → **Sleep**
- Suspend session → **After 10 minutes** *(restore from the test's "Never")*

| Scenario | Behaviour |
|---|---|
| Lid closed | s2idle @ 0.58 W → **4.6 days** runway |
| Before travel / overnight | `systemctl hibernate` — 8/8 reliable |
| Flat battery | State lost *(no backstop)* |

## §VI.2 Option B — Pseudo-timer, lid open only (works today)

Same as A, plus the hook enabled and `SLEEP_MINUTES=240`. Validated for lid-open suspends. **Will wedge the machine if it fires with the lid shut** — which makes it unsuitable as a daily default until §VI.3 is resolved.

## §VI.3 Option C — Finish the job

One hypothesis remains untested: **PowerDevil re-suspending into the hibernate window.**

Required protocol, at **60–80% charge** (never 100%, per C38):

1. KDE → On Battery → lid **Do nothing**, suspend **Never**, DPMS **Never**
2. Add a full-charge guard to the hook
3. `chmod +x`, `SLEEP_MINUTES=15`
4. Unplug, wait 3 min, `systemctl suspend`, close lid
5. Watch from a device that won't sleep (**not** WSL on a sleeping host)

**Expected cost if it fails: one more hard power-off.** Per C34, that is cheap.

**If it passes**, the final design is: lid does nothing, KDE idle-suspends after 2 min, the hook drives everything. If it fails, Option A stands and the honest conclusion is that unattended lid-closed auto-hibernate is not achievable on this hardware.

## §VI.4 Deferred

- **Hybrid-sleep** — writes the image *before* suspending, so there is no intermediate wake and the rollback failure is structurally impossible. Costs ~6 GB written per lid close and forfeits all power saving. User declined; remains the fallback if C fails.
- **KDE "Start with an empty session"** — trivial, unrelated, still not applied.

---
---

# PART VII — INSTRUMENTATION ✅ IMPLEMENTED

Part VII's design is live as `/usr/lib/systemd/system-sleep/99-sleep-audit.sh`.

```bash
#!/usr/bin/env bash
set -u
STATE=/run/sleep-audit.state
LOG=/var/log/sleep-audit.tsv
BAT=/sys/class/power_supply/BAT0
rd(){ cat "$1" 2>/dev/null || echo 0; }
acnow(){ local v; v=$(cat /sys/class/power_supply/A*/online 2>/dev/null | head -1); echo "${v:-0}"; }

case "$1" in
  pre)
    printf '%s %s %s %s\n' "$(date +%s)" "$(rd $BAT/energy_now)" "$(acnow)" \
      "$(rd /sys/power/suspend_stats/total_hw_sleep)" > $STATE
    ;;
  post)
    [ -r $STATE ] || exit 0
    read -r T0 E0 AC0 HW0 < $STATE
    T=$(date +%s); E=$(rd $BAT/energy_now); AC1=$(acnow)
    HW=$(rd /sys/power/suspend_stats/total_hw_sleep)
    DT=$((T-T0)); [ $DT -lt 60 ] && { rm -f $STATE; exit 0; }
    DE=$((E0-E)); MW=$((DE*36/(DT*10)))          # ← C17 fix: µWh→mW
    DHW=$(((HW-HW0)/1000000)); RES=$((DHW*100/DT))
    [ -s $LOG ] || printf 'date\taction\tdur_s\tdmWh\tavg_mW\thw_resid%%\tac0\tac1\n' > $LOG
    printf '%s\t%s\t%d\t%d\t%d\t%d\t%s\t%s\n' \
      "$(date -Is)" "$2" "$DT" "$((DE/1000))" "$MW" "$RES" "$AC0" "$AC1" \
      | tee -a $LOG | systemd-cat -t sleep-audit -p notice
    rm -f $STATE
    ;;
esac
exit 0
```

**Protocol:** unplug, **wait 3 min** for the gauge to settle, ensure charge is **60–80%**, suspend for ≥30 min, wake manually, read the TSV. Compare in **watts, not percent**.

---
---

# PART VIII — RESEARCH LIBRARY

*(§VIII.1–VIII.8 carried over unchanged; §VIII.9 is new)*

**Updates to existing entries:**

- **§VIII.1** — `NVreg_EnableS0ixPowerManagement=1` is no longer "the top untested candidate." **Applied, verified, worth 0.23 W.**
- **§VIII.3 (S3 DSDT patch)** — ⬇️ **substantially weakened as a power fix.** With S0i3 residency at 99.4% and every SoC block gated, there is very little left for S3 to improve. Its remaining value is that S3 uses a legacy wake path unaffected by the SMU gate — a *functionality* argument, not a battery one. The maintenance tax (re-patching after every BIOS update, brick risk) now looks poor value.
- **§VIII.5** — calibration table updated:

| Source | Figure |
|---|---|
| Framework 11th gen, s2idle "normal" | ~0.8 W |
| `batenergy` author's logged cycle | −673 mW |
| **This machine, pre-fix** | **0.81 W** |
| **This machine, post-S0ix-fix** | **0.58 W** |
| **This machine, hibernated** | **~0.2 W** |

The machine now sits **below** typical healthy s2idle for its class.

## §VIII.9 AMD `amd_pmc` and the SMU timer path *(new)*

- RTC wake from s0i3 is **not natively supported by the hardware**. AMD's remedy is a combined driver + firmware change in which the driver passes the wake time to the SMU.
- The Cezanne implementation, `amd_pmc_verify_czn_rtc()`, requires **SMU firmware ≥ 64.53** and returns immediately below that.
- Even where it works, the driver deliberately blocks the deepest CPU idle state while using the timer path (a firmware-bug workaround), so a functioning timer wake would itself cost some power.
- Useful debugfs, post-suspend only: `/sys/kernel/debug/amd_pmc/smu_fw_info` and `/sys/kernel/debug/amd_pmc/s0ix_stats`. `Hint Count` reveals how many times s0i3 was entered.
- Enable the version print with `echo 'module amd_pmc +p' > /sys/kernel/debug/dynamic_debug/control`.

---
---

# PART IX — ROLLBACK PROCEDURES

## §IX.1 Pseudo-timer hook
```bash
sudo chmod -x /usr/lib/systemd/system-sleep/50-battery-hibernate.sh   # disable
sudo rm -f /usr/lib/systemd/system-sleep/50-battery-hibernate.sh      # remove
sudo rm -f /run/battery-hibernate.*
echo 8992000 | sudo tee /sys/class/power_supply/BAT0/alarm
rfkill unblock all
```

## §IX.2 Audit hook
```bash
sudo rm -f /usr/lib/systemd/system-sleep/99-sleep-audit.sh
# /var/log/sleep-audit.tsv is your data — keep it
```

## §IX.3 S0ix power management (⚠️ costs 0.23 W)
```bash
sudo sed -i '/EnableS0ixPowerManagement/d' /etc/modprobe.d/nvidia-power-management.conf
sudo mkinitcpio -P && sudo reboot
```

## §IX.4 Hibernate fix *(don't)*
```bash
sudo rm /etc/modprobe.d/nvidia-power-management.conf
sudo mkinitcpio -P && sudo reboot
```

## §IX.5 Freeze overrides
```bash
sudo rm -f /etc/systemd/system/systemd-{hibernate,suspend-then-hibernate}.service.d/60-freeze-sessions.conf
sudo systemctl daemon-reload
```

## §IX.6 KWin GPU restriction
```bash
rm ~/.config/plasma-workspace/env/kwin-gpu.sh   # log out and back in
```
Emergency: `Ctrl+Alt+F3` → `rm` → reboot. Restores USB-C/DP output.

## §IX.7 sysrq / SSH / i2c
```bash
sudo rm /etc/sysctl.d/99-sysrq.conf
sudo systemctl disable --now sshd
sudo ufw delete allow from 192.168.1.0/24 to any port 22 proto tcp
sudo gpasswd -d ekj i2c
```

## §IX.8 🚨 Emergency: wedged NVIDIA driver
Symptoms: hot chassis, fan running, SSH dead or hanging, `sudo reboot` never completes.

1. **Hold the power button** until the fan stops (~10 s)
2. Power on normally
3. If SSH won't connect: `rfkill unblock all` at the laptop
4. `sudo chmod -x /usr/lib/systemd/system-sleep/50-battery-hibernate.sh`

Per C34, this is low-risk. Only avoid it during an active hibernation image write.

---
---

# PART X — MASTER CHANGE LOG

## ✅ TESTED AND WORKING — keep

| # | Change | Location | Evidence |
|---|---|---|---|
| K1 | 20 GiB swapfile + fstab | `/swapfile` | 8/8 hibernates |
| K2 | **`PreserveVideoMemoryAllocations=0`** | modprobe.d | **The hibernate fix.** Never delete the file — unset → `2` → broken |
| K3 | **`EnableS0ixPowerManagement=1`** | modprobe.d | **The drain fix.** 0.81 → 0.58 W. `Video Memory: Off` |
| K4 | `TemporaryFilePath=/var/tmp` | modprobe.d | Default `/tmp` is tmpfs |
| K5 | `kernel.sysrq=1` | `/etc/sysctl.d/99-sysrq.conf` | REISUB persistent (default was `16`) |
| K6 | `99-sleep-audit.sh` | system-sleep | Contamination-free measurement; caught the gauge anomaly |
| K7 | `sshd` + ufw LAN rule | — | Load-bearing all session |
| K8 | `ekj` in group `i2c` | — | Kills ~400 log lines/suspend. Suspected then exonerated |
| K9 | `10-hibernate-delay.conf` stub | sleep.conf.d | Comment-only; documents the SMU gate |
| K10 | `amdgpu.dcdebugmask=0x40000` | cmdline | Inherited; brightness fix |

## 🔬 NEEDS VALIDATION

| # | Item | Status |
|---|---|---|
| V1 | **`50-battery-hibernate.sh`** | Lid open ✅ · lid closed ⛔ ×2 · PowerDevil-disabled ⚪. **Currently `chmod -x`.** Needs a ≥95% charge guard |
| V2 | KDE battery profile (lid "Do nothing", suspend "Never") | ⚠️ Test leftover. A closed lid now stays **fully awake** — do not put it in a bag |
| V3 | Freeze overrides ×2 | Active, but hibernate passed 3/3 without them; STH is no longer used |
| V4 | `kscreenlockerrc` autolock | May have been disabled during the freeze incident. Check |

## ⚖️ COULD GO EITHER WAY

| # | Item | For | Against |
|---|---|---|---|
| M1 | `kwin-gpu.sh` | Fixes a real runtime-PM defect | Zero power benefit; costs USB-C/DP |
| M2 | 3 nvidia sleep services | Working | Upstream preset `disabled`; likely redundant on 610 |
| M3 | `resume` hook in HOOKS | Harmless | Redundant — EFI variable does the work |
| M4 | `sshd` long-term | Diagnostic lifeline | Password auth + public wifi. Move to key-only |
| M5 | Freeze override on `systemd-suspend.service` | Consistency (nvidia ships 5) | Plain suspend fine unfrozen |

## ❌ SHOULD BE REVERTED

| # | Item | Command |
|---|---|---|
| D1 | Hook executable | `sudo chmod -x /usr/lib/systemd/system-sleep/50-battery-hibernate.sh` |
| D2 | KDE suspend "Never" | GUI → restore a 10-min idle timeout |
| D3 | Test debris | `rm ~/btp-watch.log ~/lid-test*.log /tmp/wake-*.txt` |

## ↩️ ALREADY REVERTED

`rtc_cmos.use_acpi_alarm` (inert) · `HibernateDelaySec=2min` · `10-freeze-sessions.conf` (never active) · `fsck.mode=force` (one-shot, **clean**) · `99-test-lid-ignore.conf` (broke the session) · 3× limine backups · KWIN attempts A & B · `ff_rt_clk` manual enable · `pm_debug_messages` + `amd_pmc` dynamic debug

## 🔒 CLOSED — no further action

| Finding | Resolution |
|---|---|
| **Timer wake from s0i3** | ⛔ SMU 64.44 < 64.53, BIOS 416 is final. Permanently impossible |
| **`use_acpi_alarm` as a variable** | Inert — kernel quirk on 2021+ AMD |
| **`_BTP` battery wake** | ✅ Works, ±1 min, 3 confirmed self-wakes |
| **S0i3 platform health** | ✅ 99.4–99.7%, all blocks gated. Not a drain suspect |
| **`fsck`** | ✅ Clean after 5 forced power-offs |
| **KWin/dGPU as drain cause** | ❌ Refuted — real bug, no power effect |
| **Lid position as rollback cause** | ❌ Refuted by T14 |
| **`systemctl restart systemd-logind`** | 🚩 Never on a live session |
| **Aborted hibernate** | 🚩 Wedges NVIDIA; hard power-off required |
| **Gauge above 95%** | 🚩 Non-linear; can read *rising* during suspend |

---
---

# PART XI — MASTER LESSONS & GOTCHAS

## Diagnostic method
1. **Only `boot_id` and `uptime -s` prove a resume.** Restored windows prove nothing.
2. **A surviving SSH session is the strongest proof of all** — TCP cannot outlive a genuine reboot.
3. A gap between `hibernation entry` and `hibernation exit` proves real power-off.
4. A successful hibernate log looks *truncated*. Seeing `Read X kbytes` on resume means it **failed**.
5. A successful resume keeps `boot_id`, so `journalctl -b 0` is correct.
6. Error codes localise the stage: `-16`/`-22` = image never read · `-5` after a read = device quiesce.
7. **Grep widely on the first pass.**
8. **Change one variable at a time.**
9. **Verify the discriminator is what you think it is.** "Lid closed" tests were invalid twice — once because the lid was reopened, once because the instruction was physically impossible.
10. **Correlation across ≤3 runs is not causation.** Lid position correlated perfectly (open 9/9 ✅, closed 0/2 ⛔) and was still wrong.

## systemd
11. Drop-ins merge **by filename across all directories**. `/etc` wins only for *identical* names.
12. Prefix ranges: 10–40 vendor, 60–90 local.
13. Verify with `systemctl cat` / `systemctl show -p Environment`, then again in the runtime log.
14. `Environment=` does not propagate between units.
15. `HibernateDelaySec` and the low-battery trigger exist **only** inside STH.
16. **🚩 Never `systemctl restart systemd-logind` on a live desktop.** Duplicate greeter/user sessions, lock/unlock/freeze loop.
17. **`systemd-run` is too slow to win a sub-second race.**
18. `post/hibernate` hooks run **even when the hibernate rolls back**.

## Kernel / driver
19. **Unset ≠ disabled.** `PreserveVideoMemoryAllocations` defaults to `2` (auto) = enabled.
20. Verify module parameters at `/proc/driver/nvidia/params`, never the modprobe file.
21. Match parameters to driver generation.
22. **Module parameters can be silently overridden by kernel quirks.** `use_acpi_alarm=0` reached the cmdline and was ignored at probe.
23. **"GPU suspended" and "VRAM unpowered" are two separate switches.** Both must be off.
24. `nvidia-smi` wakes the GPU — never use it for PM state.
25. Runtime PM has an autosuspend delay; wait ≥2 min after resume.
26. `runtime_suspended_time` uses monotonic time, frozen during s2idle.
27. `last_hw_sleep` ≠ total; use `total_hw_sleep`.
28. `/sys/power/pm_test` does not reset itself.
29. A stale RTC wakealarm blocks STH from suspending at all.
30. **An aborted hibernate can wedge the NVIDIA driver unrecoverably.** `nv_restore_user_channels` WARN → all later suspends abort in ~127 ms → hard power-off.
31. **`amd_pmc` debugfs is meaningless before the first suspend of a boot.**
32. **`Hint Count` in `smu_fw_info`** tells you how many times s0i3 was entered — a free wake detector.
33. `power/wakeup = disabled` and "failed to suspend" are **different things**. A device can do the latter without the former.

## Measurement
34. **Measure in watts, not percent.**
35. Verify assumed capacity.
36. Log AC state on both sides of every measurement.
37. **Instrument early.** Every pre-Session-4 number carried ±20%.
38. **A fix that works is not a fix that helps** (KWin) — **and a fix that helps may look identical going in** (S0ix).
39. **🚩 Never measure drain above ~95% charge.** The gauge can read *rising*.
40. **Predictions are only as good as the constant.** The "5–10 min EC lag" was entirely an artifact of a stale 0.81 W figure.
41. **Underestimate drain when converting charge to time.** Overestimating inflates the budget and sleeps longer.

## Environment
42. `KWIN_DRM_DEVICES` splits on `:` — resolve with `readlink -f` first.
43. The shell strips backslashes when sourcing.
44. `/dev/dri/cardN` is unstable across boots — **confirmed: `card0` now exists and is the dGPU**.
45. User-scoped env beats `/etc/environment` on Wayland.
46. Auto-generated configs need override files, not edits.
47. **Set up SSH before sleep testing.** Refused = nothing listening; timeout = firewall.
48. ⚠️ Never press power during a hibernation image write.
49. **Know which component owns a policy.** PowerDevil owns the lid on KDE; the logind config was redundant *and* harmful.
50. **The power button is under the lid.** Design lid-closed tests accordingly.
51. **Don't run a watcher on a machine that sleeps.** A WSL host sleeping produced a 13-minute hole over the critical window.
52. **`rfkill` state persists across reboots** — a hard reset with radios blocked can leave you without SSH.

---
---

# APPENDIX A — Command Cookbook

## Pre-test ritual
```bash
cat /sys/power/pm_test                       # [none]
cat /sys/class/rtc/rtc0/wakealarm            # 0
cat /sys/class/power_supply/BAT0/alarm       # known value
cat /sys/class/power_supply/BAT0/capacity    # 60–80% for drain/_BTP tests
cat /sys/class/power_supply/A*/online        # 0 = on battery
swapon --show
uptime -s; cat /proc/sys/kernel/random/boot_id   # RECORD OFF-MACHINE

cd /tmp && mkdir -p hib-marker && cd hib-marker
export MARKER="test-$(date +%H%M%S)"; echo $MARKER
sleep 9999 &
```

## Post-test verification
```bash
cat /proc/sys/kernel/random/boot_id          # UNCHANGED = real resume
echo $MARKER; jobs; pwd
journalctl -b 0 -k -o short-precise --no-hostname \
  -g "suspend entry|suspend exit|hibernation entry|hibernation exit|rolling back|quiesce|NVRM|Xid"
journalctl -b 0 -t battery-hibernate --no-pager
cat /var/log/sleep-audit.tsv | tail -5
```

## AMD PMC / s0i3 *(only meaningful after a suspend)*
```bash
sudo cat /sys/kernel/debug/amd_pmc/smu_fw_info   # Hint Count, S0i3 status/times
sudo cat /sys/kernel/debug/amd_pmc/s0ix_stats
echo 'module amd_pmc +p' | sudo tee /sys/kernel/debug/dynamic_debug/control
journalctl -b 0 -k | grep -i 'SMU program'       # firmware version
```

## dGPU state *(wait ≥2 min after resume)*
```bash
cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status      # suspended
cat /proc/driver/nvidia/gpus/0000:01:00.0/power                 # Video Memory: Off
cat /proc/driver/nvidia/params | grep -iE 'preserve|suspendnotif|s0ix|temporary'
# NEVER nvidia-smi for PM state
```

## `_BTP` test
```bash
cat /sys/class/power_supply/A*/online        # must be 0
cat /sys/class/power_supply/BAT0/capacity    # 60–80%, NOT 100%
E=$(cat /sys/class/power_supply/BAT0/energy_now); echo $E
echo $((E - 150000)) | sudo tee /sys/class/power_supply/BAT0/alarm
cat /sys/class/power_supply/BAT0/alarm       # confirm it stuck
systemctl suspend
# cleanup:
echo 8992000 | sudo tee /sys/class/power_supply/BAT0/alarm
```

## Off-machine wake watcher *(run on a device that will NOT sleep)*
```bash
while :; do printf '%s ' "$(date +%H:%M:%S)"; \
  timeout 2 bash -c '</dev/tcp/192.168.1.220/22' 2>/dev/null && echo UP || echo down; \
  sleep 15; done | tee ~/watch.log
```

## Emergency
```
Hold power button 10 s        # wedged NVIDIA driver
Ctrl+Alt+F3                   # VT switch
ssh ekj@192.168.1.220
REISUB                        # sysrq, now persistent
rfkill unblock all            # if SSH dead after a hard reset
sudo loginctl unlock-sessions # locked/frozen session
```

---

# APPENDIX B — Complete Sleep-Cycle Timeline

## Sessions 1–3 *(carried over)*

| # | Date | Type | Duration | Result |
|---|---|---|---|---|
| — | 08-17 | hibernate ×2 (pm_test dry-run) | 24 s / 22 s | dry-run only |
| — | 08-17 | hibernate "real" (§9.3) | — | ⚠️ unverified |
| — | 08-18 | hibernate (pre-fix) | — | ❌ FAIL `-5` |
| 1 | 08-18 | hibernate | 51 s | ✅ |
| 2 | 08-18 | hibernate | 2 m 46 s | ✅ |
| 3 | 08-18 | hibernate (dGPU active) | 1 m 38 s | ✅ |
| 4 | 08-18 | hibernate (freeze active) | 28 m 25 s | ✅ |
| 5 | 08-18 | STH (manual wake) | 3 m 15 s + 1 m 12 s | ⚠️ no self-wake |
| 6 | 08-18→19 | suspend, overnight | 12 h 52 m | 📊 ~0.8 W |
| 7–9 | 08-19 | suspend, rtcwake ×3 | — | ❌ no RTC wake |

## Session 4

| # | Time | Type | Duration | Result |
|---|---|---|---|---|
| 10 | 12:06→12:16 | suspend (rtcwake, `use_acpi_alarm=0`) | 10 m 18 s | ❌ no wake · 99.4% S0i3 · **SMU 64.44** |
| — | 13:00 | *reboot* — fsck forced | — | ✅ **clean** |
| 11 | 12:38→13:08 | suspend (`_BTP` 300 mWh) | 30 m 11 s | ✅ **self-wake, witnessed** |
| 12 | 13:22→13:34 | suspend (`_BTP` 150 mWh) | 12 m 19 s | ✅ **self-wake, off-machine log** |
| 13 | 13:44→13:53 | suspend (KDE idle) | 9 m 55 s | ⚪ incidental |
| — | 14:12 | *reboot* — S0ix param + i2c group | — | — |
| 14 | 14:13→14:14 | **hibernate** (S0ix enabled) | 60 s | ✅ **SSH survived** |
| 15 | 14:20→14:55 | suspend | 35 m 18 s | 📊 **619 mW** |
| 16 | 15:04→15:40 | suspend (**hook armed**, lid open) | 36 m 15 s | ✅ **`_BTP` wake** · 📊 551 mW |
| 17 | 15:41→17:05 | **hibernate** (hook-triggered) | 84 m | ✅ **PSEUDO-TIMER PASS** · 📊 ~0.2 W S4 |
| 18 | 17:15→17:44 | suspend (hook, **lid closed**) | 29 m 39 s | ✅ wake · 📊 576 mW |
| 19 | 17:44→17:45 | hibernate attempt | 35 s | ⛔ **ROLLBACK** → wedge |
| — | 17:45→18:18 | ~80 aborted suspends | 127 ms each | ⛔ 19.7 Wh burned · hard power-off |
| 20 | 18:47→18:57 | suspend (hook, hardened, lid closed) | 10 m 30 s | ✅ wake |
| 21 | 18:57→18:58 | hibernate attempt | 36 s | ⛔ **ROLLBACK** → wedge · hard power-off |
| — | 19:26–19:31 | *logind restart incident* | — | 🚩 session breakage, 2 reboots |
## Appendix B, continued

| # | Time | Type | Duration | Result |
|---|---|---|---|---|
| 22 | 19:52→19:54 | suspend (KDE idle timer, lid closed) | ~2 m | ⚪ incidental |
| 23 | 19:54→20:18 | **hibernate** (lid open, post-incident) | 24 m 36 s | ✅ PASS |
| 24 | 21:21→21:22 | **hibernate, LID PHYSICALLY CLOSED** (T14) | 96 s | ✅ **PASS — lid exonerated** |
| 25 | 21:29→21:49 | suspend (hook armed, PowerDevil disabled) | 20 m 18 s | ⚪ **inconclusive** — gauge plateau at 100%, energy *rose* 48 mWh |

**Session 4 totals:** 4 hibernates attempted via normal paths, **4 passed** (#14, #17, #23, #24) · 2 hibernates attempted after a `_BTP` wake with the lid closed, **2 rolled back** (#19, #21) · 3 `_BTP` self-wakes confirmed (#11, #12, #16, #18 — four counting the lid-closed one) · 0 RTC wakes (#10) · 2 hard power-offs · 1 self-inflicted session breakage.

**Lifetime hibernate record: 8/8 on the normal path, 0/2 following a `_BTP` wake with the lid shut.**

---
---

# APPENDIX C — The Pseudo-Timer Script *(current state, disabled)*

`/usr/lib/systemd/system-sleep/50-battery-hibernate.sh` — mode `0644` (**not executable**, deliberately).

Preserved in full because it is the single most complex artifact of the investigation, and because the next session's work starts by editing it rather than rewriting it.

```bash
#!/usr/bin/env bash
# Charge-based pseudo-timer. Timer wake from s0i3 is impossible here
# (SMU 64.44 < 64.53), but the ACPI _BTP battery trip point does wake us.
set -u

### ---- CONFIG ----
SLEEP_MINUTES=15
DRAIN_MW=550             # measured. UNDER-estimate to hibernate on time (C20)
FLOOR_PCT=15
DEFAULT_ALARM=8992000
HIBERNATE_DELAY=20
### ----------------

BAT=/sys/class/power_supply/BAT0
STATE=/run/battery-hibernate.target
PENDING=/run/battery-hibernate.pending
LOOPC=/run/battery-hibernate.loopcount
LASTX=/run/battery-hibernate.lastexit

rd(){ cat "$1" 2>/dev/null || echo 0; }
log(){ echo "$*" | systemd-cat -t battery-hibernate -p notice; }
on_ac(){ [ "$(cat /sys/class/power_supply/A*/online 2>/dev/null|head -1)" = "1" ]; }

case "$1/$2" in
  pre/suspend)
    [ -e $PENDING ] && { log "hibernate pending - not re-arming"; exit 0; }

    # loop breaker: 3 consecutive suspends <60s apart = something is wrong
    NOW=$(date +%s); PREV=$(rd $LASTX); N=$(rd $LOOPC)
    if [ "$PREV" -gt 0 ] && [ $((NOW-PREV)) -lt 60 ]; then N=$((N+1)); else N=0; fi
    echo $N > $LOOPC
    if [ "$N" -ge 3 ]; then
      echo 0 > $BAT/alarm 2>/dev/null
      log "LOOP DETECTED ($N rapid cycles) - disarming, will not hibernate"
      exit 0
    fi

    rm -f $STATE
    if on_ac; then echo $DEFAULT_ALARM > $BAT/alarm 2>/dev/null
      log "on AC - no trip point armed"; exit 0; fi

    E=$(rd $BAT/energy_now); FULL=$(rd $BAT/energy_full)
    BUDGET=$(( DRAIN_MW * SLEEP_MINUTES * 1000 / 60 ))
    FLOOR=$(( FULL * FLOOR_PCT / 100 ))
    TARGET=$(( E - BUDGET ))
    [ $TARGET -lt $FLOOR ] && TARGET=$FLOOR
    [ $TARGET -ge $E ] && TARGET=$(( E - 50000 ))
    if echo $TARGET > $BAT/alarm 2>/dev/null; then
      echo $TARGET > $STATE
      log "armed: now=${E} target=${TARGET} (${SLEEP_MINUTES}min @ ${DRAIN_MW}mW)"
    else
      log "FAILED to write trip point"
    fi
    ;;

  post/suspend)
    date +%s > $LASTX
    [ -e $PENDING ] && exit 0
    [ -r $STATE ] || exit 0
    TARGET=$(cat $STATE); E=$(rd $BAT/energy_now)
    echo 0 > $BAT/alarm 2>/dev/null
    rm -f $STATE
    if on_ac; then echo $DEFAULT_ALARM > $BAT/alarm 2>/dev/null
      log "woke on AC - staying up"; exit 0; fi

    if [ "$E" -le "$TARGET" ]; then
      touch $PENDING
      log "BUDGET SPENT (${E} <= ${TARGET}) - radios off, hibernating in ${HIBERNATE_DELAY}s"
      rfkill block wifi bluetooth 2>/dev/null
      systemd-run --unit=battery-hibernate-guard \
        systemd-inhibit --what=sleep --mode=block --who=battery-hibernate \
        --why="pending hibernate" sleep $((HIBERNATE_DELAY+15)) >/dev/null 2>&1
      systemd-run --on-active=$HIBERNATE_DELAY --unit=battery-hibernate-now \
        systemctl hibernate -i >/dev/null 2>&1
    else
      echo $DEFAULT_ALARM > $BAT/alarm 2>/dev/null
      log "manual wake (${E} > ${TARGET}) - staying up"
    fi
    ;;

  post/hibernate)
    rm -f $PENDING $STATE $LOOPC
    rfkill unblock wifi bluetooth 2>/dev/null
    echo $DEFAULT_ALARM > $BAT/alarm 2>/dev/null
    log "resumed from hibernate - radios restored"
    ;;
esac
exit 0
```

## §C.1 Known defects

| # | Defect | Fix |
|---|---|---|
| **D-a** | **No full-charge guard.** Arms a target that cannot be reached above ~95%, because the gauge reads *rising* (C38) | Add `CAP=$(rd $BAT/capacity); [ "$CAP" -ge 95 ] && { log "…"; exit 0; }` before arming |
| **D-b** | `rfkill block` is ineffective (C30) and persists across reboots (lesson 52) — a hard reset with radios blocked leaves you with no SSH | Remove both `rfkill` lines |
| **D-c** | `systemd-inhibit` guard is too slow to spawn (C38/lesson 17) | Remove — dead weight |
| **D-d** | Loop breaker can only stop *arming*, not PowerDevil's retries (C29) | Keep as a safety net; it is not a solution |
| **D-e** | `DEFAULT_ALARM` restore is cosmetically odd — `post/suspend` writes `0`, then the manual-wake branch writes `8992000` | Harmless; tidy for clarity |

## §C.2 What is proven about it

- **The logic is correct.** In ~80 spurious wake cycles during the failure loop, it never once misfired a hibernate. Every cycle logged `manual wake … staying up`, correctly.
- **The arithmetic is correct.** Asked for 30 min, delivered 29 m 39 s and 36 m 15 s.
- **`DRAIN_MW=550` is well calibrated** against three independent measurements.
- **It works end to end with the lid open** (#16 → #17).

The remaining problem is not in this script.

---
---

# PART XII — WHERE THINGS STAND

## §XII.1 What was actually achieved

Across four sessions and roughly thirty hours:

| Metric | Session 1 start | Now |
|---|---|---|
| Hibernate | Broken (0/1, misdiagnosed as working) | ✅ **8/8** |
| s2idle drain | 0.81 W *(believed 1.1 W)* | ✅ **0.58 W** |
| Standby runway | 3.4 days *(believed 4.9)* | ✅ **4.6 days** |
| 8 h lid-closed cost | ~10% | **~7.2%** |
| S0i3 residency | 95% *(cumulative, imprecise)* | ✅ **99.4–99.7%, per-cycle** |
| Filesystem | 5 unclean shutdowns, unchecked | ✅ **fsck clean** |
| Automatic wake source | None known | ✅ **`_BTP`, ±1 min** |
| Auto-hibernate timer | Believed impossible | ✅ **Works, lid open** |
| Auto-hibernate, lid closed | — | ⛔ **Unsolved** |
| Measurement quality | ±20%, accidental | ✅ **Instrumented, ±2%** |

Two of the three original goals are fully met. The third — Mac-style unattended two-tier sleep — works in every configuration except the one that matters most.

## §XII.2 The single remaining problem, stated precisely

> After a `_BTP`-triggered wake from s0i3 **with the lid closed**, a subsequent `systemctl hibernate` reaches the final pre-power-off checkpoint — image written, `ACPI: EC stopped`, CPUs offlining — and then aborts with `PM: hibernation: Wakeup event detected during hibernation, rolling back`. The rollback leaves the NVIDIA driver unable to restore GPU channels, after which every suspend attempt fails in ~127 ms and the machine requires a hard power-off.
>
> The same hibernate succeeds with the lid **open** after an identical `_BTP` wake (#17), and succeeds with the lid **closed** when no `_BTP` wake preceded it (#24).

Reproduced twice. Nine hypotheses eliminated (§33.7). One untested.

## §XII.3 The untested hypothesis

**PowerDevil requests a suspend during the ~35-second hibernation image write.**

Supporting observations:
- Lid open, PowerDevil idles ~10 minutes after a `_BTP` wake — a wide, quiet window.
- Lid closed, its suspend request appears ~45 s after the wake — the write takes ~35 s.
- Its broadcast is logged *after* the rollback, but logind can only act once unwinding completes; the request itself may well have arrived mid-write.

The test is to remove PowerDevil from the equation entirely — lid **Do nothing**, suspend **Never**, DPMS **Never** — and repeat, at **60–80% charge**.

Attempt #25 was exactly this test and produced no data, because the battery was at 100% and the trip point could never be crossed.

## §XII.4 If that fails

Then the honest conclusion is that unattended lid-closed auto-hibernate is not achievable on this hardware, and the options are:

1. **Option A (§VI.1)** — manual hibernate. Everything proven, nothing experimental, 4.6-day runway. This is what most owners of MUX-less NVIDIA hybrid laptops settle for, and it was Session 1's own stated fallback.
2. **Hybrid-sleep (§VI.4)** — writes the image *before* suspending, so no intermediate wake exists and the rollback is structurally impossible. Costs ~6 GB of NVMe writes per lid close and forfeits the entire power saving. Declined once; still available.
3. **S3 DSDT patch (§VIII.3)** — now weak as a power fix, but it changes the wake path wholesale and would sidestep both the SMU gate and possibly this rollback. High maintenance, real brick risk.

## §XII.5 Immediate resting state

Two commands leave the machine in a safe, fully working configuration:

```bash
sudo chmod -x /usr/lib/systemd/system-sleep/50-battery-hibernate.sh
```

and, in **KDE → Power Management → On Battery**, restore a sane idle-suspend timeout (10 minutes) so a closed lid does not stay awake indefinitely.

Everything else — hibernate, the drain fix, the audit hook, SSH, sysrq, the clean filesystem — is stable and requires no attention.

## §XII.6 Next session, in order

1. **Add the full-charge guard** to the hook (§C.1 D-a) and strip the dead rfkill/inhibit code (D-b, D-c).
2. **Re-run §VI.3** at 60–80% charge, with PowerDevil fully disarmed and a watcher on a device that will not sleep.
3. If it passes: set `SLEEP_MINUTES=240`, run it twice more for confidence, then adopt.
4. If it fails: adopt Option A and close the investigation.
5. Housekeeping, low priority: `kscreenlockerrc` autolock state (V4) · drop M1–M3 if desired · a clean overnight S4 drain figure · KDE splash restoration (outstanding since Session 1).

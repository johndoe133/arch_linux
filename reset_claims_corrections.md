# Claims, Corrections & Open Threads

**Updated 2026-08-20 22:45.** Companion files: `system_context.txt`, `change_log.md`.

---

## Part A — Claims and their evidence status

Everything asserted during this investigation, labelled. Nothing here is a plan;
plans are in Part C.

### Verified — direct observation on this install

| Claim | Evidence |
|---|---|
| Manual `systemctl hibernate` works | 2/2, 2026-08-20. Full power-off seen by eye; `boot_id` unchanged; SSH session and two `sleep` jobs alive across both cycles; Firefox restored without reloading; Kate kept an unsaved buffer |
| `PreserveVideoMemoryAllocations` defaults to `2` here | `/proc/driver/nvidia/params` read `2` with `/etc/modprobe.d/` empty |
| `UseKernelSuspendNotifiers` is `1` by default | Same read |
| Driver is the **open** kernel module 610.57.04 | `/proc/driver/nvidia/version`, package `linux-cachyos-nvidia-open` |
| Root ext4 extends to the end of the disk | `parted`: `nvme0n1p2` runs 4297 MB → 1024 GB. No room for a swap partition |
| `HOOKS` contains `systemd` | `/etc/mkinitcpio.conf`. Hence no `resume` hook / `resume=` needed |
| logind accepts hibernation | `CanHibernate` → `"challenge"` |
| RTC alarm arms correctly | `alarm_IRQ: yes`, `alrm_time` set correctly, `alrm_pending: no` |
| RTC alarm **fires** correctly while awake | 60 s alarm: after 75 s, `alarm_IRQ` → `no` and `wakealarm` self-cleared |
| RTC alarm does **not** wake from s2idle | 0/2. Both trials entered s2idle (`PM: suspend entry (s2idle)`, timekeeping suspended 140.084 s / 156.998 s); both alarm times fell inside those unbroken blocks; neither woke |
| s0i3 itself is healthy | `Last S0i3 Status: Success`, 99.4 % residency, 1.43 s entry, 0.41 s resume |
| A stale `wakealarm` does **not** block suspend | Trial 2 began with trial 1's alarm latched and suspended normally |
| `use_acpi_alarm` is `Y` with no parameter set | `/sys/module/rtc_cmos/parameters/use_acpi_alarm` |
| RTC wake capability is advertised only up to S4 | dmesg: `rtc_cmos PNP0B00:00: RTC can wake from S4` |
| `mkinitcpio -P` does not update Limine entries | Its own warning; `limine-mkinitcpio` was required |
| `pm_test` at `freezer` neither writes an image nor powers off | Observed: process freeze, 5 s dwell, thaw |
| RTC backup cell reports dead | `/proc/driver/rtc`: `batt_status: dead`. Timekeeping still correct across real power-offs |

### Verified — external documentation

| Claim | Source |
|---|---|
| On UEFI, systemd records the hibernation location in an EFI variable and reads it back at boot, so `resume=`/`resume_offset=` are unnecessary | ArchWiki |
| systemd ignores zram devices when choosing a hibernation target | ArchWiki |
| A hibernation image cannot span multiple swap areas | ArchWiki |
| logind's `Can*` methods return `na`/`yes`/`no`/`challenge`; `challenge` means available pending authorization | systemd D-Bus docs |
| `suspend-then-hibernate` tries ACPI `_BTP` low-battery alarms first and wakes on the low-battery signal to hibernate | systemd docs |
| Without `HibernateDelaySec=`, the system stays suspended as long as possible, hibernating near depletion | systemd docs |
| Hybrid-sleep writes an image and then suspends, so state survives battery depletion | systemd docs |
| On 610-open, `NVreg_UseKernelSuspendNotifiers=1` handles VRAM save/restore | NVIDIA README |
| The `-5` NVIDIA failure manifests on the **resume** leg, after the image is read back | Multiple external reports |
| Early KMS (nvidia in initramfs) interacts badly with `PreserveVideoMemoryAllocations` | External report |
| KDE has no hybrid-sleep option in its power panel | External reports |
| PowerDevil takes a blocking `handle-lid-switch` inhibitor, suppressing logind's lid handling | External reports |

### Inferred — reasoned, not directly tested

| Inference | Basis | Risk if wrong |
|---|---|---|
| The NVIDIA modprobe fix was **necessary** on this install | No unfixed hibernate was ever run post-reinstall. Pre-reinstall sessions and external reports support it | Low — the file is harmless either way |
| The RTC alarm fires in s0i3 but cannot rouse the platform | It fires in S0; the `RTC can wake from S4` line names S4, not S0ix | Medium — could instead be that the GPE is masked during s0i3 |
| `_BTP` still works on this install | Recorded 4/4 pre-reinstall; hardware and firmware unchanged | **High** — the entire remaining plan depends on it. Untested since reinstall |
| Native `suspend-then-hibernate` is immune to the lid-closed rollback | The transition happens inside one `systemd-sleep` invocation with sessions frozen, so there is no intermediate wake to race | **High** — this is the crux of the remaining work |
| A fired alarm clears `wakealarm` | Standard behaviour, consistent with observations | Low |
| Booting the LTS entry with a 7.1.8 image pending discards the image | Kernel version mismatch | Low — just don't do it |
| Hibernating writes ~6 GB per cycle | `image_size` 6327451648, ~35 s write | Low |

### Hearsay — carried forward, unverified on this install

| Claim | Status |
|---|---|
| SMU firmware is 64.44.0 | **No evidence.** `smu_fw_info` prints no version string on kernel 7.1.8 |
| s2idle drain is 0.81 W → 0.58 W with `EnableS0ixPowerManagement=1` | Pre-reinstall measurement. Parameter is currently `0` |
| Hibernated drain ~0.2 W | Pre-reinstall |
| `_BTP` wake accurate to ±1 min | Pre-reinstall, 4 self-wakes |
| The lid-closed rollback wedges the NVIDIA driver | Pre-reinstall, reproduced twice |
| `limine-update` can reset the NVIDIA parameter to `2` | Single external anecdote; did **not** reproduce here |

---

## Part B — Corrections (append-only, never edited, never deleted)

| # | Claim | Correction |
|---|---|---|
| 1 | `system_context.txt`: `amdgpu.dcdebugmask=0x40000`, Limine `timeout: 0`, disabled `NetworkManager-wait-online`, mkinitcpio MODULES/HOOKS edits, Plymouth/Spin, Otto/Tela/Willow theming, Spectacle shortcut, `hwinfo` | All wiped by the reinstall. Confirmed: `/proc/cmdline` has no `dcdebugmask`; `/boot/limine.conf` reads `timeout: 5`. The brightness-overflow bug at 99–100 % is live again |
| 2 | Handoff Part 1: "install-time choice — real 20 GiB swap partition" | Did not happen. The install had zram only, so hibernation was impossible until a swapfile was created |
| 3 | `system_context.txt`: "GPU driver: NVIDIA proprietary" | Wrong. It is the **open** kernel module 610.57.04. Material, because the hibernate fix is specific to the open module's PM path |
| 4 | Handoff Part 4: also set `NVreg_TemporaryFilePath=/var/tmp` because the default is `/tmp` | Wrong on 610.57.04 — with `modprobe.d` empty, `params` already read `/var/tmp`. It is also only consulted when VRAM preservation is on, which the fix disables. Dropped |
| 5 | My reply 1: "decide swapfile vs. partition from `lsblk`", implying a partition might be available | No unallocated space exists; root runs to the end of the disk. A partition would need an offline shrink from live media. Swapfile was the only option |
| 6 | My reply 2: "`pri=-2` keeps it below zram" | Effective priority is **−1**; the kernel auto-assigns and ignores negative `pri=`. Conclusion holds (−1 ≪ 100), the number was wrong |
| 7 | My reply 2: "don't test tonight — the resume branch might be busybox and need the fallback entry" | That risk never existed. `HOOKS` contains `systemd`. Right advice, wrong reason |
| 8 | My reply 2 collect block: `ls -la /boot` | Failed with permission denied; the ESP is mounted `umask=0077` and needs `sudo`. My error |
| 9 | Correction 1's implication that the mkinitcpio `HOOKS` edit was lost | The live `HOOKS` line is byte-identical to the doc's "revert to" line — that is stock CachyOS. The recorded change netted to zero |
| 10 | My reply 3: the B2 command block would capture the devices test | It did not. `journalctl` raced the transition, so the "B2" output was actually B1's freezer run. No devices-test data was ever collected |
| 11 | User: "both of the hibernates booted themselves back" | Neither hibernated and neither rebooted. `pm_test` at freezer/devices never writes an image or powers off. Nothing was torn down, so nothing was restored |
| 12 | My reply 3: "B1+B2 is ~15 minutes and low-consequence" | Low-consequence yes, but it produced **one** usable result, not two, because of Correction 10 |
| 13 | My reply 4 cited "your §20.1" and "§19.2" as prior recorded experiments | **Fabricated.** The handoff has Parts 1–10 and an Appendix; no such sections exist. The conclusion happened to hold, but the citation was invented |
| 14 | My reply 3's entire B1/B2 plan as a baseline for the hibernate bug | **Structurally incapable of detecting it.** The failure occurs in the fresh boot kernel after the initramfs loads the image; `pm_test` never leaves the booted system. A clean pass would have been a false negative |
| 15 | My replies 3–4: "a failed hibernate can wedge the driver — don't test at night" | Over-cautious for that test. The documented failure mode is an ordinary cold boot; cost is unsaved work. Borne out — nothing wedged. (The wedge risk is real but requires a prior `_BTP` wake) |
| 16 | My reply 5's post-hibernate check block | **Raced the transition.** `systemctl hibernate` returns on request-accept, so `boot_id`/`MARKER`/`jobs` ran before the machine went down, both times. Second racy block written |
| 17 | Handoff Part 3: "Only `boot_id` and `jobs` count" | Incomplete. **SSH session survival** is a stronger zero-setup discriminator. And none of the three separate a genuine resume from a *rollback* — only the journal or a witnessed power-off does |
| 18 | My reply 5's command block: `sudo mkinitcpio -P` then reboot | **Insufficient on Limine.** It warns that it does not update Limine boot entries. Use `sudo limine-mkinitcpio` |
| 19 | My reply 6's rtcwake test design | Not a valid test — no instrumentation to distinguish "suspended but didn't wake" from "never suspended". Salvaged only because the journal happened to record `suspend entry` |
| 20 | My reply 7: "trial 2 was contaminated by the stale alarm" | Wrong — superseded by Correction 23 |
| 21 | Handoff Part 7.4: `rtcwake -m no -s 45 && systemctl hibernate` as the positive control for the rollback bug | **Non-functional here.** RTC alarms don't fire in s0i3, so it cannot inject a mid-write wake. The designated instrument for the unsolved bug does not exist |
| 22 | Handoff Part 3: "restored windows prove **nothing**" | Too strong. *Which* windows return proves nothing, but *how* they return is real evidence: Firefox reloads and Kate loses unsaved buffers after a cold boot. Neither happened |
| 23 | My Correction 20, and handoff landmine 12 ("a stale `wakealarm` blocks suspend entirely") | **Did not reproduce.** Trial 2 began with trial 1's alarm latched and suspended normally. Both trials are valid; RTC wake is 0/2, not 0/1 |
| 24 | Handoff Part 6.1 / Appendix: "SMU firmware 64.44.0, needs ≥ 64.53" | **Unverified.** `smu_fw_info` on kernel 7.1.8 prints statistics only, no version string. No evidence for the number |
| 25 | Handoff landmine 5 ("`rtc_cmos.use_acpi_alarm` is inert, forced on for AMD BIOS ≥ 2021"), and my counter-argument about a `FIXED_RTC` FADT flag | **Resolved against me.** `use_acpi_alarm` reads `Y` with no parameter set — the quirk applied exactly as the handoff described. Whether `=0` can override it remains untested |
| 26 | My reply 8 and 9 command blocks: `tee /mnt/data/…` | **`/mnt/data` is a path in my sandbox, not on the laptop.** `tee` failed both times. Use `~/` |
| 27 | My reply 9's outcome table: "`alarm_IRQ: yes` → armed but not delivered → platform wall, stop" | Premature. It conflated "fires in S0 but can't wake from s0i3" with "never fires at all" — two failures, two fixes, one observation |
| 28 | Handoff Part 6.1: "`amd_pmc` declines to even attempt the handoff" because SMU 64.44 < 64.53 | **Contradicted.** The alarm *is* armed and *does* fire in S0. Nothing declines to attempt anything. The conclusion (no RTC timer) stands; the mechanism was wrong |
| 29 | My reply 1, and the original step 3 ("suspend → hibernate timed by battery level") | **Not achievable as specified.** Every timed variant needs a self-wake. RTC is proven dead; `_BTP` fires near depletion, making it a backstop rather than a timer |
| 30 | My reply 10: "Do hybrid-sleep instead — it's the actual answer" | **Presented a solution as settled without checking a constraint I had not asked about.** Hybrid-sleep writes ~6 GB to the SSD on every lid close; the user rejects that. The option is withdrawn, not deferred |

---

## Part C — Open threads (max 5)

These describe things that **do not exist on the system**.

### 1. Re-confirm `_BTP` wake on this install — **blocks everything else**
Pre-reinstall it woke the machine from s0i3 4/4. Untested since. Every remaining plan
depends on it, and RTC is already proven dead, so this is the last known wake path.
Requires battery in the 60–80 % band, on battery power, lid open.
- *Works* → thread 2 is viable.
- *Fails* → there is no self-wake at all, and the only options left are plain suspend with
  manual hibernate, or lid → hibernate directly.

### 2. Native `suspend-then-hibernate` with no `HibernateDelaySec=`
The remaining candidate that satisfies "suspend first, don't write to SSD on every lid
close". Suspends at ~0.58 W; writes an image only when the battery approaches depletion.
Not a timer — the trip is near 5 %, i.e. potentially days out.
Needs: a check of which `sleep.conf` directives systemd 261 still honours (many per-mode
options were removed in recent versions), then a test starting from a low battery.
Also the natural test of the Part 4 immunity inference.

### 3. Lid routing: PowerDevil vs logind
KDE's PowerDevil holds a blocking lid inhibitor and may pre-empt logind's
`HandleLidSwitch=` even when set to "Do nothing". Needs `systemd-inhibit --list`, the
current KDE lid setting, a `logind.conf` drop-in, and a reboot (never restart logind on a
live desktop). Downstream of thread 2 — decide the *action* before wiring the *trigger*.

### 4. `NVreg_EnableS0ixPowerManagement=1` and a real drain measurement
Currently `0`. Pre-reinstall this took s2idle from ~0.81 W to ~0.58 W (~28 %), extending
standby runway from 3.4 to 4.6 days. Blocked on a 60–80 % battery, and needs a **60+
minute** window — a 10-minute test is below the gauge's noise floor. ⚠️ Re-test hibernate
afterwards: this parameter is in the same family as the one that broke it.

### 5. `amdgpu.dcdebugmask=0x40000` — brightness overflow at 99–100 %
Known-good one-liner, currently missing, bug is live. Add to `KERNEL_CMDLINE[default]` in
`/etc/default/limine`, then `sudo limine-update`. Held until the sleep work settles so it
is not a confounding variable.

### Dropped to stay at five
- **The lid-closed rollback bisect (kernel bug 218634).** Its designated positive control
  is dead (Correction 21), so it needs a new instrument before it can even be approached
  — and thread 2 may make it moot. Full description retained in `system_context.txt` §6.5.
- **Hybrid-sleep.** Withdrawn on the SSD-write constraint (Correction 30), not deferred.

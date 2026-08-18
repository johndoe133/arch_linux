# §15 — Suspend Battery Drain Investigation (Session 2)

**Date of session:** 2026-08-18
**Continues from:** §1–§14 (Hibernate investigation, 2026-08-17)
**Status:** ✅ dGPU runtime-suspend fixed · ⏳ Overnight drain re-test pending · ⏳ Lid automation still not configured

---

## 15.1 Objective

Address §12.1 — the original problem carried over from Session 1: excessive battery drain during `s2idle` suspend. Hibernate works (§9.3), but does not help short/medium closures. Target: get the RTX 3070 to actually power down so `s2idle` isn't paying a constant dGPU tax.

---

## 15.2 Drain Baselines (measured, pre-fix)

| Window | Duration | Loss | Rate | Avg power (est. 90Wh battery) |
|---|---|---|---|---|
| Overnight (17→18 Aug) | ~8 h | ~10% | 1.25 %/h | ~1.1 W |
| Workday (18 Aug) | ~8 h | ~15% | 1.88 %/h | ~1.7 W |

Both measured **before** any Session 2 change was applied. These are the numbers to beat.

---

## 15.3 Baseline Diagnostics (read-only, pre-change)

### 15.3.1 NVIDIA power state
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

### 15.3.2 PCI runtime PM
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

### 15.3.3 GPU function topology
```
$ lspci -s 01:00 -v
01:00.0 VGA compatible controller: NVIDIA GA104M [RTX 3070 Mobile / Max-Q] (rev a1)
        Kernel driver in use: nvidia
01:00.1 Audio device: NVIDIA GA104 High Definition Audio Controller (rev a1)
        Kernel driver in use: snd_hda_intel

$ for f in /sys/bus/pci/devices/0000:01:00.*/power/runtime_status; do echo "$f: $(cat $f)"; done
0000:01:00.0 (VGA):   active      🚩
0000:01:00.1 (Audio): suspended   ✅
```

**Key deduction:** the audio function under the same GPU suspends fine → not a bus-level or ACPI power-resource problem. The VGA function specifically is being held awake.

> Note: no USB-C/UCSI functions exist on this GPU (unlike many Optimus laptops) — only `.0` and `.1`.

### 15.3.4 Who holds the GPU
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
| Processes:                                                 |
|   0  N/A  N/A  1110  C+G  /usr/bin/kwin_wayland     11MiB | 🚩 ROOT CAUSE
```

`kwin_wayland` held a **live, VRAM-backed context** on the dGPU — not merely an open handle.

### 15.3.5 Platform sleep residency
```
$ cat /sys/power/suspend_stats/last_hw_sleep    → 512005920      (µs = 512 s)
$ cat /sys/power/suspend_stats/total_hw_sleep   → 34382329131    (µs = 34,382 s = 9 h 33 m)
$ cat /sys/power/suspend_stats/success          → 2
$ cat /sys/power/suspend_stats/fail             → 0
$ uptime -s                                     → 2026-08-17 20:57:30
```
Elapsed since boot ≈ 10 h 02 m (36,150 s). **Hardware-sleep ratio ≈ 95%.**

### 15.3.6 DRM device enumeration
```
$ for c in /sys/class/drm/card*/device; do echo "$c -> $(cat $c/vendor 2>/dev/null)"; done
/sys/class/drm/card1/device -> 0x10de     # NVIDIA RTX 3070
/sys/class/drm/card1-DP-1/device ->
/sys/class/drm/card1-DP-2/device ->
/sys/class/drm/card2/device -> 0x1002     # AMD Renoir iGPU
/sys/class/drm/card2-eDP-1/device ->      # internal panel
/sys/class/drm/card2-HDMI-A-1/device ->   # HDMI output
```

**Critical:** there is **no `card0`** on this system. Numbering is also **not stable across boots** (amdgpu/nvidia module load-order race).

---

## 15.4 Root Cause

**`kwin_wayland` holds a permanent render context on the RTX 3070**, preventing it from ever entering runtime suspend (D3cold) — confirmed by `runtime_suspended_time == 0` across the entire boot.

Because `s2idle` (unlike S3) does **not** force PCI devices into low-power states, a device that is `active` at the moment of suspend simply *stays* active for the duration of the sleep. The dGPU therefore drew power continuously all night despite excellent platform-level sleep residency.

This reconciles the two seemingly contradictory measurements:

| Evidence | Reading | Meaning |
|---|---|---|
| `total_hw_sleep` ratio | ~95% | ✅ CPU/SoC reached deep idle — AMD s2idle path is **healthy** |
| `runtime_suspended_time` | `0` | 🚩 dGPU never powered down — the drain source |

`KWIN_DRM_DEVICES` is a first-class KWin feature (present in KWin's DRM backend, explicitly supported for GPU-exclusion), not a workaround. Independent reports exist of the same symptom and fix on the same laptop class (incl. a CachyOS Zephyrus G16 report).

---

## 15.5 Hardware Finding — Display Output Wiring (GA503)

| Port | Wired to | Affected by restricting KWin to iGPU? |
|---|---|---|
| **HDMI** | iGPU (`card2-HDMI-A-1`) | ❌ No impact |
| **USB-C / DP Alt Mode** | dGPU (`card1-DP-1/2`) | ⚠️ Would break external output |

User confirmed: HDMI-only in practice; loss of USB-C display output acceptable. Fix applied as a **reversible session script** rather than system-wide, specifically to keep that option open.

---

## 15.6 Change Log — Attempts

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
| File | `~/.config/plasma-workspace/env/kwin-gpu.sh` |
| Content | `export KWIN_DRM_DEVICES=/dev/dri/by-path/pci-0000\:06\:00.0-card` |
| Rationale | Stable PCI path to survive card renumbering; `\:` because KWin uses `:` as its list separator |
| Problem | Shell **consumed the backslashes when sourcing** → KWin received unescaped colons → split into 3 invalid fragments |
| Outcome | **Black screen after login.** KWin crash-looped 3× with coredumps. |
| Recovery | Ctrl+Alt+F3 → `rm` the file → reboot. **Login screen was unaffected** (user-scoped, not `/etc/environment`). |
| Status | **REVERTED — no trace remains** |

Log evidence:
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

## 15.7 Verification Results (post-fix)

```
$ sudo cat /proc/$(pidof kwin_wayland)/environ | tr '\0' '\n' | grep KWIN_DRM
KWIN_DRM_DEVICES=/dev/dri/card2                    ✅ reached KWin

$ cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status
suspended                                          ✅ FIRST TIME THIS BOOT
```

`nvidia-smi` output (post-fix):
```
| N/A  49C  P3  24W / 55W |  44MiB / 8192MiB |  0% |
|   0  N/A  N/A  3248  C+G  /usr/bin/kwin_wayland  11MiB |
```

**⚠️ This output is misleading and does NOT indicate failure:**

1. **`nvidia-smi` itself wakes the GPU.** Querying forces it out of D3cold — the `P3`/`24W` reading is a measurement artifact, not the idle state. **`nvidia-smi` is unreliable as a runtime-PM diagnostic.** Command ordering proves the point: `nvidia-smi` ran first (woke GPU), then after `sleep 120` the GPU had returned to `suspended` unaided — exactly the desired behaviour, and impossible before the fix.
2. **KWin retains a lightweight handle** for PRIME buffer import even when the device is excluded as a *render* device. This no longer blocks runtime suspend, which is the only thing that matters.

**Authoritative check is sysfs, not `nvidia-smi`.**

---

## 15.8 Hypotheses Tested and Discarded (Session 2)

| Hypothesis | Verdict | Evidence |
|---|---|---|
| AMD `s2idle` platform path is broken | ❌ Healthy | `total_hw_sleep` ≈ 95% of elapsed time |
| `last_hw_sleep = 512 s` proves poor overnight sleep | ❌ Misread | It reports only the **most recent single cycle**; `total_hw_sleep` is the cumulative figure |
| A sibling GPU function (audio/USB-C) blocks D3cold | ❌ No | `01:00.1` already `suspended`; no USB-C/UCSI functions exist on this GPU |
| Missing `NVreg_EnableS0ixPowerManagement` is the primary cause | ⚠️ Deferred | Real gap, but irrelevant while KWin prevents D3cold entirely — that logic never engages |
| Fix "disables the dGPU" | ❌ No | Only stops KWin holding a context; PRIME offload unaffected |
| `card0` is the AMD iGPU | ❌ Wrong | No `card0` exists; AMD is `card2`, NVIDIA is `card1` |
| `/dev/dri/cardN` is a safe stable identifier | ❌ No | Numbering races on module load order; must use PCI path |
| Plymouth is relevant to the greeter risk | ❌ No | Plymouth = boot splash only; greeter is Plasma Login Manager (PLM) |
| `amdgpu: Runtime PM not available` indicates a fault | ❌ Benign | Known ordering artifact tied to DM backlight registration |

---

## 15.9 Environment Notes

- **Display manager:** Plasma Login Manager (PLM) — SDDM fork with a new greeter, more tightly Plasma/KWin-integrated than SDDM. Fedora 44 is switching KDE variants to it.
  - **Implication:** system-wide `/etc/environment` is *more* risky here than on SDDM. The user-scoped script avoids the greeter entirely. Confirmed empirically — Attempt B black-screened the session but the **login screen still worked**.
- **`~/.config/plasma-workspace/env/` scripts** are sourced only after successful login → PLM never reads them.

---

## 15.10 CURRENT SYSTEM STATE (consolidated — Sessions 1 + 2)

### ✅ Files created / modified — ACTIVE

| Path | Purpose | Session |
|---|---|---|
| `/swapfile` | 20 GiB hibernation target, 14 extents, prio `-1`, offset `220407808` | 1 |
| `/etc/fstab` | `/swapfile none swap defaults 0 0` | 1 |
| `/etc/mkinitcpio.conf` | `resume` hook added to `HOOKS` | 1 |
| `/etc/modprobe.d/nvidia-power-management.conf` | `NVreg_PreserveVideoMemoryAllocations=1`, `NVreg_TemporaryFilePath=/var/tmp` | 1 |
| `/etc/systemd/system/systemd-hibernate.service.d/10-freeze-sessions.conf` | `Environment=SYSTEMD_SLEEP_FREEZE_USER_SESSIONS=true` — **the hibernate fix** | 1 |
| `~/.config/plasma-workspace/env/kwin-gpu.sh` | `KWIN_DRM_DEVICES` → iGPU — **the suspend-drain fix** | 2 |

### ✅ Services enabled
`nvidia-suspend.service` · `nvidia-hibernate.service` · `nvidia-resume.service`

### ✅ Kernel parameters (unchanged)
```
quiet nowatchdog splash rw root=UUID=2b44456f-... amdgpu.dcdebugmask=0x40000
```
No `resume=` / `resume_offset=` — correctly unnecessary; systemd uses the `HibernateLocation` EFI variable.

### ❌ Reverted — no trace remains
- Attempt A: `KWIN_DRM_DEVICES=/dev/dri/card0`
- Attempt B: `KWIN_DRM_DEVICES=/dev/dri/by-path/pci-0000\:06\:00.0-card`

### ⬜ Discussed but **NOT** applied

| Item | Notes |
|---|---|
| `/usr/lib/systemd/system-sleep/battery-log.sh` | Battery %/energy logging hook |
| `/usr/lib/systemd/system-sleep/power-audit.sh` | Pre-suspend D0-device audit hook |
| `NVreg_EnableS0ixPowerManagement=1` | Would flip `S0ix Status: Disabled` → enabled |
| `NVreg_DynamicPowerManagement=0x02` + `...VideoMemoryThreshold=0` | Reserve for the D3cold-cycling bug |
| `powertop --auto-tune` | — |
| `/proc/acpi/wakeup` audit | Lid/trackpad/USB wake sources never reviewed |
| `amd_s2idle.py` / `amd-debug-tools` | Deprioritised — platform sleep already ~95% |
| `envycontrol` | Never installed |
| `HandleLidSwitch=suspend-then-hibernate` | §13 step 6 — **still outstanding** |
| `HibernateDelaySec` | Never configured |
| `fsck` from live USB | **Still pending** after 2 forced power-offs (Session 1) |
| KDE splash restoration | **Still lost** from Session 1 `mkinitcpio` regeneration |

### 🔧 Unchanged from Session 1
`zswap` (zstd) active alongside zram · lid behaviour default · KDE power settings default · UEFI Switchable Graphics

---

## 15.11 Immediate Verification TODO

Not yet run — the authoritative checks that avoid `nvidia-smi`'s wake artifact:
```bash
cat /sys/bus/pci/devices/0000:01:00.0/power_state                 # want: D3cold (D3hot = partial)
cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_suspended_time # want: nonzero AND growing
fuser -v /dev/dri/card1 /dev/dri/renderD128 2>/dev/null            # Xwayland should be off NVIDIA node
prime-run glxinfo | grep "OpenGL renderer"                         # confirm dGPU still usable
```
After `prime-run` exits, `runtime_status` should return to `suspended` within ~20 s.

---

## 15.12 Next Overnight Test

```bash
cat /sys/power/suspend_stats/total_hw_sleep
cat /sys/power/suspend_stats/success
cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_suspended_time
# + battery % before/after
```

**Predicted:** ~5–7% over 8 h (vs. 10% / 15% baselines).
**Not** near-zero — no S3 available, and `s2idle` has a hard floor: RAM self-refresh ~0.1–0.3 W, EC + keyboard/RGB ~0.1–0.3 W, plus NVMe low-power state and PMIC/VRM leakage ≈ **0.4–0.7 W total**.

If the result lands in that range, the platform is at its floor and further s2idle tuning has little left to give. The remaining gap is closed by `suspend-then-hibernate`, **not** by more suspend optimisation.

---

## 15.13 Rollback

**Suspend fix only:**
```bash
rm ~/.config/plasma-workspace/env/kwin-gpu.sh
# log out and back in
```
Restores KWin's default multi-GPU behaviour, including USB-C/DP dGPU output.

**Emergency (black screen at login):**
```
Ctrl+Alt+F3  →  rm ~/.config/plasma-workspace/env/kwin-gpu.sh  →  sudo reboot
```
Login screen remains functional because the script is user-scoped (verified during Attempt B).

**Full Session-1 rollback:** see §14.

---

## 15.14 Lessons / Gotchas

1. **`nvidia-smi` wakes the GPU** — never use it to judge runtime-PM state. Use sysfs.
2. **`last_hw_sleep` ≠ overnight total** — it is the last cycle only; use `total_hw_sleep` and compare against elapsed wall-clock.
3. **`KWIN_DRM_DEVICES` splits on `:`** — never pass a raw `by-path` name; resolve it first.
4. **Shell strips backslashes when sourcing** — escaping intended for the *variable value* will not survive.
5. **`/dev/dri/cardN` is unstable** across boots; PCI addresses are not.
6. **Never verify a device path by assumption** — Attempt A pointed at a nonexistent `card0`.
7. **User-scoped env beats `/etc/environment`** on Wayland — it cannot break the greeter. Proven under fire.
8. **Platform sleep and device sleep are independent** — 95% hardware-sleep residency coexisted with a dGPU that never suspended once.
9. **There is no per-device power profiler for suspend** — the CPU is parked. Use pre-suspend state snapshots, residency counters, `rtcwake` sampling, and A/B elimination instead.

Two things worth flagging before you file this away:

**§15.11 is unverified.** The `D3cold` vs `D3hot` check and the `runtime_suspended_time` growth check are the ones that actually confirm the fix is doing what we think — `suspended` alone doesn't distinguish "powered down" from "partially powered." Worth running before tonight's test so the overnight numbers have a clean interpretation.

**Two Session 1 items are still open and unrelated to battery:** the pending `fsck` after those two forced power-offs, and the lost KDE splash setting. Neither is urgent, but the `fsck` has been outstanding for a day now.

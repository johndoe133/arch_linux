# Change Log — Zephyrus G15 / CachyOS

**Updated 2026-08-21 10:00.** No changes applied 08-21; testing only.
**The invariant: if it is not in this table, it is not on the system.**
IDs are flat integers, never reused. Companion files: `system_context.txt`,
`claims_corrections_threads.md` (corrections now run to #54).

⚠️ **Nothing on this system has a measured *power* benefit.** The change that
allegedly produced one (`NVreg_EnableS0ixPowerManagement=1`, "0.81 → 0.58 W") is not
applied — and its premise is now doubtful: **stock s2idle drain measured 0.525 W on
this install** (1852 s unbroken block, 08-21), already below the claimed improved
figure. Re-measure before believing the parameter buys anything.

## Applied changes

| ID | Change | Why | State | Verified | Verdict | Verify |
|---|---|---|---|---|---|---|
| 1 | `sshd` enabled + started | Remote shell when the desktop wedges | Applied | ✅ 08-21 | **Proven** | `ssh ekj@192.168.1.220` |
| 2 | `/etc/sysctl.d/99-sysrq.conf` → `kernel.sysrq=1` | REISUB instead of hard power-off | Applied | ✅ 08-20 | **Insurance** | `cat /proc/sys/kernel/sysrq` → `1` |
| 3 | `journald.conf.d/60-persistent.conf` → persistent, 2 G, `RateLimitBurst=0` | Logs must survive a crash | Applied | ✅ 08-20 | **Insurance** | `journalctl -b -1 -n 3` returns lines |
| 4 | `ufw allow from 192.168.1.0/24 to any port 22 proto tcp` | Without it change 1 is inert | Applied | ⚠️ 08-20 | **Necessary** | `sudo ufw status verbose` — *never run* |
| 5 | `/swapfile` — 20 GiB, mode 0600 | Hibernation needs non-zram swap ≥ image | Applied | ✅ 08-20 | **Necessary** | `swapon --show` |
| 6 | `/etc/fstab` swapfile line | Change 5 vanishes at reboot without it | Applied | ✅ 08-20 | **Necessary** | `swapon --show` after reboot |
| 7 | `/sys/power/pm_debug_messages` = `1` | Name the aborting wakeup source | **Runtime-only** | ✅ 08-21 | **No value** | `cat /sys/power/pm_debug_messages` → `1` |
| 8 | `/sys/power/pm_test` = `freezer`, `devices` | Cheap hibernate baseline | Reverted | ✅ 08-21 | **Revert** | `cat /sys/power/pm_test` → `[none]` |
| 9 | `/etc/modprobe.d/nvidia-power-management.conf` → `NVreg_PreserveVideoMemoryAllocations=0` | Default `2` routes PM through procfs the 610 open module lacks; resume fails `-5` | Applied | ✅ 08-21 | **Proven** | `grep -iE 'preserve\|suspendnotif' /proc/driver/nvidia/params` → `0`/`1` |
| 10 | `rtc0/wakealarm` armed ×5 | Test RTC wake from s2idle | Reverted | ✅ 08-21 | **Proven** (diagnosis) | `cat /sys/class/rtc/rtc0/wakealarm` → empty |
| 11 | `BAT0/alarm` set to `12345000`, `52647000`, `1000000`, `47869000`, `47869000`; restored | Test whether ACPI `_BTP` can wake s0i3 | Reverted | ✅ 08-21 | **Untested** | `cat /sys/class/power_supply/BAT0/alarm` → `8992000` |

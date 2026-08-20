# Change Log — Zephyrus G15 / CachyOS

**Updated 2026-08-20 22:45.**

**The invariant: if it is not in this table, it is not on the system.**
Everything below is post-reinstall. IDs are flat integers, never reused, never recycled.
Companion files: `system_context.txt`, `claims_corrections_threads.md`.

---

## Applied changes

| ID | Change | State | Verdict | Verify |
|---|---|---|---|---|
| 1 | `sshd` enabled + started (`systemctl enable --now sshd`) | Applied | Proven | `ssh ekj@192.168.1.220` from another host connects |
| 2 | `/etc/sysctl.d/99-sysrq.conf` → `kernel.sysrq=1` | Applied | Proven | `cat /proc/sys/kernel/sysrq` → `1` |
| 3 | `/etc/systemd/journald.conf.d/60-persistent.conf` → `Storage=persistent`, `SystemMaxUse=2G`, `RateLimitBurst=0` | Applied | Proven | `journalctl -b -1 -n 3` returns lines from the previous boot |
| 4 | `ufw allow from 192.168.1.0/24 to any port 22 proto tcp` | Applied | Proven | `sudo ufw status verbose` shows the rule; remote SSH connects |
| 5 | `/swapfile` — 20 GiB, mode 0600, `mkswap -U clear --size 20G --file` | Applied | Proven | `swapon --show` lists `/swapfile` 20 G |
| 6 | `/etc/fstab` → `/swapfile none swap defaults,pri=-2 0 0` | Applied | Proven | after reboot, `swapon --show` still lists `/swapfile` |
| 7 | `/sys/power/pm_debug_messages` = `1` | **Runtime-only** | Neutral | `cat /sys/power/pm_debug_messages` → `1` |
| 8 | `/sys/power/pm_test` = `freezer`, then `devices` | Reverted | Neutral | `cat /sys/power/pm_test` → `[none]` |
| 9 | `/etc/modprobe.d/nvidia-power-management.conf` → `options nvidia NVreg_PreserveVideoMemoryAllocations=0` | Applied | **Proven** | `grep -iE 'preserve\|suspendnotif' /proc/driver/nvidia/params` → `0` and `1` |
| 10 | `/sys/class/rtc/rtc0/wakealarm` — armed 5× by `rtcwake` during testing, cleared each time | Reverted | Neutral | `cat /sys/class/rtc/rtc0/wakealarm` → empty |

---

## Notes per row

**1 — Proven.** Confirmed by an actual remote login, not just `is-active`. This is the
primary escape hatch; two forced power-offs in earlier sessions happened for want of it.

**2 — Proven** at runtime. Trade-off: full sysrq is a physical-access console risk
(unprivileged reboot/kill from a keyboard). Accepted deliberately — it is the difference
between a clean sync-and-reboot and a hard power-off when the NVIDIA driver wedges.
Revert: `sudo rm /etc/sysctl.d/99-sysrq.conf && sudo sysctl --system`

**3 — Proven.** `journalctl -b -1` returned pre-reboot lines after the 21:57 reboot.
Trade-off: up to 2 G of disk for logs. `RateLimitBurst=0` disables rate limiting entirely,
which is what preserved evidence during a log flood in an earlier session.
Revert: `sudo rm /etc/systemd/journald.conf.d/60-persistent.conf && sudo systemctl restart systemd-journald`

**4 — Proven for the effect that matters** (remote SSH works, and `[UFW BLOCK]` lines in
dmesg confirm ufw is actively filtering). ⚠️ **The rule text has never been read back** —
`sudo ufw status verbose` has not been run, so it is unconfirmed whether the rule is
LAN-scoped as intended rather than open to any source. Worth one command.
Revert: `sudo ufw status numbered` then `sudo ufw delete <n>`

**5 — Proven.** `CanHibernate` returns `"challenge"`, meaning logind found the swap and
considers hibernation viable. Effective priority is **−1**, not the −2 written in fstab —
the kernel auto-assigns and does not honour negative `pri=` values. No functional impact:
zram is 100, so ordinary paging stays in RAM and the swapfile remains empty for images.
Revert: `sudo swapoff /swapfile && sudo rm /swapfile` (also remove the fstab line, row 6)

**6 — Proven.** Survived the 21:57 reboot.
Revert: delete the `/swapfile` line from `/etc/fstab`

**7 — ⚠️ RUNTIME-ONLY. Dies silently at every reboot.** It survived the 22:00 hibernate
only because hibernation preserves runtime state. It has produced **no**
`PM: last active wakeup source` output on any sleep path tested so far, so it is not yet
a trustworthy instrument. To persist it:
`echo 'w /sys/power/pm_debug_messages - - - - 1' | sudo tee /etc/tmpfiles.d/pm-debug.conf`
Not done — this is diagnostic scaffolding, not configuration.

**8 — Reverted, and it must stay reverted.** A stale `pm_test` silently converts a real
hibernate into another dry run. **Confirm `[none]` before every real hibernate attempt.**
Tell-tale of a missed reset: `hibernation debug: Waiting for 5 second(s)` in the journal.

**9 — Proven.** Evidence: `params` reads `0`/`1` after reboot, and manual hibernate is
**2/2** with full power-off confirmed by eye, `boot_id` unchanged, SSH session and two
backgrounded `sleep` jobs alive across both cycles.
Caveats, all still open:
  - It is **inferred, not proven, that this file is necessary** on this install. No
    unfixed hibernate was ever run post-reinstall, so we know the current state works —
    not that `2` would have failed. Pre-reinstall sessions and external reports say it
    would.
  - `sudo mkinitcpio -P` alone is **insufficient on Limine**. It warns
    `This does not update Limine boot entries`. `limine-mkinitcpio` did the real work.
  - The same rebuild updated **both** kernels' initramfs, so the LTS kernel also carries
    the fix.
  - One external report claims `limine-update` can reset the parameter to `2`. It did not
    reproduce here, but re-run the verify after every kernel upgrade and every
    `limine-update`.
Revert: `sudo rm /etc/modprobe.d/nvidia-power-management.conf && sudo limine-mkinitcpio`
— ⚠️ **but do not.** Removing the file re-enables the broken default, it does not disable it.

**10 — Reverted.** All test alarms cleared; last read returned empty. Note that a stale
alarm did **not** block suspend on this machine, contradicting an older warning.

---

## Standing hazards (not changes — properties of the system)

- **Two kernels installed** (`linux-cachyos` 7.1.8, `linux-cachyos-lts` 6.18.42).
  Never resume a 7.1.8 hibernation image via the LTS boot entry.
- **`/etc/mkinitcpio.conf.d/10-chwd.conf` is auto-generated.** Do not edit; override.
- **`/boot/limine.conf` is auto-generated.** Edit `/etc/default/limine` and run
  `sudo limine-update` (or `limine-mkinitcpio` after an initramfs change).
- **`/mnt/data` does not exist on this machine.** Diagnostics belong in `~/`.

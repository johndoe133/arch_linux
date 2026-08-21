# Change Log — Zephyrus G15 / CachyOS

**Updated 2026-08-20 23:10.** Restructured: `Why` and `Verified` added, `Verdict`
redefined as a judgement of *value*, not of *existence*.

**The invariant: if it is not in this table, it is not on the system.**
IDs are flat integers, never reused, never recycled.
Companion files: `system_context.txt`, `claims_corrections_threads.md`.

---

## Column definitions

| Column | Meaning |
|---|---|
| **Why** | The problem this was meant to solve. If this is empty or vague, the change should not exist. |
| **State** | `Applied` (persists across reboot) · `Runtime-only` (dies at reboot) · `Reverted` |
| **Verified** | Is it *actually in effect*? A mechanical check, not a judgement. `✅` = confirmed by reading the effect, with date. `⚠️` = partially confirmed. `❌` = never checked. |
| **Verdict** | Does it *earn its place*? See vocabulary below. |
| **Verify** | The one-line command that re-confirms `Verified`. Reads the effect, never the config file. |

### Verdict vocabulary

| Verdict | Meaning |
|---|---|
| **Proven** | A measured or observed benefit, with the evidence cited. The change did something good and we watched it happen. |
| **Necessary** | A proven capability breaks without it. Weaker than Proven if the dependency is inferred rather than tested. |
| **Insurance** | No benefit unless something fails. Cannot be proven until it is needed. Legitimate, but must be labelled honestly — insurance is not evidence. |
| **No value** | Did what it was told, bought nothing. Candidate for removal. |
| **Revert** | Cost more than it returned. Row retained so it is not rediscovered and retried. |

⚠️ **Nothing currently on this system has a measured *power* benefit.** The one change
that produced a real drain improvement pre-reinstall (`NVreg_EnableS0ixPowerManagement=1`,
0.81 W → 0.58 W) is **not applied**. See open thread 4.

---

## Applied changes

| ID | Change | Why | State | Verified | Verdict | Verify |
|---|---|---|---|---|---|---|
| 1 | `sshd` enabled + started | Remote shell for when the desktop wedges; two forced power-offs in earlier sessions happened for want of one | Applied | ✅ 08-20 | **Proven** | `ssh ekj@192.168.1.220` from another host |
| 2 | `/etc/sysctl.d/99-sysrq.conf` → `kernel.sysrq=1` | REISUB instead of a hard power-off when the NVIDIA driver wedges | Applied | ✅ 08-20 | **Insurance** | `cat /proc/sys/kernel/sysrq` → `1` |
| 3 | `/etc/systemd/journald.conf.d/60-persistent.conf` → persistent, 2 G, `RateLimitBurst=0` | Kernel logs must survive a crash or rollback; a log flood destroyed evidence in an earlier session | Applied | ✅ 08-20 | **Insurance** | `journalctl -b -1 -n 3` returns lines |
| 4 | `ufw allow from 192.168.1.0/24 to any port 22 proto tcp` | ufw defaults to deny-incoming; without this, change 1 is inert | Applied | ⚠️ 08-20 | **Necessary** | `sudo ufw status verbose` |
| 5 | `/swapfile` — 20 GiB, mode 0600, `mkswap -U clear --size 20G --file` | Hibernation needs a non-zram swap ≥ image size; the install shipped with zram only | Applied | ✅ 08-20 | **Necessary** | `swapon --show` lists `/swapfile` 20 G |
| 6 | `/etc/fstab` → `/swapfile none swap defaults,pri=-2 0 0` | Change 5 would vanish at reboot without it | Applied | ✅ 08-20 | **Necessary** | after reboot, `swapon --show` still lists it |
| 7 | `/sys/power/pm_debug_messages` = `1` | Make the kernel print `PM: last active wakeup source: X` on abort paths | **Runtime-only** | ✅ 08-20 | **No value** | `cat /sys/power/pm_debug_messages` → `1` |
| 8 | `/sys/power/pm_test` = `freezer`, then `devices` | Intended as a cheap baseline for the hibernate failure | Reverted | ✅ 08-20 | **Revert** | `cat /sys/power/pm_test` → `[none]` |
| 9 | `/etc/modprobe.d/nvidia-power-management.conf` → `options nvidia NVreg_PreserveVideoMemoryAllocations=0` | Default is `2`, which routes PM through a procfs interface the 610 open module doesn't expose; hibernate then fails on the resume leg with `-5` | Applied | ✅ 08-20 | **Proven** | `grep -iE 'preserve\|suspendnotif' /proc/driver/nvidia/params` → `0` / `1` |
| 10 | `/sys/class/rtc/rtc0/wakealarm` — armed 5× by `rtcwake`, cleared each time | Test whether RTC can wake the machine from s2idle | Reverted | ✅ 08-20 | **Proven** (as diagnosis) | `cat /sys/class/rtc/rtc0/wakealarm` → empty |

---

## Notes per row

**1 — Proven, and it earned it twice over.** Not merely insurance: every hibernate test was
driven over SSH, and **SSH session survival became the primary evidence** that the 2/2
hibernates were genuine resumes rather than cold boots. TCP state lives in the hibernation
image, so a cold boot kills the session and a resume does not. It was the instrument, not
just the safety net.
Revert: `sudo systemctl disable --now sshd`

**2 — Insurance, never exercised.** No wedge has occurred post-reinstall, so the benefit is
unproven by construction. Trade-off: full sysrq is a physical-access console risk
(unprivileged reboot/kill from the keyboard). Accepted deliberately.
Revert: `sudo rm /etc/sysctl.d/99-sysrq.conf && sudo sysctl --system`

**3 — Insurance, marginally exercised.** `journalctl -b -1` was used, but only to verify the
setting itself; every substantive diagnosis so far came from the *current* boot, which
works without persistence. Its value arrives the first time something dies mid-write.
Trade-off: up to 2 G of disk. `RateLimitBurst=0` disables rate limiting entirely.
Revert: `sudo rm /etc/systemd/journald.conf.d/60-persistent.conf && sudo systemctl restart systemd-journald`

**4 — Necessary, ⚠️ only partially verified.** Remote SSH works and `[UFW BLOCK]` lines in
dmesg confirm ufw is actively filtering, so the *effect* is confirmed. But
`sudo ufw status verbose` has never been run, so **it is unconfirmed whether the rule is
LAN-scoped as intended rather than open to any source.** One command closes this.
Revert: `sudo ufw status numbered` then `sudo ufw delete <n>`

**5 — Necessary, and the dependency is proven.** `CanHibernate` returns `"challenge"`,
meaning logind found the swap and considers hibernation viable; hibernate is 2/2 on top of
it. Note the effective priority is **−1**, not the −2 written in fstab — the kernel
auto-assigns and does not honour negative `pri=`. No functional impact: zram is priority
100, so ordinary paging stays in RAM and the swapfile remains empty for images.
Revert: `sudo swapoff /swapfile && sudo rm /swapfile` (also remove the fstab line, row 6)

**6 — Necessary.** Survived the 21:57 reboot.
Revert: delete the `/swapfile` line from `/etc/fstab`

**7 — ⚠️ RUNTIME-ONLY, and it bought nothing.** It has produced **zero**
`PM: last active wakeup source` output on every sleep path tested — s2idle ×4, hibernate ×2.
Either it is not emitted on these paths or the abort conditions never arose. Either way it
is not yet a trustworthy instrument, so do not reason from its silence. It also **dies at
the next reboot**; it survived the 22:00 hibernate only because hibernation preserves
runtime state. To persist:
`echo 'w /sys/power/pm_debug_messages - - - - 1' | sudo tee /etc/tmpfiles.d/pm-debug.conf`
Not done, and not recommended — re-set it by hand only when a specific abort needs naming.

**8 — Reverted, and the verdict is Revert on the whole approach, not just the setting.**
It cost two sleep cycles and produced one trivially-true result (processes can be frozen).
Per Correction 14, `pm_test` runs entirely inside the booted system and never reaches an
initramfs, so it was **structurally incapable** of detecting the resume-leg failure it was
chosen to detect — a clean pass would have been a false negative. Do not retry this
approach for resume-leg bugs.
⚠️ Independent of the verdict: a stale `pm_test` silently converts a real hibernate into
another dry run. **Confirm `[none]` before every real hibernate.** Tell-tale of a missed
reset: `hibernation debug: Waiting for 5 second(s)` in the journal.

**9 — Proven beneficial; necessity is inferred, not proven.** Evidence for the benefit:
`params` reads `0`/`1` after reboot, and manual hibernate is **2/2** with full power-off
witnessed, `boot_id` unchanged, SSH session and two backgrounded `sleep` jobs alive across
both cycles, Firefox restored without reloading, Kate retaining an unsaved buffer.
What is *not* proven: that `2` would have failed **on this install** — no unfixed hibernate
was ever run post-reinstall. Pre-reinstall sessions and external reports say it would.
Operational caveats:
  - `sudo mkinitcpio -P` alone is **insufficient on Limine** — it warns
    `This does not update Limine boot entries`. `limine-mkinitcpio` did the real work.
  - The rebuild updated **both** kernels' initramfs, so the LTS kernel carries the fix too.
  - One external report claims `limine-update` can reset the parameter to `2`. It did not
    reproduce here, but re-run the verify after every kernel upgrade and `limine-update`.
Revert: `sudo rm /etc/modprobe.d/nvidia-power-management.conf && sudo limine-mkinitcpio`
— ⚠️ **but do not.** Removing the file re-enables the broken default; it does not disable it.

**10 — Reverted; verdict Proven as *diagnosis*, no lasting change.** The five armed alarms
produced the decisive finding that RTC wake is dead on this hardware: the alarm arms and
fires correctly while awake, but 0/2 in s2idle. That single result eliminated every timed
suspend→hibernate design and redirected the whole project. Incidental: a stale alarm did
**not** block suspend here, contradicting an older warning.

---

## Standing hazards (not changes — properties of the system)

- **Two kernels installed** (`linux-cachyos` 7.1.8, `linux-cachyos-lts` 6.18.42).
  Never resume a 7.1.8 hibernation image via the LTS boot entry.
- **`/etc/mkinitcpio.conf.d/10-chwd.conf` is auto-generated** by `chwd`. Do not edit; override.
- **`/boot/limine.conf` is auto-generated.** Edit `/etc/default/limine`, then
  `sudo limine-update` (or `limine-mkinitcpio` after an initramfs change).
- **`/mnt/data` does not exist on this machine.** Diagnostics belong in `~/`.

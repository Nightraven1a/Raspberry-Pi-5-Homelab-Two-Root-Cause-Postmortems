# Raspberry-Pi-5-Homelab-Two-Root-Cause-Postmortems
Wifi problems and login 
# Raspberry Pi 5 Homelab: Two Root-Cause Postmortems

**System:** Raspberry Pi 5, Kali Linux ARM64 (kernel `6.12.34+rpt-rpi-v8`)
**Scope:** Two unrelated, independently diagnosed failures on the same box within one week — documented together because both hinge on the same investigative discipline: verify before you diagnose, and don't let a "close enough" test stand in for the real one.

---

## Incident 1 — Wi-Fi Failure After `apt upgrade`

**Symptom:** Wi-fi unreliable/dropping after routine `apt upgrade`, recurring across multiple update cycles.
**Outcome:** Three independent, compounding issues — not a single driver bug.

### Environment
- Dual radios: onboard Broadcom (`wlan0`, `brcmfmac-nexmon` driver) + USB RTL8188EU adapter (`wlan1`, monitor-mode capable)
- NetworkManager-managed networking
- Kernel pinned to `kernel8.img` in `config.txt` for an unrelated native ARM64 Blender build (4K vs. 16K page-size fix)

### Investigation Timeline

**1. Initial triage — ruled out the obvious.**
`uname -r` vs. `/lib/modules/` confirmed no kernel/module mismatch. `lsmod` showed `brcmfmac` loaded cleanly. `rfkill list` showed no soft/hard block. `ip link` showed `wlan0` present and `DORMANT` (not crashed). Conclusion: not a driver or kernel-load failure — something more specific was happening at the NetworkManager layer.

**2. Found duplicate connection profiles.**
`nmcli connection show` revealed five separate saved profiles for the same SSID (`SpectrumSetup-5C` through `SpectrumSetup-5C 4`), all unbound to any device. NetworkManager was cloning a new profile on every failed reconnect instead of reusing the existing one. Deleting the duplicates and reactivating the base profile restored connectivity — temporarily.

**3. Duplicates came back with zero updates applied.**
Re-ran the deliberate update ritual; `apt` reported nothing changed. Yet a new duplicate profile appeared anyway, now bound to `wlan1` instead of `wlan0`. This ruled out "update" as the root trigger. Actual cause: **the saved profile had no `interface-name` binding**, so whichever radio initiated a connection attempt first could claim it, and any mismatch triggered a clone instead of a reuse.

**4. Misidentified firmware package.**
Initial hardening attempt held `firmware-brcm80211` — the standard Debian package name. It didn't exist on this system. Kali uses a monitor-mode-patched stack instead: `firmware-nexmon` + `brcmfmac-nexmon-dkms`. DKMS-built modules are more fragile across kernel updates than static firmware, since they rebuild against whatever kernel headers are currently installed. Corrected the hold to the actual installed packages.

### Root Cause
1. **No device binding on the wifi profile** — allowed either radio to claim the same SSID profile, triggering clones on mismatch.
2. **Kernel/DKMS firmware version drift** — `apt upgrade` could move the kernel and the DKMS-built module out of sync with no forced coupling.
3. **Wrong package held in the first remediation attempt** — corrected after verifying the actual installed package name.

The `apt upgrade` events weren't the root cause — they were the trigger that exposed a pre-existing binding gap.

### Permanent Fixes Implemented
```bash
# 1. Pin the wifi profile to its actual radio — eliminates binding ambiguity
sudo nmcli connection modify "SpectrumSetup-5C" connection.interface-name wlan0
sudo nmcli connection modify "SpectrumSetup-5C" connection.autoconnect yes
sudo nmcli connection modify "SpectrumSetup-5C" connection.autoconnect-priority 10

# 2. Remove the monitor-capable adapter from NetworkManager entirely
sudo nmcli device set wlan1 managed no

# 3. Lock kernel + the correct DKMS firmware stack together
sudo apt-mark hold linux-image-rpi-v8 firmware-nexmon brcmfmac-nexmon-dkms

# 4. Boot-time + event-driven watchdog as a backstop
#    (systemd oneshot service + NetworkManager dispatcher hook)
#    — deletes any duplicate SSID profile, keeps the most recently modified
```
Full installer script: [`wifi-update-safety.sh`](./wifi-update-safety.sh)

### Deliberate Update Ritual
```bash
sudo apt-mark unhold linux-image-rpi-v8 firmware-nexmon brcmfmac-nexmon-dkms
sudo apt update && sudo apt install --only-upgrade linux-image-rpi-v8 firmware-nexmon brcmfmac-nexmon-dkms
sudo reboot
dkms status | grep brcmfmac   # confirm module built for the new running kernel
nmcli device status
ping -c 3 8.8.8.8
sudo apt-mark hold linux-image-rpi-v8 firmware-nexmon brcmfmac-nexmon-dkms
```

### Verification
Post-fix, `wlan0` connects cleanly with 0% packet loss, `wlan1` sits `unmanaged`, and no duplicate profiles regenerate across reconnects or update cycles.

---

## Incident 2 — Boot Freeze Requiring Multiple Power Cycles

**Symptom:** Boot text displays, then freezes. Requires 2-3 full power-off/power-on cycles before it succeeds.
**Outcome:** Unstable overclock, no driver or hardware fault.

### Investigation Timeline

**1. Ruled out power delivery.**
`vcgencmd get_throttled` returned `0x0` — clean, no undervoltage event recorded. This eliminated the leading suspect given the peripheral load (M.2 HAT, DSI touchscreen, Arducam, Pi-top 4 shell).

**2. Ruled out NVMe and kernel-pin issues.**
NVMe enumerated in 0.38 seconds with no errors during a successful boot. No boot messages implicated the `kernel8.img` pin — RP1/camera/DSI stack initialized fine in every successful boot.

**3. Found the actual cause in `config.txt`:**
```
arm_freq=2800
gpu_freq=900
```
Stock Pi 5 ARM clock is 2400MHz. This was a ~17% overclock with **no paired `over_voltage`** to compensate — the standard recipe for intermittent instability specifically at boot, before thermal/voltage regulation fully settles.

**4. Confirmed via boot-log pattern.**
`journalctl --list-boots` showed over a dozen zero-duration boots scattered across nearly every day of available history — a chronic pattern, not a one-off. The failed boot's kernel log contained only two harmless lines before going silent — consistent with a clock-instability freeze too abrupt for journald to capture a stack trace, unlike a genuine driver panic.

### The Validation Trap
The first several "confirmation" attempts were invalid without realizing it: editing `config.txt` doesn't take effect until the next boot, and `sudo reboot` (warm reboot) doesn't fully re-arm voltage regulators the same way a cold power-on does. Multiple rounds of `vcgencmd`/`journalctl` were run against a still-running session that had never actually reloaded the new config — `uptime` was the tool that exposed this, showing hours of continuous runtime when a fresh boot was assumed.

**The lesson: a warm reboot does not validate a cold-boot bug.** The actual test required three full physical power-cycles (unplug, wait, replug) — not `sudo reboot` — because the failure only ever manifested at true cold start.

### Fix
```bash
sudo nano /boot/firmware/config.txt
# remove or comment out:
#arm_freq=2800
#gpu_freq=900
```

### Verification
Three consecutive cold power-on cycles, confirmed via `journalctl --list-boots`, each showing a clean gap between `First Entry` and `Last Entry` — zero instant-freeze boots. Stock clocks resolved the issue completely.

**If overclocking is revisited later:** never push `arm_freq` without a paired `over_voltage`, increase in small increments, and validate with the same cold-cycle stress test — not a warm reboot.

---

## Cross-Incident Lessons Learned

- **Rule out the lowest layer first, but don't stop there.** Both incidents initially looked like driver/kernel faults (dormant wifi interface, boot freeze). Neither was — one was an application-layer config gap (no device binding), the other was a firmware clock-timing fault. Surface symptoms pointed at the same suspects; the real causes were one layer removed in both directions.
- **Verify assumptions before trusting the retest.** In Incident 1, a package hold was silently ineffective because the package name was guessed rather than confirmed (`dpkg -S`, `modinfo -F firmware`). In Incident 2, several rounds of "confirmation" were run against a stale session that never reloaded the fix. Same failure mode in different clothes: checking the *expected* result instead of confirming the *actual* state (`uptime`, exact installed package names).
- **Match the test to the failure condition.** A fix only counts as verified if the test reproduces the exact condition that triggered the original bug — cold boot vs. warm reboot, real update vs. no-op update, exact package vs. assumed package name.

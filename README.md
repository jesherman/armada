# Armada — Pocket EVO 30 W Charging (fork)

A daily-rebuilt fork of [armada-os/armada](https://github.com/armada-os/armada) that carries
**30 W USB-PD direct charging for the AYANEO Pocket EVO** on top of the latest upstream
`testing` image. Built for the EVO's Snapdragon G3x Gen 2 (SM8550): the stock image charges
at ~17–18 W through the Qualcomm buck; this fork enables the device's dual HL7139 charge
pumps for **27–30 W sustained** (validated on hardware: ~30.5 W at the wall, 4.8–5.2 A into
the battery, pumps ≤ 55 °C).

Both carried changes are open pull requests upstream — [armada#258](https://github.com/armada-os/armada/pull/258)
(userspace charge policy) and [armada-packages#35](https://github.com/armada-os/armada-packages/pull/35)
(kernel: HL7139 driver + battery-manager ICL/PPS setters + EVO DTS). This fork exists so
EVO owners get the feature **now**, rebased daily, until the PRs merge.

## What's carried on top of upstream testing

- **HL7139 charge-pump driver** (kernel, in-tree) + battery-manager input-current-limit and
  PPS voltage setters + EVO device-tree nodes (from armada-packages#35)
- **Opt-in direct-charge policy** (`armada-pocketevo-charge-policy`): engages the pumps when
  a PD-PPS adapter is present, ramps to the 3 A contract ceiling, fail-closed guards
  (combined-input current, VBUS coherence, pump health) with latched faults cleared by
  cable-cycle (from armada#258)
- **`armada-charge-toggle`**: `status` / `enable` / `disable` — the opt-in switch
- **Fork-aware update hook**: the Steam "Check for Updates" flow tracks *this* repo's
  `testing` tag, so the banner reflects fork publishes, not upstream's

## Install on a Pocket EVO (from scratch or from upstream Armada)

The device is standard Armada (Fedora bootc + ostree). Any of these paths work:

### A. One command (GHCR, anonymous — recommended)

```bash
sudo bootc switch --transport registry ghcr.io/jesherman/armada:testing
sudo systemctl reboot
```

First pull is ~6 GB; subsequent updates transfer only changed layers.
Rollback: `sudo bootc rollback` (the previous upstream deployment is preserved
automatically).

### B. From the release archive (no registry)

Grab the split OCI archive from
[releases/tag/fork-image](https://github.com/jesherman/armada/releases/tag/fork-image):

```bash
cat fork-image-oci.tar.part-* > fork-image-oci.tar
sha256sum -c fork-image-oci.tar.sha256
tar -xf fork-image-oci.tar
skopeo copy oci:fork-oci:testing containers-storage:ghcr.io/jesherman/armada:testing
sudo bootc switch --transport containers-storage ghcr.io/jesherman/armada:testing
```

### C. Kernel-only (you run upstream Armada and just want charging)

[releases/tag/fork-kernel](https://github.com/jesherman/armada-packages/releases/tag/fork-kernel)
ships `armada-kernel-<ver>.tar.zst` (+ sha256). Install via the standard Armada hotfix
path (`ostree admin unlock --hotfix` → extract to `/` → `depmod` → dracut with the armada
includes → copy vmlinuz/dtbs into the deployment tree → `armada-bootimg-update`). Keep the
original versioned filename.

## Enabling 30 W charging

Charging is **opt-in** by design (direct charge bypasses the PMIC-managed path, so it's a
user decision):

```bash
armada-charge-toggle enable   # persists across reboots
```

Then just plug in a USB-PD adapter (30 W+ recommended; 45 W for headroom). The policy
starts automatically at boot and after each plug-in:

- ramps the PPS rail to the 3 A contract (~10.5 V), engages both pumps
- holds 27–30 W until ~90% SOC, then hands off to the buck for top-off
- any guard fault → pumps off, safe buck charging, fault latched; **unplug + replug the
  charger clears a latch**

`armada-charge-toggle status` shows pump health, battery state, opt-in flag, and service
state. Note: direct charge runs while **awake and in s2idle**; deep sleep suspends it
(upstream limitation).

## How updates work

- CI (in `armada-packages`' [`fork-daily.yml`](https://github.com/jesherman/armada-packages/blob/main/.github/workflows/fork-daily.yml))
  runs **every 8 hours**: rebases both carried commits onto the current upstream `main`,
  rebuilds the kernel package and the full image, publishes to `ghcr.io/jesherman/armada:testing`
  and to the rolling Releases, and notifies Discord on failure.
- Your device: `sudo systemctl enable --now armada-fork-update.timer` (optional) or simply
  click **Check for Updates** in Steam — the update hook tracks this fork. Apply = reboot.
- Never ships stale silently: if the daily rebase conflicts upstream, CI alerts and skips
  publishing.

## Building from source

The build is the upstream Armada pipeline. Kernel package: `kernel/build.sh` in
[armada-packages](https://github.com/jesherman/armada-packages) (aarch64, ~40 min on a CI
runner). OS image: the `armada` repo's Containerfile takes `KERNEL_PKG=<fork kernel image>`
as a build arg and produces the bootc image via Chunkah. CI in this repo does both end to end.

## Known issues / notes

- **Charging while asleep**: deep sleep kills the PD stack (ADSP firmware limitation);
  s2idle charges fine. Default sleep mode handles this.
- **Top-off**: direct charge stops at ~90% SOC by design; the buck finishes charging.
- **Watchdog race**: on very cold boots the policy service can lose its watchdog before
  the battery manager finishes initializing — `systemctl restart armada-pocketevo-charge-policy`
  recovers (fix queued in the carry branch).
- The device's own image signature policy accepts this repo (unverified pulls are allowed
  for non-upstream registries by the shipped `policy.json` catch-all).
- **This fork is insurance** — when armada#258 + #35 merge upstream, switch back to
  `ghcr.io/armada-os/armada:testing` and delete this from your device.

## Credits

Upstream [Armada OS](https://github.com/armada-os) by the Armada team (JPyke3 & co).
Charging subsystem reverse-engineering, driver work, and on-device validation: jesherman,
with Claude (Hermes) as pair-engineer. Charge-safety design shaped by reviewer feedback
from the upstream PRs.
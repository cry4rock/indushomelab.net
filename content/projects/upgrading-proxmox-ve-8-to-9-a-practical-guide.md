---
title: "Upgrading Proxmox VE 8 to 9: A Practical Guide"
date: 2026-05-24
draft: false
tags: ["proxmox", "homelab", "linux", "sysadmin"]
categories: ["Homelab"]
description: "A step-by-step walkthrough of upgrading Proxmox VE 8 (Bookworm) to PVE 9 (Trixie), including real-world issues encountered and how to fix them."
---

This guide documents the process of upgrading a standalone Proxmox VE 8 node to Proxmox VE 9, including every issue encountered along the way and how to resolve them. If you're running a homelab setup on a no-subscription licence, this should map closely to your experience.

---

## Step 1: Run the Pre-Upgrade Checker

Proxmox ships a built-in checker that validates your system before the upgrade. Always start here:

```bash
pve8to9
```

Work through any `FAIL` or `WARN` items one at a time, re-running the checker after each fix until you get a clean result.

---

## Step 2: Fix the `systemd-boot` Blocker (FAIL)

The checker may flag:

```text
FAIL: systemd-boot meta-package installed. This will cause problems on upgrades of other boot-related packages.
```

This is a hard blocker. Remove it:

```bash
apt remove systemd-boot
```

> See the [Proxmox wiki](https://pve.proxmox.com/wiki/Upgrade_from_8_to_9#sd-boot-warning) for background on why this conflicts with the upgrade.

---

## Step 3: Install `intel-microcode` (WARNING)

If the checker warns that the `intel-microcode` package is missing, it's because the `non-free-firmware` component isn't enabled in your apt sources.

**Enable the component:**

```bash
nano /etc/apt/sources.list
```

Find your Debian bookworm line and append `non-free-firmware`:

```text
deb http://deb.debian.org/debian bookworm main contrib non-free-firmware
```

**Then install the package:**

```bash
apt update
apt install intel-microcode
```

---

## Step 4: Disable LVM Autoactivation (NOTICE)

The checker may flag that some LVM/LVM-thin volumes still have autoactivation enabled. Starting with PVE 9, autoactivation is disabled for new volumes. Run the migration script to bring existing volumes in line:

```bash
/usr/share/pve-manager/migrations/pve-lvm-disable-autoactivation
```

---

## Step 5: Update Repositories to PVE 9 (Trixie)

This is the most common reason the upgrade fails mid-way — if the PVE 9 repository isn't configured, apt will try to *remove* `proxmox-ve` instead of upgrading it, and the Proxmox apt hook will block the operation with a warning.

Update all sources from `bookworm` to `trixie`:

```bash
# Main Debian sources
sed -i 's/bookworm/trixie/g' /etc/apt/sources.list

# Proxmox no-subscription repo
sed -i 's/bookworm/trixie/g' /etc/apt/sources.list.d/pve-no-subscription.list

# Enterprise repo (if present) — disable if you don't have a subscription
sed -i 's/^deb/# deb/' /etc/apt/sources.list.d/pve-enterprise.list
```

Verify the no-subscription file looks correct:

```bash
cat /etc/apt/sources.list.d/pve-no-subscription.list
# Expected: deb http://download.proxmox.com/debian/pve trixie pve-no-subscription
```

---

## Step 6: Run the Upgrade

```bash
apt update
apt full-upgrade
```

> Use `apt full-upgrade` — not `apt upgrade`. The full upgrade handles package replacements required by PVE 9.

Once complete, reboot:

```bash
reboot
```

---

## Step 7: Post-Upgrade Repository Fixes

After rebooting, run `pve8to9` again. You may hit two additional issues:

### Mixed repository suites (FAIL)

```text
FAIL: Found mixed old and new package repository suites, fix before upgrading!
  found suite bookworm at /etc/apt/sources.list.d/ceph.list:1
```

The Ceph repo wasn't updated. But there's an additional catch — PVE 9 uses **Ceph Squid**, not Ceph Quincy. Update the file entirely:

```bash
echo "deb http://download.proxmox.com/debian/ceph-squid trixie no-subscription" > /etc/apt/sources.list.d/ceph.list
```

### Enterprise repo 401 Unauthorized

If the enterprise repo is still enabled without a valid subscription:

```bash
sed -i 's/^deb/# deb/' /etc/apt/sources.list.d/pve-enterprise.list
```

Then refresh:

```bash
apt update
```

---

## Step 8: Clean Up Old RRD Files (Optional)

The checker may note leftover RRD metric files in the old format:

```text
INFO: Found '7' RRD files using the old format.
```

These are only used for displaying historical data. If you don't need graphs from before the upgrade, remove them:

```bash
find /var/lib/rrdcached/db/pve2-* -name "*.rrd" -delete
```

---

## Verifying a Successful Upgrade

Run the checker one final time:

```bash
pve8to9
```

A successful upgrade looks like:

```text
PASS: already upgraded to Proxmox VE 9
PASS: running new kernel '7.0.2-6-pve' after upgrade.
```

And confirm the version:

```bash
pveversion
```

---

## Summary of Changes Made

| Issue | Fix |
|---|---|
| `systemd-boot` blocking upgrade | `apt remove systemd-boot` |
| `intel-microcode` not found | Enable `non-free-firmware`, then `apt install intel-microcode` |
| LVM autoactivation notice | Run PVE migration script |
| `proxmox-ve` being removed during upgrade | Update all repos from `bookworm` → `trixie` |
| Ceph repo 404 | Switch from `ceph-quincy` to `ceph-squid` for trixie |
| Enterprise repo 401 | Comment out enterprise repo line |
| Mixed repo suites (post-upgrade) | Update `ceph.list` to trixie |

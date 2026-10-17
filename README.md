# Homelab Runbook

A rebuild-from-scratch record for my home server. If this machine dies, these notes should get a replacement running with no outside help.

**Rule for every entry:** what I did, the exact command or setting, why, and how I verified it.
**Secrets never go in this repo.** Passwords, keys, and tokens live in my password manager.

---

## 0. Hardware

| Item | Value |
|---|---|
| Machine | Dell Latitude E6530 |
| CPU | Intel Core i5-3210M (2 cores / 4 threads, 2.5 GHz, 3.1 GHz turbo) |
| RAM | 8 GB DDR3 |
| Storage | 500 GB HDD |
| Network | Wired Ethernet (gigabit onboard NIC) |
| Battery | Original, checked for swelling: none (2026-10-16). Kept in as built-in UPS. |
| Service tag | _(fill in, bottom of laptop)_ |

**Known weak point:** HDD. Slow for Minecraft chunk loading, higher failure risk running 24/7. Replace with SSD when prices allow.

---

## 1. BIOS

**BIOS version:** _(fill in: F2 → General → System Information)_
**Updated from:** _(old version, or "not updated")_

| Menu | Setting | Value | Why |
|---|---|---|---|
| General → Boot Sequence | Boot List Option | UEFI | Modern boot mode, Ubuntu supports it |
| General | Secure Boot | _(fill in: on/off/not present)_ | |
| System Configuration | SATA Operation | AHCI | Standard mode for Linux |
| System Configuration | Integrated NIC | Enabled | Server runs wired |
| Power Management | AC Behavior | Wake on AC | Auto power-on after an outage |
| Power Management | Primary Battery Charge | Primarily AC Use | Battery doesn't sit at 100% 24/7 (swelling risk) |
| Power Management | Wake on LAN | LAN Only | Remote power-on later |
| Virtualization Support | Virtualization | Enabled | Needed for VMs later |
| Virtualization Support | VT for Direct I/O | Enabled | Same |
| Wireless | WLAN / Bluetooth | Disabled | Wired only, removes unused attack surface |
| Security | Admin password | Set (in password manager) | Blocks boot-order changes by anyone with physical access |

**Verified:** _(e.g. rebooted, settings persisted)_

---

## 2. OS install

**OS:** Ubuntu Server 24.04 LTS
**Install media made with:** _(fill in)_

Steps:
1. F12 at Dell logo → boot from USB
2. _(continue as you go: storage layout, ubuntu-lv resized to full disk, hostname, OpenSSH + GitHub key import, snaps skipped)_

**Verify:**
```bash
lsblk
df -h
```
_(paste output here)_

---

## 3. Post-install

_(lid switch config, DHCP reservation, first updates)_

---

## 4. Hardening

_(next session)_

---

## Troubleshooting log

### 2026-10-17: New domains won't resolve on home network
**Symptom:** github.com and other new sites wouldn't load. Recently visited sites still worked (cached DNS).
**Diagnosis:**
- `curl -Iv https://github.com` → `Could not resolve host` (DNS failure, connection never started)
- `dig github.com` (router, 192.168.1.1) → timed out, A and AAAA
- `dig @1.1.1.1 github.com` → timed out (DNS dropped before leaving the house)
- `ping 192.168.1.1`, `ping 1.1.1.1`, `curl -I https://1.1.1.1` → all worked (internet up, only DNS broken)
- Phone on home Wi-Fi failed too → network-wide, not the Mac
**Cause:** Router DNS service hung (suspected). Not confirmed against ISP.
**Fix:** Power-cycled modem, then router.
**If it happens again:** Recurrence points at the router. Check firmware version, set manual DNS (1.1.1.1 / 9.9.9.9), and test by bypassing the router to rule out the ISP.

---

## Changelog

| Date | Change |
|---|---|
| 2026-10-16 | Hardware checked, BIOS configured |

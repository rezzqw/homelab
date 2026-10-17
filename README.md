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
| RAM | 8 GB DDR3 (7.2 GiB usable, integrated graphics reserves the rest) |
| Storage | 500 GB HDD (465.8 GiB) |
| Network | Onboard gigabit Ethernet (`eno1`), wired to router |
| Battery | Original, checked for swelling: none (2026-10-16). Kept in as built-in UPS. |
| Placement | Lid closed, hard surface, side vents unobstructed |
 
**Known weak point:** HDD. Slow for Minecraft chunk loading, higher failure risk running 24/7. Replace with SSD when prices allow.
 
**USB ports:** Boot from the **black USB 2.0 port**. The blue USB 3.0 ports fail during the Ubuntu installer (see Troubleshooting log).
 
---
 
## 1. BIOS
 
**BIOS revision:** A16
**Update:** Skipped. Newer revisions exist, but Windows was the only easy way to run Dell's updater. Accepted risk for a home server behind a router.
 
| Menu | Setting | Value | Why |
|---|---|---|---|
| General → Boot Sequence | Boot List Option | UEFI | Modern boot mode, Ubuntu supports it |
| General | Secure Boot | Off | |
| System Configuration | SATA Operation | AHCI | Standard mode for Linux |
| System Configuration | Integrated NIC | Enabled | Server runs wired |
| Power Management | AC Behavior | Wake on AC | Auto power-on after an outage |
| Power Management | Primary Battery Charge | Primarily AC Use | Battery doesn't sit at 100% 24/7 (swelling risk) |
| Power Management | Wake on LAN | LAN Only | Remote power-on later |
| Virtualization Support | Virtualization | Enabled | Needed for VMs later |
| Virtualization Support | VT for Direct I/O | Enabled | Same |
| Wireless | WLAN / Bluetooth | Disabled | Wired only, removes unused attack surface |
| Security | Admin password | Set (in password manager) | Blocks boot-order changes by anyone with physical access |
 
**Verified:** F12 boot menu shows "Boot mode is set to: UEFI; Secure Boot: OFF", BIOS Revision A16.
 
---
 
## 2. OS install
 
**OS:** Ubuntu Server 26.04.1 LTS (support through 2031)
**ISO:** `ubuntu-26.04.1-live-server-amd64.iso` (renamed to `ubuntu.iso` locally)
**ISO verified:** yes: `shasum -a 256` matched SHA256SUMS at releases.ubuntu.com/26.04/)_
 
### Making the install USB (on Mac)
balenaEtcher failed (`requestMetadata is not a function` bug). Flashed with `dd` instead:
```bash
diskutil list external              # USB was /dev/disk4, 16.1 GB, confirm before writing
diskutil unmountDisk /dev/disk4
sudo dd if=$HOME/Downloads/ubuntu.iso of=/dev/rdisk4 bs=4m status=progress
diskutil eject /dev/disk4
```
**Warning:** `dd` to the wrong disk wipes it. Always confirm the disk number with `diskutil list external` first.
 
### Installer choices
1. F12 at Dell logo → **UEFI: General UDisk** (not the Legacy USB entry), USB in black USB 2.0 port
2. Install type: **Ubuntu Server** (not minimized), no third-party drivers
3. Network: `eno1`, DHCPv4, got `192.168.1.68/24`
4. Proxy: none. Mirror: default
5. Storage: entire disk, **LVM on, encryption off**
   - Why no encryption: an encrypted disk needs a passphrase at every boot, which breaks Wake on AC recovery after an outage. Tradeoff: anyone who steals the laptop can read the drive.
   - **Edited `ubuntu-lv` to use the full disk** (installer default leaves most of it unallocated)
6. Profile: hostname `homelab`, user `rezzqw`, password in password manager
7. Ubuntu Pro: skipped
8. SSH: OpenSSH server installed, key imported from GitHub (`rezzqw`), **password authentication disabled**
9. Featured snaps: none
### Verification
```
$ lsblk
sda                         465.8G disk
├─sda1                          1G part /boot/efi
├─sda2                          2G part /boot
└─sda3                      462.7G part
  └─ubuntu--vg-ubuntu--lv   462.7G lvm  /
 
$ df -h /
/dev/mapper/ubuntu--vg-ubuntu--lv  455G  7.4G  424G   2% /
```
- Root LV uses the full disk
- `/boot/efi` present = UEFI install confirmed
- `/tmp` is tmpfs (RAM-backed) by default on 26.04
---
 
## 3. Post-install
 
### Admin workstation (Mac)
- SSH key: `~/.ssh/id_ed25519`, public key added to GitHub
- First SSH to server: **host key fingerprint verified** before accepting
```bash
  # On the server's own screen:
  ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
  # On the Mac, compare against:
  ssh-keygen -lF 192.168.1.68
```
- Connect: `ssh rezzqw@192.168.1.68`
### Updates
```bash
sudo apt update && sudo apt full-upgrade -y
```
 
### Keep running with lid closed
Drop-in file instead of editing the main config, so package updates can't overwrite it.
```bash
sudo mkdir -p /etc/systemd/logind.conf.d
printf '[Login]\nHandleLidSwitch=ignore\nHandleLidSwitchExternalPower=ignore\nHandleLidSwitchDocked=ignore\n' | sudo tee /etc/systemd/logind.conf.d/lid.conf
```
**Verify:**
```bash
systemd-analyze cat-config systemd/logind.conf | grep -i lidswitch
```
Then rebooted, closed the lid, waited 2 minutes, SSH'd in, `uptime` confirmed no suspend. **Passed.**
 
### Timezone
```bash
sudo timedatectl set-timezone America/Los_Angeles
```
Why: log timestamps need to be in local time for log analysis.
 
### Memory and swap
```
$ free -h
Mem:   7.2Gi total
Swap:  4.0Gi total
$ swapon --show
/swap.img  file  4G
```
4 GB swap file created by the installer. To do: lower swappiness so the HDD isn't used for swap unless needed.
 
### Static IP via DHCP reservation
Reserved **192.168.1.68** for the server's MAC address (`ip link show eno1` → `link/ether`) in the router admin page (192.168.1.1).
Why: the server's address must never change, or SSH and every service breaks.
 
---
 
## 4. Hardening
 
_(next session: ufw firewall, fail2ban, unattended-upgrades, sshd lockdown, swappiness)_
 
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
### 2026-10-17: `ssh -T git@github.com` appeared to hang
**Cause:** Not a hang. SSH was waiting at the first-connection host key prompt.
**Fix:** Verified the fingerprint against GitHub's published Ed25519 fingerprint (docs.github.com, "GitHub's SSH key fingerprints"), then accepted.
**Lesson:** A silent SSH session is usually waiting for input. Read the screen before assuming a network problem.
 
### 2026-10-17: balenaEtcher "Error opening source"
**Symptom:** `Error: (0 , h.requestMetadata) is not a function` when selecting the ISO.
**Cause:** Known Etcher bug (GitHub issue #4150, many forum reports).
**Fix:** Flashed with `dd` (see Section 2).
 
### 2026-10-17: Installer "unable to find a medium containing a live file system"
**Symptom:** Boot started, then repeated `/dev/sr0: no medium found` and a netboot prompt.
**Notes:** `sr0` is the empty DVD drive, a red herring. The real error was the installer losing the USB stick.
**Fix:** Answered No to netboot, powered off, moved USB from blue USB 3.0 port to **black USB 2.0 port**. Booted fine.
 
---
 
## Changelog
 
| Date | Change |
|---|---|
| 2026-10-16 | Hardware checked, BIOS configured |
| 2026-10-17 | Ubuntu Server 26.04.1 installed, SSH key-only access, lid/timezone/DHCP reservation configured |
 
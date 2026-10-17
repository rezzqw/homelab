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
**ISO verified:** _(yes/no: `shasum -a 256` matched SHA256SUMS at releases.ubuntu.com/26.04/)_

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
4 GB swap file created by the installer. Swappiness lowered to 10 (see Section 4, Step 5).

### Static IP via DHCP reservation
Reserved **192.168.1.68** for the server's MAC address (`ip link show eno1` → `link/ether`) in the router admin page (192.168.1.1).
Why: the server's address must never change, or SSH and every service breaks.

---

## 4. Hardening

**Rule for firewall and SSH changes:** keep the current SSH session open and test in a second window. Fallback access is the Dell's own keyboard and screen.

**Config pattern used throughout:** drop-in files instead of editing main config files, so package updates can't overwrite changes. Watch the precedence rule for each tool:
- **sshd:** first value read wins (lower number wins)
- **apt:** last value read wins (higher number wins)

### Step 1: Firewall (ufw)
`ufw` is a front end for the kernel firewall (nftables). Policy: deny all incoming by default, allow only what's needed.
```bash
sudo ufw status verbose                     # look before changing
sudo ufw default deny incoming              # block anything not explicitly allowed
sudo ufw default allow outgoing             # server can reach updates, DNS, time
sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp comment 'SSH from LAN only'
sudo ufw show added                         # dry run: review rules before enabling
sudo ufw enable
sudo ufw status verbose                     # verify end state
```
Why LAN-only SSH: defense in depth. Even if a port gets exposed by mistake, the internet can't reach SSH.

**Expected** _(replace with actual output)_:
```
Status: active
Default: deny (incoming), allow (outgoing), disabled (routed)
22/tcp    ALLOW IN    192.168.1.0/24    # SSH from LAN only
```
**Verified:** new SSH session from Mac in a second window succeeded.

**Notes for later:**
- SSH over Tailscale has its own rule, added in Section 5.
- **Docker bypasses ufw** for published container ports. Must be handled deliberately when Docker is installed.

### Step 2: SSH lockdown
Checked existing drop-ins first (`ls /etc/ssh/sshd_config.d/`, `sudo cat /etc/ssh/sshd_config.d/*.conf`). Named the file `10-` so it sorts before the installer's `50-cloud-init.conf`, because sshd uses the **first** value it reads.

```bash
sudo tee /etc/ssh/sshd_config.d/10-hardening.conf > /dev/null <<'EOF'
# Homelab SSH hardening (runbook Section 4)
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
AllowUsers rezzqw
MaxAuthTries 3
LoginGraceTime 30
X11Forwarding no
EOF
sudo sshd -t                                # syntax check, no output = pass
sudo systemctl restart ssh
```

| Setting | Why |
|---|---|
| PasswordAuthentication no | Keys only, passwords can be guessed |
| KbdInteractiveAuthentication no | Closes the side door passwords can sneak back through |
| PubkeyAuthentication yes | Explicit intent |
| PermitRootLogin no | Log in as a user, use sudo, leaves an audit trail |
| AllowUsers rezzqw | Allowlist, new accounts can't SSH in |
| MaxAuthTries 3 | Disconnect after 3 failures per connection |
| LoginGraceTime 30 | 30s to authenticate instead of 120s |
| X11Forwarding no | No GUI, unused feature = attack surface |

**Verify effective config** (what actually wins, not what was written):
```bash
sudo sshd -T | grep -Ei 'passwordauthentication|kbdinteractive|permitrootlogin|allowusers|maxauthtries|logingracetime|x11forwarding'
```
_(paste before and after output)_

**Test 1 (good path):** new key-based SSH session from Mac. **Passed.**
**Test 2 (bad path):** forced password attempt, refused with `Permission denied (publickey)`, never prompted for a password. **Passed.**
```bash
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password rezzqw@192.168.1.68
```

### Step 3: Automatic security updates
`unattended-upgrades` installs security updates daily.
```bash
systemctl status unattended-upgrades        # confirm active and enabled
cat /etc/apt/apt.conf.d/20auto-upgrades     # both lines must be "1"
```
Override file (apt uses the **last** value read, so `52-` beats the defaults in `50-`):
```bash
sudo tee /etc/apt/apt.conf.d/52unattended-upgrades-local > /dev/null <<'EOF'
// Homelab overrides (runbook Section 4)
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "04:00";
Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
EOF
```
**Decision: automatic reboots at 4 AM, only when an update requires one.** Kernel patches don't take effect until reboot. Cost: anyone online at 4 AM gets disconnected briefly. Accepted.

**Verify:**
```bash
sudo unattended-upgrade --dry-run --debug   # look for "Allowed origins" with -security, no errors
sudo apt-config dump | grep -i automatic-reboot   # should show "true" and "04:00"
```
**Verified:** all matched.
**Logs:** `/var/log/unattended-upgrades/unattended-upgrades.log`

### Step 4: fail2ban
Watches logs, bans IPs with too many failed logins by adding a firewall rule.

**Why it's here even though SSH is key-only and LAN-only:** the LAN isn't fully trusted (a compromised IoT device is inside the firewall), it matters more once services are exposed, and it's practice with automated intrusion prevention.

```bash
sudo apt install -y fail2ban python3-systemd    # python3-systemd lets it read the journal
```
Config in `jail.local`. **Never edit `jail.conf`**, package updates overwrite it.
```bash
sudo tee /etc/fail2ban/jail.local > /dev/null <<'EOF'
# Homelab fail2ban config (runbook Section 4)
[DEFAULT]
backend = systemd
banaction = ufw
bantime = 1h
bantime.increment = true
findtime = 10m
maxretry = 5

[sshd]
enabled = true
EOF
sudo fail2ban-client -t                     # config test
sudo systemctl enable --now fail2ban
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd            # jail running, 0 banned
```

| Setting | Why |
|---|---|
| backend = systemd | Login events are in the systemd journal on modern Ubuntu |
| banaction = ufw | Bans show up in `ufw status`, one firewall to check |
| bantime = 1h | Ban length |
| bantime.increment = true | Repeat offenders get longer bans |
| findtime = 10m, maxretry = 5 | 5 failures in 10 minutes = ban |

**Self-ban test (bad path), done at the Dell's keyboard:**
```bash
# On Mac: 6 fake login attempts
for i in 1 2 3 4 5 6; do ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password fakeuser@192.168.1.68; done
# On Dell: confirm the ban
sudo fail2ban-client status sshd
sudo ufw status
# On Dell: unban (prints the number of IPs unbanned, so "1" = success)
sudo fail2ban-client set sshd unbanip <Mac-IP>
```
**Result:** Mac IP banned and shown in ufw, unban returned `1`, SSH from Mac restored. **Passed.**
**Note:** ban history is kept, and `bantime.increment` is on, so the Mac's next accidental ban will last longer than 1 hour.

### Step 5: Swappiness
Swap is on a slow HDD. Lowering swappiness from 60 to 10 makes Linux drop file cache before moving program memory to swap. Swap stays on as a safety net.
```bash
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swappiness.conf
sudo sysctl --system                        # apply now without reboot
cat /proc/sys/vm/swappiness                 # live kernel value, should print 10
```
**Verified:** prints `10` after a full reboot.

---

## 5. Remote access (Tailscale)

**What:** a private WireGuard network (tailnet) between my devices. Each device gets a `100.x.y.z` address. Devices connect outward to Tailscale's coordination server, so **no ports are opened on the router.**

**Trust tradeoff:** Tailscale's coordination server can't read traffic, but whoever controls the login controls the tailnet. Signed in with GitHub (`rezzqw`), which has **2FA enabled**. Self-hosted alternative if needed later: Headscale.

### Install (on server)
Downloaded and read the install script before running it, instead of piping it straight into a root shell.
```bash
curl -fsSL https://tailscale.com/install.sh -o tailscale-install.sh
less tailscale-install.sh                   # read before running
sudo sh tailscale-install.sh                # adds Tailscale apt repo + signing key, installs package
sudo tailscale set --auto-update
```
**Why `--auto-update`:** unattended-upgrades only installs from Ubuntu's security repo. Tailscale comes from its own repo and would never update automatically otherwise.

### Connect
```bash
sudo tailscale up                           # prints login URL, approved from Mac
tailscale ip -4                             # server's tailnet IP
tailscale status                            # all devices and their status
```
**Server tailnet IP:** `100.___.___.___` _(fill in)_

**Not using `--ssh` (Tailscale SSH).** It's a separate login system that bypasses the hardened OpenSSH config. One auth system, hardened and tested.

### Key expiry disabled (server only)
Admin console → Machines → `homelab` → Disable key expiry.
**Why:** default re-auth every 180 days would silently drop a headless server off the tailnet while I'm away. Accepted tradeoff for this one machine. Laptop and phone keep expiry on.

### Firewall rule
```bash
sudo ufw allow in on tailscale0 to any port 22 proto tcp comment 'SSH via Tailscale'
sudo ufw status verbose
```
**Why:** Tailscale traffic arrives on the `tailscale0` interface, not `eno1`. The LAN rule doesn't cover it. Only port 22 is allowed on that interface.

**Access paths now (three independent ways in):**
1. LAN: `ssh rezzqw@192.168.1.68`
2. Tailscale, from anywhere: `ssh rezzqw@<100.x IP>`
3. Physical keyboard at the Dell

The LAN rule stays as a fallback in case Tailscale is down.

### Verification
- `tailscale status`: server and Mac online. **Passed.**
- **Off-network test:** Mac disconnected from home Wi-Fi, connected to phone hotspot, SSH to tailnet IP succeeded. Host key fingerprint matched the one verified on LAN. **Passed.**

### Notes for later
- Default tailnet policy lets all my devices reach each other on any port. Fine while it's only my devices. **Must tighten ACLs before sharing anything (like Minecraft) with friends.**

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

### Procedure: Locked out of SSH
1. Go to the Dell, open the lid, log in on its keyboard.
2. Banned by fail2ban? `sudo fail2ban-client status sshd`, then `sudo fail2ban-client set sshd unbanip <IP>`.
3. Blocked by firewall? `sudo ufw status numbered`, check the SSH rule exists and covers your IP.
4. SSH config broken? `sudo sshd -t` to find the error, fix the file in `/etc/ssh/sshd_config.d/`, `sudo systemctl restart ssh`.

---

## Changelog

| Date | Change |
|---|---|
| 2026-10-16 | Hardware checked, BIOS configured |
| 2026-10-17 | Ubuntu Server 26.04.1 installed, SSH key-only access, lid/timezone/DHCP reservation configured |
| 2026-10-17 | Hardening: ufw (LAN-only SSH), sshd lockdown, unattended-upgrades with 4 AM auto-reboot, fail2ban, swappiness 10 |
| 2026-10-17 | Tailscale: remote SSH from anywhere with zero open router ports, key expiry disabled on server |
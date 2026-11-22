# Pi-hole on Raspberry Pi 3 — Build Notes

Last updated: 2026-11-22

## Overview

Pi-hole is live: the Raspberry Pi 3 was reflashed from end-of-life Buster to Raspberry Pi OS Lite (64-bit), the Google Wifi router forwards all household DNS to it, and PADD shows live stats on the rack's 1280×400 screen.

| Item | Value |
| --- | --- |
| Hardware | Raspberry Pi 3, wired Ethernet, 1280×400 screen |
| OS | Raspberry Pi OS Lite (64-bit), headless |
| Hostname | `pihole` (`pihole.local`, `pi.hole`) |
| User | `bshot` (own account; no default `pi` user) |
| IP address | `192.168.86.39` (Ethernet only, DHCP reservation) |
| Network | Google Nest/Google Wifi, `192.168.86.0/24`, router `192.168.86.1` |
| Pi-hole upstream | Quad9 (filtered, DNSSEC), IPv4 only |
| Router DNS | Custom: primary `192.168.86.39`, secondary `9.9.9.9` |
| Desktop PC DNS | Manual: `192.168.86.39` (shows as its own client) |
| Query logging | On, privacy level 0 (show everything) |
| Rate limiting | Off (`dns.rateLimit.count 0`) |
| Admin UI | `http://192.168.86.39/admin/` |
| Screen | PADD on tty1 via console autologin |
| Remote access | SSH from LAN only; Raspberry Pi Connect left off |

Design decisions:

- **Lite, not desktop.** The Pi 3 has 1 GB RAM; a desktop plus Chromium would starve DNS. PADD gives on-screen stats from the text console.
- **Ethernet only.** Wi-Fi is disabled so Pi-hole answers on one address.
- **Pi-hole as primary DNS.** Google Home requires a secondary, so 9.9.9.9 is set as a fallback; watch for failover (see Operations).
- **Console autologin is acceptable.** The Pi sits in a private room; anyone with physical access could pull the SD card anyway.

## Troubleshooting log

Thirteen issues came up across the old image, the rebuild, the DNS cutover and the screen setup. The costliest were testing only a blocked domain before switching the router, and a DHCP reservation tied to the Wi-Fi adapter.

| Symptom | Cause | Fix / lesson |
| --- | --- | --- |
| First ping: `Reply from 192.168.86.31: Destination host unreachable` | The PC's own ARP lookup failed; the Pi was still booting | Wait and retry. A reply from your own IP means the target isn't answering ARP yet |
| `ssh: connect to host ... port 22: Connection refused` | Host up (ping TTL=64 = Linux) but sshd not listening yet on first boot | Refused = TCP reset, nothing on the port. A firewall drop would time out instead. Check with `Test-NetConnection <ip> -Port 22` |
| First SSH prompt: "authenticity of host can't be established" | New host key, not yet in `known_hosts` | Type `yes`. After any reflash, clear the old key with `ssh-keygen -R <ip>` |
| Login worked with `pi` / `raspberry` | Pre-2022 image with public default credentials | Never leave defaults on a network device |
| `/etc/os-release` showed Debian 10 (Buster), kernel 5.4 | 2020-era image, past end of life, no security updates | Reflash instead of upgrading across major versions |
| "Wi-Fi is currently blocked by rfkill" | Wi-Fi country never set | Set the country in Imager; harmless when wired |
| `hostname -I` showed two IPv4 addresses (`.186`, `.35`) | Ethernet and Wi-Fi both connected | Disable Wi-Fi so DNS answers on one address |
| Google Home: "An error has occurred while updating your DNS settings" | App rejects a blank or duplicate secondary | Use a different secondary (`9.9.9.9`) |
| Whole network lost DNS; `nslookup google.com <pi>` returned `Query refused` | Pi-hole had no upstream server ticked, so it refused anything not on the blocklist | Tick Quad9 under Settings → DNS. Always test an allowed domain (google.com) and a blocked one (doubleclick.net) before the router cutover |
| Router showed offline; phone had no internet; router-path lookups weren't blocked | Lab switch was cabled to the wrong network | Recable to the main network, re-verify the Pi's IP, then retest |
| `mv: cannot stat 'install.padd.sh'` | `curl -O` can't name a file from a URL with no path | Use `curl -sSL https://install.padd.sh -o padd.sh` |
| PADD stuck at "No password supplied. Please enter your password" | Pi-hole v6 API requires a login | Save a password to `~/.padd_secret` and start PADD with `--secret` |
| PC ping to `.186` failed after a reboot; SSH still worked over IPv6 link-local | The reservation was for the Wi-Fi MAC; with Wi-Fi off, Ethernet got `.39` from DHCP | Reserve the IP for the Ethernet MAC (`cat /sys/class/net/eth0/address`). Router DNS moved to `.39` |

## Reflash procedure

A clean flash with Raspberry Pi Imager takes about 10 minutes and erases the card.

1. Shut down the Pi with `sudo poweroff`, wait for the green LED to stop, unplug it, and remove the microSD card.
2. Insert the card into the PC and open Raspberry Pi Imager (raspberrypi.com/software).
3. Device: **Raspberry Pi 3**. OS: **Raspberry Pi OS (other) → Raspberry Pi OS Lite (64-bit)**. Storage: the SD card (double-check it isn't another drive).
4. Next → **Edit Settings**:
   - Hostname `pihole`; own username and password
   - Wi-Fi country US; no Wi-Fi network (wired only)
   - Time zone America/New_York
   - Services: **Enable SSH**, password authentication
   - Raspberry Pi Connect: **off**
5. Save → Yes. If Windows offers to format the drive, click **Cancel**.
6. Boot order: SD card in, Ethernet and screen connected, then micro-USB power (5V 2.5A supply). Red LED solid = power OK; green flicker = reading SD. First boot takes about 3 minutes.
7. From PowerShell: `ssh-keygen -R 192.168.86.39`, then `ssh <user>@pihole.local` (or the IP).

## Post-flash setup

Disable Wi-Fi first, then reserve the IP the Pi gets on Ethernet; reserving before disabling Wi-Fi tied the reservation to the wrong adapter on this build.

**1. Confirm Ethernet is up** so disabling Wi-Fi can't lock you out. `eth0` should be UP with an IPv4 address.

```bash
ip -br a
```

**2. Disable Wi-Fi permanently**, then reconnect over Ethernet. `hostname -I` should show one IPv4 address.

```bash
echo "dtoverlay=disable-wifi" | sudo tee -a /boot/firmware/config.txt
sudo reboot
```

To undo: remove that line with `sudo nano /boot/firmware/config.txt` and reboot.

**3. Reserve the Ethernet IP.** Check the Ethernet MAC, then in Google Home → Wi-Fi → Network settings → Advanced networking → DHCP IP reservations, reserve the Pi's current address (`192.168.86.39`) for that MAC.

```bash
cat /sys/class/net/eth0/address
hostname -I
```

**4. Update the system.**

```bash
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

## Pi-hole install and DNS cutover

Pi-hole installs with one script; the network switches to it with one DNS setting in the Google Home app.

**1. Run the installer.**

```bash
curl -sSL https://install.pi-hole.net | bash
```

- Static IP warning: continue (the DHCP reservation covers it)
- Upstream DNS: **Quad9 (filtered, DNSSEC)** — blocks known malware domains, nonprofit, no IP logging. Skip ECS variants and Google
- Blocklist: default
- Query logging: **on**
- FTL privacy mode: **0 — show everything**

**2. Set the admin password.**

```bash
pihole setpassword
```

**3. Confirm the upstream stuck.** Admin UI → Settings → DNS: both IPv4 boxes ticked for Quad9 (filtered, DNSSEC). The installer choice did not save on this build, which caused the outage in the troubleshooting log.

**4. Turn off rate limiting.** The router forwards every device as one client (`192.168.86.1`), which can trip the default 1,000 queries/minute limit and block the whole house.

```bash
sudo pihole-FTL --config dns.rateLimit.count 0
sudo pihole restartdns
```

**5. Test both an allowed and a blocked domain from the PC** before touching the router. Expect a real IP for google.com and `0.0.0.0` for doubleclick.net.

```powershell
nslookup google.com 192.168.86.39
nslookup doubleclick.net 192.168.86.39
```

**6. Point the router at Pi-hole.** Google Home → Wi-Fi → gear → Advanced networking → DNS → Custom: primary `192.168.86.39`, secondary `9.9.9.9` (the app refuses a blank or duplicate secondary). Save, then restart the network. Google Wifi stays the DNS server clients see (`192.168.86.1`) and forwards to Pi-hole, so the dashboard shows the whole house as one client.

**7. Verify through the router.** Expect `Server: 192.168.86.1` with `0.0.0.0`, and rows from `192.168.86.1` in the query log.

```powershell
ipconfig /flushdns
nslookup doubleclick.net
```

**8. Per-device stats (optional).** Google Wifi can't hand out Pi-hole's address, so set DNS manually on devices you want to see individually. Windows: Settings → Network & internet → Ethernet → DNS server assignment → Edit → Manual → IPv4 `192.168.86.39`. Name the device under Pi-hole Settings → Local DNS → DNS Records (e.g. `bryan-pc` → `192.168.86.31`), and reserve its IP in Google Home.

- [x] Pi-hole installed
- [x] Upstream set to Quad9 (filtered, DNSSEC)
- [x] Router DNS pointed at `192.168.86.39` (secondary `9.9.9.9`)
- [x] Blocking verified through the router
- [x] Desktop PC set to `192.168.86.39` manually
- [ ] Admin password changed with `pihole setpassword`
- [ ] Rate limiting turned off

## On-screen stats (PADD)

PADD shows live Pi-hole stats (queries, % blocked, top domains, CPU/RAM/temperature) on the text console, so no desktop is needed.

**1. Download and test PADD.**

```bash
cd ~ && curl -sSL https://install.padd.sh -o padd.sh && chmod +x padd.sh
./padd.sh   # test it, then Ctrl+C
```

**2. Give PADD a password.** Pi-hole v6 requires a login. Best practice is an app password (Admin UI → Settings → Web interface / API → Expert → Configure app password); the admin password also works. Store it readable only by you:

```bash
echo 'PASTE_PASSWORD_HERE' > ~/.padd_secret && chmod 600 ~/.padd_secret
```

**3. Enable console autologin:** `sudo raspi-config` → System Options → Boot / Auto Login → Console Autologin. Check with `who`; it should list `bshot tty1`.

**4. Start PADD on the screen at login.**

```bash
sed -i '/padd.sh/d' ~/.bashrc
echo 'if [ "$(tty)" = "/dev/tty1" ]; then ~/padd.sh --secret "$(cat ~/.padd_secret)"; fi' >> ~/.bashrc
sudo systemctl restart getty@tty1
```

**5. Fit the screen.** PADD picks a layout from the console's rows and columns; it doesn't stretch. Change the console font, then restart tty1:

```bash
sudo stty -F /dev/tty1 size          # rows cols
sudo dpkg-reconfigure console-setup  # UTF-8 → Guess optimal → Terminus → size
sudo systemctl restart getty@tty1
```

| Font | Console (cols × rows) on 1280×400 | Result |
| --- | --- | --- |
| 8x16 (default) | 160 × 25 | Small text, dashboard on the left half |
| 10x20 | 128 × 20 | Bigger text, some space on the right |
| 12x24 | 106 × 16 | Large text; PADD may switch to a compact layout |
| 16x32 | 80 × 12 | Fills the width; minimal layout |

A 3:1 screen will always leave some space, because PADD's layouts are closer to 4:3.

**Manual start from SSH** (opens PADD on a free console and switches the screen to it; `sudo chvt 1` switches back):

```bash
sudo openvt -s -- su - bshot -c "/home/bshot/padd.sh --secret \"\$(cat /home/bshot/.padd_secret)\""
```

## Operations and next steps

If Pi-hole breaks, setting DNS back to Automatic in the Google Home app restores internet for the whole network immediately.

| Task | Command / action |
| --- | --- |
| SSH in | `ssh bshot@192.168.86.39` |
| Reboot / shut down | `sudo reboot` / `sudo poweroff` (never pull power; it can corrupt the SD card) |
| Update Pi-hole | `pihole -up` |
| Update OS | `sudo apt update && sudo apt full-upgrade -y` |
| Admin dashboard | `http://192.168.86.39/admin/` |
| Unblock a site | Query log → find the red row → **Allow** |
| Test Pi-hole directly | `nslookup google.com 192.168.86.39` and `nslookup doubleclick.net 192.168.86.39` |
| Check the Pi's IP | `hostname -I` (capital I) |
| Restart DNS | `sudo pihole restartdns` |
| Restart the screen | `sudo systemctl restart getty@tty1` |
| Roll back DNS | Google Home → Advanced networking → DNS → Automatic |
| Router stuck offline | Power-cycle the main Wifi point (30 s), wait 3–5 min; no factory reset |
| After any reflash | `ssh-keygen -R 192.168.86.39` on the PC |

Known risks: Pi-hole is a single point of failure for DNS, and Google Wifi may fail over to the 9.9.9.9 secondary and stay there, silently disabling blocking. If rows from `192.168.86.1` stop appearing in the query log, give the Pi a second IP and use it as the router's secondary. The desktop PC has no fallback; if the Pi is down, switch its DNS back to Automatic.

- [ ] IPv6: if ads leak through, add the Pi's IPv6 address to the Google Home custom DNS, or disable IPv6 on the network
- [ ] SSH keys: `ssh-keygen -t ed25519` on the PC, copy the public key to the Pi, then disable password login in `/etc/ssh/sshd_config`
- [ ] Tailscale: remote SSH without open ports, and Pi-hole as tailnet DNS for ad blocking off-network
- [ ] Second Pi-hole for redundancy
- [ ] Second IP on the Pi as the router's secondary, if failover shows up
- [ ] Manual DNS on phone and TV for per-device stats


<img width="1106" alt="Screenshot 2026-11-21 234321" src="https://github.com/user-attachments/assets/7e2be887-bc1b-4aad-ac66-5983da486109" />

<img width="1090" alt="Screenshot 2026-11-21 234917" src="https://github.com/user-attachments/assets/d1f47fc4-86fb-41fe-99d0-1953277cfd6d" />

<img width="1107" alt="Screenshot 2026-11-21 235000" src="https://github.com/user-attachments/assets/72a276f8-fb18-46ef-adb2-1f5e25cf6156" />

<img width="974" alt="Screenshot 2026-11-22 000420" src="https://github.com/user-attachments/assets/a6c8a796-266e-4419-b374-f4e0de89c482" />

<img width="512" alt="Screenshot 2026-11-22 002747" src="https://github.com/user-attachments/assets/3ae88190-690b-4100-b1de-1244a3229c0b" />

<img width="1251" height="1031" alt="image" src="https://github.com/user-attachments/assets/0bbcf567-c68b-4779-87ac-a815637ce3f8" />





      






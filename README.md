# Hardening Fedora 44 KDE + Firefox

A practical, copy-paste guide to **killing WebRTC IP leaks** and locking down a
Fedora 44 (KDE Plasma) workstation running Firefox. Written for a machine that
is being actively targeted — the goal is to stop leaking your real IP and
identity, and to shrink the attack surface to almost nothing.

> Tested on: **Fedora 44**, **dnf5 5.4.x**, **firewalld**, **SELinux
> enforcing**, **Firefox 157**, KDE Plasma.
> Commands that start with `sudo` need your password. Read each section's
> **trade-off** note before pasting — some steps intentionally break
> convenience features (video calls, device discovery) in exchange for privacy.

---

## Table of contents

1. [Kill WebRTC in Firefox](#1-kill-webrtc-in-firefox)
2. [Harden Firefox (privacy & fingerprinting)](#2-harden-firefox-privacy--fingerprinting)
3. [Encrypted DNS (stop DNS leaks)](#3-encrypted-dns-stop-dns-leaks)
4. [Close your open ports](#4-close-your-open-ports)
5. [Firewall: switch to a strict zone](#5-firewall-switch-to-a-strict-zone)
6. [System hardening](#6-system-hardening)
7. [Network & VPN leak prevention](#7-network--vpn-leak-prevention)
8. [Identity & account hygiene](#8-identity--account-hygiene)
9. [Verification checklist](#9-verification-checklist)
10. [Quick command reference](#10-quick-command-reference)

---

## 1. Kill WebRTC in Firefox

WebRTC can hand your **real public IP to the other side of a call even through a
VPN** (peer-to-peer mode), and it exposes your **local network IPs** to any web
page. This is the #1 deanonymization vector in a browser.

### Fully disable it

Open `about:config` in Firefox (type it in the address bar, accept the
warning), then set:

| Preference                         | Value   | Effect                                                      |
| ---------------------------------- | ------- | ----------------------------------------------------------- |
| `media.peerconnection.enabled`     | `false` | **Disables WebRTC entirely**                                |
| `media.navigator.enabled`          | `false` | Blocks camera/mic enumeration (a fingerprint + leak vector) |
| `media.navigator.streams.fake`     | `true`  | Any site that forces getUserMedia gets a fake stream        |
| `media.peerconnection.ice.no_host` | `true`  | Belt-and-suspenders: don't expose host (LAN) candidates     |

To set one: paste the name into the search box, and for a boolean double-click
to flip it; it's saved instantly.

**Trade-off:** this breaks _all_ browser video/voice calls — Discord web calls,
Google Meet, Jitsi, **Brave Talk**. That is exactly the point in your situation:
do **not** join call links from people you don't trust. If you later need calls
for legit reasons, use a separate dedicated browser profile rather than
re-enabling WebRTC in your main one.

### If you must keep WebRTC (calls) but not leak

Instead of disabling, leave `media.peerconnection.enabled = true` and set:

| Preference                                            | Value  |
| ----------------------------------------------------- | ------ |
| `media.peerconnection.ice.default_address_only`       | `true` |
| `media.peerconnection.ice.no_host`                    | `true` |
| `media.peerconnection.ice.proxy_only_if_behind_proxy` | `true` |

These force WebRTC to use only your default (VPN) route and stop it advertising
LAN/host candidates. Full disable is still safer.

### Backstop with uBlock Origin

Install **uBlock Origin** (Firefox Add-ons). Then uBlock → Dashboard →
**Settings** → enable **"Prevent WebRTC from leaking local IP addresses."**
This protects you even if a pref gets reset.

### Verify

Visit **<https://browserleaks.com/webrtc>** — the "Public IP Address" and "Local
IP Address" fields should be **empty / n/a**. Also check
**<https://ipleak.net>** (its WebRTC section).

---

## 2. Harden Firefox (privacy & fingerprinting)

### Settings UI (Preferences)

- **Privacy & Security → Enhanced Tracking Protection → Strict.**
- **HTTPS-Only Mode → Enable in all windows.**
- **Cookies and Site Data → "Delete cookies and site data when Firefox is
  closed."**
- **Logins and Passwords → turn OFF "Ask to save logins"** (use KeePassXC or
  Bitwarden instead — see §8).
- **Firefox Data Collection → uncheck everything.**

### about:config prefs

| Preference                                   | Value   | Why                                         |
| -------------------------------------------- | ------- | ------------------------------------------- |
| `privacy.resistFingerprinting`               | `true`  | Master anti-fingerprinting (RFP)            |
| `privacy.fingerprintingProtection`           | `true`  | Newer targeted FP protection                |
| `privacy.trackingprotection.enabled`         | `true`  | Block trackers everywhere                   |
| `geo.enabled`                                | `false` | Kill geolocation API                        |
| `dom.battery.enabled`                        | `false` | Remove battery fingerprint                  |
| `beacon.enabled`                             | `false` | Block tracking beacons                      |
| `browser.send_pings`                         | `false` | Block hyperlink auditing pings              |
| `network.dns.disablePrefetch`                | `true`  | No DNS prefetch (leaks intent)              |
| `network.predictor.enabled`                  | `false` | No predictive connections                   |
| `network.prefetch-next`                      | `false` | No link prefetch                            |
| `webgl.disabled`                             | `true`  | Remove WebGL fingerprint (_breaks 3D/maps_) |
| `toolkit.telemetry.enabled`                  | `false` | No telemetry                                |
| `toolkit.telemetry.unified`                  | `false` | No telemetry                                |
| `datareporting.healthreport.uploadEnabled`   | `false` | No health report upload                     |
| `datareporting.policy.dataSubmissionEnabled` | `false` | No data submission                          |
| `app.shield.optoutstudies.enabled`           | `false` | No Shield studies                           |

**Trade-off:** `privacy.resistFingerprinting` forces your timezone to UTC,
rounds the window size, and blocks canvas reads — a few sites look odd or ask
for canvas permission. `webgl.disabled` breaks WebGL maps/3D. Both are worth it
for anti-tracking; disable individually only if a site you trust needs them.

### The easy path: arkenfox user.js

For a vetted, maintained baseline instead of setting prefs by hand, use
**arkenfox user.js** (<https://github.com/arkenfox/user.js>). Drop its `user.js`
into your profile folder (`~/.mozilla/firefox/<profile>.default-release/`) and
tune with an `user-overrides.js`. It's the gold standard for Firefox hardening.
Read its wiki first — it's aggressive.

---

## 3. Encrypted DNS (stop DNS leaks)

Plain DNS tells your ISP (and anyone on-path) every domain you visit, and it can
leak your real resolver even behind some proxies. Turn on **DNS-over-HTTPS**.

### In Firefox

`about:config`:

| Preference         | Value                                              |
| ------------------ | -------------------------------------------------- |
| `network.trr.mode` | `3` (strict — DoH only) or `2` (DoH with fallback) |
| `network.trr.uri`  | `https://dns.quad9.net/dns-query`                  |

(Quad9 blocks known-malicious domains, which is handy here. Alternatives:
Cloudflare `https://cloudflare-dns.com/dns-query`, Mullvad
`https://dns.mullvad.net/dns-query`.) Mode `3` is strongest but if a captive
portal breaks, fall back to `2`.

### System-wide (optional, via systemd-resolved)

Your box already runs `systemd-resolved`. To push DoH for the whole system:

```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
sudo tee /etc/systemd/resolved.conf.d/doh.conf >/dev/null <<'CONF'
[Resolve]
DNS=9.9.9.9#dns.quad9.net 2620:fe::fe#dns.quad9.net
DNSOverTLS=yes
DNSSEC=yes
CONF
sudo systemctl restart systemd-resolved
resolvectl status | grep -iE 'DNS Servers|DNSOverTLS|DNSSEC'
```

(systemd-resolved does DNS-over-TLS, not DoH; combined with Firefox DoH you're
covered at both layers.)

---

## 4. Close your open ports

A port scan of this machine showed several services listening on **all
interfaces** (`0.0.0.0` / `*`), meaning anything on your LAN — and potentially
beyond — can reach them. Audit and bind them to localhost.

See what's listening at any time:

```bash
ss -tulpn
```

Findings on this box and how to fix each:

### Ollama — `*:11434` (exposed LLM API)

Ollama listening on all interfaces lets anyone on the network use (or abuse)
your local models. Bind it to localhost:

```bash
# For the systemd service:
sudo systemctl edit ollama
# add:
#   [Service]
#   Environment="OLLAMA_HOST=127.0.0.1:11434"
sudo systemctl restart ollama
```

If you run Ollama in a container/pod, publish it as `127.0.0.1:11434:11434`
instead of `11434:11434`.

### Cockpit — `*:9090`

Remote web admin. If you don't use it, turn it off:

```bash
sudo systemctl disable --now cockpit.socket
```

### KDE Connect — `*:1716`

Phone↔desktop bridge. If you don't use it, remove it (or at least firewall it):

```bash
sudo dnf remove kdeconnectd        # or: kde-connect
# keep it but block from untrusted networks via the firewall (see §5)
```

### Other `*` listeners (3000, 3002, 8099, 8443, 8880)

These are your dev/app/container services. For anything you don't need reachable
off-box, bind it to `127.0.0.1` in that app's config (dev servers: `--host
127.0.0.1`; containers: publish as `127.0.0.1:PORT:PORT`). The strict firewall
zone in §5 will also block inbound to them.

### mDNS / LLMNR — `*:5355`, avahi

`avahi-daemon` is enabled (used for `.local` discovery/printers). If you don't
need network device discovery:

```bash
sudo systemctl disable --now avahi-daemon avahi-daemon.socket
```

**Trade-off:** breaks `.local` hostname resolution and some printer/file-share
auto-discovery.

---

## 5. Firewall: switch to a strict zone

firewalld is running but your default zone is **`FedoraWorkstation`**, which is
deliberately permissive — it opens high ports (1025–65535) so desktop apps
"just work." For a targeted machine, switch to the **`public`** zone, which
blocks all inbound except what you allow.

```bash
# See current state
firewall-cmd --get-active-zones
firewall-cmd --list-all

# Move your active connection to the public zone (replace <conn> with your
# NetworkManager connection name from: nmcli -t -f NAME connection show)
nmcli connection modify "<conn>" connection.zone public

# Set public as the default too
sudo firewall-cmd --set-default-zone=public

# Trim public to the minimum (drop ssh + mdns inbound; you're not serving them)
sudo firewall-cmd --permanent --zone=public --remove-service=ssh
sudo firewall-cmd --permanent --zone=public --remove-service=mdns
sudo firewall-cmd --reload

# Confirm
firewall-cmd --zone=public --list-all
```

**Trade-off:** inbound to KDE Connect (1716), local dev servers, and device
discovery stops working from the LAN. That's the goal. If you _use_ KDE Connect
on a network you trust, re-allow just that:

```bash
sudo firewall-cmd --permanent --zone=public --add-port=1714-1764/tcp
sudo firewall-cmd --permanent --zone=public --add-port=1714-1764/udp
sudo firewall-cmd --reload
```

---

## 6. System hardening

### Keep it patched — automatically

Your box has `dnf5` but no auto-updater. Install one and apply **security**
updates automatically:

```bash
sudo dnf install dnf5-plugin-automatic
sudoedit /etc/dnf/dnf5-plugins/automatic.conf
#   under [commands] set:
#     upgrade_type = security
#     apply_updates = yes
sudo systemctl enable --now dnf5-automatic.timer
systemctl list-timers dnf5-automatic.timer
```

Still run a full upgrade regularly:

```bash
sudo dnf upgrade --refresh
```

### Firmware updates

```bash
sudo fwupdmgr refresh --force
sudo fwupdmgr update
```

### SELinux — leave it enforcing

It already is (`Enforcing`). Never set it permissive/disabled to "fix" an app;
write a policy exception instead. Verify:

```bash
getenforce      # must say: Enforcing
```

### SSH

`sshd` is already **disabled** on this box — good, keep it off unless you need
remote login. If you ever enable it, harden first in a drop-in:

```bash
sudo tee /etc/ssh/sshd_config.d/99-hardening.conf >/dev/null <<'CONF'
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
CONF
# then: sudo systemctl enable --now sshd  (key-based login only)
```

Pair it with fail2ban if exposed:

```bash
sudo dnf install fail2ban
sudo systemctl enable --now fail2ban
```

### Disk encryption

Full-disk encryption (LUKS) must be set at install time. Check whether you have
it:

```bash
lsblk -o NAME,FSTYPE,MOUNTPOINT | grep -i crypt && echo "LUKS present" || echo "No LUKS detected"
```

If you have none and the data matters, the only real fix is a reinstall with
"Encrypt my data" checked. In the meantime, encrypt sensitive files/backups with
`gpg` or a LUKS container.

### Lock screen (KDE)

System Settings → **Screen Locking** → auto-lock after a few minutes, require
password. Also set a short "lock after sleep."

### Audit what else auto-starts

```bash
systemctl list-unit-files --state=enabled --type=service
```

Disable anything you don't recognize or need (research first).

---

## 7. Network & VPN leak prevention

- **Use a reputable VPN with a kill-switch** and keep it on. WireGuard-based
  providers (Mullvad, Proton, IVPN) are solid.
- **Enable the kill-switch / "lockdown mode"** so traffic is dropped if the VPN
  drops — otherwise you flash your real IP on every reconnect.
- **Randomize your Wi-Fi MAC** (stops tracking across networks). Per-connection:

  ```bash
  nmcli connection modify "<wifi-conn>" wifi.cloned-mac-address random
  nmcli connection modify "<wifi-conn>" 802-11-wireless.mac-address-randomization 2
  ```

  Or globally, create `/etc/NetworkManager/conf.d/00-macrandomize.conf`:

  ```ini
  [device]
  wifi.scan-rand-mac-address=yes
  [connection]
  wifi.cloned-mac-address=random
  ethernet.cloned-mac-address=random
  ```

  then `sudo systemctl restart NetworkManager`.

- **Verify no leaks** after connecting the VPN: <https://ipleak.net> and
  <https://dnsleaktest.com> — your IP, DNS, and WebRTC should all show the VPN,
  never your ISP.

---

## 8. Identity & account hygiene

Technical hardening is half of it. These close the human/OSINT vectors someone
is using to find and pressure you.

- **SIM-swap lock:** call your mobile carrier, set a **port-freeze / number
  lock** and a strong account **PIN**. This defeats "Verizon here we come"
  threats.
- **Kill SMS 2FA.** Move every important account (email **first**) to an
  **authenticator app or a hardware key (YubiKey)**. SMS codes fall to
  SIM-swaps.
- **WHOIS privacy:** if you own any domain, enable **domain privacy** at the
  registrar (Cloudflare Registrar, Porkbun, Namecheap include it free). Public
  WHOIS is how your real name leaked.
- **Password manager:** KeePassXC (local) or Bitwarden. Unique random password
  per site. Never reuse.

  ```bash
  sudo dnf install keepassxc
  ```

- **Break identity correlation.** Attackers link people by: reused usernames,
  typing style (stylometry), shared servers, linked social accounts, and avatars.
  A clean new identity must share **none** of these with the old one — new
  email with no name fragments, unrelated username, fresh avatar, no linked
  connections, and don't rejoin servers they're in.
- **Discord privacy:** Settings → Privacy & Safety → turn **off** DMs from
  server members, set friend requests to friends-only, enable spam filtering.
- **Preserve evidence before deleting accounts** — screenshot threats with
  usernames, user IDs, and message IDs, and save offline **before** you burn an
  account, so you can still report to Discord (dis.gd/request) and IC3
  (ic3.gov).

---

## 9. Verification checklist

Run through these after you've applied the above:

- [ ] **WebRTC**: <https://browserleaks.com/webrtc> shows no public/local IP.
- [ ] **IP/DNS**: <https://ipleak.net> and <https://dnsleaktest.com> show only your
      VPN, never your ISP.
- [ ] **Fingerprint**: <https://coveryourtracks.eff.org> reports strong protection.
- [ ] **Open ports**: `ss -tulpn` shows nothing bound to `0.0.0.0`/`*` that you
      didn't intend (dev/AI services on `127.0.0.1` only).
- [ ] **Firewall**: `firewall-cmd --get-default-zone` → `public`;
      `firewall-cmd --list-all` shows a minimal service list.
- [ ] **SELinux**: `getenforce` → `Enforcing`.
- [ ] **Updates**: `systemctl list-timers dnf5-automatic.timer` shows it active.
- [ ] **2FA**: every critical account on app/hardware 2FA, carrier SIM-locked.

---

## 10. Quick command reference

```bash
# What's listening right now
ss -tulpn

# Firewall state + lock to strict zone
firewall-cmd --get-active-zones
sudo firewall-cmd --set-default-zone=public
firewall-cmd --list-all

# Updates
sudo dnf upgrade --refresh
sudo fwupdmgr refresh --force && sudo fwupdmgr update

# Security posture
getenforce
resolvectl status | grep -iE 'DNS Servers|DNSOverTLS'

# Bind Ollama to localhost
sudo systemctl edit ollama   # Environment="OLLAMA_HOST=127.0.0.1:11434"

# Turn off remote admin you don't use
sudo systemctl disable --now cockpit.socket
```

---

_Harden deliberately, verify with the checklist in §9, and keep your evidence
before you delete anything. Report threats; don't retaliate._

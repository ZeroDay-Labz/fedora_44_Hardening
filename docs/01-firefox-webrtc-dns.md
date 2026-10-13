# Firefox — WebRTC, Privacy &amp; Encrypted DNS

> Part of the [Fedora 44 Hardening guide](../README.md).

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
[← Back to the index](../README.md)

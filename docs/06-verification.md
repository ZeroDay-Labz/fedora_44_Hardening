# Verification &amp; Quick Reference

> Part of the [Fedora 44 Hardening guide](../README.md).

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

---
[← Back to the index](../README.md)

# Fedora 44 Hardening

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

## Guides

Each part is its own page — click to read it rendered.

| Part | Topic |
|-----:|-------|
| 1 | [Firefox — WebRTC, Privacy & Encrypted DNS](docs/01-firefox-webrtc-dns.md) |
| 2 | [Closing Open Ports & the Firewall](docs/02-ports-firewall.md) |
| 3 | [System Hardening](docs/03-system-hardening.md) |
| 4 | [Network & VPN Leak Prevention](docs/04-network-vpn.md) |
| 5 | [Identity & Account Hygiene](docs/05-identity-hygiene.md) |
| 6 | [Verification & Quick Reference](docs/06-verification.md) |
| 7 | [SearXNG on Podman (self-hosted search)](docs/07-searxng-podman.md) |

---

*Harden deliberately, verify before you trust it, and keep your evidence before
you delete anything. Report threats; don't retaliate.*

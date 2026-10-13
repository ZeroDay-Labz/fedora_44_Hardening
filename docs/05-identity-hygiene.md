# Identity &amp; Account Hygiene

> Part of the [Fedora 44 Hardening guide](../README.md).

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
[← Back to the index](../README.md)

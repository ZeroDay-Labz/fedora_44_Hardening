# Network &amp; VPN Leak Prevention

> Part of the [Fedora 44 Hardening guide](../README.md).

## 7. Network & VPN leak prevention
**Use a reputable VPN with a kill-switch** and keep it on. WireGuard-based
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
[← Back to the index](../README.md)

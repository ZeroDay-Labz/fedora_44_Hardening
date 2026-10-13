# Closing Open Ports &amp; the Firewall

> Part of the [Fedora 44 Hardening guide](../README.md).

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
[← Back to the index](../README.md)

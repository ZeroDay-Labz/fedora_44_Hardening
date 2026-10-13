> Part of the [Fedora 44 Hardening guide](../README.md).

# Self-hosting SearXNG with Podman

[SearXNG](https://github.com/searxng/searxng) is a private, self-hosted
metasearch engine — it queries other search engines for you, returns the
results, and keeps no profile on you. This guide stands up a rootless,
auto-starting instance with **Podman** and **Valkey** (for rate limiting).

> **Assumes:** a Linux box with Podman installed (Fedora, RHEL, Debian/Ubuntu,
> etc.). Commands are copy-paste. The Fedora/SELinux notes matter if you're on
> Fedora/RHEL — they're called out inline.

---

## 0. Prerequisites

Install Podman (skip if you already have it):

```bash
# Fedora / RHEL
sudo dnf install -y podman

# Debian / Ubuntu
sudo apt install -y podman
```

Check it works **rootless** (no sudo needed to run containers):

```bash
podman info --format '{{.Host.Security.Rootless}}'   # should print: true
podman --version
```

We'll run everything as your normal user — no root containers.

---

## 1. Create the project folders

```bash
mkdir -p ~/searxng/config
cd ~/searxng
```

- `~/searxng/config` holds `settings.yml` (your instance's config).

---

## 2. Generate a secret key

SearXNG needs a random `secret_key`. Generate one and keep it:

```bash
openssl rand -hex 32
```

Copy the output — you'll paste it into `settings.yml` in step 4.

---

## 3. Create the pod and start Valkey

A **pod** lets the two containers share a localhost network, so SearXNG can
reach Valkey at `localhost:6379`. We publish SearXNG's web port on **localhost
only** (`127.0.0.1:8080`) — safest default; §7 covers remote access.

```bash
# Create the pod, exposing the web UI on localhost:8080
podman pod create --name searxng -p 127.0.0.1:8080:8080

# Start Valkey (Redis-compatible) inside the pod, for the rate limiter
podman run -d --name searxng-valkey --pod searxng \
  docker.io/valkey/valkey:8-alpine \
  valkey-server --save "" --appendonly no
```

---

## 4. First run of SearXNG (generates settings.yml)

Start SearXNG once so it creates a default `settings.yml` in your config folder.

> **Fedora/RHEL (SELinux):** the `:Z` on the volume is **required** — it
> relabels the folder so the container can read it. On Debian/Ubuntu `:Z` is
> harmless, leave it in.

```bash
podman run -d --name searxng --pod searxng \
  -v ~/searxng/config:/etc/searxng:Z \
  -e "SEARXNG_BASE_URL=http://localhost:8080/" \
  docker.io/searxng/searxng:latest
```

Wait ~10 seconds, then confirm the file appeared:

```bash
ls ~/searxng/config          # you should see settings.yml (and limiter.toml)
```

---

## 5. Configure it

Edit `~/searxng/config/settings.yml`. Make these changes:

1. **Paste your secret key** from step 2.
2. **Turn the limiter on** (blocks bots/abuse).
3. **Point it at Valkey.**
4. Optionally enable the **JSON API** (needed if another app will query it).

A known-good minimal top section:

```yaml
use_default_settings: true

general:
  instance_name: "My SearXNG"

server:
  secret_key: "PASTE_YOUR_openssl_KEY_HERE"
  limiter: true
  image_proxy: true

search:
  safe_search: 0
  formats:
    - html
    - json        # remove this line if you don't need the API

# Rate-limiter backend. Newer SearXNG uses the `valkey:` key; older builds use
# `redis:`. Use whichever your generated settings.yml already contains — the URL
# is the same either way:
valkey:
  url: redis://localhost:6379/0
```

> If your generated file has a `redis:` block instead of `valkey:`, just set
> `redis: { url: redis://localhost:6379/0 }` there. Don't add both.

Apply the changes:

```bash
podman restart searxng
```

Open **http://localhost:8080** in a browser — you should get the search page.
Run a test query; results mean it's talking to Valkey and the upstreams.

---

## 6. Auto-start on boot (systemd, rootless)

So the instance comes back after a reboot without you logging in.

**Let your user's services run while logged out:**

```bash
sudo loginctl enable-linger $USER
```

**Generate and enable systemd units for the pod + containers:**

```bash
mkdir -p ~/.config/systemd/user
cd ~/.config/systemd/user
podman generate systemd --new --files --name searxng
systemctl --user daemon-reload
# Enable the pod unit (it pulls in the containers):
systemctl --user enable --now pod-searxng.service
```

Check status:

```bash
systemctl --user status pod-searxng.service
```

Now it survives reboots. (Managing it: `systemctl --user restart
pod-searxng.service`.)

---

## 7. Access from other machines (optional)

By default it listens on **localhost only** — reachable just from this box.
To let your LAN or the internet use it, **put a reverse proxy with HTTPS in
front of it** rather than exposing port 8080 directly.

### Easiest: Caddy (automatic HTTPS)

If you have a domain pointing at the server:

```bash
podman run -d --name caddy --pod searxng \
  -p 80:80 -p 443:443 \
  -v ~/searxng/caddy:/data:Z \
  docker.io/library/caddy:latest \
  caddy reverse-proxy --from search.example.com --to localhost:8080
```

Then set `SEARXNG_BASE_URL=https://search.example.com/` (re-create the searxng
container with that env) and in `settings.yml` make sure `server.secret_key` is
set. Caddy fetches a Let's Encrypt cert automatically.

### Firewall

If exposing beyond localhost, open the ports you actually use:

```bash
# Fedora/RHEL
sudo firewall-cmd --permanent --add-service=https   # (and http for the cert challenge)
sudo firewall-cmd --reload
```

Don't open `8080` to the world — keep the raw app on localhost and let the proxy
handle TLS.

---

## 8. Updating

```bash
podman pull docker.io/searxng/searxng:latest
podman pull docker.io/valkey/valkey:8-alpine
systemctl --user restart pod-searxng.service    # if using systemd
# or, if running manually:
#   podman restart searxng searxng-valkey
```

Your `settings.yml` persists across updates (it's a mounted volume).

---

## 9. Handy commands

```bash
podman pod ps                 # is the pod running?
podman ps --pod               # containers + their pod
podman logs searxng           # app logs (check here if a search errors)
podman logs searxng-valkey    # limiter backend logs

# Stop / start everything
podman pod stop searxng
podman pod start searxng

# Full teardown (keeps your config folder)
podman pod rm -f searxng
```

---

## 10. Security notes

- **Rootless by design** — these containers run as your user, not root, so a
  container breakout doesn't hand over the host.
- **Localhost-bound by default** — nothing is network-reachable until you add a
  proxy in §7. Keep it that way unless you need remote access.
- **Keep the secret key secret** — don't commit `settings.yml` to a public repo.
- **SELinux stays enforcing** — the `:Z` volume flag is how you keep it happy;
  never disable SELinux to "fix" a mount.
- **Update the base images regularly** (§8) — search engines change and
  SearXNG ships fixes often.
- SearXNG itself stores no search history or user profiles; that's the point.

---

### TL;DR

```bash
mkdir -p ~/searxng/config && cd ~/searxng
openssl rand -hex 32                       # save this key
podman pod create --name searxng -p 127.0.0.1:8080:8080
podman run -d --name searxng-valkey --pod searxng docker.io/valkey/valkey:8-alpine valkey-server --save "" --appendonly no
podman run -d --name searxng --pod searxng -v ~/searxng/config:/etc/searxng:Z docker.io/searxng/searxng:latest
# edit ~/searxng/config/settings.yml: paste secret_key, limiter: true, valkey url
podman restart searxng
# http://localhost:8080
```

---
[← Back to the index](../README.md)

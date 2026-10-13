# System Hardening

> Part of the [Fedora 44 Hardening guide](../README.md).

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
[← Back to the index](../README.md)

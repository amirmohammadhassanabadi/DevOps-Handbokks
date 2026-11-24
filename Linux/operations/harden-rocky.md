## **Phase 1** - Initial System Updates & Package Sanitization

Before changing configurations, patch known vulnerabilities and strip unnecessary attack surfaces.


```bash
# Update repository metadata and upgrade all packages
sudo dnf clean all
sudo dnf upgrade -y

# Enable automatic security updates via dnf-automatic
sudo dnf install -y dnf-automatic
```

Configure automatic downloads and application of security fixes:

Edit **`/etc/dnf/automatic.conf`**:

```ini
[commands]
upgrade_type = security
apply_updates = yes
```

Enable the systemd timer:

```bash
sudo systemctl enable --now dnf-automatic.timer
```

Remove legacy/unneeded network packages and compilers if not strictly needed in production:

```bash
sudo dnf remove -y telnet rsh-server ypserv tftp-server talk-server xinetd gcc
```
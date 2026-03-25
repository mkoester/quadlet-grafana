# quadlet-grafana

Quadlet setup for [Grafana](https://grafana.com/) — an observability dashboard (`docker.io/grafana/grafana`).

This project was created with the help of Claude Code and https://github.com/mkoester/quadlet-my-guidelines/blob/main/new_quadlet_with_ai_assistance.md.

## Files in this repo

| File | Description |
|---|---|
| `grafana.container` | Quadlet unit file |
| `grafana.env` | Default environment variables |
| `grafana.override.env.template` | Template for local overrides (secrets) |
| `grafana-backup.service` | Systemd service: SQLite snapshot + rsync to backup location |
| `grafana-backup.timer` | Systemd timer: triggers the backup daily |

## Setup

```sh
# 1. Create service user (regular user, home in /var/lib)
sudo useradd -m -d /var/lib/grafana -s /usr/sbin/nologin grafana

REPO_URL=https://github.com/mkoester/quadlet-grafana.git
REPO=~grafana/quadlet-grafana
```

```sh
# 2. Enable linger
sudo loginctl enable-linger grafana

# 3. Clone this repo into the service user's home
sudo -u grafana git clone $REPO_URL $REPO

# 4. Create quadlet and data directories
sudo -u grafana mkdir -p ~grafana/.config/containers/systemd
sudo -u grafana mkdir -p ~grafana/data

# 5. Create .override.env from template and set the admin password
sudo -u grafana cp $REPO/grafana.override.env.template $REPO/grafana.override.env
sudo -u grafana nano $REPO/grafana.override.env

# 6. Symlink quadlet files from the repo
sudo -u grafana ln -s $REPO/grafana.container ~grafana/.config/containers/systemd/grafana.container
sudo -u grafana ln -s $REPO/grafana.env ~grafana/.config/containers/systemd/grafana.env
sudo -u grafana ln -s $REPO/grafana.override.env ~grafana/.config/containers/systemd/grafana.override.env

# 7. Reload and start
sudo -u grafana XDG_RUNTIME_DIR=/run/user/$(id -u grafana) systemctl --user daemon-reload
sudo -u grafana XDG_RUNTIME_DIR=/run/user/$(id -u grafana) systemctl --user start grafana

# 8. Verify
sudo -u grafana XDG_RUNTIME_DIR=/run/user/$(id -u grafana) systemctl --user status grafana
```

## Configuration

### Environment variables

`grafana.env` contains the defaults:

| Variable | Default | Description |
|---|---|---|
| `TZ` | `Europe/Berlin` | Container timezone |

`grafana.override.env` (created from template) — required and optional values:

| Variable | Required | Description |
|---|---|---|
| `GF_SECURITY_ADMIN_PASSWORD` | Yes | Admin user password (default `admin` is insecure) |
| `GF_SERVER_ROOT_URL` | No | Public URL, used in alert links and share URLs |

To apply changes after editing the override file:

```sh
sudo -u grafana nano ~grafana/quadlet-grafana/grafana.override.env
sudo -u grafana XDG_RUNTIME_DIR=/run/user/$(id -u grafana) systemctl --user restart grafana
```

## Connecting to Loki

Grafana runs in a container and reaches host-published services via `host.containers.internal`. If Loki is running on the same host and publishes on `127.0.0.1:3100`, add a Loki datasource in Grafana with:

```
URL: http://host.containers.internal:3100
```

Navigate to **Connections → Data sources → Add new → Loki** in the Grafana UI.

## Reverse proxy (Caddy)

Add a site block to your Caddyfile:

```
grafana.example.com {
    reverse_proxy localhost:3000
}
```

After editing the Caddyfile, reload Caddy:

```sh
sudo systemctl reload caddy
```

## Backup

Grafana stores its data (SQLite database, plugins, alerting state) in `/var/lib/grafana` inside the container (`~grafana/data/` on the host). See the [general backup setup](https://github.com/mkoester/quadlet-my-guidelines#backup) for the one-time server-wide setup.

The backup service uses `sqlite3 .backup` for a consistent DB snapshot, then `rsync` for the remaining files.

```sh
# 1. Create backup staging directory (owned by grafana, readable by backup-readers group)
sudo mkdir -p /var/backups/grafana
sudo chown grafana:backup-readers /var/backups/grafana
sudo chmod 750 /var/backups/grafana

# 2. Symlink the backup service and timer from the repo
sudo -u grafana mkdir -p ~grafana/.config/systemd/user
sudo -u grafana ln -s $REPO/grafana-backup.service ~grafana/.config/systemd/user/grafana-backup.service
sudo -u grafana ln -s $REPO/grafana-backup.timer ~grafana/.config/systemd/user/grafana-backup.timer

# 3. Enable and start the timer
sudo -u grafana XDG_RUNTIME_DIR=/run/user/$(id -u grafana) systemctl --user daemon-reload
sudo -u grafana XDG_RUNTIME_DIR=/run/user/$(id -u grafana) systemctl --user enable --now grafana-backup.timer
```

### On the remote (backup) machine

```sh
rsync -az backupuser@grafana-host:/var/backups/grafana/ /path/to/local/backup/grafana/
```

## Notes

- Port `3000` is bound to `127.0.0.1` only — Caddy handles TLS termination.
- All persistent data is stored at `~grafana/data/` on the host.
- `AutoUpdate=registry` is enabled; activate the timer once to get automatic image updates:
  ```sh
  sudo -u grafana XDG_RUNTIME_DIR=/run/user/$(id -u grafana) systemctl --user enable --now podman-auto-update.timer
  ```
- To prune old images automatically, enable the system-wide prune timer (see [image pruning setup](https://github.com/mkoester/quadlet-my-guidelines#image-pruning)). Replace `30` with the desired retention period in days:
  ```sh
  sudo -u grafana XDG_RUNTIME_DIR=/run/user/$(id -u grafana) systemctl --user enable --now podman-image-prune@30.timer
  ```

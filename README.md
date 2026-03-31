# rpi-ansible

Ansible playbook to configure a Raspberry Pi with Docker, Immich, and Tailscale.

## Prerequisites

- Ansible installed on your Mac: `brew install ansible`
- Required collections:
  ```bash
  ansible-galaxy collection install ansible.posix community.docker
  ```
- SSH access to the Pi (password or key-based)

## Roles

| Role | What it does |
|---|---|
| `system` | Installs LVM2, activates the photos LVM volume, mounts it |
| `docker` | Installs Docker Engine + Compose plugin, configures log rotation, schedules weekly prune |
| `tailscale` | Installs Tailscale, joins the tailnet |
| `immich` | Deploys Immich via Docker Compose |

## Configuration

All variables are in `group_vars/rpi.yml`. Most are driven by environment variables at run time.

| Env var | Default | Required | Description |
|---|---|---|---|
| `RPI_HOST` | `raspberrypi.local` | No | IP or hostname of the Pi |
| `RPI_USER` | `pi` | No | SSH user and Docker group member |
| `PHOTOS_LV_PATH` | _(none)_ | Yes | Device path of the LVM logical volume to mount (e.g. `/dev/Photos/photos`) |
| `PHOTOS_PATH` | `/mnt/photos` | No | Mount point for the photos LVM volume |
| `TAILSCALE_AUTH_KEY` | _(none)_ | Yes | Auth key from the Tailscale admin console (Settings → Keys) |
| `IMMICH_UPLOAD_LOCATION` | `/mnt/photos/immich` | No | Path where Immich stores uploaded photos and videos |
| `IMMICH_DB_DATA_LOCATION` | `/home/pi/immich/postgres` | No | Path where Postgres data is stored (must be local, not a network share) |
| `IMMICH_VERSION` | `release` | No | Immich version to deploy (e.g. `v2.6.3`); defaults to latest stable |
| `TZ` | _(UTC)_ | No | Timezone for Immich containers (e.g. `Europe/London`) |

`immich_db_password` is stored as an Ansible Vault encrypted value in `group_vars/rpi.yml`.
To set it: `ansible-vault encrypt_string 'yourpassword' --name 'immich_db_password'`

## Run

```bash
RPI_HOST=192.168.1.100 RPI_USER=pi \
PHOTOS_LV_PATH=/dev/Photos/photos \
TAILSCALE_AUTH_KEY=tskey-auth-... \
ansible-playbook playbook.yml --ask-pass --ask-become-pass --ask-vault-pass
```

## Immich

Available at `http://<pi-ip>:2283` after the playbook completes.


# rpi-ansible

Ansible playbook to configure a Raspberry Pi with Docker, Immich, Immich Public Proxy, Mindfull, Cloudflare Tunnel, and Tailscale.

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
| `immich_public_proxy` | Deploys Immich Public Proxy via a separate Docker Compose project |
| `mindfull` | Deploys Mindfull via a separate Docker Compose project |
| `cloudflared` | Deploys Cloudflare Tunnel via a separate Docker Compose project |

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
| `IMMICH_VERSION` | `v3.0.3` | No | Immich version to deploy |
| `IMMICH_PUBLISHED_HOST` | `tailscale` | No | Host interface for private Immich port publishing; `tailscale` auto-detects the Pi's Tailscale IPv4 |
| `IMMICH_PUBLISHED_PORT` | `2283` | No | Host port for private Immich |
| `IMMICH_INTERNAL_NETWORK` | `immich_default` | No | Immich Docker network used by Immich Public Proxy to reach Immich |
| `IMMICH_PUBLIC_NETWORK` | `immich_public` | No | Shared Docker network between Immich Public Proxy and Cloudflare Tunnel |
| `IMMICH_PUBLIC_PROXY_VERSION` | `2.5.0` | No | immich-public-proxy Docker image tag |
| `IMMICH_PUBLIC_PROXY_PUBLIC_BASE_URL` | `https://photos.vinayakakv.com` | No | Public URL used in shared Immich links |
| `IMMICH_PUBLIC_PROXY_HOST` | `127.0.0.1` | No | Host interface for immich-public-proxy port publishing |
| `IMMICH_PUBLIC_PROXY_PORT` | `3000` | No | Host port for immich-public-proxy |
| `IMMICH_PUBLIC_PROXY_ALLOW_DOWNLOAD_ALL` | `1` | No | Download-all behavior: `0` disables, `1` follows Immich share setting, `2` always allows |
| `IMMICH_PUBLIC_PROXY_LIGHTBOX_SHOW_DOWNLOAD` | `true` | No | Show the lightbox download button when downloads are allowed |
| `MINDFULL_VERSION` | `latest` | No | Mindfull Docker image tag; prefer an immutable `sha-<commit>` tag when available |
| `MINDFULL_BIND_ADDRESS` | `0.0.0.0` | No | Host interface for Mindfull port publishing |
| `MINDFULL_PORT` | `3001` | No | Host port for Mindfull |
| `MINDFULL_BACKUP_PATH` | `/mnt/photos/mindfull/backups` | No | Host directory for Mindfull SQLite snapshots |
| `MINDFULL_TIMEZONE` | `Asia/Kolkata` | No | Timezone used by the Mindfull backup scheduler |
| `BACKUP_LOCAL_TIME` | `03:00` | No | Daily backup time in `MINDFULL_TIMEZONE` |
| `BACKUP_DAILY_RETENTION` | `7` | No | Number of recent daily backups to retain |
| `BACKUP_WEEKLY_RETENTION` | `4` | No | Number of older weekly backups to retain |
| `CLOUDFLARED_VERSION` | `2026.6.0` | No | cloudflared Docker image tag |
| `CLOUDFLARED_TUNNEL_TOKEN` | _(none)_ | Yes | Token from the Cloudflare Tunnel setup screen |
| `TZ` | _(UTC)_ | No | Timezone for Immich containers (e.g. `Europe/London`) |

`immich_db_password` is stored as an Ansible Vault encrypted value in `group_vars/rpi.yml`.
To set it: `ansible-vault encrypt_string 'yourpassword' --name 'immich_db_password'`

`mindfull_pairing_code` should also be stored as an Ansible Vault encrypted value in `group_vars/rpi.yml`.
To set it: `ansible-vault encrypt_string 'replace-with-a-long-random-value' --name 'mindfull_pairing_code'`

## Run

```bash
RPI_HOST=192.168.1.100 RPI_USER=pi \
PHOTOS_LV_PATH=/dev/Photos/photos \
TAILSCALE_AUTH_KEY=tskey-auth-... \
CLOUDFLARED_TUNNEL_TOKEN=eyJhIjoi... \
ansible-playbook playbook.yml --ask-pass --ask-become-pass --ask-vault-pass
```

## Immich

Available at `http://<pi-tailscale-ip>:2283` after the playbook completes. This keeps the full Immich app off the public network path by default while allowing private tailnet access.

Set `IMMICH_PUBLISHED_HOST=127.0.0.1` if you want Immich reachable only from the Pi itself, or `IMMICH_PUBLISHED_HOST=0.0.0.0` if you intentionally want it bound on every interface.

The Immich role also installs `immich-compose.service`, a boot-time systemd unit that runs `docker compose up -d --force-recreate --wait` after Docker, networking, and Tailscale are available. Compose recreates the containers at boot, waits for Postgres and Redis to become healthy, and then starts Immich with the project network and service DNS in a clean state.

## Immich Public Proxy

Public shared links are served by `immich-public-proxy` at `http://127.0.0.1:3000` on the Pi by default. Put a public HTTPS tunnel or reverse proxy in front of that local endpoint for `https://photos.vinayakakv.com`.

Immich Public Proxy runs as a separate Docker Compose project from Immich. It joins the Immich internal network named by `IMMICH_INTERNAL_NETWORK` to reach `http://immich-server:2283`, and it joins `IMMICH_PUBLIC_NETWORK` so Cloudflare Tunnel can reach the proxy.

## Mindfull

Mindfull runs from `ghcr.io/vinayakakv/mind-full` as a separate Docker Compose project under `/home/<user>/mindfull`.

By default it binds to `0.0.0.0:3001`, so it is reachable on the Pi's LAN and Tailscale addresses at `http://<pi-ip>:3001`.

Daily SQLite snapshots are stored at `/mnt/photos/mindfull/backups` by default. This is on the same mounted disk as Immich but outside Immich's `/mnt/photos/immich` data directory. The live Mindfull database remains in its separate Docker volume.

The role is skipped unless the vault variable `mindfull_pairing_code` is set.

## Cloudflare Tunnel

Create a remotely managed tunnel in the Cloudflare Zero Trust dashboard, then use the token from the setup screen as `CLOUDFLARED_TUNNEL_TOKEN`.

In the tunnel's public hostname settings, route:

```text
photos.vinayakakv.com -> HTTP -> immich-public-proxy:3000
```

The `cloudflared` container runs in its own Compose project and joins the same `IMMICH_PUBLIC_NETWORK` Docker network as Immich Public Proxy. It does not publish any host ports.

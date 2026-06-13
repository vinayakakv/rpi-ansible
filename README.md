# rpi-ansible

Ansible playbook to configure a Raspberry Pi with Docker, Immich, Immich Public Proxy, Cloudflare Tunnel, and Tailscale.

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
| `IMMICH_VERSION` | `release` | No | Immich version to deploy (e.g. `v2.6.3`) |
| `IMMICH_PUBLISHED_HOST` | `127.0.0.1` | No | Host interface for private Immich port publishing |
| `IMMICH_PUBLISHED_PORT` | `2283` | No | Host port for private Immich |
| `IMMICH_PUBLIC_NETWORK` | `immich_public` | No | Shared Docker network between Immich and Immich Public Proxy |
| `IMMICH_PUBLIC_PROXY_VERSION` | `1.15.6` | No | immich-public-proxy Docker image tag |
| `IMMICH_PUBLIC_PROXY_PUBLIC_BASE_URL` | `https://photos.vinayakakv.com` | No | Public URL used in shared Immich links |
| `IMMICH_PUBLIC_PROXY_HOST` | `127.0.0.1` | No | Host interface for immich-public-proxy port publishing |
| `IMMICH_PUBLIC_PROXY_PORT` | `3000` | No | Host port for immich-public-proxy |
| `IMMICH_PUBLIC_PROXY_ALLOW_DOWNLOAD_ALL` | `1` | No | Download-all behavior: `0` disables, `1` follows Immich share setting, `2` always allows |
| `CLOUDFLARED_VERSION` | `2026.6.0` | No | cloudflared Docker image tag |
| `CLOUDFLARED_TUNNEL_TOKEN` | _(none)_ | Yes | Token from the Cloudflare Tunnel setup screen |
| `TZ` | _(UTC)_ | No | Timezone for Immich containers (e.g. `Europe/London`) |

`immich_db_password` is stored as an Ansible Vault encrypted value in `group_vars/rpi.yml`.
To set it: `ansible-vault encrypt_string 'yourpassword' --name 'immich_db_password'`

## Run

```bash
RPI_HOST=192.168.1.100 RPI_USER=pi \
PHOTOS_LV_PATH=/dev/Photos/photos \
TAILSCALE_AUTH_KEY=tskey-auth-... \
CLOUDFLARED_TUNNEL_TOKEN=eyJhIjoi... \
ansible-playbook playbook.yml --ask-pass --ask-become-pass --ask-vault-pass
```

## Immich

Available at `http://127.0.0.1:2283` on the Pi after the playbook completes. This keeps the full Immich app off the public network path by default.

Set `IMMICH_PUBLISHED_HOST` to a specific private interface, such as the Pi's Tailscale IP, if you want private remote access to the full Immich app without binding it on every interface.

## Immich Public Proxy

Public shared links are served by `immich-public-proxy` at `http://127.0.0.1:3000` on the Pi by default. Put a public HTTPS tunnel or reverse proxy in front of that local endpoint for `https://photos.vinayakakv.com`.

Immich Public Proxy runs as a separate Docker Compose project from Immich. Both projects join the shared external Docker network named by `IMMICH_PUBLIC_NETWORK`, and the proxy talks to Immich at `http://immich-server:2283` over that private Docker network.

## Cloudflare Tunnel

Create a remotely managed tunnel in the Cloudflare Zero Trust dashboard, then use the token from the setup screen as `CLOUDFLARED_TUNNEL_TOKEN`.

In the tunnel's public hostname settings, route:

```text
photos.vinayakakv.com -> HTTP -> immich-public-proxy:3000
```

The `cloudflared` container runs in its own Compose project and joins the same `IMMICH_PUBLIC_NETWORK` Docker network as Immich Public Proxy. It does not publish any host ports.

# Jellyfin

Docker Compose stack for [Jellyfin](https://jellyfin.org), served behind an existing Traefik reverse proxy. The media library is stored encrypted on an SFTP server. An [rclone](https://rclone.org) container decrypts it and serves it read-only over NFS on a private Docker network, and Jellyfin mounts that NFS export at `/media`. NVIDIA hardware transcoding (NVENC/NVDEC) is optional and works on Linux hosts only.

## How it works

```
SFTP server ──sftp──▶ rclone-secret ──nfs──▶ Docker daemon ──volume──▶ jellyfin ◀── Traefik
 (encrypted)          (crypt + VFS cache)     (mounts /media)                        (HTTPS)
```

| Service | Container | Role |
| --- | --- | --- |
| `jellyfin` | `<SERVICE_NAME>` | The media server. Mounts the NFS export read-only at `/media` |
| `rclone-secret` | `<SERVICE_NAME>-rclone-secret` | Connects to the SFTP server, decrypts the files with an rclone `crypt` remote, and serves them with `rclone serve nfs` on port 2049 |
| `watchdog` | `<SERVICE_NAME>-watchdog` | Starts Jellyfin once rclone is healthy, and restarts Jellyfin whenever rclone restarts so the NFS mount never goes stale |

The watchdog exists because Docker ignores `depends_on` when it restarts containers itself (for example when Docker Desktop launches). `depends_on` still applies to `docker compose up`.

Two `/30` networks are derived from `SUBNET_PREFIX` (default `192.168.100`):

| Network | Subnet | Addresses |
| --- | --- | --- |
| `<SERVICE_NAME>` | `<prefix>.0/30` | Gateway `.1`, Jellyfin `.2` |
| `<SERVICE_NAME>-nfs` | `<prefix>.4/30` | Gateway `.5`, rclone `.6` |

## Files

| File | Purpose |
| --- | --- |
| `docker-compose.yml` | The services, networks, volumes and Traefik labels |
| `.env.example` | Template for `.env`, which holds all the settings and is not committed |

## Prerequisites

- Docker Engine with the Compose plugin, or Docker Desktop
- A running Traefik instance that has:
  - a `websecure` entrypoint with a default (wildcard) TLS certificate
  - a `strict-path` middleware defined in its file provider (`strict-path@file`)
  - a route to Jellyfin's address (`<SUBNET_PREFIX>.2`). The Jellyfin network is a `/30` with no free address for another container, so Traefik reaches it through the gateway (for example Traefik running with host networking)
- A DNS record for `DOMAIN` that points at the Traefik host
- An SFTP account holding media that was encrypted with an rclone `crypt` remote, and the two crypt passwords used for it
- `rclone` on your workstation, to obscure the passwords

## Setup

1. Create the environment file:

   ```sh
   cp .env.example .env
   chmod 600 .env
   ```

2. Obscure the SFTP password and the two crypt passwords. rclone expects obscured values, not plaintext:

   ```sh
   rclone obscure 'sftp-password'
   rclone obscure 'crypt-password'
   rclone obscure 'crypt-salt'
   ```

   The crypt passwords must be the same ones used when the media was encrypted. Keep an offline copy of them: without them the media cannot be decrypted.

3. Edit `.env`. At minimum set `DOMAIN`, the `SFTP_*` values and the two `RCLONE_CONFIG_SECRET_PASSWORD*` values (see [Configuration](#configuration)).

4. Start the stack:

   ```sh
   docker compose up -d
   ```

5. Check the rclone logs. Its health check only tests that the NFS port is open, and rclone connects to SFTP lazily, so wrong credentials do not fail it:

   ```sh
   docker compose logs rclone-secret
   docker compose exec jellyfin ls /media
   ```

6. Open `https://<DOMAIN>` and finish the setup wizard. Add the libraries from `/media`. Then go to **Dashboard → Networking → Known proxies** and add the network gateway (`<SUBNET_PREFIX>.1`, `192.168.100.1` by default). Without this step Jellyfin logs every client as coming from the proxy, not from its real address.

## Configuration

All settings are in `.env`. The defaults in `docker-compose.yml` match `.env.example`.

### Required

| Variable | Example | Description |
| --- | --- | --- |
| `DOMAIN` | `media.example.com` | Public hostname. Used for the Traefik router and for `JELLYFIN_PublishedServerUrl` |
| `SFTP_HOST` | `storage.box.local` | SFTP server |
| `SFTP_USER` | | SFTP user name |
| `SFTP_PATH` | `media` | Folder on the SFTP server that holds the encrypted files |
| `SFTP_PASS` or `SFTP_KEY_PEM` | | Set one. `SFTP_PASS` is the **obscured** password. `SFTP_KEY_PEM` is the private key on one line, with literal `\n` for line breaks |
| `RCLONE_CONFIG_SECRET_PASSWORD` | | Obscured crypt password |
| `RCLONE_CONFIG_SECRET_PASSWORD2` | | Obscured crypt salt (password2) |

### Optional

| Variable | Default | Description |
| --- | --- | --- |
| `SERVICE_NAME` | `jellyfin` | Names the Compose project, containers, Traefik router, networks and volumes. Change it to run several stacks side by side |
| `JELLYFIN_TAG` | `12.2` | Jellyfin image tag. Pin an exact release |
| `RCLONE_TAG` | `1.75` | rclone image tag |
| `DOCKER_CLI_TAG` | `29.9.0-cli-alpine3.24` | Docker CLI image tag used by the watchdog |
| `SUBNET_PREFIX` | `192.168.100` | First three octets of the two `/30` networks. Must not overlap your LAN, VPN or other Docker networks |
| `SFTP_PORT` | `22` | SFTP port |
| `SFTP_SHELL_TYPE` | `unix` | Remote shell type. Setting it skips auto-detection on connect |
| `RCLONE_CACHE_MAX_SIZE` | `5G` | Maximum size of the rclone VFS cache |
| `RCLONE_CACHE_MAX_AGE` | `72h` | How long unused files stay in the cache |
| `RCLONE_CACHE_MIN_FREE` | `20G` | Free space rclone leaves on the Docker disk |
| `IPV4_ALLOWLIST` | `0.0.0.0/0` | IPv4 CIDR(s) allowed through Traefik |
| `IPV6_ALLOWLIST` | `::/0` | IPv6 CIDR(s) allowed through Traefik |
| `GPU_RUNTIME` | `runc` | Container runtime. Set to `nvidia` for hardware transcoding |
| `GPU_DEVICES` | `all` | GPUs exposed to the container (`NVIDIA_VISIBLE_DEVICES`). Ignored unless `GPU_RUNTIME=nvidia` |
| `GPU_CAPABILITIES` | `compute,utility,video` | Driver capabilities (`NVIDIA_DRIVER_CAPABILITIES`). `video` is required for NVENC/NVDEC |
| `TZ` | `Europe/Athens` | Time zone for all containers |
| `JELLYFIN_CPU_LIMIT` | `2` | Jellyfin CPU limit |
| `JELLYFIN_MEMORY_LIMIT` | `1G` | Jellyfin memory limit |
| `JELLYFIN_MEMORY_RESERVATION` | `512M` | Jellyfin memory reservation |
| `JELLYFIN_PIDS_LIMIT` | `128` | Jellyfin process limit |
| `RCLONE_CPU_LIMIT` | `1` | rclone CPU limit |
| `RCLONE_MEMORY_LIMIT` | `512M` | rclone memory limit |
| `RCLONE_MEMORY_RESERVATION` | `256M` | rclone memory reservation |
| `RCLONE_PIDS_LIMIT` | `128` | rclone process limit |

Software transcoding needs about 1–2 CPUs per 1080p stream, so raise `JELLYFIN_CPU_LIMIT` if you expect several streams at once and are not using hardware transcoding. Decryption is CPU-bound too: a 4K stream needs a full core, so raise `RCLONE_CPU_LIMIT` for several high-bitrate streams.

The rclone cache limits matter. In `full` cache mode rclone otherwise keeps every file it has read until the Docker disk is full.

## Hardware transcoding

Supported for NVIDIA GPUs (NVENC/NVDEC) on Linux hosts. It is off by default (`GPU_RUNTIME=runc`). Docker Desktop on macOS cannot pass a GPU to containers, so on a Mac leave it as `runc` and transcoding runs on the CPU.

1. Install the NVIDIA driver and the [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) on the host, then register the runtime with Docker:

   ```sh
   sudo nvidia-ctk runtime configure --runtime=docker
   sudo systemctl restart docker
   nvidia-smi                # should list the GPU
   ```

2. Set the runtime in `.env`:

   ```dotenv
   GPU_RUNTIME=nvidia
   ```

   To expose only specific GPUs, also set `GPU_DEVICES` (e.g. `0` or a GPU UUID).

3. Recreate the container with `docker compose up -d`, then select **NVIDIA NVENC** under **Dashboard → Playback → Transcoding**.

## Volumes

Volume names are prefixed with `SERVICE_NAME` (`jellyfin` by default).

| Volume | Mount point | Contents |
| --- | --- | --- |
| `jellyfin-config` | `/config` | Database, settings, metadata. **Back this up** |
| `jellyfin-cache` | `/cache` | Image cache and transcode segments. These can take several GB per active stream |
| `jellyfin-media` | `/media` (read-only) | NFS mount of the decrypted library, served by rclone |
| `jellyfin-fonts` | `/usr/local/share/fonts/custom` (read-only) | Extra fonts for subtitle rendering |
| `jellyfin-rclone-cache` | `/cache` in `rclone-secret` | rclone VFS cache of decrypted file chunks. Safe to delete |

The media volume is mounted with `nocopy`. Without it Docker would mount the NFS export when the container is created, before rclone is running.

The fonts volume is read-only inside the container. To add fonts, copy them in from a helper container:

```sh
docker run --rm -v jellyfin-fonts:/fonts -v "$PWD/fonts:/src:ro" alpine cp -r /src/. /fonts/
docker compose restart jellyfin
```

## Operations

```sh
docker compose ps                         # status and health
docker compose logs -f jellyfin           # follow Jellyfin logs
docker compose logs -f rclone-secret      # follow rclone logs (SFTP/decryption errors)
docker compose logs -f watchdog           # see when the watchdog starts or restarts Jellyfin
docker compose pull && docker compose up -d   # upgrade after changing an image tag
```

While the watchdog runs, it starts a stopped Jellyfin again within about 15 seconds. To keep Jellyfin down, stop the watchdog first:

```sh
docker compose stop watchdog jellyfin
```

Back up the config volume (stop Jellyfin first so the database is consistent):

```sh
docker compose stop watchdog jellyfin
docker run --rm -v jellyfin-config:/config:ro -v "$PWD:/backup" alpine \
  tar czf /backup/jellyfin-config-$(date +%F).tar.gz -C /config .
docker compose start jellyfin watchdog
```

If `/media` is empty or returns I/O errors, check that `rclone-secret` is healthy and its logs show no SFTP or crypt errors, then restart Jellyfin to remount the share.

## Security

- No ports are published on the host. All web traffic goes through Traefik, and the NFS export is only reachable on the private `<SERVICE_NAME>-nfs` network.
- rclone runs with a read-only root filesystem, all capabilities dropped and `no-new-privileges`, and serves the media read-only.
- Jellyfin runs with `no-new-privileges`. All containers have CPU, memory and process limits.
- The watchdog mounts the Docker socket, which gives it full control of the Docker host. It only runs `docker inspect`, `start` and `restart`, has no network, a read-only filesystem and no capabilities.
- `IPV4_ALLOWLIST` / `IPV6_ALLOWLIST` restrict who can reach the site. The defaults allow everyone.
- `.env` holds the SFTP credentials and the crypt passwords. Obscured values are not encrypted: anyone who reads them can recover the passwords. `.env` is git-ignored; keep it at mode `600`.

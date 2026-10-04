# Jellyfin

Docker Compose stack for [Jellyfin](https://jellyfin.org), served behind an existing Traefik reverse proxy. The media library is mounted read-only from a network share (NFS or CIFS). Intel/AMD hardware transcoding is optional.

## Files

| File | Purpose |
| --- | --- |
| `docker-compose.yml` | The Jellyfin service, its network, volumes and Traefik labels |
| `docker-compose.hwaccel.yml` | Optional override that passes `/dev/dri` into the container for hardware transcoding |
| `.env.example` | Template for `.env`, which holds all the settings and is not committed |

## Prerequisites

- Docker Engine with the Compose plugin
- A running Traefik instance that has:
  - a `websecure` entrypoint with a default (wildcard) TLS certificate
  - a `strict-path` middleware defined in its file provider (`strict-path@file`)
- A DNS record for `DOMAIN` that points at the Traefik host
- An NFS or CIFS export holding the media library, reachable from the Docker host

## Setup

1. Create the environment file:

   ```sh
   cp .env.example .env
   chmod 600 .env
   ```

2. Edit `.env`. At minimum set `DOMAIN` and the three `MEDIA_MOUNT_*` values (see [Configuration](#configuration)).

3. Start the stack:

   ```sh
   docker compose up -d
   ```

4. Connect Traefik to the `jellyfin` network so it can reach the container:

   ```sh
   docker network connect jellyfin <traefik-container>
   ```

5. Open `https://<DOMAIN>` and finish the setup wizard. Then go to **Dashboard → Networking → Known proxies** and add the network gateway (`JELLYFIN_GATEWAY`, `172.30.0.1` by default). Without this step Jellyfin logs every client as coming from the proxy, not from its real address.

## Configuration

All settings are in `.env`.

### Required

| Variable | Example | Description |
| --- | --- | --- |
| `DOMAIN` | `media.example.com` | Public hostname. Used for the Traefik router and for `JELLYFIN_PublishedServerUrl` |
| `MEDIA_MOUNT_TYPE` | `nfs` | `nfs` or `cifs` |
| `MEDIA_MOUNT_O` | `addr=192.168.1.10,nfsvers=4.1,ro,hard` | Mount options |
| `MEDIA_MOUNT_DEVICE` | `:/export/media` | Export path (`:/path` for NFS, `//host/share` for CIFS) |

CIFS example:

```dotenv
MEDIA_MOUNT_TYPE=cifs
MEDIA_MOUNT_O=addr=192.168.1.10,username=u,password=p,ro
MEDIA_MOUNT_DEVICE=//192.168.1.10/media
```

### Optional

| Variable | Default | Description |
| --- | --- | --- |
| `JELLYFIN_TAG` | `12.1` | Image tag. Pin an exact release |
| `TZ` | `Europe/Athens` | Container time zone |
| `JELLYFIN_SUBNET` | `172.30.0.0/16` | Subnet of the `jellyfin` network. Must not overlap other Docker networks |
| `JELLYFIN_GATEWAY` | `172.30.0.1` | Gateway of that network, which is the address Traefik connects from |
| `IPV4_ALLOWLIST` | `0.0.0.0/0` | IPv4 CIDR(s) allowed through Traefik |
| `IPV6_ALLOWLIST` | `::/0` | IPv6 CIDR(s) allowed through Traefik |
| `JELLYFIN_CPU_LIMIT` | `2` | CPU limit |
| `JELLYFIN_MEMORY_LIMIT` | `2G` | Memory limit |
| `JELLYFIN_MEMORY_RESERVATION` | `512M` | Memory reservation |
| `JELLYFIN_PIDS_LIMIT` | `1024` | Process limit |

Software transcoding needs about 1–2 CPUs per 1080p stream, so raise `JELLYFIN_CPU_LIMIT` if you expect several streams at once and are not using hardware transcoding.

## Hardware transcoding

Supported for Intel (QSV/VA-API) and AMD (VA-API) GPUs.

1. Check that the host exposes a render node:

   ```sh
   ls -l /dev/dri            # should list renderD128
   stat -c %g /dev/dri/renderD128
   ```

2. Uncomment these lines in `.env`, with `RENDER_GID` set to the GID printed above:

   ```dotenv
   COMPOSE_FILE=docker-compose.yml:docker-compose.hwaccel.yml
   RENDER_GID=993
   ```

3. Recreate the container with `docker compose up -d`, then pick the method under **Dashboard → Playback → Transcoding**.

## Volumes

| Volume | Mount point | Contents |
| --- | --- | --- |
| `jellyfin-config` | `/config` | Database, settings, metadata. **Back this up** |
| `jellyfin-cache` | `/cache` | Image cache and transcode segments. These can take several GB per active stream |
| `jellyfin-media` | `/media` (read-only) | The network share |
| `jellyfin-fonts` | `/usr/local/share/fonts/custom` (read-only) | Extra fonts for subtitle rendering |

The fonts volume is read-only inside the container. To add fonts, copy them in from a helper container:

```sh
docker run --rm -v jellyfin-fonts:/fonts -v "$PWD/fonts:/src:ro" alpine cp -r /src/. /fonts/
docker compose restart jellyfin
```

## Operations

```sh
docker compose ps                 # status and health
docker compose logs -f jellyfin   # follow logs
docker compose pull && docker compose up -d   # upgrade after changing JELLYFIN_TAG
```

Back up the config volume (stop the container first so the database is consistent):

```sh
docker compose stop jellyfin
docker run --rm -v jellyfin-config:/config:ro -v "$PWD:/backup" alpine \
  tar czf /backup/jellyfin-config-$(date +%F).tar.gz -C /config .
docker compose start jellyfin
```

## Security

- The container runs with `no-new-privileges` and has CPU, memory and process limits.
- No ports are published on the host. All traffic goes through Traefik.
- `IPV4_ALLOWLIST` / `IPV6_ALLOWLIST` restrict who can reach the site. The defaults allow everyone.
- `.env` may contain share credentials. It is git-ignored; keep it at mode `600`.

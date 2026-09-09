# SoulSync + slskd on MOS

This guide documents a practical MOS deployment of
[SoulSync](https://github.com/Nezreka/SoulSync) together with
[slskd](https://github.com/slskd/slskd).

The goal is to keep both containers independent while sharing the same download
directory, so SoulSync can process files downloaded by slskd and move organized
music into the final media library.

## Template status

- Upstream reviewed
- MOS templates prepared
- MOS validation pending

## Recommended layout

A simple MOS layout is:

```text
slskd
  |
  | downloads
  v
/mnt/user/downloads/soulsync
  |
  +--> slskd:    /downloads
  |
  +--> SoulSync: /app/downloads
                    |
                    v
               processing
                    |
                    v
                /app/Transfer
                    |
                    v
             final music library
```

The important part is that both containers see the **same host directory** for
downloads.

## SoulSync persistent directories

Recommended mappings:

| Host path | Container path | Purpose |
|---|---|---|
| `/mnt/cache/appdata/soulsync/config` | `/app/config` | SoulSync configuration |
| `/mnt/cache/appdata/soulsync/data` | `/app/data` | Persistent application/database data |
| `/mnt/cache/appdata/soulsync/logs` | `/app/logs` | Logs |
| `/mnt/cache/appdata/soulsync/staging` | `/app/Staging` | Staging/import area |
| `/mnt/user/downloads/soulsync` | `/app/downloads` | Shared slskd download directory |
| `/mnt/user/music` | `/app/Transfer` | Final organized music library |
| `/mnt/user/music-videos` | `/app/MusicVideos` | Optional music-video destination |

The public template uses generic MOS paths. Adjust them to match your actual
storage layout.

## User and group IDs

The MOS template defaults to:

```text
PUID=500
PGID=500
UMASK=022
```

Both SoulSync and slskd should be able to read and write the shared download
directory.

Using compatible UID/GID values across both containers makes shared-directory
permissions much easier to manage.

## slskd integration

For slskd, map the same host directory used by SoulSync:

```text
Host:
/mnt/user/downloads/soulsync

Container:
/downloads
```

For SoulSync:

```text
Host:
/mnt/user/downloads/soulsync

Container:
/app/downloads
```

From the host's point of view, both paths refer to the same files.

This avoids copying downloads between containers.

## Final music library

SoulSync writes organized music to:

```text
/app/Transfer
```

A typical MOS mapping is:

```text
/mnt/user/music -> /app/Transfer
```

Media servers can then mount the same host library read-only.

Example:

```text
Navidrome:
/mnt/user/music -> /music:ro
```

The same principle can be used with Jellyfin or Plex.

## Networking between SoulSync and slskd

There are two reasonable approaches.

### Shared Docker network

The preferred approach is to attach both containers to the same user-defined
Docker network.

SoulSync can then reach slskd using its container name or network alias.

Example:

```text
http://slskd:5030
```

This avoids depending on the MOS host address.

### Host gateway

The SoulSync template also includes:

```text
--add-host=host.docker.internal:host-gateway
```

This makes `host.docker.internal` available inside the container.

It can be useful when SoulSync must reach a service exposed through the MOS host
rather than through a shared Docker network.

Use the shared Docker network when possible.

## OAuth callback ports

The template exposes:

```text
8888/TCP  Spotify OAuth callback
8889/TCP  Tidal OAuth callback
```

The callback port configured in SoulSync must match the Docker port mapping and
the redirect URI configured with the provider.

If these integrations are not used, the ports can be left unused.

# Migrating an existing SoulSync installation

This section applies when moving an existing Docker, Unraid, NAS, or Linux
SoulSync deployment to MOS.

## 1. Stop SoulSync

Stop the current SoulSync container before copying persistent data.

This prevents the database or configuration from changing while it is being
copied.

## 2. Identify every existing mount and Docker volume

Older or customized SoulSync deployments may use a mixture of bind mounts,
named volumes, and anonymous Docker volumes.

Do not assume that an anonymous volume is empty.

Inspect the existing container and identify anything mounted at locations such
as:

```text
/app/config
/app/data
/app/logs
/app/downloads
/app/Transfer
/app/MusicVideos
/app/Staging
/app/scripts
```

Only after the contents have been inspected should an old volume be considered
disposable.

## 3. Back up the existing state

Back up the current configuration, database, and any other persistent SoulSync
state before migration.

At minimum, preserve the data required for:

```text
/app/config
/app/data
```

Also preserve any custom scripts or other files that are actually in use.

## 4. Copy persistent data to MOS

Recommended destinations:

```text
/mnt/cache/appdata/soulsync/config
/mnt/cache/appdata/soulsync/data
/mnt/cache/appdata/soulsync/logs
/mnt/cache/appdata/soulsync/staging
```

Make sure the migrated files are accessible to the UID/GID configured in the
MOS template.

## 5. Keep internal paths stable during the first migration

Avoid changing several things at once.

For the first MOS startup, preserve the important container-side paths whenever
possible:

```text
/app/downloads
/app/Transfer
/app/config
/app/data
```

Once the migrated installation is confirmed working, host paths can be cleaned
up separately if desired.

## 6. Validate the shared download path

Before running any automation, verify that a test file created in the shared
host directory appears in both containers.

For example:

```text
slskd:    /downloads
SoulSync: /app/downloads
```

Both must reference the same underlying host files.

## 7. Validate the final library path

Confirm that SoulSync can write to:

```text
/app/Transfer
```

and that the resulting files are immediately visible to the media server using
the same host library.

## Deep Scan caution

Before running a Deep Scan on a migrated library, confirm that:

- the SoulSync database has migrated correctly
- `/app/Transfer` points to the expected existing music library
- `/app/Staging` points to the intended staging directory
- file permissions are correct
- a backup exists

Do not use Deep Scan as the first migration test against an important library.

Start with normal application startup and a small controlled test.

# Validation checklist

Before considering the MOS deployment complete, verify:

- SoulSync Web UI opens on port `8008`
- the existing configuration is present
- the database is present
- slskd is reachable from SoulSync
- the shared download directory is visible to both containers
- SoulSync can process a small test download
- the organized file reaches `/app/Transfer`
- Navidrome/Jellyfin/Plex can see the resulting file
- OAuth integrations work if used
- logs persist after container restart
- configuration persists after container restart

# Rollback

Keep the previous SoulSync deployment and its persistent data unchanged until
the MOS installation has passed validation.

If rollback is required:

1. Stop the MOS SoulSync container.
2. Do not alter the original backup.
3. Restart the previous deployment with its original mounts and volumes.
4. Investigate the MOS paths, permissions, or networking before trying again.

# Security notes

- Do not publish Soulseek, Spotify, Tidal, or other credentials in the template.
- Keep private tokens and passwords out of GitHub.
- Prefer a shared Docker network for container-to-container communication.
- Expose only the ports actually required.
- Avoid unnecessarily exposing the Web UI to the public Internet.
- Back up `/app/config` and `/app/data`.

# Upstream resources

- SoulSync: https://github.com/Nezreka/SoulSync
- SoulSync issues: https://github.com/Nezreka/SoulSync/issues
- slskd: https://github.com/slskd/slskd

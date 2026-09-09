# qBittorrent on MOS

This template deploys the official headless qBittorrent image:

```text
qbittorrentofficial/qbittorrent-nox:latest
```

It is intended for both new MOS installations and migrations of existing qBittorrent Docker deployments.

## Template status

- Upstream reviewed
- MOS template prepared
- MOS validation pending

## Persistent data

The public MOS template maps:

```text
/mnt/cache/appdata/qbittorrent -> /config
/mnt/user/downloads            -> /downloads
```

`/mnt/cache/...` is used as the generic MOS Hub convention for appdata.

Replace `cache` with the name of the pool used for appdata on your MOS system when required.

## Critical migration data

The most important directory to preserve is:

```text
/config
```

For existing qBittorrent installations, copy the complete configuration directory.

Do not copy only `qBittorrent.conf`.

The session and torrent state stored under:

```text
/config/qBittorrent/BT_backup
```

is especially important because it contains the torrent metadata and fast-resume state required to restore existing torrents correctly.

## Downloads path

The public template uses:

```text
/downloads
```

as the main container-side download path.

During migration, preserve existing internal container paths whenever possible.

If the old qBittorrent deployment uses additional paths such as:

```text
/data
/downloads
/media
```

do not change those paths during the first migration if existing torrents reference them.

Changing the platform and torrent paths at the same time can cause torrents to appear missing or require manual path correction.

# Legal notice

The official qBittorrent container requires acknowledgment of its legal notice.

The MOS template exposes:

```text
QBT_LEGAL_NOTICE
```

Leave it empty by default.

After reading the notice, enter:

```text
confirm
```

when you want to acknowledge it.

The public template must not pre-fill this value.

# Web UI

The template defaults to:

```text
QBT_WEBUI_PORT=8080
```

with the mapping:

```text
8080 -> 8080/TCP
```

The internal qBittorrent WebUI port and Docker port mapping must match.

On recent qBittorrent releases, the first startup may generate a temporary WebUI password and print it in the container logs.

Review the startup logs when performing a fresh installation.

# BitTorrent listening port

The public template defaults to:

```text
QBT_TORRENTING_PORT=6881
```

with:

```text
6881 -> 6881/TCP
6881 -> 6881/UDP
```

Both TCP and UDP mappings should use the same listening port.

During migration, preserve the existing torrent port if possible.

For example, an existing deployment using:

```text
51822/TCP
51822/UDP
```

should initially keep:

```text
QBT_TORRENTING_PORT=51822
```

and map:

```text
51822 -> 51822/TCP
51822 -> 51822/UDP
```

This avoids unnecessary changes to router, firewall, NAT, tracker, or client configuration.

# User and group IDs

The MOS template defaults to:

```text
PUID=500
PGID=500
UMASK=022
```

These values follow the MOS convention for containers supporting configurable UID/GID values.

Make sure both `/config` and the download directories are writable by the configured user and group.

Change the IDs if the migrated files use different ownership.

# Graceful shutdown

The template uses:

```text
--stop-timeout 1800
```

This gives qBittorrent enough time to shut down cleanly and flush session state before Docker stops the container.

This is particularly important for installations with many active torrents.

The template also uses:

```text
--tmpfs /tmp
```

for temporary files.

# Migrating an existing qBittorrent installation

## 1. Stop qBittorrent cleanly

Stop the old qBittorrent container and allow it to shut down normally.

Do not copy the configuration while qBittorrent is actively writing session state.

## 2. Back up the complete configuration

Back up the entire directory mapped to:

```text
/config
```

Verify that the backup includes:

```text
/config/qBittorrent/BT_backup
```

Keep the original installation unchanged until MOS has been fully validated.

## 3. Record the existing path mappings

Before migration, record all current host-to-container mappings.

Examples:

```text
Host path                    Container path
------------------------------------------------
/mnt/user/downloads          /downloads
/mnt/user/data               /data
```

Existing torrents store save paths using the container-visible paths.

Preserve these paths during the first MOS migration.

## 4. Record the listening port

Record the current incoming BitTorrent port and keep it unchanged during migration when practical.

Preserve both TCP and UDP mappings.

## 5. Copy `/config` to MOS

The generic MOS destination is:

```text
/mnt/cache/appdata/qbittorrent
```

Adjust the pool name if necessary.

Copy the complete old `/config` contents into the new MOS appdata directory before starting qBittorrent.

## 6. Recreate required download mappings

Recreate every container-side path still referenced by active torrents.

For an existing deployment, this may mean temporarily adding additional paths beyond the simple public template.

The goal of the first migration is compatibility, not cleanup.

## 7. Check ownership and permissions

Ensure the copied configuration and download paths are accessible by:

```text
PUID
PGID
```

configured in the MOS template.

Do not recursively change permissions on a large media library unless necessary.

## 8. Start qBittorrent and inspect logs

Start the MOS container and immediately review the logs.

Check for:

- permission errors
- invalid configuration
- WebUI startup errors
- temporary WebUI password information
- torrent path errors
- port binding errors

## 9. Validate existing torrents

Before making further changes, verify:

- all expected torrents are present
- completed torrents remain completed
- seeding torrents remain available
- save paths are correct
- categories are present
- tags are present
- tracker URLs are present
- ratios and statistics look reasonable
- automatic torrent management behaves as expected
- paused/running states are sensible

Do not trigger a full recheck of every torrent unless it is actually required.

## 10. Test new downloads

Add a small legal test torrent and verify:

- download starts
- files are written to the expected path
- permissions are correct
- incoming connectivity works
- the torrent survives a container restart

# Migration strategy

Avoid changing multiple variables at once.

Recommended order:

```text
1. Preserve /config
2. Preserve internal paths
3. Preserve torrent port
4. Start on MOS
5. Validate all existing torrents
6. Validate restart persistence
7. Only then clean up paths or change ports
```

This makes rollback and troubleshooting much easier.

# Validation checklist

Before considering the migration complete, verify:

- WebUI is reachable
- existing torrents are present
- no torrents unexpectedly show as missing
- expected seeding torrents are active
- trackers respond normally
- categories and tags are preserved
- save paths are correct
- TCP and UDP listening ports are correct
- incoming connectivity works
- new downloads work
- container restart preserves state
- graceful shutdown completes without corruption

# Security notes

- Do not expose the WebUI directly to the public Internet.
- Prefer LAN, VPN, or an authenticated reverse proxy.
- Use a strong WebUI password.
- Keep `/config` backed up.
- Expose only the torrenting ports that are actually required.
- Do not publish credentials or private tracker information in the public MOS repository.

# Rollback

Keep the old qBittorrent installation unchanged until the MOS deployment has been fully validated.

If rollback is required:

1. Stop the MOS qBittorrent container.
2. Do not modify the original backup.
3. Restart the previous qBittorrent container with its original `/config`.
4. Restore the original port mappings and paths if they were changed.
5. Confirm that torrents return to their previous state.

# Upstream resources

- qBittorrent: https://www.qbittorrent.org/
- Official Docker image: https://hub.docker.com/r/qbittorrentofficial/qbittorrent-nox
- Docker project: https://github.com/qbittorrent/docker-qbittorrent-nox
- Docker image issues: https://github.com/qbittorrent/docker-qbittorrent-nox/issues

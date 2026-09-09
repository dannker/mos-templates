# Duplicacy Web on MOS

This template deploys [Duplicacy Web](https://duplicacy.com/) using the
`saspus/duplicacy-web:mini` container image.

The template is intended for both new installations and migrations of existing
Duplicacy Web deployments.

## Template status

- Upstream reviewed
- MOS template prepared
- MOS validation pending

## Persistent directories

| Container path | MOS default path | Purpose |
|---|---|---|
| `/config` | `/mnt/cache/appdata/duplicacy` | Configuration and persistent machine ID |
| `/cache` | `/mnt/cache/appdata/duplicacy/cache` | Temporary cache |
| `/logs` | `/mnt/cache/appdata/duplicacy/logs` | Duplicacy Web logs |

The `/config` directory is the most important directory to preserve when
backing up or migrating an existing installation.

The `/cache` directory contains temporary data and can be recreated.

## User and group IDs

The MOS template defaults to:

```text
USR_ID=500
GRP_ID=500
```

These values follow the MOS convention for containers supporting configurable
user and group IDs.

Change them if the ownership of your source data requires different IDs.

## Duplicacy Web version

The template defaults to:

```text
DUPLICACY_WEB_VERSION=Stable
```

The `mini` image downloads the selected Duplicacy Web release during startup.

`Stable` is recommended for normal installations.

# Migrating an existing installation

## Preserve `/config`

Stop the existing Duplicacy container before copying its data.

Copy the complete directory currently mapped to:

```text
/config
```

to:

```text
/mnt/cache/appdata/duplicacy
```

Do not copy only individual configuration files. Preserve the complete contents
of `/config`.

## Preserve the container hostname

Duplicacy licensing uses information associated with the machine running the
application.

The container persists its machine ID inside `/config`, while the Docker
hostname can also affect an existing license.

For a fresh MOS installation the template uses:

```text
duplicacy-mos
```

When migrating an existing licensed installation, use the hostname from the
previous Duplicacy container instead.

Therefore, preserve both:

```text
/config
container hostname
```

before starting Duplicacy on MOS.

Do not start an empty installation using the final migration hostname before
copying the existing `/config`.

# Backup source directories

Backup source directories are intentionally not included in the public MOS
template because every installation has a different storage layout.

Add the directories you want Duplicacy to access after installing the
container.

Example:

```text
Host:
/mnt/data/documents

Container:
/backuproot/documents

Mode:
ro
```

For normal backup jobs, mounting source data as read-only is recommended.

This prevents the backup container from accidentally modifying source files.

## Restores

A read-only mount is suitable for backups but cannot be used as a restore
destination.

When restoring directly into a mounted filesystem, temporarily provide a
writable destination:

```text
Mode:
rw
```

Only grant write access where it is required.

# Validation after migration

Before removing the previous Duplicacy installation, verify:

- The Web UI opens on port `3875`
- Existing storage definitions are present
- Existing backup jobs are present
- Existing schedules are present
- The expected license is detected
- Backup source directories are accessible
- A test backup completes successfully
- A test restore to a temporary directory succeeds

# Rollback

Keep the original Duplicacy installation and its appdata unchanged until the
MOS installation has been fully validated.

If rollback is required, stop the MOS container and restart the previous
Duplicacy installation using its original `/config` directory and hostname.

The backup repositories themselves do not need to be recreated simply because
the Duplicacy Web container has moved to MOS.

# Security

Do not expose the Duplicacy Web interface directly to the public Internet.

Prefer access through your LAN, VPN, or an authenticated reverse proxy.

Backup source directories should be mounted read-only whenever possible.

Never publish storage credentials, encryption passwords or other secrets in a
MOS template or public repository.

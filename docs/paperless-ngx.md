# Paperless-ngx on MOS

This template deploys [Paperless-ngx](https://docs.paperless-ngx.com/) using the official container image:

```text
ghcr.io/paperless-ngx/paperless-ngx:latest
```

It is intended for both new MOS installations and migrations of existing Paperless-ngx deployments.

## Template status

- Upstream reviewed
- MOS template prepared
- MOS validation pending

## Storage layout

The public MOS template uses the following generic paths:

```text
/mnt/cache/appdata/paperless-ngx/data
/mnt/user/paperless/media
/mnt/user/paperless/consume
/mnt/user/paperless/export
```

`/mnt/cache/...` is used as a generic MOS Hub template convention.

Replace `cache` with the name of the pool used for appdata on your MOS system when required.

For example:

```text
/mnt/services/appdata/paperless-ngx/data
```

may be appropriate on a system whose appdata pool is named `services`.

## Persistent directories

| Container path | Purpose |
|---|---|
| `/usr/src/paperless/data` | Application data, search index and SQLite database when SQLite is used |
| `/usr/src/paperless/media` | Original documents, archived documents and thumbnails |
| `/usr/src/paperless/consume` | Incoming documents for automatic consumption |
| `/usr/src/paperless/export` | Export destination |

All four directories should be backed up when they contain data you want to preserve.

## Database mode

This MOS template defaults to:

```text
PAPERLESS_DBENGINE=sqlite
```

This keeps the deployment simple and makes it suitable for a single-container Paperless installation with an external Valkey/Redis-compatible broker.

SQLite is also useful when migrating an existing SQLite-based deployment because it avoids changing both the platform and the database backend at the same time.

A later migration to PostgreSQL can be performed separately if desired.

## Valkey / Redis broker

Paperless requires a Redis-compatible broker.

The template exposes:

```text
PAPERLESS_REDIS
```

A recommended MOS layout is to run a dedicated Valkey instance for Paperless on the same user-defined Docker network.

Example:

```text
valkey-paperless
paperless-ngx
```

Example connection string:

```text
redis://:PASSWORD@valkey-paperless:6379
```

Prefer container-to-container networking instead of publishing Valkey port `6379` to the LAN.

## Secret key

Paperless requires:

```text
PAPERLESS_SECRET_KEY
```

Generate a long random value before first startup.

Do not commit this value to GitHub or include it in public MOS templates.

If a secret key has previously been exposed in screenshots, logs, chats or public files, replace it before completing the migration.

## OCR language

The template exposes:

```text
PAPERLESS_OCR_LANGUAGE
PAPERLESS_OCR_LANGUAGES
```

`PAPERLESS_OCR_LANGUAGE` defines the OCR language or languages used for document processing.

Example:

```text
spa+eng
```

`PAPERLESS_OCR_LANGUAGES` is for additional Tesseract language packages that are not already included in the container image.

Do not confuse the two settings.

## Time zone

The public template defaults to:

```text
PAPERLESS_TIME_ZONE=UTC
```

Change this to the local timezone used by your installation when required.

Example:

```text
Europe/Madrid
```

## Public URL

If Paperless is accessed through a reverse proxy or public hostname, configure:

```text
PAPERLESS_URL
```

Example:

```text
https://paperless.example.com
```

Use the canonical URL that users will actually use to access Paperless.

## File ownership

The template defaults to:

```text
USERMAP_UID=500
USERMAP_GID=500
```

These values follow the MOS convention for containers that support configurable host file ownership.

Make sure the mounted directories are accessible by the configured UID and GID.

Change the values if the ownership of your existing data requires different IDs.

# Migrating an existing Paperless-ngx installation

This section is for users moving an existing Docker, Unraid, NAS, or Linux Paperless deployment to MOS.

## 1. Back up the existing installation

Before migration, back up:

```text
/usr/src/paperless/data
/usr/src/paperless/media
/usr/src/paperless/consume
/usr/src/paperless/export
```

Also record the current environment variables and the broker configuration.

Do not remove the previous installation until the MOS deployment has been fully validated.

## 2. Stop Paperless and its broker

Stop Paperless before copying its persistent data.

For the cleanest migration, also stop or isolate the Redis/Valkey broker used by the existing installation.

This prevents state from changing during the copy.

## 3. Identify the current database backend

Before migration, determine whether the existing deployment uses:

```text
SQLite
PostgreSQL
MariaDB/MySQL
```

This public MOS template is intentionally configured for SQLite.

If the existing deployment already uses PostgreSQL or another supported database, do not convert it to SQLite just to match this template.

Adjust the deployment architecture instead.

## 4. SQLite migration

For an existing SQLite-based installation, preserve the complete Paperless data directory.

The SQLite database is stored inside:

```text
/usr/src/paperless/data
```

Copy the full persistent data directory rather than only the database file.

The MOS destination used by the generic template is:

```text
/mnt/cache/appdata/paperless-ngx/data
```

Adjust the pool name if necessary.

## 5. Media migration

Copy the existing Paperless media directory to the host path mapped to:

```text
/usr/src/paperless/media
```

This directory contains the managed document library and must not be treated as disposable cache.

## 6. Consume and export directories

The `consume` and `export` paths are installation-specific.

Preserve the existing host paths when practical during the first migration.

Changing storage layout and application platform at the same time makes troubleshooting harder.

After the MOS installation is stable, host paths can be reorganized separately if desired.

## 7. Broker migration

Paperless requires a working Redis-compatible broker.

For a new MOS deployment, using a dedicated Valkey instance is recommended.

Example:

```text
valkey-paperless
```

Attach both containers to the same user-defined Docker network and configure `PAPERLESS_REDIS` using the Valkey container name.

Do not expose port `6379` to the public Internet.

## 8. Preserve the secret key

An existing Paperless installation should normally keep its existing:

```text
PAPERLESS_SECRET_KEY
```

during migration so sessions and cryptographic application state remain consistent.

If that key has been exposed, rotate it deliberately and understand the consequences before doing so.

Never copy the real key into the public repository.

## 9. Version upgrades

Do not combine a major Paperless upgrade with the MOS migration unless necessary.

If the existing deployment is significantly older than the target container version, review the official Paperless migration and release documentation first.

For major-version upgrades, follow the required upstream upgrade path before or separately from the platform migration.

## 10. First startup on MOS

Before starting the new container, verify:

- all required directories exist
- ownership and permissions are correct
- `PAPERLESS_REDIS` points to a reachable broker
- `PAPERLESS_SECRET_KEY` is configured
- the selected database backend matches the migrated data
- OCR settings are valid
- the public URL is correct if a reverse proxy is used

Then start Paperless and monitor the logs.

# Validation checklist

Before considering the migration complete, verify:

- the Web UI opens on port `8000`
- existing users can log in
- existing documents are visible
- document previews work
- tags, correspondents and document types are present
- search works
- OCR works on a small test document
- the consume directory imports a test document
- the export directory works
- the Valkey/Redis connection is healthy
- the container survives a restart
- data remains intact after restart

Do not delete the previous installation until these checks have passed.

# Optional later migration to PostgreSQL

If the MOS deployment is initially migrated as SQLite, keep the database migration as a separate maintenance task.

Recommended order:

```text
1. Migrate Paperless to MOS
2. Validate Paperless on SQLite
3. Back up the working MOS deployment
4. Plan the database migration separately
5. Migrate SQLite -> PostgreSQL
6. Validate again
```

This makes rollback and troubleshooting much easier.

# Security notes

- Keep `PAPERLESS_SECRET_KEY` private.
- Keep Valkey/Redis passwords private.
- Do not publish database connection strings containing credentials.
- Prefer a private Docker network for Paperless and Valkey.
- Protect public access with HTTPS and an authenticated reverse proxy.
- Back up both `data` and `media`.
- Review permissions on `consume` and `export`.
- Do not commit real document paths, credentials, or personal URLs to the public MOS repository.

# Rollback

Keep the previous Paperless deployment unchanged until the MOS installation has been validated.

If rollback is required:

1. Stop the MOS Paperless container.
2. Stop the MOS Valkey instance if it was created only for Paperless.
3. Restart the previous Paperless deployment.
4. Restore its original broker connection if necessary.
5. Confirm that users, documents and search behave normally.

# Upstream resources

- Documentation: https://docs.paperless-ngx.com/
- Configuration: https://docs.paperless-ngx.com/configuration/
- Project: https://github.com/paperless-ngx/paperless-ngx
- Issues: https://github.com/paperless-ngx/paperless-ngx/issues

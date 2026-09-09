# Valkey on MOS

This template deploys [Valkey](https://valkey.io/) using the official container image:

```text
valkey/valkey:8.0-alpine
```

It is intended for application caching, queues, session storage, and other Redis-compatible workloads.

## Template status

- Upstream reviewed
- MOS template prepared
- MOS validation pending

## Persistent data

The MOS template maps:

```text
/mnt/cache/appdata/valkey -> /data
```

The `/data` directory contains Valkey persistence files when persistence is enabled.

Back up this directory if the Valkey instance contains data that must survive recreation or migration.

## Runtime user

The template runs Valkey as:

```text
UID=500
GID=500
```

using Docker's `--user=500:500`.

This follows the MOS convention for containers that support configurable user and group ownership.

Make sure the host path mapped to `/data` is writable by UID/GID `500:500`.

## Network access

The template does not publish port `6379` to the host by default.

This is intentional.

For most MOS deployments, Valkey is used by other containers on the same Docker network.

Example:

```text
Docker network: infra

valkey-immich
immich

valkey-nextcloud
nextcloud

valkey-paperless
paperless
```

Applications can connect to Valkey using the container name or network alias.

Example:

```text
redis://:PASSWORD@valkey-paperless:6379
```

Publishing port `6379` to the LAN is usually unnecessary.

If host or LAN access is required, add a TCP port mapping manually and use a strong password.

## Password

The template exposes:

```text
VALKEY_PASSWORD
```

The container starts Valkey with:

```text
--requirepass "$VALKEY_PASSWORD"
```

Use a strong password for any instance that is reachable by other containers or networks you do not fully trust.

Do not commit passwords to GitHub or public MOS templates.

## Maximum memory

The template exposes:

```text
VALKEY_MAXMEMORY
```

Default:

```text
2gb
```

This limits the amount of memory Valkey may use for dataset storage.

Choose a value appropriate for the host and workload.

Do not allocate most of the host RAM to a single Valkey instance unless that is intentional.

## Memory eviction policy

The template exposes:

```text
VALKEY_MAXMEMORY_POLICY
```

Default:

```text
noeviction
```

Two commonly useful policies are:

### `noeviction`

When the configured memory limit is reached, Valkey refuses writes that require additional memory rather than deleting existing keys.

This is often appropriate when applications expect cached or queued data not to disappear unexpectedly.

### `allkeys-lru`

When memory is full, Valkey may evict less recently used keys to make room for new data.

This is appropriate for workloads where Valkey is used primarily as a disposable cache.

Choose the policy based on the application using the instance.

Do not assume the same policy is correct for every application.

## Append-only persistence

The template exposes:

```text
VALKEY_APPENDONLY
```

Default:

```text
no
```

Set:

```text
VALKEY_APPENDONLY=yes
```

to enable append-only persistence.

When append-only persistence is disabled, applications may still be able to repopulate Valkey after a restart if the data is only cache or transient state.

Whether persistence is required depends on the application.

## Recommended deployment model

Use one Valkey container per application when practical.

Example:

```text
valkey-immich
valkey-nextcloud
valkey-paperless
```

This gives each workload independent:

- password
- memory limit
- eviction policy
- persistence settings
- restart lifecycle
- troubleshooting

Avoid sharing one Valkey database between unrelated applications unless the applications explicitly support that deployment model.

# Migrating an existing Valkey or Redis-compatible instance

This section is for users moving an existing Valkey deployment to MOS.

## 1. Identify the current workload

Before migrating, determine:

- which application uses the instance
- whether persistence is enabled
- the current memory limit
- the current eviction policy
- whether authentication is enabled
- whether the application uses a specific database number
- whether the instance is used only as cache or contains important state

Do not copy settings blindly from another application.

## 2. Stop dependent applications

For a clean migration, stop the applications that write to the Valkey instance before copying persistent data.

This avoids changes occurring while the data is being migrated.

## 3. Back up `/data`

If the existing instance persists data, back up its complete `/data` directory or equivalent persistent storage.

Do not delete the previous instance until the MOS deployment has been validated.

## 4. Copy persistent data to MOS

The default MOS destination is:

```text
/mnt/cache/appdata/valkey
```

This is mapped to:

```text
/data
```

inside the container.

Make sure ownership and permissions match the configured runtime UID/GID.

## 5. Recreate the application-specific settings

Configure the MOS instance with the values appropriate for the workload:

```text
VALKEY_PASSWORD
VALKEY_MAXMEMORY
VALKEY_MAXMEMORY_POLICY
VALKEY_APPENDONLY
```

If the old deployment used a different port or Docker network, preserve application connectivity first and simplify the network later.

## 6. Reconnect the application

Attach Valkey and the dependent application to the same user-defined Docker network.

Configure the application to connect using the Valkey container name.

Example:

```text
valkey-paperless:6379
```

Prefer container-to-container networking over exposing `6379` on the MOS host.

## 7. Validate

Before removing the previous instance, verify:

- the Valkey container stays running
- the application can connect
- authentication works
- the expected database is being used
- memory usage stays within the configured limit
- the selected eviction policy is appropriate
- persistence behaves as expected
- application sessions, queues, or cache behave normally
- the application survives a Valkey restart

# Security notes

- Do not expose port `6379` to the public Internet.
- Prefer a private Docker network.
- Use a strong password.
- Do not commit passwords or connection strings containing secrets.
- Run the container without root privileges.
- Back up `/data` when the workload requires persistence.
- Use separate Valkey instances for unrelated applications when practical.

# Rollback

Keep the previous Valkey deployment and its persistent data unchanged until the MOS instance has been validated.

If rollback is required:

1. Stop the MOS Valkey container.
2. Reconnect the dependent application to the previous instance.
3. Start the previous Valkey container if necessary.
4. Confirm that the application works normally.
5. Investigate the MOS network, permissions, password, or persistence settings before trying again.

# Upstream resources

- Valkey: https://valkey.io/
- Docker image: https://hub.docker.com/r/valkey/valkey
- Project: https://github.com/valkey-io/valkey
- Issues: https://github.com/valkey-io/valkey/issues

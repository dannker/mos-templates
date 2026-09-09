# Mumble Server on MOS

This template deploys the official Mumble server image:

```text
mumblevoip/mumble-server:latest
```

It is intended for both new MOS installations and migrations from existing Mumble Docker deployments.

## Template status

- Upstream reviewed
- MOS template prepared
- MOS validation pending

## Persistent data

The MOS template maps:

```text
/mnt/cache/appdata/mumble -> /data
```

The `/data` directory contains the persistent Mumble server database and configuration.

Back up this directory before changing or migrating the container.

## Network ports

Mumble uses the same port for TCP and UDP:

```text
64738/TCP
64738/UDP
```

The public MOS template maps:

```text
64738 -> 64738/TCP
64738 -> 64738/UDP
```

If an existing installation already uses a different host port, that port can be preserved during migration.

For example:

```text
64700 -> 64738/TCP
64700 -> 64738/UDP
```

Keeping the existing host port during the first migration avoids unnecessary changes to clients, firewall rules, NAT rules, or DNS documentation.

## User and group IDs

The template defaults to:

```text
PUID=500
PGID=500
```

These values follow the MOS convention for containers that support configurable user and group IDs.

Change them if the ownership of the migrated `/data` directory requires different values.

## SuperUser password

The template exposes:

```text
MUMBLE_SUPERUSER_PASSWORD
```

This value is optional.

When configured, it sets the password for the Mumble `SuperUser` account.

Keep this value private and do not commit it to a public repository.

## Server password

To require a password for normal users connecting to the server, configure:

```text
MUMBLE_CONFIG_SERVER_PASSWORD
```

Leave it empty for a server without a connection password.

## Maximum users

The template exposes:

```text
MUMBLE_CONFIG_USERS
```

The default value is:

```text
100
```

Adjust it according to the expected number of simultaneous users.

# Migrating an existing Mumble installation

This section is for users moving an existing Mumble Docker deployment to MOS.

It is especially relevant when the previous deployment uses a different Docker image.

## 1. Stop the old container

Stop the current Mumble container before copying its persistent data.

Do not run the old and new Mumble containers against the same database or data directory.

## 2. Back up the existing configuration

Back up the complete persistent directory used by the existing installation.

Preserve:

- the Mumble database
- configuration files
- certificates, if stored there
- channel and user state
- ACL configuration
- registration data
- any custom server settings

Do not delete the old installation until the MOS deployment has been fully validated.

## 3. Identify the current container image

Check which Docker image the existing deployment uses.

Migration from another image to:

```text
mumblevoip/mumble-server:latest
```

may require reviewing paths, ownership, and configuration variables.

Do not assume environment-variable names from another image are compatible with the official image.

## 4. Copy persistent data to MOS

The default MOS destination is:

```text
/mnt/cache/appdata/mumble
```

This directory is mapped to:

```text
/data
```

inside the official Mumble container.

Copy the required existing data before starting the migrated container.

## 5. Check ownership and permissions

Ensure the persistent data can be read and written by the UID and GID configured in the template.

Default MOS values:

```text
PUID=500
PGID=500
```

Adjust them when required by the ownership of the migrated files.

## 6. Review custom configuration files

The official Mumble Docker image supports custom configuration files.

If the deployment uses:

```text
MUMBLE_CUSTOM_CONFIG_FILE
```

review it carefully before migration.

When a custom configuration file is used, settings that would normally be supplied through `MUMBLE_CONFIG_*` environment variables may no longer be applied in the same way.

Do not configure the same option in both places without understanding which configuration source is authoritative.

For migrations, prefer preserving the existing working configuration first. Modernize the configuration only after the server is running correctly on MOS.

## 7. Preserve the current public port when useful

A migration does not require changing the externally visible port.

If the current server uses a host port such as:

```text
64700
```

it can initially remain:

```text
64700 -> 64738/TCP
64700 -> 64738/UDP
```

After the migration is stable, the host port can be changed separately if desired.

## 8. Re-enter secrets

Do not copy passwords or other secrets into the public MOS template.

Re-enter private values directly in MOS when required:

```text
MUMBLE_SUPERUSER_PASSWORD
MUMBLE_CONFIG_SERVER_PASSWORD
```

If a credential has previously been exposed in screenshots, logs, chats, or public files, replace it before completing the migration.

# Validation after migration

Before removing the previous Mumble installation, verify:

- the container remains running
- the server accepts TCP connections
- UDP voice traffic works
- existing channels are present
- registered users are present
- ACLs and permissions are correct
- the expected server name and welcome message are present
- the SuperUser account works
- the server password works if enabled
- existing clients can reconnect
- voice quality and latency are normal
- the server survives a container restart
- persistent data remains intact after restart

A successful connection alone is not enough. Verify that the existing server state has also migrated correctly.

# Rollback

Keep the old Mumble installation unchanged until the MOS deployment has been validated.

If rollback is required:

1. Stop the MOS Mumble container.
2. Restore the previous container configuration if necessary.
3. Start the previous Mumble container using its original persistent data.
4. Confirm that users and channels are intact.

# Security notes

- Do not publish SuperUser or server passwords in GitHub.
- Expose only the TCP and UDP ports required by Mumble.
- Restrict Internet exposure when the server is intended only for LAN or VPN users.
- Keep the persistent `/data` directory backed up.
- Review custom configuration files before publishing examples.
- Avoid running the container as root unless there is a specific requirement.

# Upstream resources

- Docker image: https://hub.docker.com/r/mumblevoip/mumble-server
- Docker project: https://github.com/mumble-voip/mumble-docker
- Mumble project: https://www.mumble.info/
- Docker image issues: https://github.com/mumble-voip/mumble-docker/issues

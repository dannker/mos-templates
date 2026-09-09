# Hermes Agent on MOS

This template deploys [Hermes Agent](https://hermes-agent.nousresearch.com/) using the official Docker image:

```text
nousresearch/hermes-agent:latest
```

It is intended for both new MOS installations and migrations of an existing Hermes Agent container.

## Template status

- Upstream reviewed
- MOS template prepared
- MOS validation pending

## Persistent data

Hermes uses `/opt/data` as the persistent data directory.

The MOS template maps:

```text
/mnt/cache/appdata/hermes-agent -> /opt/data
```

`/opt/data` is the main source of persistent Hermes state and may contain:

- configuration
- API keys and secrets
- profiles
- sessions
- memories
- skills
- cron jobs
- hooks
- logs
- plugins and other user-managed files

Back up this directory before changing or migrating the container.

Do not share the same `/opt/data` directory between two running Hermes containers.

## User and group IDs

The template defaults to:

```text
PUID=500
PGID=500
```

Hermes supports `PUID` and `PGID` as aliases for its runtime UID and GID settings.

Change these values if the ownership of the MOS appdata directory requires different IDs.

## Gateway mode

The template starts Hermes with:

```text
gateway run
```

This is the persistent gateway mode intended for long-running Docker deployments.

## Dashboard

The built-in dashboard uses port:

```text
9119
```

Enable it with:

```text
HERMES_DASHBOARD=1
```

A dashboard reachable over the network requires an authentication provider.

For a trusted LAN or VPN deployment, Hermes supports basic authentication using:

```text
HERMES_DASHBOARD_BASIC_AUTH_USERNAME
HERMES_DASHBOARD_BASIC_AUTH_PASSWORD
HERMES_DASHBOARD_BASIC_AUTH_SECRET
```

The session secret should be a persistent strong random value so dashboard sessions remain valid across container restarts.

Do not expose an unauthenticated Hermes dashboard to the Internet.

## OpenAI-compatible API

Hermes can optionally expose its OpenAI-compatible API on port:

```text
8642
```

The API server is disabled unless:

```text
API_SERVER_ENABLED=true
```

When exposing it outside the container, configure:

```text
API_SERVER_HOST=0.0.0.0
API_SERVER_KEY=<strong-token>
```

Do not publish the API without authentication.

## First-time setup

A fresh Hermes installation must be configured before normal unattended use.

The persistent configuration belongs in `/opt/data`.

If Hermes reports missing or invalid configuration, open a terminal in the container and complete the Hermes setup using the official setup workflow before relying on `gateway run`.

After configuration, restart the container and verify that the gateway starts normally.

# Migrating an existing Hermes Agent installation

This section is for users moving an existing Hermes Agent deployment to MOS.

It is also the procedure intended for migrating an existing Unraid, Docker, NAS, or Linux installation.

## 1. Stop the existing container

Stop the old Hermes Agent container before copying its persistent state.

Do not run the old and new containers simultaneously against the same data directory.

## 2. Identify the existing persistent data

Modern Hermes Docker installations use:

```text
/opt/data
```

as the persistent data directory.

Before migration, inspect the existing container and identify every persistent mount or Docker volume that contains Hermes state.

Older or customized deployments may have additional mounts. Do not assume they are empty or disposable.

## 3. Back up the existing installation

Back up all persistent Hermes data before making changes.

The important goal is to preserve the contents that belong in the new `/opt/data` directory.

Do not copy secrets into the public MOS template or repository.

## 4. Copy data to MOS

The default MOS destination is:

```text
/mnt/cache/appdata/hermes-agent
```

This directory is mapped to:

```text
/opt/data
```

inside the container.

Copy the required existing Hermes state into this directory before the first normal startup of the migrated instance.

## 5. Check permissions

Ensure the MOS appdata directory is writable by the UID and GID configured in the template.

Default template values:

```text
PUID=500
PGID=500
```

Adjust them when required by the ownership of the migrated files.

## 6. Review old custom mounts

Existing installations may contain extra mounts for:

- projects
- external documents
- Obsidian vaults
- imported data
- custom tools
- package-manager directories

These are installation-specific and are intentionally not included in the public MOS template.

Add only the mounts that are actually required.

Do not blindly migrate package-manager or software-installation directories into the new container. Prefer rebuilding required tools from the current Hermes image or a controlled custom image.

## 7. Rotate exposed credentials

If credentials, API keys, gateway tokens, or other secrets have ever been exposed in logs, screenshots, chat messages, or public files, rotate them before completing the migration.

Never commit secrets to the MOS Hub repository.

## 8. Start and validate

Start the MOS container and verify:

- Hermes starts without configuration errors
- `/opt/data` contains the expected state
- profiles are present
- memories and sessions are available where expected
- required skills are present
- the gateway remains running
- configured messaging integrations work
- the dashboard works when enabled
- dashboard authentication is active
- the API works only when intentionally enabled
- required external mounts are accessible

Keep the previous installation available until these checks have passed.

## Rollback

If the migration fails:

1. Stop the MOS Hermes container.
2. Do not modify the original backup.
3. Restart the previous Hermes installation using its original persistent data.
4. Investigate the MOS configuration before attempting the migration again.

# Security notes

Hermes is an agent capable of interacting with tools and external services, so treat its persistent data and network interfaces as sensitive.

Recommended practices:

- keep API keys and tokens out of public templates
- protect the dashboard with authentication
- require an API key when enabling the API server
- prefer LAN or VPN access
- avoid unnecessary host mounts
- do not run the container as root unless there is a specific justified requirement
- back up `/opt/data`
- review permissions after migration

## Upstream documentation

- Docker: https://hermes-agent.nousresearch.com/docs/user-guide/docker/
- Dashboard: https://hermes-agent.nousresearch.com/docs/user-guide/features/web-dashboard/
- Environment variables: https://hermes-agent.nousresearch.com/docs/reference/environment-variables/
- Project: https://github.com/NousResearch/hermes-agent

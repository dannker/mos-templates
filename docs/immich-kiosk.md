# Immich Kiosk on MOS

This template deploys [Immich Kiosk](https://docs.immichkiosk.app/) using the official container image:

```text
ghcr.io/damongolding/immich-kiosk:latest
```

It is intended for both new MOS installations and migrations of existing Immich Kiosk deployments.

## Template status

- Upstream reviewed
- MOS template prepared
- MOS validation pending

## Persistent configuration

The MOS template maps:

```text
/mnt/cache/appdata/immich-kiosk -> /config
```

This directory can contain:

```text
/config/config.yaml
```

and any other local configuration files used by Immich Kiosk.

The public MOS template does not contain personal Immich URLs, API keys, album IDs, person IDs, or passwords.

## Required settings

Immich Kiosk needs access to an Immich server.

The two main settings are:

```text
KIOSK_IMMICH_URL
KIOSK_IMMICH_API_KEY
```

Example:

```text
KIOSK_IMMICH_URL=https://immich.example.com
KIOSK_IMMICH_API_KEY=<your-api-key>
```

The API key is a secret and should never be committed to a public repository.

## Configuration methods

Immich Kiosk can be configured using:

- environment variables
- `/config/config.yaml`

The MOS template exposes the most commonly used options as environment variables.

If the same option is configured in more than one place, review the effective configuration before migration so that old values do not unexpectedly override new ones.

## Main template options

The MOS template includes:

```text
KIOSK_DURATION
KIOSK_ALBUMS
KIOSK_PEOPLE
KIOSK_SHOW_TIME
KIOSK_TIME_FORMAT
KIOSK_SHOW_DATE
KIOSK_DATE_FORMAT
KIOSK_IMAGE_FIT
KIOSK_BACKGROUND_BLUR
KIOSK_TRANSITION
KIOSK_SHOW_PROGRESS_BAR
KIOSK_CACHE
KIOSK_PREFETCH
KIOSK_PASSWORD
```

Only configure the options you actually need.

## Healthcheck

The template uses the built-in healthcheck command:

```text
/kiosk --healthcheck
```

This allows Docker/MOS to detect whether Immich Kiosk is responding correctly.

# Migrating an existing installation

This section is for users moving an existing Immich Kiosk deployment to MOS.

## 1. Stop the old container

Stop the existing Immich Kiosk container before copying or changing its configuration.

## 2. Back up the current configuration

Back up the existing configuration directory and record the environment variables currently in use.

If the existing installation uses a `config.yaml`, preserve it until the MOS installation has been fully validated.

## 3. Copy configuration to MOS

The default MOS destination is:

```text
/mnt/cache/appdata/immich-kiosk
```

This is mapped inside the container as:

```text
/config
```

If you use `config.yaml`, place it at:

```text
/mnt/cache/appdata/immich-kiosk/config.yaml
```

## 4. Review renamed settings

Older Immich Kiosk deployments may use setting names that have since changed.

Common migrations include:

| Older setting | Current setting |
|---|---|
| `KIOSK_REFRESH` | `KIOSK_DURATION` |
| `KIOSK_ALBUM` | `KIOSK_ALBUMS` |
| `KIOSK_PERSON` | `KIOSK_PEOPLE` |
| `KIOSK_SHOW_PROGRESS` | `KIOSK_SHOW_PROGRESS_BAR` |

Do not blindly copy old environment variables into the new MOS template.

Review each setting and use the current variable name.

## 5. Re-enter secrets

For security, do not place secrets in the public template or documentation.

Re-enter the following directly in MOS when required:

```text
KIOSK_IMMICH_API_KEY
KIOSK_PASSWORD
```

If an API key has previously been exposed in screenshots, logs, chats, or public files, revoke it in Immich and create a new one before completing the migration.

## 6. Validate the Immich connection

After starting the MOS container, confirm that:

- the Web UI opens on port `3000`
- Immich Kiosk can reach the configured Immich server
- images load correctly
- album filters work when configured
- people filters work when configured
- slideshow timing is correct
- time/date overlays work when enabled
- image fitting and transitions behave as expected
- the container healthcheck reports healthy

## 7. Validate configuration precedence

If both environment variables and `config.yaml` are used, confirm that the values actually applied by Immich Kiosk match your intended configuration.

Once the MOS deployment is working, remove obsolete settings from the old configuration to avoid confusion.

# Security notes

- Keep `KIOSK_IMMICH_API_KEY` private.
- Do not commit API keys or passwords to GitHub.
- Prefer LAN, VPN, or an authenticated reverse proxy for remote access.
- Use `KIOSK_PASSWORD` when an additional access layer is useful.
- Revoke and replace any credential that has been exposed.

# Rollback

Keep the previous Immich Kiosk installation and configuration unchanged until the MOS deployment has been validated.

If rollback is required:

1. Stop the MOS container.
2. Restart the previous Immich Kiosk container.
3. Restore its original configuration and environment variables if necessary.

# Upstream documentation

- Documentation: https://docs.immichkiosk.app/
- Project: https://github.com/damongolding/immich-kiosk
- Issues: https://github.com/damongolding/immich-kiosk/issues

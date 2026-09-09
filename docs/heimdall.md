# Heimdall on MOS

## Persistent data

Heimdall stores its persistent configuration in:

`/config`

The MOS template maps this to the configured MOS AppData location.

Example:

`/mnt/services/appdata/heimdall -> /config`

## Migrating an existing Heimdall installation

An existing Heimdall installation can normally be migrated by copying
the complete contents of its `/config` volume.

For example, an Unraid installation may use:

`/mnt/user/appdata/heimdall -> /config`

The complete directory should be copied to the Heimdall AppData
directory on MOS while the source and destination containers are stopped.

### PUID / PGID and permissions

LinuxServer containers use `PUID` and `PGID` to determine which user
owns and writes to files stored under `/config`.

When migrating existing AppData, the numeric ownership of the files
must match the `PUID` and `PGID` configured in the MOS container.

For example, a typical Unraid installation may use:

- `PUID=99`
- `PGID=100`

while a new LinuxServer installation commonly uses:

- `PUID=1000`
- `PGID=1000`

Do not blindly use the template defaults when importing existing data.

If Heimdall fails to start and logs errors similar to:

`Permission denied`

for paths under:

`/app/www/storage/`

or the SQLite database, check the ownership of the migrated `/config`
directory and make sure it matches the configured PUID/PGID.

Example:

`stat -c '%u:%g %n' /path/to/heimdall`

The important values are the numeric UID and GID, not the displayed
user/group names.

## Internal service requests

By default the template uses:

`ALLOW_INTERNAL_REQUESTS=false`

Heimdall blocks requests to private and reserved IP addresses in this
mode.

If you use Heimdall Enhanced Apps that need to query services on your
local network (for example `192.168.x.x` addresses), you may need:

`ALLOW_INTERNAL_REQUESTS=true`

Only enable this when required and when you trust how Heimdall is
exposed, as allowing requests to internal addresses increases SSRF risk.

## Ports

The container exposes:

- `80/tcp` - HTTP Web UI
- `443/tcp` - HTTPS Web UI

The host-side ports can be changed freely in MOS to avoid conflicts
with the MOS Web UI or other services.

## Docker templates

| Application | Category | Status | Documentation |
|---|---|---|---|
| AdGuardHome-Sync | Network | ✅ Tested on MOS | [Guide](docs/adguardhome-sync.md) |
| Duplicacy | Backup | ✅ Tested on MOS | [Guide](docs/duplicacy-migration.md) |
| Heimdall | Utilities | ✅ Tested on MOS | Upstream |
| Hermes-Agent | AI | 🧪 Awaiting MOS validation | [Guide](docs/hermes-agent.md) |
| Immich-Kiosk | Media | ✅ Tested on MOS | [Guide](docs/immich-kiosk.md) |
| Nukkit | Game Server | 🧪 Awaiting MOS validation | Upstream |
| Mumble Server | Network | ✅ Tested on MOS | [Guide](docs/mumble-server.md) |
| Navidrome | Media | ✅ Tested on MOS | Upstream |
| NetAlertX | Network | ✅ Tested on MOS | Upstream |
| Obsidian | Productivity | 🧪 Awaiting MOS validation | Upstream |
| OpenClaw | AI / Productivity | 🧪 Awaiting MOS validation | [Guide](docs/openclaw.md) |
| Paperless-ngx | Productivity | 🧪 Awaiting MOS validation | [Guide](docs/paperless-ngx.md) |
| PS3NetSrv | Game Server | ✅ Tested on MOS | Upstream |
| qBittorrent | Downloader | ✅ Tested on MOS | [Guide](docs/qbittorrent-migration.md) |
| slskd | Downloader / Media | ✅ Tested on MOS | [Guide](docs/soulsync-slskd.md) |
| SoulSync | Media / Downloader | ✅ Tested on MOS | [Guide](docs/soulsync-slskd.md) |
| Valkey | System | ✅ Tested on MOS | [Guide](docs/valkey.md) |
| YoutubeDL-Material | Downloader / Media | 🧪 Awaiting MOS validation | Upstream |

## MOS storage paths

Templates use `/mnt/cache/appdata/...` as the generic MOS Hub appdata path.

Replace `cache` with the name of the pool used for appdata on your MOS system
when required.

## Status

**✅ Tested on MOS**

The template has been deployed and functionally validated on MOS.

**🧪 Upstream reviewed / awaiting MOS validation**

The template has been reviewed against current upstream documentation but has
not yet completed a clean MOS deployment test.

# AdGuardHome-Sync on MOS

## Same-host AdGuard Home with ipvlan

If AdGuardHome-Sync runs on a Docker bridge network and the replica
AdGuard Home container uses an ipvlan network with its own LAN IP,
the sync container may not be able to reach the replica through that LAN IP.

Example:

- MOS host: 192.168.1.79
- AdGuard Home: 192.168.1.78 on `eth0` / ipvlan
- AdGuardHome-Sync: bridge network

Trying to use:

REPLICA1_URL=http://192.168.1.78

may result in:

no route to host

## Recommended solution

Create a user-defined bridge network in MOS, for example:

adguard-internal

Keep AdGuard Home on its normal ipvlan network:

- Network: eth0
- Custom IP: 192.168.1.78

Then add this to AdGuard Home's Extra Parameters:

--network name=adguard-internal,alias=adguard-home,gw-priority=-1

This gives AdGuard Home two networks:

- `eth0` / ipvlan → LAN access and DNS service
- `adguard-internal` → internal Docker communication

Configure AdGuardHome-Sync to use:

- Network: adguard-internal
- REPLICA1_URL=http://adguard-home

The sync container does not require an additional LAN IP.

The `gw-priority=-1` option keeps the ipvlan network as AdGuard Home's
primary/default network.

This configuration survives container recreation by MOS because the
secondary network is included in Extra Parameters.

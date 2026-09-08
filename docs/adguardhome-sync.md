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

Create a dedicated user-defined Docker bridge network in MOS so
AdGuard Home and AdGuardHome-Sync can communicate directly by container
name without assigning another LAN IP to AdGuardHome-Sync.

### 1. Create the Docker network in MOS

Go to:

**Settings → Docker service → Docker Networks → Add**

Create a network with:

- Name: `adguard-internal`
- Driver: `bridge`
- IPv4: enabled
- Subnet: leave empty
- Gateway: leave empty

MOS/Docker will automatically assign an available private subnet
(for example `172.18.0.0/16`).

> Note: despite the name `adguard-internal`, this is not a Docker
> `--internal` network. It is a normal user-defined bridge network,
> allowing AdGuardHome-Sync to communicate both with AdGuard Home
> and with other hosts on the LAN.

### 2. Configure AdGuard Home

Keep AdGuard Home on its normal ipvlan network:

- Network: `eth0`
- Custom IP: your AdGuard Home LAN IP

For example:

`192.168.1.78`

Add the following to **Extra Parameters**:

`--network name=adguard-internal,alias=adguard-home,gw-priority=-1`

This connects AdGuard Home to two Docker networks:

- `eth0` / ipvlan → LAN access and DNS service
- `adguard-internal` / bridge → communication with AdGuardHome-Sync

The `gw-priority=-1` setting keeps the ipvlan network as the primary
/default network for AdGuard Home.

### 3. Configure AdGuardHome-Sync

Configure the Sync container with:

- Network: `adguard-internal`
- Custom IP: leave empty
- `REPLICA1_URL=http://adguard-home`

The name `adguard-home` is the Docker network alias configured above.

AdGuardHome-Sync therefore does not need an additional LAN IP.

### Persistence

Because the secondary network is included in AdGuard Home's
**Extra Parameters**, MOS recreates the container with both networks
after an update or container recreation.

No manual `docker network connect` command is required after recreation.

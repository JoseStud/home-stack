# Phone media server (Poco F3) and Plex failover

An old Xiaomi Poco F3 (`alioth`) stores the movie and TV library and runs a
second, standby Plex server. The Pi's media stack mounts the phone's storage
over sshfs, so the phone is the source of truth and the Pi is the front end.

Last verified 2026-09-29.

## Layout

```
Pi (Docker, media/ stack)                 Phone (Termux sshd :8022, Docker, Plex)
  sonarr / radarr / bazarr / plex  --sshfs-->  /storage/emulated/0/network-storage/{movies,tv}
  /mnt/media/{movies,tv}                       plex (standby) reads the same folders directly
```

| Piece | Where | Notes |
| --- | --- | --- |
| Media files | phone `/storage/emulated/0/network-storage/{movies,tv}` | about 13 GB |
| Pi mounts | `/mnt/media/movies`, `/mnt/media/tv` | sshfs via systemd automounts in `/etc/fstab`, `port=8022`, `reconnect` |
| `MEDIA_DIR` | `/mnt/media` in `media/.env` | so `media/` reaches the phone through this path |
| Phone address | `192.168.1.15` | reserve it in the router; a DHCP change breaks the mounts |
| Standby Plex | `http://192.168.1.15:32400/web` | server name "Phone Plex (backup)" |

## Failover behaviour

- Both Plex servers are signed in to the same Plex account with the same library
  names. Plex clients list both.
- If the Pi loses power, its Plex disappears and the phone's Plex keeps serving
  the same files. Pick the phone's server in the client.
- Clients do not switch on their own, and watch history is **not** synced
  between the two servers.
- Do not copy the Pi's Plex config to the phone. Two servers with the same
  machine identifier confuse plex.tv.
- Sonarr, Radarr, Bazarr, Prowlarr and qBittorrent run on the Pi only, so
  nothing new is downloaded or imported during a Pi outage.
- Keep the phone on a charger.

## Phone software

| Item | State |
| --- | --- |
| ROM | LineageOS 23.2 (Android 16), official nightly 2026-09-23 |
| Root | Magisk 30.7, patched into `boot_a` |
| Kernel | LineageOS `4.19.325` source at commit `71b13e62f057` with Docker options added: `PID_NS`, `IPC_NS`, `SYSVIPC`, `POSIX_MQUEUE`, `CGROUP_DEVICE`, `CGROUP_PIDS`, `BRIDGE_NETFILTER`, `NETFILTER_XT_MATCH_ADDRTYPE`, `MACVLAN` |
| Known kernel gap | `rmnet_shs` and `rmnet_perf` modules do not load (mobile-data tuning only). `USER_NS` is off, which rootful Docker does not need |
| Docker | 29.8.1 static aarch64 build in `/data/local/docker/bin` |
| Plex | container `plex`, image `plexinc/pms-docker`, host networking, config in `/data/local/plex/config` |
| Disabled apps | Android Auto, Digital Wellbeing, Google Restore, Google app (`pm disable-user`) |

### How Docker runs on Android

- Android's `/` is read-only and has no `/run`. `/data/local/docker/start-dockerd.sh`
  starts dockerd in a private mount namespace on a writable overlay of `/`
  (upper layer in `/data/local/docker/ovl`). Real `/dev`, `/proc`, `/sys`,
  `/data` and other mounts are bound in.
- The script also writes a `resolv.conf` (router and 1.1.1.1), sets
  `SSL_CERT_DIR` to Android's CA store, and sets `DOCKER_RAMDISK=true` so runc
  does not use `pivot_root`.
- It uses cgroup v2 (`memory`, `pids`) and `overlay2` on f2fs.
- **Host networking only.** Docker's bridge and iptables are off so it cannot
  change Android's firewall or routing.
- `/data/adb/service.d/docker.sh` (Magisk) starts it about 20 seconds after boot.
  The Plex container comes back through `--restart unless-stopped`.

### Plex container

```bash
docker run -d --name plex --restart unless-stopped --network host \
  -e TZ=America/Bogota -e PLEX_UID=0 -e PLEX_GID=0 \
  -e ADVERTISE_IP=http://192.168.1.15:32400/ \
  -v /data/local/plex/config:/config \
  -v /data/local/plex/transcode:/transcode \
  -v /data/media/0/network-storage/movies:/movies:ro \
  -v /data/media/0/network-storage/tv:/tv:ro \
  plexinc/pms-docker:latest
```

It runs as root inside the container so it can read Android's media files, and
mounts from `/data/media/0` (the real storage) instead of the FUSE view.
There is no hardware transcoding: Plex on Linux needs VAAPI, NVENC or Quick
Sync, and the phone has none of them. Prefer Direct Play.

### Running docker commands on the phone

```bash
adb -s 192.168.1.15:5555 shell 'su -c "export DOCKER_HOST=unix:///data/local/docker/run/docker.sock PATH=/data/local/docker/bin:\$PATH; docker ps"'
```

## Remote access to the phone

- `boot_a` carries a small init script (`overlay.d/init.adbwifi.rc`, run by
  Magisk) that enables `adbd` over TCP on port 5555 and installs the admin PC's
  adb key at boot. The phone's screen is broken, and Android blocks USB data
  while locked, so this is the only way in. It is key-authenticated but open to
  the LAN.
- Termux runs `sshd` on port 8022 through Termux:Boot (`~/.termux/boot/start-sshd.sh`).
  It must run in Termux's own SELinux context. If it is started from adb as
  root it runs as `magisk` and gets *Permission denied* on `/storage`, and the
  Pi's mounts fail.

## Recovery

Restore the phone's share after a phone reboot or address change:

```bash
ssh <pi> 'sudo umount -l /mnt/media/movies /mnt/media/tv; sudo systemctl restart mnt-media-movies.automount mnt-media-tv.automount; ls /mnt/media/movies | head -3'
ssh <pi> 'docker start radarr sonarr bazarr'
```

If the Docker kernel misbehaves, boot with the stock kernel by flashing the
Magisk-only image over USB (`fastboot flash boot_a boot_a.magisk-stockkernel.img`),
or the untouched stock image `boot_a.los23.img` to drop root as well.
A `fastboot boot <image>` test flashes nothing and a reboot returns to the
installed kernel.

## Checks

```bash
curl -s http://192.168.1.15:32400/identity          # Plex up, claimed="1"
ssh <pi> 'findmnt /mnt/media/movies; ls /mnt/media/movies | head -3'
```

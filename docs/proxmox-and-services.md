# Proxmox and services

## The host

| Item | Value |
|---|---|
| Hardware | AMD Ryzen 7 9800X3D, 64 GB DDR5, ASUS X870E board, 1000 W PSU |
| Hypervisor | Proxmox VE 9.2 |
| Root pool | ZFS `rpool` on a 1 TB NVMe (mirror planned) |
| Graphics | Ryzen Radeon iGPU only, passed to the media container |
| NIC | Onboard Realtek RTL8126 5GbE (`nic0`), on a 2.5G port for now |
| IOMMU | On (AMD-Vi) |

Choices and why:

- **Single node, no cluster.** HA and corosync services are disabled and SPICE is masked.
- **Memory overclock (DOCP) off.** Memory overclocking on a hypervisor is a known source of rare, random instability.
- **BIOS Restore on Power Loss: On,** not "Last State". After a clean UPS halt the last state is "off", so the host would stay dark.
- **Storage `local` holds only ISOs, templates and snippets.** Guest disks can land only on ZFS, and backups can never land on the host's own disk.
- **ZFS scrub and trim** come from Debian's `zfsutils-linux` cron (scrub second Sunday, trim first Sunday). I checked it before adding my own schedule and did not need one.

### Access control

- SSH: key only (ed25519 with a passphrase), password and keyboard-interactive login off. Config is validated with `sshd -t` before every reload.
- Web UI: two-factor on every account. `root@pam` is break-glass with a YubiKey (WebAuthn), TOTP and paper recovery keys. WebAuthn is bound to the host's DNS name, which needed a real certificate first.
- Admin access is allowed in three places, and every admin subnet goes in all three: the UniFi zone policy, `pveproxy`'s `ALLOW_FROM`, and the Proxmox firewall `management` IPset.

```text
# /etc/pve/firewall/cluster.fw (sanitized)
[OPTIONS]
enable: 1
policy_in: DROP
policy_out: ACCEPT

[IPSET management]
ADMIN_VLAN_SUBNET
SERVER_VLAN_SUBNET
VPN_SUBNET

[RULES]
IN ACCEPT -p tcp -dport 8006 -source +management
IN ACCEPT -p tcp -dport 22 -source +management
IN ACCEPT -p icmp -source +management
```

`policy_forward` is deliberately left unset: a datacenter-level forward drop filters traffic crossing the bridge and breaks guest networking.

### Certificate

The host gets its own Let's Encrypt certificate through the built-in ACME client and a Cloudflare DNS-01 plugin, with its own scoped token. It is deliberately **not** behind the reverse proxy, so the hypervisor stays reachable with a trusted certificate when the Docker VM is down.

### Networking

`vmbr0` is VLAN aware. The host's own address is untagged, so its switch port must have the Server VLAN as native. Guests on the Server VLAN leave the VLAN tag blank; only the game VM carries a tag. The switch trunk carries only what is used.

## Guests

| ID | Name | Type | Resources | VLAN | Purpose |
|---|---|---|---|---|---|
| 100 | Docker | VM, Debian 13 | 4 cores, 16 GB | Server | Application stacks |
| 101 | Minecraft | VM, Debian 13 | 6 cores, 14 GB | Minecraft (tagged) | Java and Bedrock servers |
| 102 | adguard | LXC, unprivileged | 1 core, 2 GB | Server | House DNS |
| 103 | jellyfin | LXC, unprivileged | 4 cores, 8 GB, 32 GB disk | Server | Media server |
| 104 | test farm | VM, Debian 13 cloud image | 8 cores, ballooning | Server | Headless game test runs |

Common VM settings: CPU type `host`, UEFI, disks on ZFS with discard, SSD emulation and IO threads, QEMU guest agent installed. Addresses are UniFi reservations, so there is one place to change them.

**Boot order** (with delays so dependencies are ready): AdGuard first, then the Docker VM, then the game VM, then Jellyfin (it needs the NAS mount), then the test farm last.

**Updates.** Every Debian guest runs `unattended-upgrades` limited to the Debian-Security origin, with automatic reboot off. Docker, Jellyfin and Proxmox packages and point releases are never installed automatically. I reboot by hand when `/var/run/reboot-required` appears, checking for active players or test runs first.

```text
# /etc/apt/apt.conf.d/52unattended-upgrades-local
Unattended-Upgrade::Origins-Pattern {
#clear
        "origin=Debian,codename=${distro_codename}-security,label=Debian-Security";
};
Unattended-Upgrade::Automatic-Reboot "false";
```

## Docker VM stacks

Every stack lives in `/opt/stacks/NAME/` with a `compose.yaml` and its data folder: one place to back up, one place to look. Docker CE and Compose come from Docker's own repository.

| Stack | Purpose | Notes |
|---|---|---|
| npm | Nginx Proxy Manager | The only thing published on 80 and 443 |
| immich | Photo library | Library on NFS, database on local disk |
| homeassistant | Home automation | Host networking for mDNS and SSDP discovery |
| uptime-kuma | Monitoring | Admin UI on localhost, reached through the proxy |
| dockge | Compose manager | Admin UI on localhost, reached through the proxy |
| actual | Budgeting | No host port at all; proxy only |

**Every image is pinned** to an exact version (and checked against the registry digest). Nothing floats, so an upgrade is a deliberate act: change the tag, `docker compose pull`, `docker compose up -d`. Never `restart` after an environment change, because restart reuses the old container.

**Docker DNS** is set in `/etc/docker/daemon.json` to the VLAN gateway, so containers never inherit an empty resolver list at boot ([troubleshooting entry 3](troubleshooting.md#3-docker-dns-failures-and-the-boot-race)).

### Reverse proxy

- **Nginx Proxy Manager** with one wildcard certificate for `*.example.com` and `example.com`, issued and renewed through DNS-01 with a token that can only edit DNS for that one zone. A new service needs a proxy host, never a new certificate. A renewal dry run against a copy of the certificate folder passed after the 2.16 upgrade.
- **A shared `proxy` Docker network,** created outside any stack. Services on it are addressed by container name; services on other hosts (Jellyfin's container, the gateway, the NAS, the UPS card) by address.
- **Admin pages** (gateway, NAS, UPS card, Dockge, Uptime Kuma and the proxy's own UI) sit behind an access list that allows only the Server, Server Admin and VPN networks. Everyone else gets 403. Their host ports are bound to localhost, so the access list cannot be bypassed.
- **AdGuard's admin page is not proxied:** browser safe-browsing flags its login page as dangerous under any hostname, so it stays on the Server VLAN only.

Immich gets the proxy network through an override file, so the upstream compose file stays untouched on upgrades:

```yaml
# /opt/stacks/immich/compose.override.yaml
services:
  immich-server:
    networks:
      - default   # required: naming any network drops the project default
      - proxy

networks:
  proxy:
    external: true
```

## Services

### AdGuard Home (CT 102)

Network-wide DNS filtering. Details in [network.md](network.md#dns). It runs as a systemd service in its own container so editing a compose stack can never take house DNS down. Memory is 2 GB because AdGuard holds every rule in RAM and the lists are large.

### Jellyfin (CT 103)

- Unprivileged container, no UID mapping needed. The NAS exports with a squash mode that makes everything inside the share world-readable, so bind-mounting each library subfolder (not the share root) lets any user in the container read it.
- Every media mount is **read-only**. A compromised container cannot delete the library. Anything that writes next to media (NFO savers, artwork in media folders) stays off.
- The iGPU's render node is passed in with `--dev0 /dev/dri/renderD128,gid=...`. Hardware acceleration is VAAPI with decode for H.264, HEVC, HEVC 10-bit, VP9, VP9 10-bit and AV1, and encode for H.264 and HEVC, matching exactly what `vainfo` reports.
- Throttling and segment deletion on, with a 32 GB disk and an 85% disk alert, after transcodes filled the disk ([entry 2](troubleshooting.md#2-jellyfin-playback-pauses-until-a-page-refresh)).
- Behind the proxy with WebSockets on, the proxy set as a known proxy (so client addresses are real), remote access from the internet refused, Quick Connect off, one admin user.

### Immich (VM 100)

- `UPLOAD_LOCATION` on the NAS over NFS. `DB_DATA_LOCATION` on the VM's local disk, because Immich does not support a network share for its database.
- Database password generated on the VM with `openssl rand -hex` and written straight into `.env`, never displayed.
- Proxy host allows unlimited body size and long timeouts, so video uploads do not fail with 413.
- Version pinned exactly. Upgrades follow the official order: phone apps first, then the server, after a database dump, a protected VM backup and a NAS snapshot.

### Home Assistant (VM 100)

Container install, managed like every other stack. Wall switches run fully local through Tuya Local. The AC is driven through an IR hub. Discovery protocols do not cross Docker's bridge, hence host networking.

### Uptime Kuma and Dockge (VM 100)

Monitoring and stack management. Kuma monitors use container names for services on the proxy network. Kuma's DNS-type monitor resolves a real name through AdGuard, which proves DNS answers, not just that a port is open.

### Actual Budget (VM 100)

Needs a secure context to run at all ([entry 28](troubleshooting.md#28-budget-app-would-not-load-sharedarraybuffer)). Reachable only through the proxy.

### Minecraft (VM 101)

- `itzg/minecraft-server` (Java 8 tag for Forge 1.12.2, about 190 mods, 8 GB heap with Aikar's flags) and a vanilla Bedrock container.
- Whitelist enforced, one operator, Bedrock cheats off, telemetry (snooper) off.
- Log4Shell checked, not assumed: the image injects a runtime patcher and the bundled Log4j has lookups disabled.
- 120-second stop grace period so the world saves during a UPS shutdown.
- The VM sits alone in its zone. It never gets NFS access to the NAS.

### Test farm (VM 104)

Built from the Debian 13 cloud image with cloud-init (key-only login; I removed cloud-init's passwordless sudo). It runs headless Godot test runs for a personal game project on a timer and writes results to one folder on the NAS over SMB. Low CPU weight (`cpuunits 10`) so it yields to the other guests under contention, and memory ballooning so the host keeps headroom. It is not in the nightly backup because it can be rebuilt and its results live on the NAS. needrestart never restarts its service ([entry 39](troubleshooting.md#39-test-farm-runs-killed-by-automatic-updates)).

## Restore and rollback habits

- A protected backup or snapshot before every upgrade, with a note saying why.
- A dated `.bak` copy of any config file before editing it, and a `--check-config` or `config` dry run before restarting a service.
- Every change written down with its rollback before it is applied.

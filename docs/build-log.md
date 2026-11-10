# Build log

What I actually built, configured or changed, in order. Plans that were never carried out are not in the timeline; they are in the short "Planned" list at the end. Problems and their fixes are summarised here and written up in full in [troubleshooting.md](troubleshooting.md) (linked as "T" plus the entry number).

## Early September 2026: network foundation

- Replaced an earlier plan built around a firewall appliance and a separate managed switch with UniFi gear. The old appliance and switch went to the shelf as lab practice gear.
- Brought up 5 Gig XGS-PON fiber with the ISP gateway bypassed: an SFP+ ONU module in the UDM Pro Max's WAN port. Found the "no DHCP" fault was the ONU in the LAN cage (T40). Recorded an optical baseline. IPv6 left off on purpose.
- Router speed test: 3.69 Gbps down, 5.01 Gbps up.
- Created the VLANs and custom firewall zones: MGMT, Server, Trusted, IoT, Guest, Minecraft (its own zone) and an isolated BlackList VLAN with no DHCP and no internet.
- Set the addressing convention per VLAN: low addresses by hand, the rest DHCP. Fixed IPs are UniFi reservations, not statics inside guests.
- Security baseline on the gateway: local-only admin account, UPnP off, Teleport VPN removed, IPS on, Content Filter and Encrypted DNS off (both would hijack DNS from AdGuard), SafeOps recovery options on.
- Registered the domain at Cloudflare (DNS only, no proxying). Set up UniFi's native Cloudflare DDNS for the one public name, with a token scoped to DNS edit on that zone.

## 12 to 13 September: hypervisor and first services

- Installed Proxmox VE on a ZFS root pool (`rpool`, `ashift=12`, compression on). Switched to the no-subscription repository, installed AMD microcode, kept `local` storage off the disk-image path.
- Disabled cluster and HA services and masked SPICE (single node).
- SSH key-only (ed25519, passphrase), password login off. Daily web admin is a Proxmox realm user with TOTP; `root@pam` kept as break-glass. Web UI bound to the host's address with an allow list, plus the Proxmox firewall with a management IPset (default inbound drop).
- VLAN-aware bridge on the host. The server's switch port became a trunk: Server native, game VLAN tagged. Converted the switch port first, verified, then the bridge, with no outage.
- Built VM 100 (Docker, Debian 13) with Docker CE and Compose from Docker's own repository. Stacks live under `/opt/stacks/NAME/`.
- Deployed Dockge, Uptime Kuma, Home Assistant (container, host networking) and Nginx Proxy Manager.
- Issued a wildcard Let's Encrypt certificate (`*.example.com` and `example.com`) through Cloudflare DNS-01, with its own scoped token.
- Built Actual Budget, the app that made the reverse proxy necessary (T28, T29).
- Built VM 101 for Minecraft on its own VLAN: a modded Forge 1.12.2 Java server (about 190 mods, 8 GB heap) and a vanilla Bedrock server, each in Docker. Port forwards for the two game ports only, protocol narrowed. Java whitelist and a single operator. Checked Log4Shell rather than assuming: the image's runtime patcher and the bundled Log4j version both close it.

## 14 to 15 September: admin VLAN, VPN and DNS filtering

- Added the Server Admin VLAN so reaching servers means deliberately joining an admin network, and moved my workstation onto it. Added the new subnet to all three admin allow lists (T19).
- Started narrowing what Trusted (household devices) can reach on the Server VLAN, beginning with Home Assistant on its one port.
- Set up the UniFi WireGuard server with the DDNS name as the endpoint, IPv6 off. Peers: a phone and a GL.iNet travel router. Found and fixed an allow rule saved as Block and a missing return-traffic setting (T20).
- Configured the travel router: firmware updated first, defaults changed, WireGuard client with kill switch, guest network excluded from the tunnel, IPv6 and cloud management off.
- Built AdGuard Home in its own unprivileged LXC (CT 102): DoH upstreams (Quad9 and Cloudflare), no plaintext fallback, blocklists added as custom URLs including a DoH/VPN bypass list. Raised its memory to 2 GB for the larger lists.
- Rolled AdGuard out to Trusted, Guest and Server Admin with DHCP DNS, NAT redirects and port 53/853 blocks. Kept Server, MGMT and the game VLAN on the gateway on purpose. Reverted IoT after it broke the AC controller hub (T26).
- Added rules so the Server Admin VLAN reaches only the game and SSH ports on the game VM.

## 16 to 20 September: NAS, UPS, media and photos

- Commissioned the used UPS network card: factory reset, hardened services, public NTP (T23).
- Installed NUT on the host reading the UPS over SNMPv3 (T21, T22), with my own runtime and charge thresholds and a Discord notify script. Baseline: 222 W, 34 minutes.
- Commissioned the UNAS Pro 8: RAID 10 on 4x 24 TB, about 48 TB usable. Moved it from MGMT to the Server VLAN after the setup wizard. Encryption off on purpose (NFS needs it off, and unattended recovery needs no unlock).
- Load with the NAS: 256 W, 26 minutes. That became the design number.
- Created four NAS shares up front (backups, media, photos, a personal SMB-only drive), NFS connections per client IP with sync writes, and Proxmox backup storage on NFS. Cleaned up a stale mount from a failed first attempt (T25).
- Moved the nightly backup job to the NAS (21:00, snapshot, zstd).
- Restore drill: a container backup restored to a spare ID at 257 MiB/s, booted, then destroyed. Avoided the duplicate-MAC trap.
- Wrote the ordered UPS shutdown script and drilled it on a real power cut: 26 seconds end to end (T5). Added the NAS to the chain over SSH. Set BIOS Restore on Power Loss to On, set Over Discharge Protection to 20 minutes, and added a boot unit that re-enables backup storage.
- Ran four tests to make the UPS cut its output on command. None worked; the SNMP path is closed (T5).
- Fixed guest auto-start and boot order (T17). Added a 120-second stop grace period for the Java server.
- Built CT 103 Jellyfin: unprivileged, AMD iGPU render node passed through, media bind-mounted read-only per library. Set VAAPI options to match `vainfo` (T33).
- Found IPS in "Notify and Block" dropping internal traffic and set it to Notify (T10).
- Put Jellyfin behind the proxy with WebSockets on and known proxies set.
- Built Immich on VM 100: library on the NAS over NFS, Postgres on local SSD, random DB password generated on the VM and never displayed, and the `proxy` network added through a compose override file. Proxy host with upload size and timeout settings. Phone backup set up (about 8,400 assets).
- Built the Proxmox notification path to Discord, which had never existed (T6). Set ZED to report scrubs.
- Fixed Docker DNS on VM 100 with `daemon.json` (T3).

## 21 to 30 September: hardening pass and reboot drills

- Found and closed the Proxmox mail bounce loop (T6).
- Hardening pass: SMB access through File Service Credentials (T24), Immich pinned to an exact version, an unused Proxmox API key and the dead SMTP target deleted, Docker DNS on VM 101.
- Reboot drill found the VM 101 qxl hang (T4) and the boot race behind the Docker DNS and NFS failures (T3). Both fixed and reboot-verified.
- Moved the AP to the ceiling (T8) and fixed the cable run it exposed.
- Added an Uptime Kuma DNS monitor that resolves a name through AdGuard, proving it answers, not just listens.
- Finished narrowing Trusted to three rules: Home Assistant, the reverse proxy on 443 and DNS to AdGuard. Admin ports (hypervisor, Dockge, Uptime Kuma, the NAS) need the Server Admin VLAN or the VPN.
- Re-read the whole firewall policy table from the live config and corrected the docs to match it. Rebuilt the Server Admin port 53 block. Deleted paused rules instead of leaving them greyed out. Moved the IoT DHCP range above the hand-assigned addresses. Confirmed why camera live view works with IoT-to-Trusted blocked (it rides the return path of the Trusted-to-IoT allow).
- Added Uptime Kuma monitors for the proxy and the budget app.

## Early October: media library and DNS outage

- Fixed a DNS outage on my workstation: it was listed in AdGuard's disallowed clients (T27).
- Diagnosed game disconnects (NAT hairpin and a client mod crash) (T32).
- Renamed two series for Jellyfin with dry-run-first Python scripts (366 files and 67 files) (T34).

## 12 to 19 October: 10G backbone

- Bought a USW Pro Aggregation, two copper SFP+ modules, two 3 m DACs and a USB-C 5GbE adapter.
- Upgraded Nginx Proxy Manager 2.15.1 to 2.16.0 to close a CVE, with a compose backup.
- Read the UDM, switch and AP firmware versions: all current.
- Installed the aggregation switch. Moved the host and the workstation to it at 5 Gbps through the copper modules.
- The first USB-C 5GbE adapter failed on two PCs and was returned (T1).

## 21 October

- Built VM 104 (test farm) from the Debian 13 cloud image with cloud-init: CPU weight lowered, memory ballooning on, SMB mount of a single folder on the NAS, security-only auto updates. Removed cloud-init's passwordless sudo. Added ping and SSH monitors. Added it as the first step of the UPS shutdown chain (not drilled yet).
- Put the six admin UIs (gateway, NAS, UPS card, Dockge, Uptime Kuma, the proxy) behind the reverse proxy with the wildcard certificate and an access list allowing only the Server, Server Admin and VPN networks. Tested that other networks get 403.
- Checked that only the two game ports are forwarded and nothing exposes 443.
- Ran a certificate renewal dry run against a copy of the proxy's certificate folder. Passed.
- Set backup retention, which had never been set, and pruned 89 old backups (T16).
- Found the Proxmox certificate had never been deployed and ordered it (T18).
- A Cloudflare token was printed by a command; started the cleanup (T9).
- Turned on daily snapshots for the personal NAS drive.
- Set up RDP to the workstation from the laptop only through the travel router's tunnel: a UniFi rule and the Windows firewall scoped to that path, NLA on, Duo second factor on RDP (trial licence), account lockout. Outside test passed the next day.

## 22 October

- Host link failure and a port "bounce" outage (T1, T13).
- Finished the token cleanup: leaked token deleted, tokens renamed per consumer (T9).
- Upgraded Immich v3.2.2 to v3.3.5 (security fix) with a DB dump, a protected VM backup and a NAS snapshot first. Phone app updated first. Test upload worked.
- Upgraded Jellyfin 12.1 to 12.2 with a protected backup first. Proved hardware transcoding (104 fps, 4K HEVC to 720p H.264 on VAAPI). Raised the container to 4 cores and 8 GB live. Confirmed exactly one admin user.
- Pinned every remaining floating image (Dockge, Uptime Kuma, Home Assistant, Actual) to the exact running version, digest checked. Removed the budget app's host port.
- Set up security-only unattended upgrades with no automatic reboot on every Debian guest. Installed the waiting security updates (with backups) and rebooted the two VMs with players checked first.
- Fixed nightly backup notices being rejected by Discord (T15). Added sender names.
- Added healthchecks.io dead-man's switches for the nightly backup (a vzdump hook script), the Immich dump (a daily check of age, size and gzip integrity) and the test farm timer.
- Two-factor on the Proxmox root login: YubiKey WebAuthn, TOTP and paper recovery keys.
- Deleted the old local backups from the host disk after archiving the first-build backup (T16).
- Turned on daily snapshots for the photos, media and backups shares with per-share limits.
- Set every SNMPv1 community on the UPS card to Forbidden after checking nothing used v1.
- Moved the workstation to the UniFi 5 Gbps USB-C adapter with a fixed IP and power saving off (T1).
- Host lost power twice from a loose cable (T14).

## 23 October

- Turned on client isolation for the guest Wi-Fi.
- Added a 5-minute "host alive" heartbeat to healthchecks.io and confirmed the UniFi device-offline alarm exists.
- Fixed the UPS low-battery alert storm (T12).
- Upgraded Docker CE 29.8.0 to 29.9.2 and Compose 5.5.1 to 5.6.1 on both VMs (players checked first). All containers came back healthy.
- Ran a read-only privacy and exposure review (live reads, the 41-day DNS log, certificate logs, a passive outside lookup).

## 24 October: privacy changes

- AdGuard: replaced one large list with a tracker list plus Apple, Amazon, Microsoft and TikTok tracker lists; query log 90 days to 14; NXDOMAIN for iCloud Private Relay and the Firefox DoH canary. Config checked before each restart.
- Domain: SPF `-all`, DMARC reject, null MX, CAA for Let's Encrypt, DNSSEC signing enabled.
- IoT moved onto AdGuard properly, with the hub exempted from filtering (T26).
- Outbound commercial WireGuard VPN on the gateway for the Trusted and Server Admin VLANs only, through a dedicated least-privilege local admin account. A weekday schedule on the host turns the route off at 08:00 and on at 17:00, tested both ways. Kill switch tested and found not to hold (T37). Tunnel-down alert installed (T36).
- Fixed guest DNS, which had never worked (T7).
- Pointed both VPN peers at AdGuard and fixed a DNS typo (T38).
- Raised the test farm VM from 6 to 8 cores.

## 25 October: security review 2

- Ran a second, deeper read-only review, then fixed its top 10 one at a time with a test after each (T42):
  - Trusted-to-proxy rule narrowed to TCP 443.
  - Admin UIs moved off host ports (T30).
  - Proxmox and gateway MFA checked; media server Quick Connect off.
  - IoT and game zones blocked from the gateway except DHCP, DNS and NTP.
  - Game server telemetry off.
  - Internal names removed from public DNS.
  - Server Admin DNS block resumed; DNS redirects fixed (T11).
  - Unused switch ports moved to BlackList.
  - Certificate tokens IP-filtered, with a home-IP-change alert.
  - Trunks trimmed to only the VLANs each port needs.
- Stopped needrestart from killing test farm runs (T39).
- Host link failed again; moved the host to a 2.5G port and added a per-minute link check (T1, T35).
- Purchase check before buying four more NAS drives and a rack (read-only). Confirmed the NAS can take them as a second RAID group in the same pool.

## 26 to 27 October

- After one more overnight failure on the 5G path, the host stays on 2.5G. Decided on an Intel SFP+ card with a DAC for 10G (T1).
- The link check now posts a "was bad from X to Y" summary when its alert could not get out (T35).
- Fixed Jellyfin playback pauses and added a disk alert for its container (T2).

## Late October to 1 November

- Bought the UniFi Power Kit (UniFi UPS 2U plus Power Distribution Pro).
- Moved the rack on 1 November 2026.

## 9 November

- Built a **local NVR** for an IP camera (full write-up in [local-nvr.md](local-nvr.md)): isolated the camera on the IoT VLAN, added a firewall policy to block its outbound internet so it can no longer reach its overseas vendor cloud, and drove it by its local RTSP stream. Stood up Frigate in Docker on the Docker VM, recording to a dedicated NFS share on the NAS with a retention policy. Added go2rtc to pull each stream once and serve an HD live view over MSE, which works from the admin VLAN with no WebRTC port opened. Added a mosquitto broker and a small notifier so a person detection sends a push alert. Traced two issues live: the cross-VLAN live view (WebRTC could not cross the VLANs, switched to MSE plus a transcoded audio track) and person detection not firing (it was running on the low-bitrate sub-stream, moved to the main stream). Secrets kept in `chmod 600` env files, nothing in version control.
- **DNSSEC now validating.** The DS record is published at the registry, the chain validates (a validating resolver returns the authenticated-data flag) and the published digest matches the zone's signing key.
- Added a UniFi alarm that sends a **mobile push if the server or NAS drops off the network**.

## Planned (not done yet)

- An Intel SFP+ card (X710 class) and a DAC for the host (not bought yet).
- Install the UniFi Power Kit and re-drill the shutdown chain.
- Mirror the ZFS boot pool.
- Offsite copy of the NAS.
- Configs and stacks in git, metrics and dashboards.
- Four more 24 TB NAS drives as a second RAID group.
- UniFi Protect cameras and a doorbell on their own VLAN with no internet.

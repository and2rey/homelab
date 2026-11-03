# Troubleshooting log

Every real problem I hit and fixed in this lab, with the evidence. Each entry has the same six parts: **Symptom**, **Investigation**, **Root cause**, **Fix**, **Verified**, **Lesson**. Where a cause was not fully proven, I say so. Where something is a known limitation rather than a fix, I say that too.

Addresses are sanitized. "The gateway" is the UniFi Dream Machine, "the host" is the Proxmox server, "AdGuard" is the DNS container, and VLANs are named, not numbered by subnet.

## Contents

1. [Server 5GbE link drops through a copper SFP+ module](#1-server-5gbe-link-drops-through-a-copper-sfp-module)
2. [Jellyfin playback pauses until a page refresh](#2-jellyfin-playback-pauses-until-a-page-refresh)
3. [Docker DNS failures and the boot race](#3-docker-dns-failures-and-the-boot-race)
4. [VM hangs forever on reboot (qxl)](#4-vm-hangs-forever-on-reboot-qxl)
5. [UPS shutdown chain and the killpower investigation](#5-ups-shutdown-chain-and-the-killpower-investigation)
6. [Proxmox never sent an alert, then a bounce loop](#6-proxmox-never-sent-an-alert-then-a-bounce-loop)
7. [Guest VLAN had no DNS for 41 days](#7-guest-vlan-had-no-dns-for-41-days)
8. [Poor Wi-Fi coverage was a 6 GHz band problem](#8-poor-wi-fi-coverage-was-a-6-ghz-band-problem)
9. [Leaked API token and credential cleanup](#9-leaked-api-token-and-credential-cleanup)
10. [IPS silently dropping one TCP port](#10-ips-silently-dropping-one-tcp-port)
11. [DNS redirects that never redirected](#11-dns-redirects-that-never-redirected)
12. [UPS low-battery alert storm while on mains](#12-ups-low-battery-alert-storm-while-on-mains)
13. [A switch port "bounce" took the Server VLAN offline](#13-a-switch-port-bounce-took-the-server-vlan-offline)
14. [Host lost power twice in one evening](#14-host-lost-power-twice-in-one-evening)
15. [Nightly backup notices rejected by Discord](#15-nightly-backup-notices-rejected-by-discord)
16. [Backups were never pruned](#16-backups-were-never-pruned)
17. [Guests did not start after a reboot](#17-guests-did-not-start-after-a-reboot)
18. [Hypervisor TLS certificate was never actually deployed](#18-hypervisor-tls-certificate-was-never-actually-deployed)
19. [Proxmox refused a new admin subnet](#19-proxmox-refused-a-new-admin-subnet)
20. [Allow rules that time out](#20-allow-rules-that-time-out)
21. [UPS card unreachable: rule order](#21-ups-card-unreachable-rule-order)
22. [UPS SNMPv3 passphrase silently truncated](#22-ups-snmpv3-passphrase-silently-truncated)
23. [Commissioning a used UPS network card](#23-commissioning-a-used-ups-network-card)
24. [NAS rejected every SMB password](#24-nas-rejected-every-smb-password)
25. [Stale NFS mount after a share path change](#25-stale-nfs-mount-after-a-share-path-change)
26. [IoT DNS filtering broke the AC controller hub](#26-iot-dns-filtering-broke-the-ac-controller-hub)
27. [Workstation could reach almost nothing: blocked in AdGuard](#27-workstation-could-reach-almost-nothing-blocked-in-adguard)
28. [Budget app would not load: SharedArrayBuffer](#28-budget-app-would-not-load-sharedarraybuffer)
29. [Reverse proxy 502: "could not resolve" an IP address](#29-reverse-proxy-502-could-not-resolve-an-ip-address)
30. [Admin UIs reachable on host ports, around the proxy](#30-admin-uis-reachable-on-host-ports-around-the-proxy)
31. [Minecraft server crash loop from one whitelist name](#31-minecraft-server-crash-loop-from-one-whitelist-name)
32. [Minecraft disconnects from inside the LAN](#32-minecraft-disconnects-from-inside-the-lan)
33. [Jellyfin hardware decode settings that did not match the GPU](#33-jellyfin-hardware-decode-settings-that-did-not-match-the-gpu)
34. [Media library misread by Jellyfin](#34-media-library-misread-by-jellyfin)
35. [Host link alert that could not report its own outage](#35-host-link-alert-that-could-not-report-its-own-outage)
36. [VPN tunnel-down alert gave a false reading](#36-vpn-tunnel-down-alert-gave-a-false-reading)
37. [VPN kill switch failed its test](#37-vpn-kill-switch-failed-its-test)
38. [Remote VPN client DNS: a typo and a "leak" that was not one](#38-remote-vpn-client-dns-a-typo-and-a-leak-that-was-not-one)
39. [Test farm runs killed by automatic updates](#39-test-farm-runs-killed-by-automatic-updates)
40. [ONU stick gave "no DHCP"](#40-onu-stick-gave-no-dhcp)
41. [Smart switch pairing for local control](#41-smart-switch-pairing-for-local-control)
42. [Findings fixed from a read-only security review](#42-findings-fixed-from-a-read-only-security-review)
43. [Operator traps worth writing down](#43-operator-traps-worth-writing-down)

---

## 1. Server 5GbE link drops through a copper SFP+ module

**Symptom.** In mid October I moved the Proxmox host to the new aggregation switch at 5 Gbps. Its onboard Realtek RTL8126 NIC ran over Cat6 into a UniFi copper SFP+ module (UACC-CM-RJ45-MG). From then on the link dropped every 12 to 48 hours, and a replug fixed it for a while. On 22 Oct the host and every guest on it showed 33 to 95% packet loss with round trips near one second. Because AdGuard runs on the host, the whole house lost DNS. On 25 Oct it happened again (13 to 53% loss, about 500 ms). After I moved the host back for a retest, the link flapped twice and then lost almost all traffic overnight: 458 of 480 one-minute checks showed 100% loss to the gateway.

**Investigation.**
- I compared targets on the same VLAN. The NAS (same VLAN, other switch) and the gateway answered at 0% loss and 0 ms while the host did not. That put the fault on the host's own link.
- `ethtool -S nic0` showed receive errors in the millions (about 5.9 million by 22 Oct). After one reboot it counted about 850,000 errors in 13 hours with zero link drops. During the 25 Oct fault it reached 2.24 million.
- The switch side of the same port showed **0 errors, 0 drops**. The errors were only in the direction module to cable to host NIC. Copper SFP+ modules report no temperature or DDM data in UniFi, so the switch had nothing to show.
- The kernel log showed link drops on 15, 16, 17, 19 and 20 Oct, with no hang, transmit timeout, reset or PCIe error messages. Driver: in-kernel `r8169`, firmware `rtl8126a-3_0.0.5`.
- My workstation sat on the adjacent port through the same module type. Its port logged 44 link-downs in 7.5 days, and one drop came within 6 seconds of the host's. An earlier USB-C 5GbE adapter (RTL8157) on that port was defective: it failed on two different PCs, so I returned it.
- A cable reseat held for about 25 minutes. Moving the host to a plain 2.5G RJ45 port on the access switch made the errors stop at once. A one-hour watch after moving back to 5G was clean, and then it failed again that night. So the fault followed the module path and was reproducible.

**Root cause.** The host's RTL8126 on the `r8169` driver does not run reliably at 5 Gbps through the UACC-CM-RJ45-MG copper SFP+ module. I did not separate the module, the cable run and the NIC driver from each other completely. The evidence points at the module and NIC pairing: the same module type on the workstation port stopped failing once the workstation used a UniFi 5 Gbps USB-C adapter (UACC-Adapter-RJ45-USBC-5GE) on its end.

**Fix.**
- I configured a free 2.5G port on the access switch first (native Server VLAN, Minecraft VLAN tagged, PoE off), then moved the cable. The host stays there permanently.
- The old 5G port is not used again: it is the known-bad path.
- Workstation: the UniFi 5 Gbps USB-C adapter with the Realtek driver, USB selective suspend off and "allow the computer to turn off this device" off. It runs stable at 5 Gbps.
- Planned permanent fix for the host (not bought yet): an Intel SFP+ card (X710 class) with a DAC to the aggregation switch, so no copper module sits in the path.

**Verified.** On the 2.5G port: link up at 2.5 Gbps full duplex, 0 new receive errors, 0% loss to the gateway. AdGuard, the reverse proxy, the NFS mounts and the Minecraft VM (on a tagged VLAN) all checked good after the move. A per-minute link check on the host keeps watching it and alerts if loss returns.

**Lesson.** Read the counters on both ends of a link. A switch port with zero errors does not mean the link is healthy. Prefer DACs over copper SFP+ modules. Configure a switch port before plugging into it, and have a known-good port ready to fall back to. A clean one-hour watch is not proof; this fault needed an overnight run to show again.

---

## 2. Jellyfin playback pauses until a page refresh

**Symptom.** In the browser, Jellyfin playback stopped about 20 minutes into a 4K movie or episode. Only a page refresh brought it back.

**Investigation.**
- The container's root disk was 100% full. `/var/cache/jellyfin/transcodes` held 11 GB: 575 segments of one job.
- `encoding.xml` had **segment deletion off** and **throttling off**.
- The source was 4K HEVC Main 10 HDR, transcoded for the browser at about 35 to 45 Mbps (3-second segments of 12 to 17 MB). ffmpeg ran 1.45x ahead of playback and kept every segment, so about 28 minutes of video filled the 12 GB disk.
- When the disk filled, ffmpeg exited with code 134, the Jellyfin database could not write, and the reverse proxy logged HTTP 500 on the playback-progress call. A refresh ended the job, Jellyfin deleted its segments, space came back and playback resumed.
- I ruled out the NAS and NFS (no NFS errors), the network (0 new errors, link check quiet), CPU and memory (13% CPU at most, 1.8 GB of 8 GB, no swap), and DNS and the proxy path (LAN only, WebSocket headers present).

**Root cause.** The transcode cache lived on a small container disk with no throttling and no segment cleanup.

**Fix.**
1. Turned on Throttle Transcodes and Delete Segments, with a backup of `encoding.xml` first. An earlier toggle in the dashboard had never been saved (no save request in the proxy log, file unchanged), so I set it through Jellyfin's configuration API, the same call the dashboard's Save button makes.
2. Grew the container disk from 12 GB to 32 GB live with `pct resize`, with no restart. ZFS made this a quota change.
3. Added an Uptime Kuma push monitor for the container's disk. A host cron job reads the ZFS `referenced` and `refquota` values every 5 minutes and pushes "down" at 85%.

**Verified.** A 31-minute 4K transcode played to the end of the episode with no pause. The transcode folder levelled off at about 6.5 GB (about 305 segments: 12 minutes kept plus 3 ahead) and stayed steady. Disk use peaked at 25% of 32 GB. No ffmpeg crash and no database error. The disk monitor pushes "up" every 5 minutes.

**Lesson.** A playback problem can be a storage problem. Size scratch space for the worst case (here, 4K HDR to a browser), and alert on the disk that holds it.

---

## 3. Docker DNS failures and the boot race

**Symptom.** A new Uptime Kuma monitor using a hostname went straight to DOWN with `getaddrinfo EAI_AGAIN`. Every other monitor used an IP and was green. A week later a reboot drill showed the Immich photo library mount missing, and Immich in a restart loop.

**Investigation.**
- Inside the container, `/etc/resolv.conf` read `# NO EXTERNAL NAMESERVERS DEFINED`. Four containers (the reverse proxy, the budget app, Dockge and Uptime Kuma) were in that state. Containers created later were fine.
- Docker's embedded resolver snapshots the host's nameservers when a container is created and never refreshes them. This was urgent: the reverse proxy renews its wildcard certificate through DNS-01 and must reach the Cloudflare API. It would have failed silently around day 60.
- After a reboot, `df -h /mnt/photos` fell through to the root filesystem, so the NFS export was not mounted. The journal said `mount.nfs: Network is unreachable` one second into boot.
- `/etc/network/interfaces` used `allow-hotplug` and `ifupdown-wait-online` was disabled. `networking.service` finished in the same second it started, before DHCP gave the interface an address.

**Root cause.** `network-online.target` fired before the network was online. Everything ordered after it ran against a dead interface: containers were created with an empty resolver list, and the `_netdev,nofail` NFS mount failed and was never retried.

**Fix.**
- Treated the symptom durably: `/etc/docker/daemon.json` sets the VLAN gateway as the Docker DNS server on both VMs.
- Fixed the cause: `allow-hotplug` to `auto` for the interface, and `ifupdown-wait-online.service` enabled, so the target means what it says.
- Deliberately did **not** make Docker hard-require the NFS mount. A failed mount would then have taken down every service on the VM instead of only the photo app.

```json
{ "dns": ["SERVER_VLAN_GATEWAY"] }
```

**Verified.** Containers report `# ExtServers: [gateway]` from `daemon.json`, not inherited from the host. After the fix, a reboot came back whole on its own: the NFS mount present, all nine containers healthy, Immich up with no intervention. It was the first reboot that evening to do so.

**Lesson.** Fix the cause, not each symptom. The DNS fix alone left the NFS mount failing the same way. `nofail` trades a failed boot for a silent failure, so a `nofail` mount needs something that notices when it is missing. Immich's marker files turned a slow disaster (photos written to the VM's local disk) into a visible outage.

---

## 4. VM hangs forever on reboot (qxl)

**Symptom.** `sudo reboot` on the Minecraft VM dropped SSH and the VM never came back. Fifteen minutes later SSH said `Connection refused`, and Proxmox said the VM was `running`.

**Investigation.**

| Signal | Meaning |
|---|---|
| `Connection refused`, not a timeout | The guest kernel was alive and sending resets; userspace was gone |
| `qm status` running | QEMU was fine; the problem was inside the guest |
| Disk reads flat over 19 s | Not a filesystem check |
| No new boot in `journalctl --list-boots` | It shut down and never restarted |

The previous boot's journal (read with `sudo`, because without it the journal truncated silently) showed `[TTM] Buffer eviction failed` every 15 seconds, after a hung task blocked in `write()`. The VM used the `qxl` display adapter.

**Root cause.** A known Debian 13 bug with the qxl driver: on shutdown it retries buffer eviction forever, and systemd never reaches the reset. It blocks shutdown too, so this VM would also have stalled the UPS shutdown chain.

**Fix.** `qm set 101 --vga std`, then a hard stop and start (the display device changes only on a full power cycle; a clean shutdown hits the same hang). I checked that no other VM used qxl.

**Verified.** The reboot that had failed now completes: back over SSH on its own in about two minutes, both game containers healthy.

**Lesson.** "Refused" and "timed out" are different diagnoses. Working is not the same as rebootable; drill reboots on purpose before a power event does it for you.

---

## 5. UPS shutdown chain and the killpower investigation

**Symptom.** The default NUT shutdown (`shutdown -h +0`) stops guests on Proxmox's own schedule and gives a heavily modded game world no time to save. The NAS has no NUT client of its own. And after the first real drill, when mains came back, **nothing powered back on**, because the UPS never cut its output.

**Investigation.**
- I wrote and timed an ordered shutdown script, then ran a real drill: wall plug pulled, with the trigger threshold raised temporarily so it fired after two minutes instead of sixteen.
- For automatic recovery the UPS must drop its output after the host halts. I tried four ways:

| Attempt | Method | Power state | Card response | Output cut |
|---|---|---|---|---|
| 1 | NUT systemd shutdown hook | mains | none: networking already down at that stage | no |
| 2 | NUT `driver.killpower` over SNMPv3 | battery and mains | `OK` | no |
| 3 | `snmpset` over SNMPv1, sleep OID | mains | value echoed back | no |
| 4 | `snmpset` over SNMPv1, sleep and load-off OIDs | battery | values echoed back | no |

- Driver debug output showed the SET leaving the host. On SNMPv3 the card silently discards writes; on SNMPv1 it authenticates, accepts, echoes and still does nothing.

**Root cause.** The RMCARD205 on firmware 1.6.0 will not drop its own output on an SNMP command, on battery or on mains.

**Fix.**
- `ups-shutdown.sh`, run by `upsmon` at 10 minutes of runtime left: stop the test farm VM, the Minecraft VM (180 s budget, with a 120 s container stop grace period so the world saves), the Docker VM, then AdGuard last; disable the NFS backup storage and unmount NFS (the mounts are `hard`, so the NAS must not vanish first); power off the NAS over SSH with a 30-second timeout; halt the host. Every step is `|| true` so nothing can stop the host from halting. The Jellyfin container is not in the list; the host halt stops it.
- Long-outage recovery: the card's Over Discharge Protection (20 minutes at 0%) puts the UPS to sleep, so when mains returns, output restores, BIOS "Restore on Power Loss: On" boots the host and the NAS boots itself.
- A boot-time systemd unit re-enables the backup storage once the NAS answers, because disabling it persists across reboots and backups would otherwise stop silently.
- Closed the SNMP path afterwards: all SNMPv1 communities set to Forbidden.

**Verified.** The real drill ran end to end in **26 seconds** against a 600-second budget, with the Discord messages in order and no data loss. The test farm step was added after this drill and has not been drilled yet. A cold AC test proved the NAS boots on its own. The re-enable unit was tested by disabling the storage and restarting the unit, and after a real reboot.

**Known limitation.** A short outage (mains back before the battery empties) still needs a person to press the power button. The next candidates are the vendor's own agent or remote power control.

**Lesson.** A protocol-level acknowledgement proves the packet was parsed, never that the device obeyed. The only proof is the physical effect.

---

## 6. Proxmox never sent an alert, then a bounce loop

**Symptom.** While testing whether a ZFS pool failure would reach Discord, I found Proxmox had **never notified anything**. Every alert in the channel came from Uptime Kuma or the UPS script, which post to the webhook directly. Five days after I fixed that, "Undelivered Mail Returned to Sender" started arriving in Discord.

**Investigation.**
- `/etc/pve/notifications.cfg` had one matcher routing everything to local mail and no webhook target at all. A Gmail SMTP target existed, but it could never authenticate (Google stopped accepting account passwords for SMTP in 2022) and no matcher pointed at it.
- The target Test button tests only the target and skips matchers, which is how a target can look perfect and still receive nothing.
- For the bounces: the mail queue held 12 messages, the oldest from the night the webhook started working, all stamped around 21:03, three minutes after the nightly backup. `root@pam`'s address was an external mailbox, and outbound port 25 is blocked on residential lines.

**Root cause.** First, no notification path existed. Second, the built-in matcher still sent everything to a sendmail target that could not deliver; each bounce became a new notification, which the new Discord matcher delivered.

**Fix.**
- Added a webhook target with a body that uses the `{{ json }}` helper so quotes and newlines in a backup log cannot break the JSON. Added a new catch-all matcher rather than editing the built-in one.
- Disabled the built-in matcher, flushed the queue (`postsuper -d ALL`, 12 messages), and deleted the unusable SMTP target.
- Set `ZED_NOTIFY_VERBOSE=1` so ZFS reports a finished scrub too, which proves the path works while nothing is failing.

**Verified.** `echo test | mail -s test root` arrives in Discord. That is the exact call ZED makes, so the chain ZED to mail to `pvemailforward` to the notification system to Discord is proven. Re-tested after disabling the built-in matcher.

**Lesson.** Configured is not delivering. Alerting that has never fired has never been tested. A cluster of timestamps is a diagnosis: twelve mails at 21:03 named the backup job without reading one message body.

---

## 7. Guest VLAN had no DNS for 41 days

**Symptom.** A phone on the guest Wi-Fi connected but had no internet. When I analysed AdGuard's 41-day query log (1.2 million queries), not one query had come from the Guest VLAN. An earlier browse test from a guest phone had been recorded as a pass, so that test had proved less than it seemed.

**Investigation.**
- Guest DHCP handed out AdGuard as DNS.
- In the guest-to-servers policy list, **"Block Guest to Servers" sat above "Allow guest to AdGuard DNS"**. Policy order beats specificity.
- The guest DNS redirect (DNAT) sends any other resolver to AdGuard too, so every DNS path ended at the same block.
- A packet capture on AdGuard's interface saw no guest DNS at all. The IoT VLAN had the same ordering problem.

**Root cause.** A redirect to AdGuard plus a block on AdGuard's zone ordered above the allow means no DNS at all.

**Fix.**
- Moved the AdGuard allow above the block.
- Guest to gateway: allow DHCP only, then block (this also closed the gateway's login page to guests).
- Guest to the internet: block all private ranges, block DNS on 53 and 853 to anything but AdGuard.
- One guest device enforces its own DNS-over-TLS profile; it got a narrow per-device 853 exception and its own AdGuard client with filtering off.
- Two rules were refused by UniFi when created with "auto allow return traffic" (it is not allowed on rules to the Gateway or External zones); recreated without it.

**Verified.** A capture showed 313 DNS queries from the guest device reaching AdGuard, and its apps worked. Must-fail tests from the guest Wi-Fi: AdGuard's web page and both gateway login pages do not load.

**Lesson.** Prove DNS in the query log, not by browsing. Rule order is part of the rule. See also [entry 11](#11-dns-redirects-that-never-redirected): the redirects themselves turned out to be bound to the wrong interface.

---

## 8. Poor Wi-Fi coverage was a 6 GHz band problem

**Symptom.** Three clients had very poor signal "in the room next door": -79 to -87 dBm.

**Investigation.** I looked at what the failing clients had in common before measuring anything. **All three were on 6 GHz, and nothing on 2.4 or 5 GHz was bad** (those sat between -48 and -69 dBm). 6 GHz loses more to walls, and indoor 6 GHz is power-limited by regulation. Clients also cling to the band they joined until it fails outright. The AP was flat on a table, where its downward antenna pattern points at the floor and ceiling.

**Root cause.** 6 GHz is a same-room band, and the AP was in close to the worst spot.

**Fix.** Mounted the AP on the ceiling, centred on the space that matters.

**Verified.** The coverage problem resolved after the move.

**Second problem the move exposed.** After mounting, the AP's switch port came up at 1 Gbps instead of 2.5 Gbps while its neighbours were at 2.5. I did not force the speed: auto-negotiation had *found* the limit, and a forced 2.5G link against an AP that cannot be pinned causes errors. I worked from the cheapest test up: was it right before the move (yes, so the new run was suspect), reseat both ends, check every joint, then prove the AP end with a short known-good patch cable. The cable run was the fault; after fixing it the port came up at 2.5G.

**Lesson.** Look for what failing clients share before measuring. Auto-negotiation reports the limit, it does not cause it. A port whose colour does not match its neighbours is a free physical-layer diagnostic.

---

## 9. Leaked API token and credential cleanup

**Symptom.** Running `pvenode acme plugin list` without options printed the Proxmox ACME plugin's data, including the Cloudflare API token in plain text.

**Investigation.**
- Three Cloudflare tokens existed (DDNS, reverse proxy, Proxmox), all with the same default name. The dashboard shows no token IDs, so I could not tell from the list which row was the leaked one.
- I wrote a small script that verifies a stored token and prints **only its ID and status**, never the value. I ran it once against the reverse proxy's stored token so I knew which ID must survive, and used each row's "Last used" date to tell the rows apart.

**Root cause.** A command that prints secrets by default, run bare.

**Fix.**
1. Created a new token scoped to DNS edit on the one zone, set it in Proxmox and ordered the certificate with it.
2. Deleted the leaked token, then re-verified the reverse proxy token was still active.
3. Renamed every token after its one consumer, so "Last used" is no longer needed to identify them.
4. Later: IP-filtered the two tokens used for certificates to the home IP, and added an hourly check that alerts if the home IP changes (it stores only a SHA-256 hash of the IP, never the IP).

**Verified.** The reverse proxy token verified active before and after the deletion, a certificate renewal dry run passed, and the Proxmox certificate was issued with the new token. Both IP-filtered tokens verified from their hosts afterwards.

**Other credential cleanups, all done:**
- The UPS card's SNMPv3 passphrase appeared in a screenshot; I rotated it on the card and in NUT.
- A WireGuard phone peer's QR code appeared in a screenshot; I removed and recreated the peer.
- Deleted an unused Proxmox API key and the dead SMTP target.

**Lesson.** One token per consumer, named for it. Know which commands print secrets, and verify tokens by ID only. A VPN config or QR code is a credential: an exposed one is replaced, not reused.

---

## 10. IPS silently dropping one TCP port

**Symptom.** Jellyfin's setup wizard hung on "Next". Ping and SSH to the container worked, but its web port from the Trusted VLAN timed out, and only some of the time.

**Investigation.** I ruled out, in order: the web root, DNS, outbound HTTPS, browser extensions and cached state, the zone policies, an IP conflict (`arping` showed one MAC, five of five), and the Proxmox firewall (it applies to the host, and no per-guest rules existed). Then `tcpdump` inside the container during a failing attempt: SYNs to port 22 arrived, SYNs to the web port **never arrived**, and the container kept retransmitting FINs that got no answer.

**Root cause.** UniFi Intrusion Prevention was in "Notify and Block" with the Server and Trusted networks selected, and it was dropping internal traffic to a new service.

**Fix.** IPS stays on, in Notify mode, so every signature still logs. Later the service moved behind the reverse proxy on 443.

**Verified.** With IPS in Notify, the port worked at once and tcpdump showed a clean HTTP exchange.

**Lesson.** Ping working while one TCP port dies, intermittently, on a newly added service, means intrusion prevention before it means firewall rules. Diagnostic order: test a second known-good port on the same host, `curl` from the same VLAN, then tcpdump on the target.

---

## 11. DNS redirects that never redirected

**Symptom.** During a hardening pass I re-enabled a port 53 block on the Server Admin VLAN and tested it. A lookup sent to a public resolver timed out. If the DNAT redirect were working, AdGuard would have answered it.

**Investigation.** A blocked ad domain asked of a public resolver timed out, while the same name asked of AdGuard returned `0.0.0.0`. So the block caught the query and the redirect did not. All four redirect rules had their interface set to the WAN ("Internet 1"), so they applied to traffic arriving from the internet, not to queries leaving a VLAN.

**Root cause.** The DNAT redirects were bound to the WAN interface. On Trusted and Guest the forcing had really been done by the port 53 blocks alone.

**Fix.** Changed one redirect's interface to its own network and tested it. When that worked, changed the other three the same way. Also added a redirect and a port 853 block for IoT.

**Verified.** From the Server Admin VLAN, the blocked domain asked of a public resolver now returns AdGuard's `0.0.0.0`, and normal names resolve. Household devices and IoT devices were retested afterwards and worked.

**Lesson.** Test the mechanism you think is doing the work, not just the outcome. Two layers that both "work" can hide that one of them never did anything.

---

## 12. UPS low-battery alert storm while on mains

**Symptom.** Discord got "UPS battery is low" every 10 to 25 minutes for an evening, 20 alerts in 90 minutes, while the UPS was on mains at 100% charge.

**Investigation.**
- `upsc`: status `OL`, charge 100%, no on-battery, comms-lost or stale-data events in the journal.
- The NUT config uses `ignorelb` with my own thresholds (runtime under 600 s or charge under 20%). With charge at 100%, the only way to raise low-battery on mains is a runtime estimate under 600 s.
- I ran a read-only sampler (`upsc` every 5 s for 29 minutes). It caught `OL LB` at **52% load with a 540 s runtime estimate**, and 420 s at 61% load. Load spiked when the test farm VM used nearly six cores.

**Root cause.** Real load peaks, not a glitch. The card's uncalibrated runtime estimate dropped under my 600 s threshold during load spikes.

**Fix.** `nut-notify.sh` now re-checks a LOWBATT event after 45 seconds. On battery it still alerts at once. On mains it logs status, charge, runtime, load and voltages to syslog and skips Discord. Shutdown logic was not touched (`upsmon` decides shutdown, not the notifier).

**Verified.** Tested three ways with backups taken first: the real UPS by hand (cleared within 45 s, Discord skipped), and a dummy NUT UPS set to `OL LB` (syslog only) then `OB LB` (posted at once, no shutdown). The dummy was removed and the configs restored byte-identical. Then five real events on mains were handled as designed.

**Lesson.** Sample before you tune. The data showed the alert was telling the truth about a thin margin at peak load, which is now a known number for the power plan.

---

## 13. A switch port "bounce" took the Server VLAN offline

**Symptom.** While troubleshooting the host's flaky link (entry 1), the host, the Docker VM, AdGuard and every other Server VLAN guest dropped off the network entirely.

**Investigation.** The UniFi audit log showed that setting the port to Disabled and then Active in the Port Settings form had reset it: native VLAN Server to empty to **MGMT**, tagged list to Allow All. The host's address is untagged on its bridge, so its traffic landed on the wrong VLAN.

**Root cause.** In UniFi, "Disabled then Active" in the port form is not a link bounce. It clears the port's VLAN settings and fills in defaults.

**Fix.** Set the native VLAN back to Server; the switch was restarted.

**Verified.** All pings 0% loss, SSH to the host worked, and DNS was restored on every VLAN.

**Lesson.** Snapshot a port's full settings before changing it, and read the audit entry (old and new values) after saving. To reset a link, replug or restart the switch.

---

## 14. Host lost power twice in one evening

**Symptom.** The host rebooted twice in about ten minutes, with no shutdown sequence in the journal.

**Investigation.** The journal ended mid-session; `last -x` marked both boots "crash". The UDM's traffic-flow log showed the host's guests still opening connections for about seven minutes after the last journal line, so the journal had lost its final minutes to an unsynced power cut. I ruled out the UPS (on mains, no power or transfer events, other devices on it stayed up), a software reboot (no task, command or watchdog) and hardware (no MCE, EDAC or BERT records, pstore empty).

**Root cause.** A loose power cable at the host. Reseating it caused the second cut.

**Fix.** Reseated the cable.

**Verified.** Third boot clean: every guest running, all containers healthy, NFS mounts back on their own, backup job intact, ZFS healthy, all monitors green.

**Lesson.** A power cut loses the end of the journal, so use the router's flow log to see when a host really went silent. Check every power cable is seated when working in the rack.

---

## 15. Nightly backup notices rejected by Discord

**Symptom.** Short Proxmox notices arrived in Discord, so the channel looked healthy. A full month of nightly backup notices never arrived.

**Investigation.** The host journal showed `could not notify via target 'discord': http status: 400` after every nightly backup. The backup notice puts the whole job log into the message, and the webhook body sent that as the embed description, which Discord caps at 4,096 characters. The sizes matched exactly: logs of 1,106, 1,353 and 2,872 characters were delivered; 3,219 and the roughly 5,700-character four-guest log were rejected.

**Root cause.** Backup logs longer than Discord's embed limit.

**Fix.** A second webhook target that sends only the title and severity, a matcher that routes backup notices to it, and the catch-all matcher changed to exclude backups. Full logs stay in Proxmox task history. I also added sender names so each source is easy to read.

**Verified.** The next nightly run logged "notified via target discord-short" with no 400, and the message arrived.

**Lesson.** "Some alerts arrive" is not "alerts work". Check the sender's log for delivery errors, not just the channel.

---

## 16. Backups were never pruned

**Symptom.** The backup share held 34 backups each of three guests and 31 of the fourth, going back a month: 733.9 GiB.

**Investigation.** The job had no retention of its own and fell back to the storage setting, which was `prune-backups keep-all=1`.

**Root cause.** Retention was never actually set.

**Fix.** Ran `pvesm prune-backups --dry-run` with the target values, checked the list, then set `keep-daily=7,keep-weekly=4,keep-monthly=3` and pruned. Separately, I found 11 old backups on the host's own boot disk from before the NAS existed: archived the first-build backup to an unpruned folder on the NAS (SHA-256 verified), deleted the rest, and removed "backup" from the local storage's allowed content so it cannot happen again.

**Verified.** The prune removed 89 backups (497 GiB) and left 44 (236.9 GiB), exactly matching the dry run. The boot pool went from 64.1 GB to 34.9 GB used.

**Lesson.** Read the effective config file, not the GUI's summary line. A dry run before a destructive change.

---

## 17. Guests did not start after a reboot

**Symptom.** After a host reboot only the Docker VM came back.

**Investigation.** `onboot` was set on that one guest only, so AdGuard and the game server would never have restarted after a power event.

**Root cause.** Missing start-at-boot settings.

**Fix.** Start order with delays: AdGuard first (DNS, 15 s wait), the Docker VM second (30 s wait), then the game VM, then the media container (it needs the NAS mount), and the test farm last.

```bash
pct set 102 --onboot 1 --startup order=1,up=15
qm  set 100 --onboot 1 --startup order=2,up=30
qm  set 101 --onboot 1 --startup order=3
```

**Verified.** On a real reboot, `journalctl -u pve-guests` showed exactly the configured order and spacing.

**Lesson.** Run `date` next to a post-boot `qm list`: a listing taken mid-sequence looks just like a failure.

---

## 18. Hypervisor TLS certificate was never actually deployed

**Symptom.** My notes said the Proxmox certificate was done. A check showed port 8006 still served the self-signed node certificate under the host's old name.

**Investigation.** The custom certificate file did not exist, and every night the update job logged "ACME config found for node, but no custom certificate exists. Skipping ACME renewal".

**Root cause.** The ACME account and plugin were configured but a certificate was never ordered.

**Fix.** Ordered it with `pvenode acme cert order` over DNS-01 through Cloudflare (DNS-01 because the host is not reachable from the internet). It is deliberately not behind the reverse proxy, so it stays reachable with a trusted certificate if the Docker VM is down.

**Verified.** The browser loads the Proxmox UI by name with no warning; the Let's Encrypt certificate shows the right name and expiry; nightly renewal is active. This also unblocked WebAuthn (YubiKey) login, which needs a real hostname.

**Lesson.** Verify the served result (`openssl s_client`, the browser padlock), not the config screen.

---

## 19. Proxmox refused a new admin subnet

**Symptom.** From a new admin subnet, Proxmox alone reported "server stopped responding" while everything else on the Server VLAN worked. It happened three times: with the workstation's subnet, the VPN subnet and the Server Admin subnet.

**Investigation.** Stopping the host firewall brought SSH back but not the web UI, which looked like two faults. The SSH banner's "Last login from" line showed the workstation was on a different subnet than the plan assumed.

**Root cause.** Admin access is allowed in three independent places: the UniFi zone policy, `pveproxy`'s `ALLOW_FROM`, and the Proxmox firewall `management` IPset. Each new subnet was missing from one or two of them.

**Fix.** Added each admin subnet to all three, always from a network that already worked, then removed old entries as a separate step.

**Verified.** The web UI and SSH load from every admin network; from Trusted they do not.

**Lesson.** Confirm the client's subnet from the client itself before writing an allow list. Keep a checklist of every place an allow list lives.

---

## 20. Allow rules that time out

**Symptom.** SSH to the game VM over the VPN timed out while everything else on the tunnel worked.

**Investigation.** I watched the policy's Hits counter while retrying. I hit all three of these on this build, and all three look like a timeout, not an error:
1. A rule named "Allow VPN to Minecraft" had been saved with Action **Block**. The name does not constrain the action.
2. An Allow with Connection State "New + Invalid" matches only the first packet; established traffic falls to the zone default.
3. "Auto Allow Return Traffic" was unticked, so the reply direction (game zone to VPN) hit the default block.

**Root cause.** Case 3 for the SSH timeout; cases 1 and 2 on other rules.

**Fix.** Corrected each rule. Allows use connection state All with return traffic; blocks use New + Invalid only.

**Verified.** SSH over the VPN works. Hits increment on the intended rules.

**Lesson.** Hits not moving means something above is matching first. Hits moving while the connection fails means one of the three settings is wrong.

---

## 21. UPS card unreachable: rule order

**Symptom.** NUT on the host could not reach the UPS card on the MGMT VLAN, though my workstation could.

**Investigation.** The Allow for the host to the card was correct but sat below a blanket "Block Servers to Mgmt". That block's hit counter showed 130 hits, 6 minutes ago, matching the failed attempts exactly.

**Root cause.** Policy order beats specificity.

**Fix.** Moved the narrow Allow (host to card only) above the block. In UniFi, reordering is per zone pair.

**Verified.** NUT reads the card; the block still applies to everything else.

**Lesson.** Check hit counters first.

---

## 22. UPS SNMPv3 passphrase silently truncated

**Symptom.** NUT said `Requested 'mibs' value 'cyberpower' did not match this device`, and `snmpget` said only `Timeout`. This was the longest debugging session of the build.

**Investigation.** An hour of plausible wrong guesses, then two minutes of `tcpdump -ni any host UPS_CARD and udp port 161`: GetRequest out, Report back (engine discovery worked), authenticated request out, silence. That pattern means an authentication or privacy key mismatch and nothing else.

**Root cause.** The card silently truncates an SNMPv3 passphrase longer than its field limit. The NUT error means "no answer", not "wrong MIB".

**Fix.** A passphrase within the card's limit, set on both sides.

**Verified.** `upsc` returns full readings; `battery.runtime` matches the card's web UI exactly.

**Lesson.** Reach for a packet capture early. Error messages describe what the software assumed, not what happened.

---

## 23. Commissioning a used UPS network card

**Symptom.** The second-hand card did not answer on its documented default address, and later its clock would not sync.

**Investigation.** The vendor's network utility found it by broadcast on a previous owner's subnet. A static-IP device stayed invisible in UniFi until it sent traffic. For time, the gateway hands out NTP by DHCP but does not answer NTP queries.

**Root cause.** A previous owner's configuration, and a gateway that does not serve NTP.

**Fix.** Factory reset, static address on the MGMT VLAN, HTTPS and SSH on, HTTP, Telnet, FTP and Modbus off, cipher suites trimmed, default viewer account disabled, configuration exported. NTP pointed at public servers by IP so DNS is not a variable.

**Verified.** The card serves HTTPS, keeps correct time and is read by NUT.

**Lesson.** Treat used gear as untrusted until reset, and find it by broadcast.

---

## 24. NAS rejected every SMB password

**Symptom.** Mapping the NAS from Windows prompted for credentials and rejected both the console's root login and my SSO account.

**Investigation.** The first theory was the firewall (workstation and NAS are on different VLANs). One command disproved it: `Test-NetConnection NAS -Port 445` returned `TcpTestSucceeded : True`.

**Root cause.** UniFi Drive keeps SMB authentication separate from console login. Each user needs a File Service Credential.

**Fix.** Created a File Service Credential and used it for SMB only.

**Verified.** The share maps and reads from Windows.

**Lesson.** You cannot be prompted by a service you cannot reach. A password prompt that rejects you is a credential problem, so test the port before touching the firewall.

---

## 25. Stale NFS mount after a share path change

**Symptom.** After a failed attempt to add NFS storage, `pvesm status` showed no such storage, `findmnt` showed it mounted, `ls` returned "Stale file handle", and `rmdir` said "Device or resource busy".

**Root cause.** The failed add had mounted the export and rolled back the storage config, leaving the mount behind.

**Fix.** `findmnt`, `umount` (then `-f`, then `-l` if needed), then `rmdir`. Then I created all four NAS shares up front before attaching any storage, so paths would not change again.

**Verified.** The storage added cleanly and mounts at boot.

**Lesson.** Never `rm -rf` under `/mnt/pve/` without `findmnt` first: on a live mount it deletes data on the NAS.

---

## 26. IoT DNS filtering broke the AC controller hub

**Symptom.** When I first pointed the IoT VLAN at AdGuard, the IR hub that controls the air conditioner stopped working, in both its own app and Home Assistant.

**Investigation (first pass).** The hub's status LED was the diagnostic that mattered: the vendor's documentation says a flashing light means no internet. With AdGuard protection off, all blocks paused and all redirects paused, it still failed. Only setting IoT DHCP DNS back to the gateway fixed it, so I reverted IoT and wrote down that the mechanism was unknown instead of inventing one.

**Root cause (found later).** During the guest DNS investigation (entry 7) I found IoT had the same rule order: the block to the Server zone sat above the allow to AdGuard. That is the most likely cause of the original failure.

**Fix.** Moved the IoT allow above the block, gave the hub its own AdGuard client with filtering off, and set IoT DHCP DNS to AdGuard. The next day I added the IoT DNS redirect and an 853 block.

**Verified.** IoT devices appear in AdGuard's log. The voice assistant, camera live view, wall switches and the AC hub all tested working after each step.

**Lesson.** A device's own status light can answer in seconds what an hour of firewall reading does not. When the timing of recent changes lines up, still widen the search.

---

## 27. Workstation could reach almost nothing: blocked in AdGuard

**Symptom.** My workstation could load only a couple of large sites.

**Investigation.** I first blamed the NAT redirects and then the reverse-DNS upstream. Both were wrong. TCP 53 connections to AdGuard completed, but UDP queries got silence.

**Root cause.** The workstation was listed in AdGuard's disallowed clients. AdGuard drops a blocked client at the query layer, so the TCP handshake succeeds and nothing answers. I found it myself in AdGuard's access settings.

**Fix.** Removed the entry.

**Verified.** Normal resolution returned at once.

**Lesson.** When DNS breaks, check AdGuard's Query Log and access settings first. It would have shown the cause immediately.

---

## 28. Budget app would not load: SharedArrayBuffer

**Symptom.** Actual Budget died on load: "Actual requires access to SharedArrayBuffer".

**Root cause.** Browsers expose `SharedArrayBuffer` only in a secure context, and plain HTTP to an IP is not one. No compose setting fixes it.

**Fix.** Stood up Nginx Proxy Manager with a wildcard Let's Encrypt certificate (DNS-01) and served the app at `budget.example.com`. The app sends its own cross-origin isolation headers and the proxy passes them through. Later I removed the app's host port entirely so the proxy is the only way in.

**Verified.** The app loads over HTTPS; nothing listens on its old host port.

**Lesson.** Some apps need a real certificate to run at all, not just to look secure. That made the reverse proxy the right investment for every service after it.

---

## 29. Reverse proxy 502: "could not resolve" an IP address

**Symptom.** The proxy returned 502, and its per-host error log said `UPSTREAM_IP could not be resolved (3: Host not found)`: nginx claiming it could not resolve a literal IP. `curl` from inside the proxy container to the same IP and port returned 200 the whole time.

**Root cause.** Nginx Proxy Manager puts the upstream in a variable, and `proxy_pass` built from a variable is resolved at request time through nginx's resolver. That container also had no working resolver at the time (entry 3), which produces the same error.

**Fix.** A shared Docker network called `proxy`, created outside any stack. Every service behind the proxy joins it and is addressed by container name. Services on other hosts (Jellyfin in its own container, the gateway, the NAS) are addressed by IP.

**Verified.** All proxy hosts return 200.

**Lesson.** The real reason for a 502 is only in the per-host error log, not in the browser or `docker logs`.

---

## 30. Admin UIs reachable on host ports, around the proxy

**Symptom.** A read-only security review found that Dockge, Uptime Kuma and the proxy's own admin UI were still published on the Docker VM's host ports. That bypassed the proxy's admin access list.

**Fix.**
1. Moved the two Uptime Kuma monitors that used those ports to container names first, so they would not turn red.
2. Bound the three ports to `127.0.0.1` only, with compose backups, then `docker compose up -d`.
3. Kept a documented fallback: an SSH tunnel to the admin UI if the proxy is ever down.

```yaml
ports:
  - "127.0.0.1:5001:5001"
```

**Verified.** From the workstation the three host ports refuse connections, the three admin names load through the proxy, and all 12 Kuma monitors were up after the restart.

**Lesson.** An access list only protects the path that goes through it. Check every published port.

---

## 31. Minecraft server crash loop from one whitelist name

**Symptom.** After adding a player, the Java server never finished starting; players saw "Server is still starting".

**Investigation.** The container log: `Could not resolve user from Playerdb`. The image validates every whitelist name against Mojang before the server starts, and one unresolvable name aborts startup and restarts the container forever. The name added was an Xbox gamertag, not a Java profile name. Bedrock does not validate names at all.

**Root cause.** A gamertag in the Java whitelist.

**Fix.** Removed the name and used the person's real Java profile name (read from the launcher). The procedure now checks the whitelist file and the log after every change, and uses `docker compose up -d`, never `restart` (restart ignores environment changes).

**Verified.** The server starts; the whitelist entry count went up by one; the log is silent about Playerdb.

**Lesson.** Validate inputs that can take a whole service down before applying them.

---

## 32. Minecraft disconnects from inside the LAN

**Symptom.** Players inside the house were disconnected repeatedly.

**Investigation.** Two different causes in one evening. Two drops were TCP resets ("forcibly closed") on connections made through the public hostname from inside the LAN. A later clean "Disconnected" was VanillaFix catching a client-side mod crash.

**Root cause.** NAT hairpin through the public name, plus a separate client mod crash.

**Fix.** Inside the LAN, connect to the game VM's internal address, not the public name. Added firewall allows for the Server Admin VLAN to reach only the game and SSH ports.

**Verified.** LAN players stay connected; outside players use the public name.

**Lesson.** Test public forwards from cellular, never from inside the network.

---

## 33. Jellyfin hardware decode settings that did not match the GPU

**Symptom.** Some files stuttered before falling back to software decode.

**Investigation.** I read the GPU's real capabilities with the `vainfo` bundled in jellyfin-ffmpeg, from inside the container (there is no system `vainfo` there). It lists decode and encode for H.264 and HEVC (including Main 10) and decode only for VP9 and AV1. VC1 had been ticked even though no VC1 profile exists, and plain VP9 was unticked while VP9 10-bit was ticked.

**Root cause.** Checkbox settings that did not match the hardware.

**Fix.** Ticked exactly what `vainfo` lists and nothing else. AV1 encode off (no AV1 encode support).

**Verified.** A forced low-quality stream showed "Transcoding (hw)", and a test as the `jellyfin` user turned 4K HEVC into 720p H.264 entirely on VAAPI at 104 fps.

**Lesson.** Never pipe a diagnostic into `grep` on the first run: an earlier `vainfo 2>/dev/null | grep VLD` returned nothing and looked like a GPU failure, because the pipe hid "command not found".

---

## 34. Media library misread by Jellyfin

**Symptom.** Two anime series sorted wrongly or showed as several broken series.

**Investigation.** One series used per-season episode numbers with an absolute number in brackets. The other had season folders Jellyfin could not parse, no season marker in some file names, an unrecognised `ONA` folder and no year on the show folder.

**Root cause.** Naming that did not follow Jellyfin's rules.

**Fix.** Two Python rename scripts, each with a dry run by default, collision checks and an `--apply` flag. The first renamed 366 files to Season 1 with three-digit absolute numbers. The second moved 67 files into a `Show (Year)/Season NN` layout with specials in Season 00 and extras in `extras/`, and fixed a mislabelled episode title along the way. The empty old folders were removed afterwards.

**Verified.** Dry runs showed all files matched and none unmatched before applying. Jellyfin now shows each as one series with the right seasons; the file count and total size were unchanged after the cleanup.

**Lesson.** Write rename scripts that default to a dry run, and take a snapshot of the share first (the media share has daily snapshots for this reason).

---

## 35. Host link alert that could not report its own outage

**Symptom.** After the link failure in entry 1, I added a per-minute link check that alerts after 3 bad minutes. During the next overnight failure, no alert from the host ever reached me.

**Investigation.**
- The log showed zero "alert sent" lines. The check had tried to post after 3 bad minutes, but the post had to leave over the same dead link. When the link came back, the counter reset without a "clean again" post, because nothing had been sent.
- Separately, the check's receive-error figure always read 0: cron runs without `/usr/sbin` in `PATH`, so `ethtool` was not found. The ping trigger itself worked.
- The healthchecks.io "host alive" heartbeat (an outside check, pinged every 5 minutes) **did** alert that night.

**Root cause.** An inside alert cannot report the failure of the path it needs, and a cron `PATH` bug.

**Fix.** The check now records "alert could not be sent" and, on the first clean minute, posts once "link was bad from X to Y (N min). The alert could not be sent at the time", retried until it gets out. Added a `PATH` line to the script. The outside heartbeat stays as the real-time alert for a dead host link.

**Verified.** Run with cron's environment (`env -i PATH=/usr/bin:/bin`), the error figure now matches `ethtool`. No-post simulations covered the dead-link, recovery and normal cases.

**Lesson.** Every alert path needs an outside dead-man's switch. Test scripts with cron's environment, not your shell's.

---

## 36. VPN tunnel-down alert gave a false reading

**Symptom.** One minute after the outbound VPN route came back on, my tunnel-down check counted the tunnel as down while traffic was flowing.

**Investigation.** The check also used "sending with nothing coming back" from UniFi's rate fields. A three-minute sample showed those fields update only every one to two minutes and sit at 0 in between. Two such readings in a row would have sent a false alert. I wanted a WireGuard handshake-age rule instead, but UniFi's API exposes no handshake field.

**Root cause.** Rate fields that refresh too slowly to use as a liveness signal.

**Fix.** The check now uses only the tunnel status UniFi reports, with two bad checks in a row before alerting, one alert when down and one when back, and no alert while the route is paused. It now also alerts if the host cannot reach UniFi for 5 minutes, and a failed Discord post is retried instead of being marked sent.

**Verified.** No-post simulations covered down, repeat, recovery and unreachable cases. It has run without false readings since.

**Lesson.** Know how often a metric really updates before you alert on it. A known gap remains: a tunnel that dies while UniFi still says "connected" is not caught.

---

## 37. VPN kill switch failed its test

**Symptom.** Two VLANs leave through a commercial WireGuard VPN on the gateway, with UniFi's kill switch on. I tested it: a watcher on the workstation recorded the egress (VPN, home IP or none) every 3 seconds while a script turned the VPN client off for 40 seconds.

**Investigation.** Traffic went out through the **home IP for about 37 seconds**, then came back through the VPN when the client was re-enabled.

**Root cause.** On this UniFi version the kill switch does not hold when the VPN client is disabled. Another UniFi user reported the same for WireGuard clients.

**Decision.** Accepted as a UniFi limitation. Mitigated with the tunnel-down alert in entry 36, which says plainly that those VLANs are using the home IP.

**Verified.** The watcher log showed the exact timeline. The route and client were back to normal afterwards and the test script was removed.

**Lesson.** Test a privacy control by failing it on purpose, and run the watcher as a separate process so it keeps recording if your session drops.

---

## 38. Remote VPN client DNS: a typo and a "leak" that was not one

**Symptom.** After pointing the phone's WireGuard tunnel at AdGuard for DNS, the photo app's hostname would not load over cellular, though AdGuard's own page did. Separately, a DNS leak test from the phone on home Wi-Fi showed the ISP's resolvers.

**Investigation.**
- Every resolver answered the hostname correctly, and the proxy logged no requests from the phone. A 15-minute capture on AdGuard's and the Docker VM's interfaces, filtered to the phone's tunnel address, showed nothing. The tunnel's DNS line had a typo: an address one digit off, where nothing listens.
- For the leak test: DHCP on that VLAN hands out AdGuard only; lookups sent to the gateway ended at Cloudflare, not the ISP; and the "unknown" resolver was in the same /24 as Quad9, AdGuard's other upstream. The ISP rows came from the phone's cellular connection (iOS Wi-Fi Assist).

**Root cause.** A typo in the client config, and a phone quietly using cellular during the test.

**Fix.** Corrected the tunnel's DNS address on the phone, and set the same DNS on the travel router's tunnel. WireGuard DNS lives in each device's config, so changing the server's field does not change existing peers. Wi-Fi Assist turned off on the phone.

**Verified.** A capture showed 71 queries from the phone reaching AdGuard (8 of them blocked, so filtering applies), and the photo app loaded. The leak test repeated with cellular data off showed only Cloudflare and Quad9.

**Lesson.** Check the client config for typos before the network. A leak test is only as good as knowing which interface the device used.

---

## 39. Test farm runs killed by automatic updates

**Symptom.** A long automated test run on the test farm VM stopped two seconds after a security update installed a library.

**Investigation.** The unattended-upgrades log showed `systemctl restart` of the farm service right after the library update. `needrestart` was set to restart affected services automatically. Earlier I had also found unattended-upgrades set to reboot at 05:15, which could cut a run.

**Root cause.** Automatic service restarts and reboots on a VM whose jobs run for hours.

**Fix.** Turned automatic reboot off (reboots happen by hand between runs when `/var/run/reboot-required` appears). Added a needrestart override so updates still install but this one service is never restarted; the next run picks up the new libraries.

```perl
# /etc/needrestart/conf.d/60-farm.conf
$nrconf{override_rc}{qr(^test-farm\.service$)} = 0;
```

**Verified.** needrestart loads the file, the service matches the rule with value 0, and a list-only run restarted nothing.

**Lesson.** Security updates should install automatically; restarts should follow the workload's schedule.

---

## 40. ONU stick gave "no DHCP"

**Symptom.** On commissioning the fiber, the router got no DHCP lease from the ISP.

**Investigation.** Over an hour in, a `tcpdump` on the ONU stick's interface showed LAN ARP traffic from the router's LAN address. The ONU was seeing the LAN.

**Root cause.** The ONU stick was in the router's LAN SFP+ cage and the DAC was in the WAN cage.

**Fix.** Swapped them. The ISP also holds a DHCP lease against the hardware ID for 20 to 30 minutes, so I waited before changing anything else.

**Verified.** WAN up with a public address; the router's speed test showed 3.69 Gbps down and 5.01 Gbps up. I recorded an optical baseline (Rx and Tx power, temperature, laser bias) for future comparison.

**Lesson.** If the ONU sees LAN traffic, it is in the wrong port. Record a baseline on day one.

---

## 41. Smart switch pairing for local control

**Symptom.** Home Assistant's Tuya Local integration rejected the wall switches' app account: "please use the designated app".

**Root cause.** The cloud-assisted setup (used once to fetch device IDs and local keys) authenticates only against a Smart Life or Tuya Smart account, not the switch brand's own app.

**Fix.** Re-paired the switches into Smart Life, fetched the keys once, and then control runs fully locally. Installed the integration by hand (it is not in the HACS default search under that name), and did not run the similar `localtuya` project alongside it, because Tuya devices accept one local connection at a time.

**Verified.** The switches respond locally from Home Assistant.

**Lesson.** Read which account type an integration expects before re-pairing hardware.

---

## 42. Findings fixed from a read-only security review

I ran two read-only reviews (privacy and exposure, then security and anonymity) from live reads plus web research, ranked the findings, and then fixed the top items one at a time, each with a rollback and a test. These are the ones that are done and verified:

| Finding | Fix | Verified by |
|---|---|---|
| Internal hostnames published in public DNS with private addresses | Moved them to AdGuard rewrites and UniFi local DNS records, deleted the public records and an unused CNAME | Cloudflare answers NXDOMAIN; every VLAN, the VPN and the containers still resolve the names |
| Unused switch ports defaulted to MGMT with every VLAN | All unused ports set to an isolated BlackList VLAN, no tagged VLANs, PoE off | Same ports up before and after; no device lost |
| Trunks carried every VLAN | Each port now carries only what it needs (for example the AP: Trusted, IoT and Guest only) | A capture on the host saw only its one tagged VLAN, and no guest broadcasts reached AdGuard's port |
| Trusted-to-proxy rule allowed all protocols | Narrowed to TCP 443 | Photos load from Trusted; admin ports do not |
| IoT and game zones could reach the gateway's login page | Allow only DHCP, DNS, NTP (and mDNS for IoT), then block | Login page and SSH unreachable from those zones; devices and game server still work |
| Server Admin port 53 block paused and misnamed | Renamed and resumed | Lookups to public resolvers are answered by AdGuard |
| Admin UIs on host ports | Bound to localhost (entry 30) | Ports refuse; names load |
| Certificate API tokens usable from anywhere | IP-filtered to the home IP, with a home-IP-change alert | Tokens verify from their hosts |
| Media server Quick Connect on | Turned off | API reports it disabled |
| Game server telemetry (snooper) on | Turned off | Server properties read back |
| Domain could be spoofed in email | SPF `-all`, DMARC reject, null MX, CAA | Records read back from Cloudflare's name server |
| Guest Wi-Fi without client isolation | Client device isolation on | AP stayed up, no client dropped |

---

## 43. Operator traps worth writing down

Small mistakes that cost time, recorded so they are not repeated:

- A multi-line paste that starts with `sudo` loses its second line to the password prompt; the rest runs and the change silently does not exist. Run `sudo -v` first and verify the file, not the absence of an error.
- The Proxmox web shell flattens multi-line pastes (it once produced a file literally named `99-local.confsshd -t && systemctl reload ssh`). Use SSH for multi-line work.
- Never put `<placeholder>` text in a command meant to be pasted: bash reads `<` as a redirect. Use a bare UPPERCASE word.
- `journalctl -u UNIT -t TAG` ANDs the two filters; query them separately.
- `pct reboot` exists; `pct restart` does not.
- A restored guest keeps the original's MAC unless "Unique" is ticked; on a fixed-IP network that could have taken DNS down. Restore drills go to a spare ID with networking disconnected.
- `qm shutdown` returns at once when a guest agent is present; poll `qm status` instead of timing the command.
- The ZFS event daemon's unit is `zfs-zed`, not `zed`; `journalctl -u zed` returns "No entries", which looks like a dead daemon.
- Never analyse a 2 GB DNS query log inside a 2 GB container; stream it to the host instead.

# Network

How the network is built and why. Addresses are left out on purpose; VLANs are described by name.

## WAN

- 5 Gig XGS-PON fiber. The ISP's gateway is bypassed and shelved: an SFP+ XGS-PON ONU module sits directly in the UDM Pro Max's SFP+ WAN port. No double NAT, and WireGuard listens directly on the WAN.
- Public IPv4 by DHCP, kept current in DNS by UniFi's native Cloudflare DDNS. IPv6 is off everywhere on purpose, so nothing routes around the IPv4 controls.
- An optical baseline (receive and transmit power, module temperature, laser bias) was recorded at commissioning. A downward drift in receive power is the early warning for a dirty connector.
- Router speed test: 3.69 Gbps down, 5.01 Gbps up. The router's IPS is rated near 5 Gbps, which matches the plan.

## Switching

| Layer | Device | Links |
|---|---|---|
| Edge | UDM Pro Max | SFP+ WAN (ONU), 10G DAC to the core |
| Core | USW Pro Aggregation | 10G SFP+ backbone, uplink to the access switch |
| Access | USW Pro Max 24 PoE | 2.5G PoE++ for the AP, 10G SFP+ for the NAS, 2.5G for the hypervisor |
| Wireless | U7 Pro Max | 2.5G PoE uplink, ceiling mounted |

Why the host is on 2.5G today: its onboard 5GbE NIC was not reliable through a copper SFP+ module (see [troubleshooting entry 1](troubleshooting.md#1-server-5gbe-link-drops-through-a-copper-sfp-module)). The plan is an Intel SFP+ card (X710 class, not bought yet) with a DAC. DACs are preferred over copper or optical modules: less heat, no compatibility surprises.

The NAS array (four 7200 RPM drives in RAID 10) delivers roughly 400 to 500 MB/s sequential, so 10G to the NAS is plenty and the host link is the real bottleneck. The restore drill read at 257 MiB/s, which is about 92% of a 2.5G link.

Port policy:

- Every port that carries a device is configured before anything is plugged in.
- Each trunk carries only the VLANs its device needs. The hypervisor port has Server native and only the game VLAN tagged. The AP port carries only the three SSID VLANs. Single-purpose devices (NAS, UPS card, camera base station, workstation) get no tagged VLANs.
- Every unused port is named "Unused", sits on the isolated BlackList VLAN with no tagged VLANs, and has PoE off.
- The hypervisor's old 5G port is kept out of use: it is the known-bad path from troubleshooting entry 1.

## VLANs and zones

| VLAN | Name | Who lives there | DNS |
|---|---|---|---|
| 1 | MGMT | Network gear, UPS card | Gateway |
| 20 | Server | Hypervisor, guests, NAS | Gateway (see below) |
| 30 | Trusted | Household phones and laptops | AdGuard |
| 40 | IoT | Smart switches, AC hub, camera base station, voice assistant | AdGuard |
| 50 | Guest | Visitors, internet only, client isolation on | AdGuard |
| 60 | Minecraft | The public game VM, alone in its own zone | Gateway |
| 70 | Server Admin | Admin workstation | AdGuard |
| 999 | BlackList | Unused ports; no DHCP, no internet | none |
| (VPN) | VPN zone | WireGuard peers | AdGuard |

Each VLAN has its own custom firewall zone. New custom zones default to block everything except the internet and the gateway.

### Firewall design

I describe the design here rather than the rule table.

- **Default deny between zones.** Access is granted with narrow allow rules: one source zone, one destination, the fewest ports that work.
- **Allows are stateful.** Connection state All with automatic return traffic. The destination zone cannot start anything back.
- **Blocks match new and invalid connections only,** so they never cut established flows that an allow already permitted.
- **Household (Trusted) devices** reach only three things on the Server VLAN: Home Assistant, the reverse proxy on HTTPS and DNS. Admin interfaces need the Server Admin VLAN or the VPN.
- **IoT** cannot start connections to any other internal zone. Home Assistant reaches into IoT, not the other way round. Camera live view works because the phone opens the connection and the reply rides the return path; no IoT exception is needed.
- **Guest** reaches the internet and AdGuard's DNS port only. Guests are blocked from the gateway (except DHCP), from every internal zone and from all private address ranges.
- **The game zone** has no path into Server or Trusted. It is the only zone with inbound port forwards (the two game ports, nothing else).
- **IoT and the game zone** can reach the gateway only for DHCP, DNS and NTP (plus mDNS for IoT).
- **Order matters more than specificity.** The policy Hits counter is the check, not whether a page loads.

Lessons from building it are in [troubleshooting entries 19 to 21](troubleshooting.md#19-proxmox-refused-a-new-admin-subnet): three ways an allow rule fails as a silent timeout, rule order, and admin allow lists that live in more than one place.

### Why the Server VLAN uses the gateway for DNS

AdGuard is a guest on the hypervisor, which sits on the Server VLAN. If the hypervisor depended on a container it hosts, a cold boot would have a window with no name resolution. The Server and MGMT VLANs and the game VLAN therefore use the gateway. The game VLAN stays on the gateway so there is no path from the public game VM into the Server VLAN.

## DNS

### Filtering

- AdGuard Home in its own LXC container. It is infrastructure, so it does not share a VM with application stacks.
- Upstreams are DNS-over-HTTPS only (Quad9 and Cloudflare), load balanced. **No plaintext fallback:** a fallback would silently downgrade queries when the encrypted upstreams are slow.
- Blocklists: AdGuard's own filter, HaGeZi Pro, a threat-intelligence list, Apple, Amazon, Microsoft and TikTok tracker lists, and HaGeZi's DoH/VPN/proxy bypass list. Three or four content lists is the right number; more add false positives faster than coverage.
- Query log kept 14 days. NXDOMAIN rules for iCloud Private Relay and the Firefox DoH canary so those clients fall back to the house resolver.
- Per-device exceptions use AdGuard persistent clients with filtering off, for example the AC hub.

### Forcing every client through AdGuard

A device on a filtered VLAN can either use AdGuard or have no DNS. Three layers per VLAN:

1. DHCP hands out AdGuard as the only DNS server. A second entry would defeat it, because clients use whichever answers first.
2. A destination-NAT redirect sends any port 53 traffic to AdGuard. Each redirect is bound to its own network as the interface (binding it to the WAN made it do nothing; see [entry 11](troubleshooting.md#11-dns-redirects-that-never-redirected)).
3. Blocks for port 53 and port 853 (DNS-over-TLS) to the internet. The DoH bypass list handles DNS-over-HTTPS, which looks like normal web traffic.

**The one rule never to build:** a port 53 block from the Server zone to the internet. AdGuard's bootstrap resolvers use plain DNS on port 53, and blocking them takes DNS down for the whole house.

**Single point of failure, accepted.** If the AdGuard container stops, every filtered VLAN loses DNS. Mitigations: an Uptime Kuma DNS monitor that resolves a real name through AdGuard, the container starts first at boot, and a documented fallback (tick "Auto DNS server" on each network in the UniFi app).

### Internal names

Internal service names (`photos.example.com`, `jellyfin.example.com` and the admin names) are not in public DNS. They resolve inside the house in two places:

- AdGuard DNS rewrites, for every client that uses AdGuard, including VPN peers.
- UniFi local DNS records on the gateway, for the Server VLAN, which uses the gateway.

Every web name points at the reverse proxy's address, never at the service itself. Adding a service means a proxy host, an AdGuard rewrite and a UniFi record; no new certificate, because the wildcard covers it. The only public record is the game server's name.

## Wi-Fi

- Three SSIDs: Trusted, IoT and Guest. Guest has client device isolation on. mDNS is proxied only between the VLANs that need discovery, never Guest.
- The AP is ceiling mounted, centred on the living space. 6 GHz is treated as a same-room band (see [entry 8](troubleshooting.md#8-poor-wi-fi-coverage-was-a-6-ghz-band-problem)).
- The AP's 2.5G uplink is sized for real throughput: Wi-Fi is half duplex and shared, and real-world totals are 40 to 60% of the advertised PHY rates.

## Outbound privacy route

The Trusted and Server Admin VLANs leave through a commercial WireGuard VPN client on the gateway, with a policy-based route for those two VLANs only. The gateway itself and the Server, IoT, Guest and game VLANs never use it. A host cron job turns the route off at 08:00 and back on at 17:00 on weekdays, through a dedicated local UniFi account limited to Network site admin. A per-minute check alerts if the tunnel drops while the route is on, because UniFi's kill switch does not hold in every case ([entry 37](troubleshooting.md#37-vpn-kill-switch-failed-its-test)).

## Example: verifying DNS forcing from a client

```text
ipconfig /flushdns
nslookup doubleclick.net             -> Server: adguard, 0.0.0.0
nslookup doubleclick.net 8.8.8.8     -> answered by AdGuard (0.0.0.0), via the redirect
```

Run it from a client on the VLAN under test. A server-VLAN VM cannot prove anything about a rule written for another zone.

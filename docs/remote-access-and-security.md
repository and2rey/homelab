# Remote access and security

This page describes the security design and the controls that are in place. It does not list firewall rule tables, and it describes fixes only where they are done and verified.

## What the internet can see

- **Two forwarded ports,** both for the game servers, both pointing at a VM that sits alone in its own zone. UPnP and NAT-PMP are off.
- The gateway's WireGuard port is the one other open port. WireGuard does not answer packets that fail authentication, so it does not show up in scans.
- A passive outside lookup (a public scan database, not my own scan) showed only those two ports and no known vulnerabilities.
- **No web service is published.** Photos, media and every admin page are reachable only from inside the house or over the VPN.
- **Public DNS holds only the game server's name.** Internal names were removed from public DNS and now resolve only inside the house.
- Certificate transparency logs show only the wildcard and apex certificates, not individual service names. That is a side benefit of using one wildcard.
- WHOIS contact details are redacted by the registrar.

## Remote access: WireGuard

- The WireGuard server runs on the UDM. Clients use the DDNS name as their endpoint, so an IP change needs no client edits.
- Two peers: a phone (full tunnel) and the GL.iNet travel router.
- VPN clients land in their own zone, which defaults to block. They are allowed to the servers, household, IoT and game zones; MGMT and Guest stay blocked.
- Peers use AdGuard for DNS, so filtering and internal names work on the road. That setting lives in each client's config, not on the server.
- IPv6 is off on the tunnel and the travel router, so nothing can route around it.
- **A client config or QR code is a credential.** It goes from screen to device with nothing in between. One peer whose QR appeared in a screenshot was deleted and recreated.

### Travel router

- Firmware updated before setup (older builds had a WireGuard rekey bug that looks like a misconfiguration).
- Kill switch on, so devices lose connectivity rather than falling back to the hotel network.
- The tunnel covers only the main LAN. The travel router's guest network goes straight to the local internet and never enters the tunnel, so people I hand the guest Wi-Fi to cannot reach home.
- Admin panel not reachable from the WAN; cloud management, WireGuard server and OpenVPN server all off.

### Remote desktop to the workstation

- RDP is allowed only from the travel router's tunnel path: one UniFi rule from the VPN zone to the workstation's RDP port, and the Windows firewall scoped to the same source.
- Network Level Authentication on, RDP logon right limited to one account, account lockout after 5 attempts.
- A Duo second factor (Verified Push) on RDP logins, on a trial licence for now.
- An outside test over a phone hotspot passed end to end.

## Admin access

| System | Protection |
|---|---|
| Hypervisor web UI | Two-factor on every account. Root: YubiKey WebAuthn, TOTP and paper recovery keys |
| Hypervisor SSH | Keys only (ed25519 with a passphrase), password login off |
| Hypervisor allow lists | Admin subnets only, in all three places (UniFi zone policy, `pveproxy`, Proxmox firewall IPset) |
| UniFi cloud account | Hardware passkey and authenticator app |
| UniFi local break-glass admin | Kept on purpose for when cloud login is unavailable |
| Admin web pages | Behind the reverse proxy with an access list (Server, Server Admin and VPN networks only); host ports bound to localhost |
| Least-privilege automation | A dedicated local UniFi account limited to Network site admin, used only by the route schedule and tunnel check |

## TLS

- **Every service behind the reverse proxy has a trusted certificate,** including the admin pages of the gateway, the NAS and the UPS card, which only ship self-signed ones.
- Let's Encrypt through **DNS-01** with Cloudflare. DNS-01 issues real certificates for hosts that are not reachable from the internet, and it allows a wildcard, so a new service never needs a new certificate.
- Two issuers on purpose: the reverse proxy holds the wildcard; the hypervisor holds its own certificate so it stays reachable with a trusted certificate even when the Docker VM is down.
- Renewal is automatic. I ran a dry run against a copy of the proxy's certificate folder to prove the DNS plugin still worked after an upgrade.
- CAA records limit issuance to Let's Encrypt.

## Secrets

- **One API token per consumer,** each scoped to DNS edit on the one zone, each named for its consumer. Revoking one breaks nothing else.
- The two tokens used for certificates are IP-filtered to the home address. An hourly check compares the public address with the last one seen (stored only as a SHA-256 hash) and alerts if it changes, so the filters can be updated before a renewal fails.
- Secrets live in root-only files (mode 600). Scripts read them at run time and never print or log them. Webhook and ping URLs are treated as secrets too.
- Tokens are verified by ID only, with a small script that prints the ID and status and never the value.
- Leaked or exposed credentials were replaced, not reused: a printed Cloudflare token, a UPS SNMP passphrase and a VPN peer QR code that appeared in screenshots ([troubleshooting entry 9](troubleshooting.md#9-leaked-api-token-and-credential-cleanup)).
- Unused credentials were deleted: an old hypervisor API key and a dead SMTP target.

## Patching and versions

- Debian guests install security updates automatically every day, with no automatic reboot. Reboots are manual and checked against active users and jobs.
- Container images are pinned to exact versions. I track advisories for each service and upgrade on purpose. Done so far: the reverse proxy (CVE fix), Immich (a high-severity advisory), Jellyfin, Docker Engine on both VMs. I also checked one advisory that did not apply (it needs a feature I do not use) and wrote down why.
- Network device firmware is read and recorded; all devices were current when checked.

## Domain and email hygiene

The domain sends no mail, and says so: SPF `v=spf1 -all`, DMARC `p=reject` with strict alignment, and a null MX. CAA allows only Let's Encrypt. DNSSEC signing is enabled; the DS record is still waiting to be published at the registrar.

## Privacy

- AdGuard blocks ads, trackers and telemetry for every filtered VLAN, including guests, and blocks known DoH, VPN and proxy bypass hosts. Device-specific exceptions are explicit persistent clients, not global allow rules.
- iCloud Private Relay and the Firefox DoH canary get NXDOMAIN, so those clients fall back to the house resolver instead of going around it.
- Query log retention cut from 90 days to 14.
- Game server telemetry off. Home Assistant analytics and Jellyfin metrics confirmed off.
- Two VLANs leave through a commercial WireGuard VPN on the gateway on a schedule, with a tunnel-down alert ([network.md](network.md#outbound-privacy-route)).

## Review process

I ran two read-only reviews before changing anything: a privacy and exposure review, then a deeper security and anonymity review. Each combined live reads (gateway API, Cloudflare, every guest, the NAS), a 41-day analysis of 1.2 million DNS queries, certificate logs, a passive outside view, and web research on current best practice. Findings were ranked by benefit against effort. I then fixed the top items one at a time: each change shown with its rollback, applied, tested, and recorded. The list of what was fixed is in [troubleshooting entry 42](troubleshooting.md#42-findings-fixed-from-a-read-only-security-review).

Rules I follow while doing this:

- Never print the public IP, a token, a password or a webhook. Outputs that could contain them are masked before display.
- Read-only first, then one change at a time with a rollback ready.
- Test from the zone the rule is about, and include must-fail tests, not only must-work tests.

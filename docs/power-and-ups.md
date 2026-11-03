# Power and UPS

## Hardware

| Item | Detail |
|---|---|
| UPS | CyberPower CP1500PFCRM2U, 1500 VA / 1000 W, pure sine on battery, 2U rack |
| Battery | 24 V (2x 12 V 9 Ah cartridge) |
| Network card | RMCARD205, firmware 1.6.0, on the MGMT VLAN |
| Second UPS and PDU | UniFi Power Kit (UniFi UPS 2U plus Power Distribution Pro): bought, install planned |

The UPS and card were bought second-hand. I factory reset the card, found it by broadcast (it was not on its documented default address), gave it a static address, turned on HTTPS and SSH, turned off HTTP, Telnet, FTP and Modbus, trimmed the cipher suites to three GCM suites, disabled the default viewer account and exported the configuration. The gateway does not answer NTP, so the card uses public NTP servers by address.

## Load and runtime

| Reading | Load | Runtime |
|---|---|---|
| Before the NAS | 222 W (22%) | 34 min |
| With the NAS (design number) | 256 W (25%) | 26 min |
| Later, with more running | about 36 to 40% | about 18 to 20 min |
| Load spikes (sampled every 5 s) | 52% | 540 s estimate |

Sampling showed that at about 52% load the card's runtime estimate drops under the 10-minute shutdown threshold. That is why the second UPS is planned: moving network gear onto it lowers the load on this one. Filling the switch's PoE budget with cameras is the one change that could overload the 1000 W output, so load is re-measured after any big addition.

## NUT on the hypervisor

NUT runs in standalone mode on the host and reads the card over **SNMPv3 (authPriv, SHA and AES)**. SNMPv1 has no access at all: every community is set to Forbidden.

```ini
# /etc/nut/ups.conf (sanitized)
[ups]
    driver = snmp-ups
    port = UPS_CARD_ADDRESS
    mibs = cyberpower
    snmp_version = v3
    secLevel = authPriv
    authProtocol = SHA
    privProtocol = AES
    ignorelb
    override.battery.runtime.low = 600
    override.battery.charge.low = 20
```

`ignorelb` plus the two overrides tells NUT to ignore the UPS's own low-battery flag (which asserts near the end of the discharge curve) and use my numbers: start shutdown with 10 minutes left, with 20% charge as a backstop. Ordinary blips never get close.

**Alerts.** `nut-notify.sh` posts every UPS event to Discord. A low-battery event on mains is re-checked after 45 seconds and only logged, because short load spikes push the runtime estimate under the threshold ([troubleshooting entry 12](troubleshooting.md#12-ups-low-battery-alert-storm-while-on-mains)). On battery it alerts at once. `upsmon` alone decides shutdown, so the notifier cannot affect safety.

## Ordered shutdown chain

`upsmon` runs `/usr/local/bin/ups-shutdown.sh` at 10 minutes of runtime left:

1. Stop the test farm VM (60 s). Nothing on it needs saving, and stopping it releases its SMB mount before the NAS goes.
2. Stop the game VM (180 s). The Java container has a 120-second stop grace period so the world saves.
3. Stop the Docker VM (120 s).
4. Stop AdGuard last of the guests (60 s). Nothing needs DNS while everything is shutting down.
5. Disable the NFS backup storage and unmount both NFS mounts. They are hard mounts, so the NAS must not disappear first or the host's own shutdown hangs.
6. Power off the NAS over SSH (`ubnt-systool poweroff`) with a 30-second timeout. Failure only logs and notifies.
7. Halt the host.

Every step ends in `|| true`, so no single failure can stop the host from halting. Guests are stopped with `qm shutdown` / `pct shutdown`, then polled with `qm status`, then forced if they exceed their budget. The Jellyfin container is not in the list; it stops when the host halts.

```bash
stop_guest() {
  local kind=$1 id=$2 name=$3 wait=$4 cmd i
  cmd=$([ "$kind" = vm ] && echo qm || echo pct)
  $cmd shutdown "$id" --timeout "$wait" >> "$LOG" 2>&1 || true
  i=0
  while [ "$($cmd status "$id")" != "status: stopped" ] && [ "$i" -lt "$wait" ]; do
    sleep 2; i=$((i+2))
  done
  [ "$($cmd status "$id")" = "status: stopped" ] || $cmd stop "$id" >> "$LOG" 2>&1 || true
}
```

**The NAS in the chain.** The UNAS Pro 8 has no NUT client of its own, so the host shuts it down over SSH with a password kept in a root-only file. A password, not a key, on purpose: the NAS auto-updates its firmware, and an update can reset the vendor-managed key store. A silently broken key would mean the NAS never shuts down, and I would find out during an outage. The command runs with `</dev/null` because it behaves differently without a terminal, and `upsmon` has none.

### Drill results

| Step | Elapsed |
|---|---|
| Chain starts | 0 s |
| Game VM down | 4 s |
| Docker VM down | 16 s |
| AdGuard down | 18 s |
| NFS released | 18 s |
| NAS power-off accepted | 21 s |
| Host halting | **26 s** |

A real pull-the-plug drill (threshold raised temporarily so it fired after two minutes). 26 seconds against a 600-second budget, Discord messages in order, no data loss. The NAS was also proven to boot on its own when AC returns. The test farm step (step 1) was added after this drill and has not been drilled yet.

## Recovery after power returns

| Piece | Setting |
|---|---|
| Host | BIOS Restore on Power Loss: On |
| NAS | Boots itself when AC returns (tested with a 30-second AC cut) |
| Backup storage | A systemd unit waits up to five minutes for the NAS, then re-enables the NFS backup storage (disabling it in the chain persists across reboots) |
| Guests | Start in dependency order with delays (DNS first) |
| Long outage | Over Discharge Protection (20 minutes at 0%) puts the UPS to sleep, so output restores when mains returns and everything boots |

**Known limitation:** after a short outage (mains back before the battery empties) the UPS never cuts its output, so the halted host and NAS stay off until someone powers them on. I tried four ways to make the card cut its output on command; it accepts the SNMP writes and does nothing ([entry 5](troubleshooting.md#5-ups-shutdown-chain-and-the-killpower-investigation)). Next options: the vendor's own agent, or remote power control through the new PDU.

## Plan for the UniFi Power Kit

- **UniFi UPS 2U:** the router and both switches, plugged straight into battery-backed outlets, never through the PDU. Estimated 110 to 140 W, which gives a long runtime.
- **CyberPower:** the hypervisor, the NAS and the workstation, which stay on the pure-sine unit.
- The network must outlast the server chain, because the chain powers the NAS off over the network and the UPS card is reached through a switch.
- **PDU (later):** on the CyberPower side, for remote power-cycling, with power-on defaults staggered: NAS first, the host about two minutes later so NFS is up.
- Move one device at a time, avoiding the backup and dump windows. Then re-measure both loads, add a monitor for the new UPS, and re-drill the full chain.

The rack was moved on 1 November 2026, ahead of this install.

# Storage and backups

## Hypervisor storage

- ZFS root pool `rpool` on a single 1 TB NVMe (`ashift=12`, compression on). It holds the OS, every guest disk, the Immich database and the game world.
- About 35 GB used after cleanup, growing under 1 GB a week.
- Scrub on the second Sunday and trim on the first Sunday of each month, from Debian's own `zfsutils-linux` cron. ZED reports a finished scrub to Discord, so silence is never mistaken for health.
- SMART on the NVMe is checked and recorded (wear, spare, media errors, power-on hours).
- **Known gap:** a single disk means a scrub can detect corruption but not repair it. The nightly backups make a failure survivable, at the cost of a rebuild. The fix is planned: attach a second NVMe of at least the same byte capacity with `zpool attach`.

## NAS: UniFi UNAS Pro 8

| Item | Value |
|---|---|
| Pool | One storage pool, RAID 10, 4x 24 TB 7200 RPM |
| Usable | About 48 TB |
| Free bays | Four (the pool can take a second RAID group) |
| Network | 10G SFP+ on the Server VLAN |
| Encryption | Off on purpose |

**RAID 10 over RAID 6.** Both give the same capacity from four drives. RAID 6 survives any two failures; RAID 10 survives two only in different mirror pairs. The deciding factor at 24 TB is rebuild time: RAID 10 copies one disk to one disk, while RAID 6 recalculates parity across every surviving drive for much longer, which is a bigger window for a second failure.

**Encryption off.** UniFi Drive cannot export an encrypted drive over NFS, and an encrypted pool needs unlocking after every power event, which would break unattended recovery.

### Shares

| Share | Used for | Access |
|---|---|---|
| backups | Proxmox backup target | NFS to the hypervisor only |
| media | Jellyfin libraries | NFS to the hypervisor only, bind-mounted read-only into Jellyfin |
| photos | Immich library and database dumps | NFS to the Docker VM only |
| personal | My own files and the test farm's folder | SMB only, never NFS |

Design notes:

- **Separate shares, not folders,** because NFS exports are per share. Anything granted media must not also get backups.
- **NFS access is per client address,** and that list is the access control. It is never widened to a whole subnet, and the public game VM is not on it at all.
- **sync, not async:** async acknowledges writes before they reach disk, so a backup could be recorded as finished while its tail was still in RAM.
- **Hard mounts, not soft:** soft mounts can silently corrupt writes. The hang risk is handled by unmounting before the NAS powers off in the UPS chain.
- **The personal drive is SMB only.** NFS authenticates nothing but the source address, so share permissions on an NFS-exported drive would be decoration. SMB uses File Service Credentials, separate from the console login.
- **The test farm mounts only its own folder,** to limit what a mistake can touch.

Example host mount (sanitized):

```text
NAS_ADDRESS:/var/nfs/shared/media  /mnt/media  nfs  defaults,_netdev,nofail  0  0
```

## Backups

### Nightly guest backups

| Setting | Value |
|---|---|
| Schedule | Daily at 21:00 |
| Mode and compression | Snapshot, zstd |
| Guests | Docker VM, game VM, AdGuard, Jellyfin (the rebuildable test farm is left out on purpose) |
| Target | NFS storage on the NAS |
| Retention | keep-daily 7, keep-weekly 4, keep-monthly 3 (set on the storage) |
| Alerting | Proxmox notice to Discord, plus a healthchecks.io dead-man's switch through a vzdump hook script |

The hook script pings healthchecks.io on job start, records any guest that failed, and sends success or failure at job end. Its ping URL lives in a root-only file and is never logged. It always exits 0, so a ping problem can never fail a backup.

Retention was never set until I checked the effective config ([troubleshooting entry 16](troubleshooting.md#16-backups-were-never-pruned)). One first-build backup is kept in an archive folder outside the pruned path on purpose.

### Immich database

- Immich's own `pg_dump` every night at 02:00, keeping 14, written to the photos share.
- A daily check on the Docker VM confirms the newest dump exists, is under 26 hours old, is not empty and passes `gzip -t`, then pings healthchecks.io.
- Why both this and the VM backup: a snapshot of a running Postgres directory is not a guaranteed-consistent dump, and the dump alone does not carry the OS and the other stacks.
- The dumps need the VectorChord-capable Postgres image to restore; the exact image and digest are recorded in my as-built notes.

### NAS snapshots

Daily at 03:00, after the Immich dump:

| Share | Kept | Why |
|---|---|---|
| photos | 30 | Covers deletions and bad jobs in the photo app |
| media | 14 | Covers rename scripts |
| backups | 7 | The hypervisor can delete over NFS but cannot touch NAS snapshots, so this survives a bad prune or a compromised host |
| personal | 30 | Undo for the test farm and my own files |

Snapshots live on the same pool, so they are not a backup of the NAS itself.

### Restore drill

A container backup was restored to a spare ID on the hypervisor: 757 MiB read back over NFS at 257 MiB/s, extracted, booted to a login prompt, stopped and destroyed. The procedure that avoids the main trap: restore to an unused ID, disconnect the network or tick "Unique" (otherwise the copy keeps the original's MAC address and, with a fixed-IP reservation, could take DNS down), confirm, then destroy it by the spare ID.

### What is not covered yet

- **There is no offsite or second copy of the NAS.** The photo library is the irreplaceable data, and RAID 10 is not a backup. An offsite copy is the top item in [What's next](../README.md#whats-next).
- The boot pool mirror (above).

## Housekeeping done

- Pruned 89 old backups (497 GiB) after a dry run matched the plan.
- Deleted 11 pre-NAS backups from the host's boot disk after archiving the first-build backup with a checksum check, and removed "backup" from the host disk's allowed content.
- Cleaned up a stale NFS mount the safe way (`findmnt` first, then `umount`, then `rmdir`; never `rm -rf` under `/mnt/pve/`).

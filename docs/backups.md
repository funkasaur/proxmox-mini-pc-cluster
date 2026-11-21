# Backups

Three layers: ZFS replication inside the cluster (fast failover, not a backup), nightly encrypted backups to a
Proxmox Backup Server, and an off-site copy of the PBS datastore in cloud storage.

## Proxmox Backup Server

- One HP Mini with a SATA SSD (ZFS `rpool`), running since 2026-10-26. On a fresh setup, run
  `zfs create rpool/datastore` before adding datastore `main` at `/rpool/datastore`. These are the toolkit defaults.
- **Client-side encryption** on the PVE storage entry. The key and its paper copy are kept off the cluster. Without the
  key, every backup (and the off-site copy) is unreadable.
- **The PVE API token can back up but not delete** (`DatastoreBackup` only, a deliberate choice). Retention runs on PBS,
  and the PVE backup jobs keep all. A compromised node can add backups but can't remove old ones.
- Nightly vzdump job for the cluster VMs, snapshot mode.
- **Standalone nodes back up to the same PBS** (since 2026-11-18): copy the PBS storage block from
  `/etc/pve/storage.cfg` and its `/etc/pve/priv/storage/<id>.pw` / `<id>.enc` files from a cluster node (same token, same
  key; compare hashes), then give each node its own backup job at its own time.

### Datastore maintenance

| Job | Schedule | Settings |
|---|---|---|
| Prune | Daily at 05:30 | Keep 14 daily, 8 weekly and 6 monthly snapshots |
| Garbage collection | Saturdays at 04:00 | Reclaim unused chunks |
| Verify | Sundays at 06:00 | Ignore already verified snapshots and re-verify after 30 days |

Prune removes snapshot metadata. Garbage collection frees the space occupied by unused chunks once they pass the
default 24 h 5 min access-time (`atime`) grace period. An active backup writer can require an older cutoff.
[PBS maintenance tasks](https://pbs.proxmox.com/docs/maintenance.html#garbage-collection)

### Moving PBS to a bigger disk

Same method as a node boot disk ([proxmox-cluster.md](proxmox-cluster.md#swapping-the-boot-disk-without-a-reinstall)):
the new SSD on a USB adapter, copy the GPT, full-size partition 3, `zpool attach`, `proxmox-boot-tool format/init`, then
swap it into the SATA bay (2026-11-11; 70 GB resilvered in 10 minutes, 0 errors). Check for running PBS tasks before
powering off.

## Off-site copy

Script: [toolkit `pbs/pbs_offsite.sh`](https://github.com/funkasaur/proxmox-cluster-toolkit#6-pbs-off-site-copy-pbspbs_offsitesh).
It runs daily on a machine outside the cluster: a ZFS snapshot of the datastore dataset, `zfs send -c` piped straight
into `rclone rcat` (a full stream weekly, incrementals between), the stream's md5 checked against the remote's, two
chains kept.

**Why ZFS streams and not the chunk files:** the first version mirrored the datastore's chunk files with rclone. A PBS
datastore is thousands of small chunk files (7.7k at the time), and the cloud storage API took them at about 10 files a minute. Don't
retry that design. As streams, the first full (17.8 GB) took 6 minutes and the first incremental was 52 KB (2026-10-27).

### Restore test

Done end to end on 2026-10-27, the day the off-site copy started: both streams downloaded and received with
`zfs receive` on the PBS host, mounted as a temporary datastore (`--reuse-datastore`), `verify` with 0 errors, then one
VM disk restored and decrypted on a cluster node with the cluster key. Its sha256 matched the same disk restored from
the live datastore. The temporary datastore, ACLs and dataset were then removed. Steps are in the toolkit README.

## Monitoring the backups

- `pbs-backup-age` (toolkit, on the PBS host, every 5 minutes) writes the newest completed snapshot per backup group
  as a textfile metric. The toolkit's `PbsBackupStale` rule alerts when the latest completed backup for a reported
  group is older than 26 hours. [Toolkit backup-age setup](https://github.com/funkasaur/proxmox-cluster-toolkit#pbs-backup-age)
- `pbs_offsite.sh` writes its own status metrics; PbsOffsiteStale fires on a failure or no success within its window.
- PBS datastore free space below 20 % and any ZFS pool not ONLINE alert too.

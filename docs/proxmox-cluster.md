# Proxmox VE cluster

## Install

- **Check the installer's `hdsize` field** before accepting the disk layout.
- Repositories: enterprise and Ceph enterprise off, `pve-no-subscription` on. Keep copies of the original `.sources`
  files.
- `openipmi.service` fails on every boot (these desktops have no BMC). It is harmless; it is masked on every box to keep
  the failed-units list clean (`systemctl unmask openipmi` undoes it).
- Root SSH by key only.
- Pin NIC names with `.link` files (see [hardware.md](hardware.md#network-cards)).

## The cluster

Three nodes, one cluster. The cluster was first formed with two nodes (by 2026-10-27); with two nodes there was only
storage replication and manual failover (steps in the
[toolkit README](https://github.com/funkasaur/proxmox-cluster-toolkit#5-node-maintenance-maintenancepve-maint)).
The third node joined on 2026-10-30 (`pvecm add <existing-node> --use_ssh`), and Proxmox HA was turned on.

Decisions (2026-11-06):

- **No QDevice.** The cluster deliberately runs with three expected votes and a quorum of two. Do planned node
  maintenance one node at a time. See [the QDevice decision](lessons-learned.md#what-id-do-differently).
- **No second cluster** for the standalone boxes. node4, node5 and standby are standalone Proxmox installs: they host
  things that don't need HA, are the first to get updates, or get powered off for hardware work.

### Corosync on two links

Corosync started on a single link on the LAN. Since 2026-11-15 it runs on two (Kronosnet, passive mode):

| Link | Network | Priority |
|---|---|---|
| link0 | the LAN, on the 5GbE cards | 10 |
| link1 | the management VLAN (example: VLAN 20, `198.51.100.0/24`), on each node's onboard 1G port, through its own small switch | **20 (preferred)** |

In passive mode knet sends over the highest-priority link that is up. link1 is a separate physical path with nothing
else busy on it, so corosync latency doesn't depend on backup or migration traffic on the LAN.

The router serves the VLAN untagged on one port to the small managed switch. Each node's onboard port plugs into that
switch and has a static address with no gateway. The original plan (2026-10-28) put the VLAN on the core switch.
Check that all nodes can reach each other on the new link before adding it to Corosync.

For cluster-wide network maintenance that may interrupt quorum, Proxmox VE 9.2 supports
`ha-manager crm-command disarm-ha freeze`. Wait for `ha-manager status` to show fencing as `disarmed` before
starting the work. Automatic fencing and recovery are suspended in this state. Once connectivity and quorum are
restored, run `ha-manager crm-command arm-ha` and verify the fencing status.
[Proxmox VE cluster maintenance](https://pve.proxmox.com/pve-docs/chapter-ha-manager.html#ha_manager_disarm)

Copy the cluster config to `/etc/pve/corosync.conf.new` and edit the copy.

```sh
cp /etc/pve/corosync.conf /etc/pve/corosync.conf.new
nano /etc/pve/corosync.conf.new
```

Merge the settings below into the existing `nodelist` and `totem` blocks. Keep the existing node names, IDs, link0
addresses and other settings. The addresses below are examples. Set `config_version` higher than its current value
(`2` assumes the current value is `1`).

```text
nodelist {
  node {
    name: node1
    nodeid: 1
    ring0_addr: 192.0.2.11
    ring1_addr: 198.51.100.11
  }
  node {
    name: node2
    nodeid: 2
    ring0_addr: 192.0.2.12
    ring1_addr: 198.51.100.12
  }
  node {
    name: node3
    nodeid: 3
    ring0_addr: 192.0.2.13
    ring1_addr: 198.51.100.13
  }
}

totem {
  config_version: 2
  token_coefficient: 125
  link_mode: passive
  interface {
    linknumber: 0
    knet_link_priority: 10
  }
  interface {
    linknumber: 1
    knet_link_priority: 20
  }
}
```

Back up the active config, then move the edited copy into place and check quorum and link status.
[Proxmox VE Corosync editing procedure](https://pve.proxmox.com/pve-docs/chapter-pvecm.html#pvecm_edit_corosync_conf)

```sh
cp /etc/pve/corosync.conf /etc/pve/corosync.conf.bak
mv /etc/pve/corosync.conf.new /etc/pve/corosync.conf
pvecm status
corosync-cfgtool -s
```

Test it: take link1 down on one node (`ip link set <onboard-nic> down`). The cluster should fall back to link0 and stay
quorate; bring it back and corosync returns to link1. This test passed on 2026-11-15.

Monitor it: with two links, a dead link is silent until the second one fails too. The toolkit's
`pve_corosync_link_connected{link,peer}` metric and PveCorosyncLinkDown alert cover that.

## HA

- Every service VM on the cluster is HA-managed, with fencing armed.
- HA uses the default software watchdog, `softdog`. A supported hardware watchdog is optional.
- HA uses the default `shutdown_policy=conditional`. It freezes HA services for a reboot so they can restart on
  the same node, and stops them locally for a poweroff. `migrate` requires another node to stay available and
  quorate during migration, which is a poor fit for [the shared UPS](power.md#setup).
- **Node-affinity rules** (Proxmox VE 9 "HA rules") give each VM a preferred node, with failback on, so the VMs spread
  over the three nodes and return home after maintenance. An alert fires if a VM sits away from its preferred node for
  30 minutes.
- Guests that are not HA-managed are left alone by node maintenance: shut them down or migrate them first.

### Node maintenance

```sh
ha-manager crm-command node-maintenance enable <node>    # HA live-migrates its services away
# reboot / work on the node
ha-manager crm-command node-maintenance disable <node>   # services fail back per their affinity rules
```

The toolkit's `pve-maint` wraps this: it waits until the node is empty and lists running guests HA won't move.

**This is a live migration, not a restart.** Measured downtime was 26-55 ms (2026-11). But afterwards Proxmox shows a
fresh VM uptime, a `qmstart` task and a "starting" state, which looks exactly like a restart. Check the guest's own
`/proc/uptime` before concluding it rebooted. (An earlier note in this build recorded HA VMs as "restarting, about a
minute down" during maintenance; that reading came from the Proxmox side and was later corrected.)

First live-migration test (2026-10-27, over 1G): 30 ms and 96 ms of downtime for two VMs; a once-a-second HTTP probe
saw one failed request at each switchover.

Leave nodes in maintenance as briefly as possible; an alert fires after 3 hours.

## Storage replication

ZFS storage replication every 15 minutes, **each HA VM to both other nodes**, so HA can restart it anywhere.

- The first full sync is large (19 GB in 4.5 min, 2026-11-01); incrementals take seconds.
- A rate limit of 60 MB/s was set while corosync and replication shared one 1G link.
- **A replicated VM can only live-migrate to a node that already holds its replica.** To move a VM to a new node: create
  the replication job, let the full sync finish, then migrate.
- After a migration, Proxmox flips the job direction; the old source stops reporting it. The toolkit's replication
  metrics are written by each source node for that reason.
- If HA restarts a VM after a node failure, its disk is as old as the last sync (up to 15 minutes).

## Updates

node5 is the staging node: Proxmox updates and new kernels go there first. On the cluster, one node at a time through
node maintenance. Before each: a tarball of `/etc`, the `dpkg -l` list and the upgrade log in `/root/backups/`. After
each reboot: quorate, all pools healthy, HA services back on their preferred nodes.

## Swapping the boot disk without a reinstall

How the nodes moved from SATA SSDs to NVMe (2026-11-10), with the node in maintenance:

1. Back up `/etc` (tarball), the old disk's GPT (`sgdisk --backup`) and `/etc/kernel/proxmox-boot-uuids`.
2. Copy the partition layout to the new disk (`sgdisk -R <new> <old>`, then `sgdisk -G <new>`). If the replacement
   is larger and partition 3 is last, grow only its end with `parted <new> resizepart 3 100%`. Replace `<new>` with
   the whole replacement disk path. Verify that partition 3's start sector and GPT type are unchanged before attaching it.
3. `proxmox-boot-tool format <new>-part2` and `proxmox-boot-tool init <new>-part2`.
4. `zpool attach rpool <old>-part3 <new>-part3`, wait for the resilver, check 0 errors.
5. Reboot and confirm the firmware booted from the new ESP (`LoaderDevicePartUUID` in efivars matches it).
6. `zpool detach rpool <old>-part3`, remove the old ESP from `/etc/kernel/proxmox-boot-uuids`.
7. If partition 3 grew: `zpool online -e rpool <new>-part3`.

Gotchas: `proxmox-boot-tool init` failed with "wrong filesystem" until `udevadm trigger` ran; `partprobe` isn't
installed, use `partx -u`. Wipe the old disk before it goes into any other node (duplicate `rpool` labels break boot).

## Moving a VM in from ESXi

The first VM came over from ESXi on 2026-10-26: shut it down cleanly, stream the flat `.vmdk` over SSH with `dd` into a
pre-created zvol of the same size, and compare sha256 on both ends. Recreate the VM as q35/OVMF, VirtIO SCSI, a virtio
NIC with the **same MAC**, guest agent on. The guest's NIC name changes (new PCI path), so its network config needs
updating. Downtime was 13 minutes for a 32 GiB disk. Keep the old VM powered off as the rollback until the new one has
a verified backup.

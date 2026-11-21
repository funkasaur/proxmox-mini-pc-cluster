# Lessons learned

## Timeline

| Date (2026) | What happened |
|---|---|
| 10-26 | Two-node cluster set up (live migration tested 10-27); PBS up; SMART monitoring on every box; first VM moved in from ESXi (13 min down, sha256-verified disk copy) |
| 10-26 | First node-down alert (ping-based) stayed silent through a 21-minute outage: AMT answers ping with the host off. Rebuilt on exporter scrapes |
| 10-26 | A CPU repaste wiped one node's CMOS: VT-x off, BIOS clock 196 days wrong and on local time |
| 10-27 | Nightly encrypted backups, 15-minute replication, off-site copy of the PBS datastore with a full restore test. The backup server's board lost its coin-cell holder and the disk moved to another board (NIC `.link` files had to follow the new MACs) |
| 10-27 | First chunk-file off-site mirror ran at about 10 files a minute; replaced by ZFS send streams |
| 10-30 | Third node joined; Proxmox HA on |
| 11-01 | PVE 9.2.24 rolled out node by node. UPS software powered a node off with the UPS on mains ("runtime insufficient" trigger). 2.5GbE M.2 card test passed |
| 11-02 | BIOS 02.50 + ME 11.8.95 on all six HPs; F10 network update found dead, USB method used |
| 11-06 | Decided: one NVMe per node, no SATA mirrors, no QDevice, no second cluster. Vulnerability scanner VM on the staging node |
| 11-07 | First RTL8126 5GbE cards fitted and soaked |
| 11-09 | PCIe ASPM causes correctable PCIe errors on the SN730: PCIe power management off in BIOS |
| 11-10 | Boot disks moved to NVMe on four nodes without a reinstall. Stock `hp-bioscfg` write bug found and patched |
| 11-11 | bios-guard live (daily BIOS + AMT drift check and fix) |
| 11-14 | IP-KVM keyboard works at HP POST only with a Dell keyboard identity and virtual media off |
| 11-15 | 5GbE cards + heatsinks + blower fans on the cluster nodes. Corosync link1 added on a dedicated management VLAN on the onboard ports. Patched BIOS driver found unloaded for 4 days. Standby monitor with automatic alerting failover |
| 11-18 | Standalone nodes back up to PBS too |

## Gotchas

Grouped by where they bite. Details are in the linked docs.

**Hardware**
- The 5GbE M.2 card reaches about 100 °C bare; a heatsink alone gains ~3 °C, a heatsink plus the HP drive-cage blower
  brings it to the low 60s idle ([hardware](hardware.md#cooling-the-5gbe-card)).
- Fragile coin-cell holder; after any CMOS loss, recheck VT-x, power-loss behavior, fan and clock.
- A reseated cable can negotiate 100 Mb/s: check link speed after hands-on work.
- Old disks with `rpool` labels break the boot import on another node: wipe them.

**BIOS / AMT**
- PCIe power management on: NVMe and NIC errors. Off on every node with an NVMe or 5GbE card.
- The stock `hp-bioscfg` driver can read but not write without a setup password (buffer-size bug).
- A DKMS module installed in the same apt run that brings in a new kernel may not get built for that kernel. Build for
  every installed kernel, and make the alert say what is broken, not what state it's in.
- i915 turns off unconnected outputs: AMT KVM is black unless an output is forced on the kernel command line, and
  `echo detect` on the forced port undoes it.
- AMT in Client Control Mode needs on-screen consent for KVM: use Admin Control Mode on headless boxes.
- AMT answers ping with the host off, and its address is not the OS address.

**Proxmox**
- Check the installer's `hdsize` field before accepting the disk layout.
- Node maintenance is a live migration even though the UI afterwards looks like a VM restart. Check the guest's uptime.
- A replicated VM live-migrates only to a node that already holds its replica.
- `proxmox-boot-tool init` may need `udevadm trigger` first on a new disk.

**Monitoring and alerting**
- Node-down must mean "exporters unreachable", not "no ping".
- With two corosync links, one dead link is invisible without a per-link metric.
- Grafana rules with no-data-as-alerting must always return a value.
- Grafana 13 reports a group's `lastEvaluation` as `0001-01-01` if it holds a paused rule.
- Stock smartd configs often mail root on a box that can't send mail: nothing visible happens.

**Backups**
- Cloud storage APIs are slow with many small files: ship ZFS streams, not PBS chunk files.
- An encrypted backup is only as recoverable as the copy of its key. Keep the key off the cluster.
- Restore-test the off-site copy on day one.

**Power**
- UPS software "runtime insufficient" triggers can shut nodes down on mains. Keep that one off.

## What I'd do differently

These are opinions from this build, not measured results.

1. **Pick one model of boot NVMe and one NIC before building**, instead of migrating SATA → NVMe and 1G → 2.5G → 5G
   on a live cluster. Every migration was done safely through node maintenance, but each cost an evening.
2. **Cool the M.2 NIC from day one.** The blower fan costs little and the difference is 30+ °C.
3. **Turn PCIe power management off in the BIOS before the first boot**, and put the BIOS targets under bios-guard
   right away. Drift happened more than once before the checker existed.
4. **Build the dedicated corosync network before turning on HA**, and firewall the management VLAN (AMT, switch
   management) from the rest of the LAN at the same time.
5. **Base the node-down alert on exporters from the start.** The ping-based version would have missed every real
   outage on AMT hardware.
6. **Match the QDevice decision to the node count.** Proxmox supports QDevices for even node counts, recommends them
   for two-node clusters that need higher availability, and discourages them for odd node counts. In an odd-sized
   cluster of N nodes, the QDevice contributes N−1 votes. If it becomes unreachable, all N nodes must stay online
   to keep quorum. This three-node cluster deliberately runs without a QDevice and can keep quorum with two nodes.
   [Proxmox QDevice guidance](https://pve.proxmox.com/pve-docs/chapter-pvecm.html#_corosync_external_vote_support)

## Not verified / open at the time of writing (2026-11-21)

- Pulling a node's 5GbE cable (losing the LAN path while link1 stays up) has not been tested.
- Long-run 5GbE card temperatures during the nightly backup window have not been reviewed.
- The IP-KVM "storage first, keyboard later" POST fix has not been tried on hardware.

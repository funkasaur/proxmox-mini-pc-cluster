# Proxmox Mini PC Cluster

A three-node Proxmox VE HA cluster built from used HP EliteDesk 800 G3 Mini business desktops, plus two standalone
Proxmox nodes, a Proxmox Backup Server and a standby monitoring box. This is the build guide and the record of
decisions: what the hardware is, how the BIOS, AMT, cluster, backups and monitoring are set up, and what went wrong
along the way.

Used business mini PCs are cheap, quiet and plentiful, but they were not designed to be headless hypervisors with
add-in 5GbE cards. Most of the interesting problems here were not in Proxmox: they were a BIOS that forgets its
settings, a kernel driver that can't write BIOS settings, a remote console that shows a black screen, PCIe power
management that throws errors on NVMe and NICs, and M.2 network cards that run at 100 °C in still air.
[`docs/lessons-learned.md`](docs/lessons-learned.md) collects those, with dates.

**Code is not in this repo.** The monitoring collectors, Grafana alert rules, maintenance helper, PBS off-site copy and
BIOS drift checker live in
[funkasaur/proxmox-cluster-toolkit](https://github.com/funkasaur/proxmox-cluster-toolkit). This repo explains why they
exist and how they fit together.

Built October-November 2026. Versions and measurements carry the date they were taken.

---

## Contents

| | |
|---|---|
| [`docs/hardware.md`](docs/hardware.md) | The boxes, disks, RAM, 5GbE cards and how they are cooled |
| [`docs/bios-and-remote-console.md`](docs/bios-and-remote-console.md) | BIOS settings and why, `hp-bioscfg` from Linux, bios-guard, Intel AMT, headless console, IP-KVM |
| [`docs/proxmox-cluster.md`](docs/proxmox-cluster.md) | Install, cluster, corosync on two links, HA rules, node maintenance, replication, boot-disk swaps |
| [`docs/backups.md`](docs/backups.md) | Proxmox Backup Server, encryption, retention, off-site copy, restore test |
| [`docs/monitoring.md`](docs/monitoring.md) | Prometheus + Grafana, textfile metrics, SMART, alerts, standby monitor, vulnerability scans |
| [`docs/power.md`](docs/power.md) | UPS and automatic shutdown, power-loss behavior |
| [`docs/lessons-learned.md`](docs/lessons-learned.md) | **Gotchas by date, and what I'd do differently** |

---

## Layout

| Name here | Hardware | Role |
|---|---|---|
| node1, node2, node3 | HP EliteDesk 800 G3 Mini, i7-7700T, 32 GB | Proxmox VE HA cluster; all service VMs run here |
| node4 | HP EliteDesk 800 G3 Mini, i7-7700T, 32 GB | Standalone Proxmox: one VM with no HA needs; also the NIC test bench |
| node5 | HP EliteDesk 800 G3 Mini, i7-7700T, 32 GB | Standalone Proxmox: staging (Proxmox updates and kernels go here first), vulnerability scanner VM, spare node |
| pbs | HP EliteDesk 800 G3 Mini, i5-7500T, 16 GB | Proxmox Backup Server |
| standby | Dell OptiPlex 7060 Micro, i7-8700T, 32 GB | Standalone Proxmox: standby Prometheus + Grafana VM that takes over alerting if the primary monitor fails |

The primary Prometheus + Grafana run on a separate, bigger server outside the cluster.

```
                       core switch (LAN, 10G)
                       |                    |
            multi-gig switch (5GbE)        router (routes the management VLAN)
            |   |   |   |   |   |                    |
   RTL8126 5GbE M.2 card in each HP: vmbr0     small managed switch, untagged access
   node1-3, node4, node5, pbs                  on the management VLAN (e.g. VLAN 20)
                                                     |   |   |   |   |   |   |
                                      onboard Intel I219 1G of every box:
                                      corosync link1 (node1-3) + Intel AMT (all)
```

- **Cluster LAN and VM traffic**: the 5GbE card on each node. Corosync **link0** also runs here.
- **Corosync link1** (preferred) and **Intel AMT**: the onboard 1G port, on a separate small switch in its own VLAN
  and subnet. See [`docs/proxmox-cluster.md`](docs/proxmox-cluster.md#corosync-on-two-links).
- **No QDevice and no second cluster** (decision 2026-11-06). The three-node cluster has three expected votes and
  needs two for quorum. [Proxmox discourages QDevices for odd node counts](https://pve.proxmox.com/pve-docs/chapter-pvecm.html#_corosync_external_vote_support).

## Software (2026-11-15)

| | |
|---|---|
| Proxmox VE | 9.2.24, kernel 7.0.14 (`-28` on the cluster) |
| Proxmox Backup Server | 4.2 |
| Boot disks | One NVMe per node, ZFS `rpool`, no mirror (see [hardware](docs/hardware.md#disks)) |
| HP BIOS | 02.50 on every G3 (latest available); ME firmware 11.8.95.4551 |
| Monitoring | Prometheus node-exporter (Debian package) + smartctl_exporter, Grafana-managed alerting |

## Related repos

| Repo | What |
|---|---|
| [proxmox-cluster-toolkit](https://github.com/funkasaur/proxmox-cluster-toolkit) | Corosync/HA/replication textfile metrics, SMART + cluster alert rules as code, dashboard builder, `pve-maint`, PBS off-site copy, bios-guard + patched `hp-bioscfg` driver |
| [grafana-standby](https://github.com/funkasaur/grafana-standby) | Watchdog that un-pauses a standby Grafana's copied alert rules when the primary monitor is unhealthy |
| [dual-3090-llm-server](https://github.com/funkasaur/dual-3090-llm-server) | The server that runs the primary Prometheus + Grafana (and the reason this repo's format looks familiar) |

## License

MIT (`LICENSE`).

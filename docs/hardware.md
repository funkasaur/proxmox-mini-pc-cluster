# Hardware

## The boxes

Six HP EliteDesk 800 G3 Mini (35 W "T" CPUs) and one Dell OptiPlex 7060 Micro. All were bought used.

| Name | CPU | RAM | Boot disk | LAN |
|---|---|---|---|---|
| node1-3 (cluster) | i7-7700T (4C/8T) | 32 GB DDR4 | WD SN730 512 GB NVMe | RTL8126 5GbE M.2 card |
| node4, node5 (standalone) | i7-7700T | 32 GB DDR4 | WD SN730 512 GB NVMe | RTL8126 5GbE M.2 card |
| pbs | i5-7500T | 16 GB DDR4 | WD Blue 500 GB SATA SSD | RTL8126 5GbE M.2 card |
| standby | i7-8700T | 32 GB DDR4 | Micron 3400 512 GB NVMe (with heatsink) | 2.5GbE RTL8125 |

Most nodes started with i5-6500T/7500T chips and 8-32 GB; the i7-7700T and 32 GB upgrades came in early November 2026.
RAM in the G3 runs at 2133 MT/s with 6th-gen CPUs and 2400 MT/s with 7th-gen.

**Burn-in for every CPU/RAM change** (2026-11-06 onward): PassMark MemTest86 (the Microsoft-UEFI-CA-signed free build)
installed as a one-shot boot entry on the node's ESP next to Proxmox, four passes, then `stress-ng` for 30 minutes
checking for machine-check errors. Proxmox stays the default boot entry.

## Network cards

The G3 Mini has one onboard Intel I219-LM 1G port. The only free PCIe is the **M.2 A+E (Wi-Fi) slot, which is PCIe
Gen3 x1 (8 GT/s)**: checked with `lspci -vv` on 2026-10-28. That is enough for a 5GbE card.

| Card | Result |
|---|---|
| RTL8125B 2.5GbE, M.2 A+E (2026-11-01 test) | PASS: 2.35 Gb/s each way, 30-min soak with 0 errors, 74 °C. Limits: one RX queue, 256-slot ring (rx_missed under small-packet load). Kept as the fallback |
| **RTL8126 5GbE, M.2 A+E** (2026-11-07 on) | **In use on every HP.** In-kernel `r8169` driver, nothing to install. 4.70 Gb/s each way in `iperf3 --bidir`, 0 link flaps over 20-minute soaks |

The onboard I219 stays in use: it carries corosync link1 and Intel AMT (AMT only works on the onboard port, and that
port must be link-up). See [proxmox-cluster.md](proxmox-cluster.md#corosync-on-two-links).

Interface names are pinned with systemd `.link` files (`/usr/local/lib/systemd/network/50-pmx-nic*.link`, matching on
MAC). **When you move a disk to another board, update those files** and run `update-initramfs -u -k all`, or the
bridge comes up on the wrong port (2026-10-27).

On Proxmox VE, run `pve-network-interface-pinning generate` to generate `.link` files for unpinned physical
interfaces. Review the generated `.link` files and staged `.new` configuration files before rebooting to apply
them. [Proxmox VE interface pinning](https://pve.proxmox.com/pve-docs/chapter-sysadmin.html#network_override_device_names)

Physical mounting details for the M.2 5GbE card, RJ45 jack and cable routing are not documented here.

### Cooling the 5GbE card

The RTL8126 runs hot. It sits under the 2.5" drive cage in still air.

| Setup | Idle | Under 4.7 Gb/s bidirectional |
|---|---|---|
| Bare card | 96-100 °C | 99-107 °C (reported by the `r8169` hwmon sensor) |
| Small copper heatsink only (2026-11-10) | | about 3 °C cooler than bare in a same-run comparison |
| **Copper heatsink + HP's 5 V "HDD" blower fan in the drive cage** (2026-11-15) | 62-65 °C | 72-76 °C, 0 errors |

The blower is the G3 Mini's optional second fan (HP part 914256-001 / L21471-001), which mounts in the 2.5" drive cage
over the M.2 slots. The heatsink alone barely helps; moving air does. Keep the drive cage on any node that gets the
fan. The heatsink was fitted with thermal paste on the chip and Kapton tape around it.

Compare temperatures only within one test run: earlier readings with two pairs of nodes under load at once were higher
than any single-pair run.

## Disks

- **One NVMe per node, no mirror** (decision 2026-11-06). SATA mirrors were dropped as slower. A dead NVMe gets a spare
  put in; meanwhile HA restarts the VMs on another node from their replicas (every HA VM replicates every 15 min to both
  other nodes). Same-model drives across nodes keep spares interchangeable; a larger spare can replace any boot disk
  with `zpool replace`.
- **PBS on a SATA SSD.** A QLC NVMe (Intel 660p) was ruled out for the backup target because of its write cliff
  (2026-11-11). The off-site copy covers the PBS disk.
- **Disable PCIe power management for NVMe nodes** (see [BIOS](bios-and-remote-console.md#bios-settings)): with ASPM L1
  on, an SN730 logged correctable PCIe receiver errors within minutes idle (2026-11-09).
- A thermal pad under a single-sided M.2 drive does little: the chips are on the label side.

New drives get a SMART check, a short self-test and a large sequential read before they go into a pool.

## Switches (generic)

- A 10G core switch (Brocade ICX 7250).
- A small multi-gig switch for the 5GbE cards, uplinked to the core over 10G.
- A small managed 8-port switch (TP-Link TL-SG108E) for the management VLAN: corosync link1 and AMT. It is a plain
  untagged access network from one router port.

A 10G copper switch for the nodes was not worth it (2026-10-28): revisit if storage moves to fast shared storage, VMs
grow to hundreds of GB, or transfers start making someone wait.

## Remote hands

- An 8-port HDMI/DP KVM switch with network control (TESmart HKS801), feeding a GL.iNet Comet PoE IP-KVM, gives
  BIOS-level access to every box in the rack.
- Intel AMT (vPro) on each HP gives remote power, serial-over-LAN, a KVM screen and IDE-R (boot from a remote ISO).

Details: [bios-and-remote-console.md](bios-and-remote-console.md).

## Physical gotchas

- **The G3 Mini's coin-cell holder is fragile.** It is a solder-only cradle; prying the cell upward can lift it off the
  board. One board lost the holder's ground pad (2026-10-27) and was retired as the backup server. Use a small flathead
  from the side facing the M.2 slot, with a light pry.
- **After any CMOS loss** check VT-x/VT-d (one node came back with VT-x off and no `/dev/kvm`), After Power Loss, fan
  speed, and the clock: the BIOS clock had been set to local time; fix with `hwclock --systohc --utc`.
- **A bad reseat can leave the NIC at 100 Mb/s.** It happened twice after battery work (2026-10-27): reseat the cable
  and check the negotiated speed before trusting the box.
- An old disk that still carries `rpool` ZFS labels breaks the boot-time pool import (two pools with the same name).
  Wipe drives before reusing them in another node.

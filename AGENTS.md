# Agent instructions for proxmox-mini-pc-cluster

## Scope

Documentation for a three-node Proxmox VE HA cluster, standalone nodes, Proxmox Backup
Server and standby monitoring.

## Repository layout

- `docs/hardware.md` records nodes, disks, memory, NICs and cooling.
- `docs/bios-and-remote-console.md` covers BIOS settings, Intel AMT and remote console
  access.
- `docs/proxmox-cluster.md` covers cluster setup, corosync, HA, replication and node
  maintenance.
- `docs/backups.md` and `docs/power.md` cover backups, restore tests and automatic shutdown.
- `docs/monitoring.md` and `docs/lessons-learned.md` contain the monitoring design and dated
  findings.

## Validation and use

- Use the README contents to find the relevant setup chapter.
- Check the dated software and hardware observations before changing the guide.
- This repository is a build guide with no build entry point or automated tests.
- The companion proxmox-cluster-toolkit repository contains the executable collectors,
  maintenance helpers and alert tooling.

## Documentation conventions

- Use the existing `node1` through `node5` labels and record when versions and measurements
  were observed.
- Preserve the distinction between the three HA members and standalone machines.
- Keep the two corosync paths, management network and standby monitoring roles clear.
- Preserve documented choices such as testing Proxmox updates and kernels on the staging
  node first.
- Keep network examples generic and keep private endpoint and account details out of this
  public guide.

## Reviewing changes

- Use the hardware chapter for component details and the power chapter for UPS behavior.
- Check the monitoring and backup chapters together when changing explanations of standby or
  recovery.
- Keep version observations dated and link supporting details where the README summarizes a
  decision.

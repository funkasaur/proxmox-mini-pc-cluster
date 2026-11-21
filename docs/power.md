# Power and UPS

## Setup

- The rack runs from one rack-mount UPS (CyberPower CP2000PFCRM2U) with a network management card (RMCARD205).
- **PowerPanel Business** (4.12.2) on the three cluster nodes and the backup server (since 2026-11-01), paired with the
  UPS's network card, shuts them down on a long outage:
  - Utility power failure: shut down after 20 minutes on battery.
  - Critical events: shut down at once.
  - **"Available runtime is insufficient": keep it Inactive.** Set to "Instant", it powered a cluster node off on
    2026-11-01 while the UPS was on mains, and that node's VM was down for 37 minutes.
- HA uses the default `shutdown_policy=conditional`. On poweroff, it stops managed guests locally. The `migrate`
  policy tries to move guests while the same UPS is shutting down their destinations. If quorum is lost while HA
  is still migrating, the watchdog can fence the node. See [cluster HA settings](proxmox-cluster.md#ha).
- **BIOS "After Power Loss" = Power On** on every node ([BIOS](bios-and-remote-console.md#bios-settings)), so the
  cluster comes back by itself when power returns. bios-guard checks it daily, because these BIOSes have reset it
  to Off.
- The UPS's network card is scraped by the monitoring server (SNMP), so on-battery state and load are in Prometheus
  alongside the nodes.

## Notes

- Mini PCs make the UPS math easy: 35 W TDP CPUs and one SSD each. Measured figures for the whole rack are not
  recorded in this repo.
- Put the internet edge (modem/ONT and router) on its own UPS. A failing small UPS feeding the modem took the internet
  path down on 2026-11-15 while the rack stayed up.
- Test a UPS under load before trusting it. A small consumer UPS elsewhere in the homelab went from 100 % to low-battery in about
  90 seconds at 100 W in a pull test (2026-11-16).

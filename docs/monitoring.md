# Monitoring

Prometheus scrapes every Proxmox box; **Grafana owns alerting** (Grafana-managed rules, no Alertmanager). The primary
Prometheus + Grafana run on a server outside the cluster, so the monitor doesn't go down with what it watches. A standby
monitor on another box takes over alerting if the primary fails.

## What runs on each node

| | |
|---|---|
| `prometheus-node-exporter` | Debian package, `:9100`, with the textfile collector directory |
| `smartctl_exporter` | Release binary from prometheus-community (not in the Debian/PVE repos), `:9633` |
| smartd | Self-tests only: short daily, long weekly, **each node at its own hour** so they never overlap |
| Toolkit textfile timers | corosync links, HA state, replication age (cluster nodes); backup age (PBS) |

Everything is in the [toolkit](https://github.com/funkasaur/proxmox-cluster-toolkit). Two Prometheus jobs
(`node-pve`, `smartctl-pve`), each target with a `host` label; the dashboard and node-down alerts select by job, so a
new node shows up by itself.

**The stock smartd config did nothing visible** (2026-10-26): it had no `-a` and mailed root on a box with no mail
relay. smartd now only schedules tests; alerts come from the exporter. A failed self-test shows up in the smartctl exit
status bits, since the exporter has no self-test metric.

## Alerts

22 rules in two groups, kept as code in the toolkit (`grafana/cluster_rules.py`). The ones that taught something:

- **PveNodeDown = both exporters down for 5 minutes, not ping.** Intel AMT answers ping with the host powered off. The
  first, ping-based version of this alert stayed silent through a 21-minute node outage (2026-10-26).
- **Standalone boxes get a warning-only down alert** after 10 minutes: they are powered off for hardware work on
  purpose.
- **PveCorosyncLinkDown**: one corosync link lost a peer. With two links nothing else would notice.
- **PveReplicationStale**: a job older than its limit, failing, a node's metrics timer stopped, or no data at all.
- **PveHaServiceProblem**, **PveHaOffPreferredNode**, **PveHaNodeMaintenance** (left on for 3 h), **PveQuorumLost**.
- **SMART**: health failed, error counters, attributes near threshold, temperature, NVMe media errors and wear, and
  drive writes against about 80 % of the drive's TBW rating.

SMART notes from the baselines (2026-10-26/30):

- Attribute 169 (`Total_Bad_Blocks` on some SSDs) ships non-zero from the factory: alert on growth only.
- Attribute 241 (writes) is in GiB on some drives and 512-byte LBAs on others; set limits per drive.
- Attribute 230 on one SATA SSD model reads near its threshold out of the box and means nothing there; use program/erase
  cycles or host writes versus TBW for wear instead.
- NVMe drives don't have SATA attributes at all; they need their own rules (critical warning, media errors, error-log
  growth over a baseline, percentage used, spare).

Rule-writing lessons:

- With "no data = alerting", make every expression return a value (0 when healthy). An empty result leaves the rule
  in NoData/pending instead of firing or resolving.
- Grafana's rule-group API replaces the whole group: the toolkit refuses to write a group that holds rules it doesn't
  know about.
- Never put `X or on() ...` after a branch that can return 0: the 0-valued series wins and hides the rest.

## Dashboard

One cluster dashboard (built by the toolkit's `cluster_dashboard.py`): uptime, CPU, RAM counting the ZFS ARC as used,
temperatures, HA services not OK, corosync links down, quorum, replication age, ARC, network, disk I/O, pool state.

## Standby monitor (since 2026-11-15)

A VM on the standby box runs its own Prometheus + Grafana, seeded from the primary (same rules, all paused).
[grafana-standby](https://github.com/funkasaur/grafana-standby) checks the primary every minute: after 3 failed checks
it un-pauses the copied rules and notifies; after 5 healthy checks it pauses them again. A daily sync copies rules,
dashboards and notification policies, but never contact points or data sources. The Prometheus config is edited in one
place and pushed to the standby. Failover and failback were tested live the day it was built: active about 3 minutes
after the primary went down, paused again 5 minutes after it came back.

A watchdog should be tested against the live primary before it goes in: a wrong health check means a false failover
three minutes later.

**Grafana 13 gotcha:** in the Prometheus-compatible rules API, a group's `lastEvaluation` reads `0001-01-01` whenever
the group holds a paused rule. Judge freshness per rule, not per group.

## Vulnerability scanning

node5 runs Greenbone Community Edition (the official compose file, in a VM) with a weekly "Full and fast" scan at low
concurrency (one host at a time, four checks at once) so it doesn't load the nodes.

## BIOS and AMT drift

bios-guard reports daily; see [bios-and-remote-console.md](bios-and-remote-console.md#bios-guard).

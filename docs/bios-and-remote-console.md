# BIOS, Intel AMT and the remote console

## BIOS version

All G3 Minis run BIOS **02.50** (HP SoftPaq sp153868, August 2024, the newest for this board family), flashed on
2026-11-02. **The F10 "network update" no longer works**: it fails with "protocol defined in the url is not
supported" because the HP FTP source is gone. What worked: build the USB stick with the SoftPaq's tool on Windows,
then copy the `.bin` into `EFI\HP\BIOS\New\` on the stick by hand, and update from F10 → from USB.

After flashing, check from Linux: `dmidecode -s bios-version`, `grep -c vmx /proc/cpuinfo` (VT-x), DMAR units in
`dmesg` (VT-d), and that the RTC is on UTC.

ME firmware is 11.8.95.4551 on all six (2026-11-02).

## BIOS settings

| Setting | Value | Why |
|---|---|---|
| After Power Loss | **Power On** | Nodes come back by themselves after an outage. The BIOS has been seen to reset this to Off |
| PCI Express Power Management | **Disable** on NVMe nodes | With ASPM L1 on, an SN730 logged correctable PCIe RxErr (physical layer) in about 2 minutes idle; with ASPM off, 0 errors through a 100 GB read (2026-11-09). On the backup server, PM on put the 5GbE card's link into L1 and it logged 3,480 rx_errors under iperf, versus 44 in 1.45 billion packets on a node with PM off (2026-11-10) |
| SATA Power Management | Disable | Set together with the above |
| Increase Idle Fan Speed (%) | 30 (40 on one node on purpose) | More airflow over the M.2 slots |
| VT-x, VT-d | Enable | Lost after a CMOS reset more than once |

The kernel could not override ASPM on the backup server (no `l1_aspm` link control in sysfs), so the BIOS is the place
to fix it. After a reboot, `lspci -vv` on both ends of the link should say `ASPM Disabled`, and the device's
`aer_dev_correctable` counters should stay at 0.

## Reading and writing BIOS settings from Linux

The G3s load the `hp-bioscfg` kernel driver, which exposes every BIOS setting under
`/sys/class/firmware-attributes/hp-bioscfg/attributes/<Name>/current_value`. Names contain spaces: quote them.
Reading works with the stock driver and is a good way to check BIOS state without a monitor.

**Writing did not work** (found and fixed 2026-11-10). Every write failed, and the driver labeled the BIOS error
"Invalid command type". The cause is a driver bug: when no BIOS setup password is configured, the driver reserves 4 bytes for the
security buffer but writes a 20-byte payload into it. The firmware gets a truncated buffer and returns code 4
("failed"), and the driver overruns its heap allocation by 16 bytes. The fix is one function. The patched driver,
packaged as a DKMS module, and the upstream patch are in
[proxmox-cluster-toolkit/bios-guard/dkms](https://github.com/funkasaur/proxmox-cluster-toolkit/tree/main/bios-guard/dkms/hp-bioscfg-fix-1.0).
It was verified by changing a setting from Linux, rebooting, and reading it back with the stock driver.

With Secure Boot enabled, the module must be signed with a trusted key. If DKMS uses its own Machine Owner Key
(MOK), request enrollment with `mokutil --import /var/lib/dkms/mok.pub` as root (use the configured public
certificate path if it differs), then confirm enrollment in MokManager on the next boot.
[DKMS Secure Boot documentation](https://github.com/dkms-project/dkms#secure-boot)

## bios-guard

These BIOSes lose settings (power loss, CMOS battery work). bios-guard
([toolkit](https://github.com/funkasaur/proxmox-cluster-toolkit#7-bios-guard-bios-guard)) runs daily from the
monitoring host: it reads every attribute over SSH, compares with a targets file, writes drifted settings back (`--fix`)
and reports "reboot pending" without ever rebooting. It also checks each node's AMT configuration over WS-Man with a
read-only AMT account. It went live on 2026-11-11.

**Gotcha (2026-11-15): the patched module was not loaded for four days.** The installer's `apt` step pulled in a new
kernel before the DKMS source was registered, so the kernel's DKMS hook skipped it, and the next boot loaded the stock
read-only module. The daily report did say "stock module (read-only)", but in a line that was easy to skip. Two fixes:
the installer now builds for every installed kernel, and the report line now says **PATCHED MODULE NOT LOADED, BIOS
fixes are off**. A warning that names the consequence gets acted on; one that names a state doesn't.

## Intel AMT

Every box is set to:

- **Admin Control Mode** (ACM), so **no user consent** is needed for KVM. Client Control Mode activation from the OS
  needs someone at the screen to read a consent code, which is useless on a headless box. If AMT drops out of ACM,
  fix it in MEBx (Ctrl+P at boot) or with an AMT admin tool.
- Serial-over-LAN and IDE-R on, KVM on.
- On the management VLAN (DHCP with fixed leases), on the onboard I219 port, which must be link-up: each node has
  `auto <onboard-nic>` in `/etc/network/interfaces` even where Proxmox doesn't use the port for anything else.

Use a separate read-only AMT account for monitoring.

**Pinging an AMT box tells you nothing about the OS.** AMT answers ping (and its web ports) with the host powered off.
Its address is also different from the OS address: probing the AMT address for SSH and concluding the server is down
led to needless power cycles once.

### Making the AMT screen work headless

The `i915` driver turns off every output that has no monitor, so the AMT KVM shows a black screen on a headless node.
Force an output on from the kernel command line. Current command line on every node (`/etc/kernel/cmdline`, then
`proxmox-boot-tool refresh`):

```
console=ttyS4,115200 console=tty0 video=DP-1:1920x1080@60 video=DP-2:1920x1080@60e
```

- `DP-2` with the trailing `e` is **forced on**; AMT KVM shows that output.
- `DP-1` is where the rack KVM cable goes, because the HP BIOS, POST screen and MEBx draw **only** on DP-1.
- `ttyS4` is the AMT serial port (the "KT" PCI device) on the G3. Run
  `systemctl enable --now serial-getty@ttyS4.service` for a login over serial-over-LAN. Press Enter in the SOL
  window to get the prompt.
- The forced output only exists once Linux runs: AMT KVM is blank in BIOS setup. BIOS changes need the rack KVM.

Two gotchas:

1. **Never write `detect` to the forced port** (`/sys/class/drm/cardN-DP-2/status`): it cancels the force and AMT goes
   blank. `echo on` to the same file restores it (at 640x480 until reboot). Read `status`/`enabled` only.
2. **A blank rack-KVM picture after switching ports:** re-probe the KVM port (DP-1, not the forced one):
   `echo off > /sys/class/drm/cardN-DP-1/status; sleep 2; echo detect > /sys/class/drm/cardN-DP-1/status`.
   Disabling i915's HPD storm detection did not help. (`cardN` was `card1` on the G3s.)

A remote-management server running in a VM can't reach the AMT of the node it runs on (shared NIC); use a client on
another machine for that node. AMT KVM can take about a minute after boot to come up.

## Rack KVM and IP-KVM

The rack KVM switch takes serial-style hex commands over TCP to switch ports, so the IP-KVM user can pick a box
remotely. Behind it, a GL.iNet Comet PoE IP-KVM gives the screen and keyboard from POST onward.

**HP POST ignores the IP-KVM keyboard by default** (2026-11-14). With the IP-KVM's default USB identity, keys only
worked once the OS had booted. What works: set the IP-KVM's device identity to a Dell keyboard and **turn its virtual
media off**. With virtual media on (USB disk or CD-ROM), the HP firmware rejects the composite keyboard + storage
device at POST and the keyboard stops working again. A community-reported cause is that the firmware rejects HID and
mass storage in one device; that fix (storage only at POST, then add the keyboard) has **not been tested here**. For
OS installs on the HPs, use AMT IDE-R instead of the IP-KVM's virtual media.

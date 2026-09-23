# fujitsu - Proxmox VE host

The virtualisation host of the lab, and the platform my bachelor thesis runs on. Built 15.08.2026, headless, administered over SSH.

| | |
|---|---|
| Hardware | Fujitsu Esprimo D757, i5-6500 (4c/4t), 16 GB RAM |
| Boot disk | WD Black SN770 250 GB NVMe (M.2) - the only internal drive; the 2.5" SATA bay is empty |
| External | WD Elements 2 TB USB - data tier, mounted on the host, not yet passed to a guest |
| Hypervisor | Proxmox VE 9 on Debian 13 "trixie" |
| Root filesystem | ext4 + LVM-thin |
| Address | static IPv4, assigned outside the router's DHCP pool |

## Install decisions

Three choices were made before installing anything, each written down with the option rejected. On a 250 GB disk and 16 GB of RAM, none of them were free.

**Root filesystem: ext4 + LVM-thin, not ZFS.** The workload this host exists for is a self-hosted photo library, which runs PostgreSQL, Redis and a machine-learning container for face and object recognition; the ML container is the hungry part, and 16 GB is not much room. ZFS would put its ARC cache in competition with that workload for RAM. My first reasoning was that ARC takes half of system RAM, which is the plain ZFS-on-Linux default. That is not what this installer does: since Proxmox VE 8.1 a new install caps ARC at 10% of RAM (about 1.6 GB here) in `/etc/modprobe.d/zfs.conf`, and the cap is tunable either way. So the RAM argument is weaker than I first wrote it. The choice still stands on a single disk: ZFS would have brought checksums but no redundancy (RAID0 on one disk heals nothing), plus ARC tuning to watch under a memory-hungry guest, while ext4 + LVM-thin keeps snapshots and needs no tuning. What that costs me, stated honestly: ZFS checksumming, so silent bit-rot detection has to come from the backups, once they exist.

**Guest disks live on the same pool as the host,** because there is one internal drive. The obvious risk is a shared blast radius. The three that bite in practice are less obvious: I/O contention, since guests and hypervisor share one device queue and a busy guest makes the host itself sluggish; capacity coupling, since an unpruned snapshot or a runaway guest disk fills root, and Proxmox with a full root does not fail gracefully; and write amplification on consumer NVMe, because the boot device is now also the busiest device. The external USB disk is the data tier (and later the backup target), not a guest tier - spinning-disk latency under random I/O makes VMs crawl, and USB storage can drop off the bus, which to a running VM is indistinguishable from its disk vanishing. With budget, the fix is a second internal drive in the empty SATA bay.

**Static IP on the host, not a DHCP reservation.** Both give a stable address; they differ in where the truth lives. Static means the host knows its own identity and boots correctly even if the router is rebooted or replaced. A reservation centralises addressing, which is nicer at scale, but makes the hypervisor depend on the router being healthy at boot. For a single host I SSH into, static wins.

The address was derived rather than invented: keep the netmask and gateway the installer already filled in from DHCP, change only the host part to an address outside the router's DHCP pool, then prove it free by pinging it from another machine before committing. DHCP had offered an address from inside the pool; a low one outside it was taken instead, and confirmed silent before being committed. Thirty seconds that prevents a duplicate-address bug later.

Interface names were pinned at install time rather than left to predictable-naming drift.

## Problem - `apt-get update` fails on a brand-new install

**Symptom.** The first update task failed in the web UI with:

```
TASK ERROR: command 'apt-get update' failed: exit code 100
```

That message is useless on its own. 100 is apt's generic failure code, and the GUI swallowed the actual output.

**Diagnosis.** Re-ran `apt-get update` in a shell and read what it printed. The `pve-enterprise` and Ceph enterprise repositories are enabled by default on a fresh install and require a paid subscription. Without a key they return `401 Unauthorized`, and one failing repository fails the whole update.

**Fix.** Disabled both enterprise repositories and enabled `pve-no-subscription`. The next run returned `Hit` on the three Debian repositories, `Get` on the Proxmox one, and `TASK OK`. Then `apt full-upgrade` and a reboot.

**The transferable part.** The GUI gave me an exit code; the shell gave me the cause. First move on any Proxmox task failure is to re-run the underlying command in a terminal, because the web UI reports that something failed and the CLI reports why. This is the same shape as the GRUB failure on [textboxer](textboxer.md) - the error text contained the answer, once I actually read it.

One habit kept from this: if a kernel was installed, check `uname -r` before and after the reboot. Installing a kernel and continuing to run the old one is a quiet and common mistake.

## Storage

The 2 TB USB disk that serves as the data tier is documented separately: [labs/usb-data-tier.md](../labs/usb-data-tier.md).

## Verification

Headless operation was tested rather than assumed. Monitor and keyboard unplugged, cold boot, then SSH straight in from the laptop - no console login needed first. That confirms the thing worth understanding: systemd starts enabled services at boot regardless of whether anyone logs in, and the console login prompt is just another service, not a precondition for the machine being up.

## Not done yet

- Snapshot rollback drill, and a `vzdump` backup restored *and booted* - a snapshot is not a backup
- The 2 TB USB disk passed into a container as a bind mount - the host-side mount is done and reboot-proven, the container half is not
- Confirm the USB 3.0 connector stays fully seated; it silently fell back to 480M once
- Both hosts monitored from textboxer

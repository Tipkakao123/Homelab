# The 2 TB USB data tier on fujitsu

Making an external USB disk behave like an internal one: partitioned, formatted, addressed by UUID, mounted at boot, and proven by an unattended reboot. Host is the Proxmox VE box described in [docs/fujitsu.md](../docs/fujitsu.md).

## What it is

This disk acts as my main storage for photos and videos. This is its only purpose - act as my "cloud" storage, a Google Photos alternative. The reason it is USB and not internal has only to do with pricing: recent price hikes have finally reached hard drives as well, and it was cheaper to get an external drive than an internal one.

The numbers I bought against, in Estonia in July 2026, when storage prices had risen sharply on AI datacentre demand and the rise had spread to mechanical drives too:

| Option | Price | Availability |
|---|---|---|
| Toshiba L200 2.5" 1 TB internal | €93.76 | one seller, ~3 week lead |
| Seagate Barracuda 2.5" 1 TB internal | €175.90 | special order |
| **WD Elements Portable 2 TB external USB** | **€110.80** | in stock, 1-5 days |

That is **€55.40 per TB** external against €93.76-175.90 per TB internal. Twice the capacity, available in days instead of weeks, and cheaper than one of the two internal drives I could actually have bought. The boot drive next to it is a WD Black SN770 250 GB NVMe at €62.90, so the whole storage tier came to €173.70.

The drive also has a second job already planned: once internal drives are affordable again it becomes the backup target instead of the primary pool. So this is not a stopgap I have to undo later.

## Decisions

**Reserved blocks - `-m 1`.** By default `mkfs.ext4` reserves 5%, nearly 100 GB on this drive. As this is a pure photo pool and nothing system-critical (OS or similar) lives on it, I reserved 1% instead - roughly 18 GB. The reserve has two jobs: a full disk still lets root log in and daemons write their logs, because a filesystem at literal 100% is how systems become unrecoverable; and the allocator keeps free space to find contiguous runs and avoid fragmentation. On a filesystem at 99.9% full there is nothing but scattered gaps, so every new write fragments and the fragments make future allocation worse. Although not critical for a dedicated photo pool, I would rather leave some room that is always reclaimable than write the disk full and risk fragmentation.

**Inode tables.** A separate, unchosen cost. The default ratio assumes one file per 16 KB, giving 122 million inodes at ~31 GB of tables, against maybe 600,000 photos. Fixed at `mkfs`; only a reformat changes it.

**Boot behaviour - `nofail`.** `nofail` ensures that when systemd cannot find the storage device, the failed mount does not take `local-fs.target` down with it and drop the host into `emergency.target` - which would need a monitor and a direct hardware connection to recover from. On a headless box that is the difference between a missing disk and a trip to the machine.

**Device wait - `x-systemd.device-timeout=10`.** This exists because USB is slow to enumerate. The default wait for a device to appear is 90 seconds, so an unplugged USB disk would stall boot for the full time; this caps it at 10. It does not make good boots faster - a device that appears in 4 seconds costs 4 seconds either way. It caps the failure case.

## The fstab line

```
UUID=8d054ba4-812c-4815-80e0-aa76f80e1670  /mnt/photopool  ext4  defaults,nofail,x-systemd.device-timeout=10  0  2
```

`/dev/sdX` letters are assigned in enumeration order, not bound to a device. If I plugged in a second USB disk, or had the existing one enumerate slower, `sda` would become `sdb` - and fstab would mount the wrong disk or fail. The filesystem UUID is a property of the filesystem itself and does not move.

## Problem - the disk vanished after the first reboot

After the first reboot the mount was gone - and not just unmounted, the disk itself was not there. There was no light on the enclosure either, which pointed at hardware before anything else. So I checked it in layers: is the device on the USB bus at all, and if it is not, what did the kernel say while trying to bring it up. That order matters, because if the device never appears then nothing in fstab is relevant, and editing fstab would have been fixing the wrong thing.

The host booted normally and `lsblk` showed no `sda` at all. `lsusb` listed only the two root
hubs - no device on the bus:

```
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
```

The kernel log showed what had been tried:

```
usb 1-3: new full-speed USB device number 2 using xhci_hcd
usb 1-3: device descriptor read/64, error -71
usb 1-3: device descriptor read/64, error -71
usb usb1-port3: attempt power cycle
usb 1-3: Device not responding to setup address.
usb 1-3: device not accepting address 4, error -71
usb usb1-port3: unable to enumerate USB device
```

`full-speed` is the clue that cracked it. Full-speed is USB 1.1, 12 Mbit/s. This is a USB 3.0 drive, so it should come up as SuperSpeed, or high-speed at worst. Coming up at full-speed means the kernel could not establish the faster signalling at all, and that is a physical layer problem rather than a configuration one. `-71` is `-EPROTO`, a protocol error: the kernel asked for the device descriptor and did not get a valid answer back. It retried, power-cycled the port, and gave up.

After reseating the cable the device enumerated cleanly and spun up:

```
usb 1-2: Product: Elements 2621
usb 1-2: Manufacturer: Western Digital
usb-storage 1-2:1.0: USB Mass Storage device detected
sd 6:0:0:0: [sda] Spinning up disk...
...ready
sd 6:0:0:0: [sda] 3906963456 512-byte logical blocks: (2.00 TB/1.82 TiB)
 sda: sda1
sd 6:0:0:0: [sda] Attached SCSI disk
```

It came back at USB 2.0 speed, on the 2.0 root hub:

```
/:  Bus 001.Port 001: Dev 001, Class=root_hub, Driver=xhci_hcd/16p, 480M
    |__ Port 002: Dev 006, If 0, Class=Mass Storage, Driver=usb-storage, 480M
/:  Bus 002.Port 001: Dev 001, Class=root_hub, Driver=xhci_hcd/8p, 5000M
```

Reseating it fully moved it to the SuperSpeed hub, same physical port:

```
/:  Bus 001.Port 001: Dev 001, Class=root_hub, Driver=xhci_hcd/16p, 480M
/:  Bus 002.Port 001: Dev 001, Class=root_hub, Driver=xhci_hcd/8p, 5000M
    |__ Port 002: Dev 003, If 0, Class=Mass Storage, Driver=usb-storage, 5000M
```

One partially seated USB 3.0 Micro-B connector caused all of it. That connector is really two connectors side by side - the normal USB 2.0 part, plus an extra section carrying the SuperSpeed pins - and when the second part is not making contact the drive falls back to USB 2.0 silently instead of failing outright. That single fault explains all four symptoms: the `-71` errors, the full-speed fallback, the dead light, and the 480M. My first guess was that the reboot had not fully de-powered the port, which fitted the timing but did not survive the speed data. If a diagnosis needs a different story for every symptom, it is usually not the diagnosis.

## Proof

Rebooted, then straight back in over SSH. Nothing typed to mount anything:

```
root@pve:~# uptime
 20:25:21 up 0 min,  1 user,  load average: 0.55, 0.15, 0.05
root@pve:~# lsblk
NAME               MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda                  8:0    0   1.8T  0 disk
└─sda1               8:1    0   1.8T  0 part /mnt/photopool
sr0                 11:0    1  1024M  0 rom
nvme0n1            259:0    0 232.9G  0 disk
├─nvme0n1p1        259:1    0  1007K  0 part
├─nvme0n1p2        259:2    0     1G  0 part /boot/efi
└─nvme0n1p3        259:3    0   231G  0 part
  ├─pve-swap       252:0    0     8G  0 lvm  [SWAP]
  ├─pve-root       252:1    0  67.7G  0 lvm  /
  ├─pve-data_tmeta 252:2    0   1.4G  0 lvm
  │ └─pve-data     252:4    0 136.5G  0 lvm
  └─pve-data_tdata 252:3    0 136.5G  0 lvm
    └─pve-data     252:4    0 136.5G  0 lvm
root@pve:~# lsusb -t
/:  Bus 001.Port 001: Dev 001, Class=root_hub, Driver=xhci_hcd/16p, 480M
/:  Bus 002.Port 001: Dev 001, Class=root_hub, Driver=xhci_hcd/8p, 5000M
    |__ Port 002: Dev 003, If 0, Class=Mass Storage, Driver=usb-storage, 5000M
```

`up 0 min` with `sda1` already at `/mnt/photopool` is the whole claim. Still holding 14 hours later:

```
root@pve:~# uptime && lsblk
 10:54:48 up 14:14,  1 user,  load average: 0.03, 0.05, 0.00
sda                  8:0    0   1.8T  0 disk
└─sda1               8:1    0   1.8T  0 part /mnt/photopool
```

## What I would do differently

`mkfs.ext4 -T largefile` would have allocated ~1.8 million inodes instead of 122 million, giving most of that 31 GB back. The defaults assume small files; `-i` / `-T largefile` is the lever for a media pool. As this is prep work for my bachelor thesis it stays as is for now and gets revisited later.

Always check that cables and devices are properly seated in their USB or PCI slots. Errors indicating a software or protocol issue - especially different errors under the same conditions - might just be a loosely connected hard drive.

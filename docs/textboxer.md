# textboxer — manual Arch Linux build

A spare ThinkPad that was lying around doing nothing. After experimenting with different Linux flavours I landed on what I actually needed: a CLI-only host for practicing Linux on the command line, working toward the LFCS certification. The box is deliberately disposable — if something breaks, I fix it or reinstall without any risk to the future Proxmox host. Built by hand, June 2026, while sick.

## The build

Booted the install ISO and connected to wifi with `iwctl`, verified with a ping. The Arch wiki happened to be returning 403 errors that night, so parts of the install went ahead without the official documentation.

Partitioning: started with `fdisk`, found the single-letter interface unfriendly, and switched to `cfdisk` — same job, arrow-key menus, much easier mid-install. Wiped the disk's previous partition layout and wrote a fresh GPT: a 1 GB EFI System Partition (FAT32) mounted at `/boot`, plus an ext4 root. No swap — a swapfile can be added later if ever needed.

Installed the base system with `pacstrap` (`base`, `linux`, `linux-firmware`), then after `arch-chroot` added the toolkit with plain `pacman`: bootloader (`grub`, `efibootmgr`, `intel-ucode`), networking (`networkmanager`, `openssh`, diagnostics like `nmap` and `traceroute`), and admin quality-of-life (`sudo`, `vim`, `tmux`, `htop`, man pages).

Generated the fstab, which confused me at first — until I understood it's the table the system reads at every boot to know which partition mounts where. Without it, no working system. Set locale to English system messages (`LANG=en_US.UTF-8`) on an Estonian keyboard (`KEYMAP=et`), set the root password, created my user, hostname `textboxer`.

## Snag 1 — GRUB refused to install

**Symptom:** `grub-install` failed, complaining it couldn't find the EFI directory.

**Diagnosis:** `ls` showed the files were there; `findmnt` confirmed the ESP was mounted at `/boot` as vfat. So the partition was fine — GRUB was looking in the wrong place.

**Cause:** `grub-install` assumes the ESP lives at `/boot/efi` by default. Mine was mounted at `/boot`.

**Fix:** point it at the actual location:

```
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
```

then generated the config with `grub-mkconfig`, successfully.

**Lesson:** 
Tools ship with default assumptions — verify them against your actual layout instead of assuming they fit.

## Snag 2 — first boot, network dead

**Symptom:** after rebooting into the installed system, no connectivity. All interfaces down, and `ping` returned `temporary failure in name resolution`.

**Diagnosis / cause:** honestly, a missed checklist step — NetworkManager was installed but never *enabled* inside the chroot. The live ISO's networking (`iwctl`) doesn't carry over to the installed system, and a fresh install brings nothing up on its own: raising interfaces is a service's job, and no service had been given that job. The error chain made sense once seen that way: no link → no route → no DNS.

**Fix:**

```
systemctl enable --now NetworkManager
```

then connected to wifi with `nmcli`. NetworkManager saved the connection profile, so it reconnects automatically on every boot — verified.

**Lesson:** 
Stuff that works before might not work later, services that were enabled might not be live when changing environments.

## Going headless

Enabled `sshd`, and now administer the box over SSH from my main laptop — lid closed, no monitor or keyboard attached.

I'll admit the honest confusion here: my first reaction was *"what's the point of SSHing into my own box?"*, why not just use the host laptop. The answer reframed the whole machine for me — you SSH *from* your daily computer *into* the server, which is exactly how every real server is administered. Remote administration over SSH isn't a gimmick; it's the normal daily motion of the job this lab is preparing me for. textboxer isn't a laptop I log into anymore — it's a small server I operate remotely.

## What I'd do differently

- **Follow the checklist to the end.** The one skipped step (enabling NetworkManager in the chroot) was exactly the one that bit at first boot.
- **Verify, don't assume.** The first SSH login from my main laptop went unconfirmed for three weeks before I actually tested it.
- **Document while doing, not after.** Reconstructing this write-up weeks later took logs (`/var/log/pacman.log`), old chat records, and effort that five minutes of notes during the install would have saved. The system keeps receipts — but notes are cheaper.


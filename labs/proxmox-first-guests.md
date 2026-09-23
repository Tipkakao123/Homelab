# First guests on Proxmox - one LXC from the CLI, one VM

Two guests on the Fujitsu Proxmox host: a Debian 13 LXC container built entirely from the command line, and a Debian 13 VM installed through the web UI. First time doing either.

I built the container from the CLI on purpose. The GUI would have been faster, but a command is something I can version, paste into a runbook and hand to someone else - a click is not. I will not pretend I understood every flag as I typed it. The end result worked and I understand it now, which is the honest version.

## The command

```
pct create 100 local:vztmpl/debian-13-standard_13.6-1_amd64.tar.zst \
  --hostname lab05-ct --memory 512 --cores 1 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --storage local-lvm --unprivileged 1
```

The part that confused me first time: **there are two different storages in one command.** The template is read from `local`, but the container's root disk is created on `local-lvm`. That is not a preference, it is forced by what each storage is:

```
dir:     local      content vztmpl,backup,import,iso     <- no rootdir
lvmthin: local-lvm  content rootdir,images
```

`local` is a directory, so it holds files - ISOs, templates, backups. `local-lvm` is LVM-thin, so it hands out block devices, which is what a root filesystem needs. `local` cannot hold a container rootfs because `rootdir` is not in its content list.

Two other things I got wrong before I got them right: I retyped the template filename from memory instead of listing it with `pveam list local`, and I put a space inside the `--net0` value. That value is a single argument - a comma-separated `key=value` list with no whitespace anywhere in it. It looks like prose and it is not.

## Problem - systemd degraded in an unprivileged container

**Symptom.** `pct create` finished with a warning:

```
WARN: Systemd 257 detected. You may need to enable nesting.
```

The container started anyway, but inside it:

```
# systemctl is-system-running
degraded
```

**Diagnosis.**

```
# systemctl --failed
● dev-mqueue.mount   failed  POSIX Message Queue File System
● run-lock.mount     failed  Legacy Locks Directory /run/lock
● tmp.mount          failed  Temporary Directory /tmp
```

Three failures, and all three are `.mount` units. Not a service, not networking - every one of them is systemd calling `mount()` and being refused.

**Cause.** I chose `--unprivileged 1` deliberately. In a privileged container, root inside is effectively root on the host as far as the kernel is concerned. An unprivileged container maps container-root to an unprivileged UID on the host instead, so a breakout lands as an unprivileged host user. The cost of that choice is exactly what I was looking at: that UID has no business mounting kernel filesystems, so systemd 257 could not set up the mounts it wanted.

**Fix.**

```
pct set 100 --features nesting=1
pct stop 100 && pct start 100
```

**Verification.**

```
# systemctl --failed
0 loaded units listed.
# systemctl is-system-running
running
```

**What nesting actually does, and what it does not.** It permits the container to create its own nested namespaces and mount filesystems inside them. It does **not** change the UID mapping - container-root is still an unprivileged user on the host, and it still cannot touch host filesystems. So `tmp.mount` succeeds because the container may now mount a tmpfs in its own mount namespace, which was always harmless and was blocked as collateral damage from a blanket restriction. Enabling it did not undo the reason I picked unprivileged in the first place.

**Lesson.** Picking the more restrictive option means meeting its costs, and the useful part is meeting them knowingly. I predicted before testing that whatever failed would be something the container was not allowed to do to the kernel, and all three failures were mounts. Being right about the class of failure is worth more than the fix, because the fix is one line in a wiki and the reasoning transfers.

Also worth separating: `degraded` is not `failed`. The system booted, I was logged into it, and it worked. Three optional mounts had not happened. That distinction is the difference between a real incident and a warning.

## VM against LXC, proved rather than asserted

Same distribution, both Debian 13, one command each:

```
# in the VM
$ uname -r
6.12.107+deb13-amd64

# in the container
# uname -r
7.0.14-12-pve
```

The VM runs Debian's own kernel. The container runs the Proxmox host's kernel, because it shares it.

That single fact explains both of the things above. It is why the container could not mount its own filesystems until I granted nesting - there is one kernel and it is not the container's. And it is why a Windows Server VM could never be a container: Windows needs the NT kernel, an LXC gets the host's Linux kernel, and there is no mechanism by which that could work. It is not a support boundary or a design preference, it is impossible.

The sizes say the same thing more cheaply. Debian 13 as a container template is 124 MB. Debian 13 as a bootable ISO is 791 MB. The container image carries no kernel, no bootloader, no initramfs and no installer, because it does not boot - it starts inside a kernel that is already running.

## What I would do differently

- **List, do not retype.** `pveam list local` prints the volume ID in exactly the form `pct create` wants. Typing a filename from memory cost me a failed command for no reason.
- **A decision I do not encode is a decision I did not make.** I chose an unprivileged container, said so out loud, and then left `--unprivileged 1` out of the command. It would have come up privileged and nothing would have told me. I now check with `pct config 100 | grep unprivileged` rather than trusting that I typed what I meant.
- **Name things at creation.** The container is `lab05-ct` because I named it. The VM is still `VM 101` and its hostname is `debian`, which will tell me nothing in three months.
- **A text login prompt is not a broken install.** I thought I had ruined the VM install because it came up to a console instead of a desktop. That is what a correctly installed headless server looks like. My expectation was wrong, not the install.

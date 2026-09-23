# Homelab

Hands-on homelab documenting my build toward Linux/infrastructure work - testing and working with different tools, and first attempts at implementing my bachelor thesis infrastructure. Failures and learnings included, for my future self.

## Estate

| Machine | Role | OS | Status |
|---------|------|----|--------|
| textboxer | ThinkPad lab workstation - runs labs and experiments; first built by hand on Arch Linux in June 2026 | Ubuntu | live |
| fujitsu | Proxmox VE host - virtualisation estate and the platform for my bachelor thesis | Proxmox VE 9 (Debian 13) | live |

fujitsu runs headless and is administered over SSH. Build notes: [textboxer](docs/textboxer.md), [fujitsu](docs/fujitsu.md), [network](docs/network.md).

## Running guests

On fujitsu: an unprivileged Debian 13 LXC container (`lab05-ct`) built from the CLI, and a Debian 13 VM. Both are lab guests, not services yet - see [first guests on Proxmox](labs/proxmox-first-guests.md).

## Roadmap

Everything below builds toward an IaC-managed lab and the platform for my bachelor thesis.

- [x] Proxmox VE on the fujitsu host
- [x] First guests: Debian VM + LXC container (LXC built from the CLI)
- [ ] Snapshot rollback and a `vzdump` backup restored *and booted*
- [x] External data tier: 2 TB USB disk, UUID-mounted, survives an unattended reboot
- [ ] Data tier passed into a container as a bind mount
- [ ] Docker on textboxer
- [ ] Uptime Kuma - monitoring for the estate
- [ ] Windows Server + AD VM - mixed-estate practice
- [ ] k3s - CNCF-certified Kubernetes distribution, right-sized for this hardware
- [ ] Zabbix + Grafana
- [ ] Ansible
- [ ] Terraform (evaluating)

## Labs

Worked problems with the commands, the failure and the proof:

- [The 2 TB USB data tier](labs/usb-data-tier.md) - partition, fstab by UUID, a USB fault diagnosed from the kernel log, reboot proof
- [First guests on Proxmox](labs/proxmox-first-guests.md) - an LXC from the CLI, a VM, and why systemd degraded until nesting was enabled

## Docs

Deep dives per machine live in [docs](docs/).

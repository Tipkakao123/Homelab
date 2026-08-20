# Homelab

Hands-on homelab documenting my build toward Linux/infrastructure work — testing and working with different tools, and first attempts at implementing my bachelor thesis infrastructure. Failures and learnings included, for my future self.

## Estate

| Machine | Role | OS | Status |
|---------|------|----|--------|
| textboxer | ThinkPad lab workstation - CLI-only, headless over SSH; runs labs and experiments | Arch Linux | live |
| fujitsu | Proxmox VE host - virtualisation estate and the platform for my bachelor thesis | Proxmox VE 9 (Debian 13) | live |

Both boxes run headless and are administered over SSH. Build notes: [textboxer](docs/textboxer.md), [fujitsu](docs/fujitsu.md), [network](docs/network.md).

## Running services

No guest workloads yet - the hypervisor is up, first VM and LXC are next.

## Roadmap

Everything below builds toward an IaC-managed lab and the platform for my bachelor thesis.

- [x] Proxmox VE on the fujitsu host
- [ ] First guests: Debian VM + LXC container (LXC built from the CLI)
- [ ] Snapshot rollback and a `vzdump` backup restored *and booted*
- [ ] External data tier: 2 TB USB disk, UUID-mounted, passed into a container
- [ ] Docker on textboxer
- [ ] Uptime Kuma - monitoring for the estate
- [ ] Windows Server + AD VM - mixed-estate practice
- [ ] k3s - CNCF-certified Kubernetes distribution, right-sized for this hardware
- [ ] Zabbix + Grafana
- [ ] Ansible
- [ ] Terraform (evaluating)

## Docs

Deep dives per machine live in [docs](docs/).

# Homelab

Hands-on homelab documenting my build toward Linux/infrastructure work — testing and working with different tools, and first attempts at implementing my bachelor thesis infrastructure. Failures and learnings included, for my future self.

## Estate

| Machine | Role | OS | Status |
|---------|------|----|--------|
| textboxer | ThinkPad lab workstation — CLI-only, headless over SSH; runs labs and experiments | Arch Linux | live |
| fujitsu (name pending) | Proxmox host — the main brain of the lab and my thesis platform | Proxmox VE | waiting on parts |

## Running services

None yet — Uptime Kuma is first up (see roadmap).

## Roadmap

Everything below builds toward an IaC-managed lab and the platform for my bachelor thesis.

- [ ] Docker on textboxer
- [ ] Uptime Kuma — monitoring for the estate
- [ ] Proxmox VE on the fujitsu host
- [ ] k3s — CNCF-certified Kubernetes distribution, right-sized for this hardware
- [ ] Zabbix + Grafana
- [ ] Ansible
- [ ] Terraform (evaluating)

## Docs

Deep dives per machine live in [docs](docs/).

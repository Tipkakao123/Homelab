# network - addressing and a wifi diagnosis

Two hosts, one flat network, done properly this time. This documents the addressing decisions and the wifi failure that came with them - a diagnosis, not just a working config.

| | |
|---|---|
| Gateway / DNS | `192.168.3.1` (Huawei WiFi Mesh 3+) |
| Static band | `.2` - `.99` |
| DHCP pool | `.100` - `.254` |
| `pve` | `192.168.3.10`, static, `ssh pve` |
| `textboxer` | `192.168.3.11`, static, `ssh textboxer` |

## Addressing decisions

**A static band, not per-device reservations.** Both approaches stop an address from drifting. A reservation lives in the router's binding table - it's centralised and nice to browse, but it means every future static device needs a trip into the router UI, and the convention only exists if you remember to look. Narrowing the router's own DHCP pool to `.100-.254` states the convention once, in one place, and it holds even for a device that's never touched the router's admin page. `.2-.99` is now simply "infrastructure lives here".

**Static on the host, not a reservation on the router, for the same reason fujitsu already settled this one:** both give a stable address; they differ in where the truth lives. A reservation makes the host's identity depend on the router being up and correctly configured at boot. A server should know who it is without asking. For two boxes I administer over SSH, static wins on both.

**The `nmcli` list trap - the near-miss worth keeping.** Setting textboxer's static IP, the first attempt wrote:

```
nmcli con mod <profile> ipv4.addresses "192.168.3.11/24, 192.168.3.1"
```

`ipv4.addresses` is a list property - it takes CIDR addresses, comma-separated if there's more than one. It has no concept of "and here's the gateway." Applied as written, NetworkManager would have tried to assign the router's own address, `192.168.3.1`, to the laptop as a second address on the same interface - a duplicate-address conflict on the one machine every other device on the network depends on to route anywhere. That's not "textboxer loses wifi," that's "the flat loses internet" until someone finds the laptop and disconnects it. Caught before applying, by reading what the property actually expects rather than assuming a space-separated shorthand still worked. The gateway is a separate property, `ipv4.gateway`, and the older combined syntax was removed from NetworkManager for exactly this reason - it let two different concepts collapse into one string.

## Problem - wifi wouldn't reconnect after the household PSK got changed

**Symptom.** textboxer came back up asking for wifi secrets on every attempt. `nmcli device wifi connect` kept failing with no more detail than "secrets required."

**Diagnosis.** `journalctl -u NetworkManager` had the actual line:

```
4WAY_HANDSHAKE -> DISCONNECTED, psk mismatch reported by supplicant
```

That one line rules things out faster than guessing does. Getting as far as the 4-way handshake means the access point was found, the SSID matched, and cipher negotiation succeeded - the whole WPA3/SAE-incompatibility theory dies right there, before touching a single router setting. A handshake failure that specifically reports a PSK mismatch means the credential is wrong, not the protocol.

**Cause.** A typo in the new PSK, sitting in a stale connection profile. Not more interesting than that - which is the point. This one took minutes to find because the log said exactly what was wrong, against the hour it would have taken reconfiguring the router on a wrong theory.

**Fix.** Deleted the stale profile and reconnected with the corrected PSK.

## Headless secrets - `psk-flags`

By default NetworkManager can store a wifi secret as agent-owned, meaning it's held by a user-session secret agent and typed in (or unlocked) at login. That's fine on a desktop with someone sitting at it. It's a guaranteed failure on a console-only box: no session, no agent, no prompt - the exact "server doesn't come back after a power cut" scenario. Set:

```
nmcli con mod <profile> wifi-sec.psk-flags 0
```

`psk-flags 0` makes the secret system-owned - stored in the connection file itself, so the box can associate on its own at boot with nobody logged in. The tradeoff is the obvious one: the secret sits in a file, readable by root, which is the right owner for a box that's meant to come back unattended.

## Naming - WSL and the generated hosts file

Wanted `ssh pve` and `ssh textboxer` to resolve from inside WSL, the same as from Windows. Editing WSL's own `/etc/hosts` directly didn't stick - WSL regenerates that file from the Windows hosts file on every start (`generateHosts` in `wsl.conf` can turn that off, but disabling it turns into a maintenance burden: every DNS/network change in Windows now has to be re-applied by hand). The names went into the actual source instead: `C:\Windows\System32\drivers\etc\hosts`, which propagates into WSL and also resolves the names in the Windows-side browser for the Proxmox web UI. Host+user mapping (`ssh pve` instead of `ssh root@192.168.3.10`) lives in `~/.ssh/config`, permissions `600`.

## Verification

Both `ssh pve` and `ssh textboxer` resolve and connect from a cold laptop - fresh boot, no prior session, names and keys both working.

## Not done yet

- No segmentation - one flat `/24`, one broadcast domain. The narrowed DHCP pool gives predictable addressing, not isolation: any host can reach any other. This starts to matter with the first guests, where VM traffic sits on the same L2 as the host's management interface. The fix is a management VLAN separated from workloads and from household devices, which needs 802.1Q-capable kit. A MikroTik is the plan - fully configurable, and the usual recommendation for learning networking hands-on rather than the minimum box that would do the job. Not bought yet.

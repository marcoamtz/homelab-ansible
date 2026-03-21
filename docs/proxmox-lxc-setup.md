# Proxmox LXC Container Setup

This documents the Proxmox LXC container configuration required before running the Ansible playbooks. Replace all placeholder values (`<...>`) with your own.

## CT 100 — DNS Gateway

Create an unprivileged Debian LXC container with the following settings:

| Setting | Value |
|---------|-------|
| OS | Debian |
| Arch | amd64 |
| Cores | 1 |
| Memory | 512 MB |
| Swap | 512 MB |
| Disk | 8G |
| Hostname | `dns-gateway` |
| Unprivileged | Yes |
| Start on boot | Yes |
| Startup order | 1 |
| Features | `nesting=1` |

### Network (net0)

| Setting | Value |
|---------|-------|
| Bridge | `vmbr0` |
| Firewall | Yes |
| IPv4 | `<DNS_SERVER_IP>/24` |
| IPv4 Gateway | `<ROUTER_IP>` |
| IPv6 | `<ULA_PREFIX>::<HOST_ID>/64` |
| IPv6 Gateway | `<ULA_PREFIX>::1` |

### DNS

| Setting | Value |
|---------|-------|
| Nameserver | `<PUBLIC_DNS>` (e.g. `9.9.9.9`, only needed for initial setup) |
| Search domain | `<YOUR_DOMAIN>` (e.g. `home.arpa`) |

> After the DNS playbook runs, the container resolves through itself (`127.0.0.1`). The nameserver here is only used during first boot and package installation.

---

## CT 101 — Tailscale Bridge

Create an unprivileged Debian LXC container with the following settings:

| Setting | Value |
|---------|-------|
| OS | Debian |
| Arch | amd64 |
| Cores | 2 |
| Memory | 512 MB |
| Swap | 512 MB |
| Disk | 8G |
| Hostname | `tailscale-bridge` |
| Unprivileged | Yes |
| Start on boot | Yes |
| Startup order | 2 (starts after DNS gateway) |
| Features | `nesting=1` |

### Network (net0)

| Setting | Value |
|---------|-------|
| Bridge | `vmbr0` |
| Firewall | Yes |
| IPv4 | `<TAILSCALE_NODE_IP>/24` |
| IPv4 Gateway | `<ROUTER_IP>` |
| IPv6 | `auto` (SLAAC) |

> IPv6 is set to `auto` so the container gets a public GUA via SLAAC, which Tailscale needs for IPv6 connectivity. The Ansible playbook assigns a static ULA address alongside SLAAC.

### DNS

| Setting | Value |
|---------|-------|
| Nameserver | `<DNS_SERVER_IP> <DNS_SERVER_IPV6>` |
| Search domain | `<YOUR_DOMAIN>` (e.g. `home.arpa`) |

### TUN device passthrough

Tailscale requires `/dev/net/tun` access. The container must be **stopped** before adding these lines. Add them to the container config on the Proxmox host (`/etc/pve/lxc/<CTID>.conf`):

```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

Or via the Proxmox shell:

```bash
pct set <CTID> -lxc.cgroup2.devices.allow "c 10:200 rwm"
pct set <CTID> -lxc.mount.entry "/dev/net/tun dev/net/tun none bind,create=file"
```

Restart the container after applying.

---

## Firewall

Both containers have `firewall=1` on their network interface. The firewall rules are managed by the `deploy-proxmox-firewall.yml` playbook — see the main [README](../README.md) for details.

The Tailscale container also requires an `ipfilter-net0` IPSet to allow its SLAAC-assigned IPv6 addresses through the Proxmox firewall. This is handled by the firewall playbook via the `ipfilter_v6_prefixes` variable in `group_vars/proxmox_hosts.yml`.

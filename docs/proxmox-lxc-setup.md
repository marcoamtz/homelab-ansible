# Proxmox LXC Container Setup

This documents the Proxmox LXC container configuration required before running the Ansible playbooks. Replace all placeholder values (`<...>`) with your own.

## Proxmox Post-Install

After installing Proxmox, run the [community post-install script](https://community-scripts.org/scripts/post-pve-install) from the Proxmox shell:

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/misc/post-pve-install.sh)"
```

This configures the no-subscription repository, removes the enterprise subscription nag, and updates the system. Answer `yes` to all prompts and reboot when finished.

---

## CPU Governor — Powersave

Set the CPU governor to `powersave` to reduce power consumption and heat. Install `linux-cpupower` and create a systemd service to apply it on boot:

```bash
apt install linux-cpupower
```

Create `/etc/systemd/system/cpupower.service`:

```ini
[Unit]
Description=Apply CPU Power Management Settings
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/usr/bin/cpupower frequency-set -g powersave
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```bash
systemctl enable --now cpupower.service
```

---

## Storage — ZFS Pool

Create a ZFS pool on the NVMe drive for container storage. In the Proxmox web UI:

1. Go to **Disks > ZFS > Create: ZFS**
2. Select the NVMe drive
3. Set the name to `tank`
4. RAID level: Single Disk
5. Click **Create**

The `tank` pool will appear under **Datacenter > Storage** and is used as the storage target for all LXC containers below.

---

## CT 100 — DNS Gateway

Create an unprivileged Debian LXC container with the following settings:

| Setting | Value |
|---------|-------|
| OS | Debian |
| Arch | amd64 |
| Cores | 1 |
| Memory | 512 MB |
| Swap | 512 MB |
| Storage | `tank` (ZFS) |
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
| Storage | `tank` (ZFS) |
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

## CT 102 — Docker Server

Create an unprivileged Debian LXC container with the following settings:

| Setting | Value |
|---------|-------|
| OS | Debian |
| Arch | amd64 |
| Cores | 6 |
| Memory | 8192 MB |
| Swap | 2048 MB |
| Storage | `tank` (ZFS) |
| Disk | 16G |
| Hostname | `docker-server` |
| Unprivileged | Yes |
| Start on boot | Yes |
| Startup order | 3 (starts after DNS and Tailscale) |
| Features | `nesting=1,keyctl=1` |

> `nesting=1` is required for Docker-in-LXC. `keyctl=1` is required for certain container runtimes and systemd features inside Docker containers.

### Network (net0)

| Setting | Value |
|---------|-------|
| Bridge | `vmbr0` |
| Firewall | Yes |
| IPv4 | `<DOCKER_SERVER_IP>/24` |
| IPv4 Gateway | `<ROUTER_IP>` |
| IPv6 | `auto` (SLAAC) |

> IPv6 is set to `auto` so the container gets a public GUA via SLAAC, which is required for incoming IPv6 torrent peer connections (qBittorrent). The `ipfilter-net0` IPSet in the firewall config must cover the SLAAC-assigned prefixes — this is handled by the `ipfilter_v6_prefixes` variable in `group_vars/proxmox_hosts.yml`.

### DNS

| Setting | Value |
|---------|-------|
| Nameserver | `<DNS_SERVER_IP>` |
| Search domain | `<YOUR_DOMAIN>` (e.g. `home.arpa`) |

### Storage layout

The root disk (16G) holds Docker images, container configs, and Dockge data. Large or fast-changing data uses separate mounts:

- **`mp0:` — Fast NVMe scratch space** — A separate ZFS dataset on `tank` for qBittorrent incomplete downloads and Emby transcoding temp. Uses the fast NVMe and is not limited by the root disk size.
- **`mp1:` — Synology NFS (media)** — Bind mount from the Proxmox host's NFS mount for the Emby media library.
- **`mp2:` — Synology NFS (complete)** — Bind mount from the Proxmox host's NFS mount for qBittorrent completed downloads.

Create the ZFS dataset on the Proxmox host and fix ownership for the unprivileged container (UID 0 inside maps to 100000 on the host):

```bash
zfs create tank/subvol-<CTID>-tank
chown -R 100000:100000 /tank/subvol-<CTID>-tank
```

Then add these to `/etc/pve/lxc/<CTID>.conf` while the container is **stopped**:

```
mp0: tank:subvol-<CTID>-tank,mp=/mnt/tank
mp1: <PROXMOX_NFS_MEDIA_PATH>,mp=/mnt/nas/media
mp2: <PROXMOX_NFS_COMPLETE_PATH>,mp=/mnt/nas/complete
```

Where `<PROXMOX_NFS_MEDIA_PATH>` and `<PROXMOX_NFS_COMPLETE_PATH>` are the Synology NFS mounts on the Proxmox host. The mount points must match `qbittorrent_incomplete_dir`, `emby_transcode_dir`, `nas_media_path`, and `nas_complete_path` in `group_vars/docker_hosts.yml`.

### GPU passthrough (Intel Quick Sync)

Emby uses Intel Quick Sync for hardware video transcoding. The container must be **stopped** before adding these lines to `/etc/pve/lxc/<CTID>.conf`:

```
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
```

Restart the container after applying.

The GPU devices also need world-readable permissions on the Proxmox host — unprivileged LXC UID mapping makes the render group inaccessible inside the container. This is handled by the `deploy-proxmox-host.yml` playbook, which installs a udev rule to persist the permissions across reboots.

---

## Firewall

All containers have `firewall=1` on their network interface. The firewall rules are managed by the `deploy-proxmox-firewall.yml` playbook — see the main [README](../README.md) for details.

The Tailscale and Docker containers also require an `ipfilter-net0` IPSet to allow their SLAAC-assigned IPv6 addresses through the Proxmox firewall. This is handled by the firewall playbook via the `ipfilter_v6_prefixes` variable in `group_vars/proxmox_hosts.yml`.

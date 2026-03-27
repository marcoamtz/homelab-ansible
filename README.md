# Homelab Ansible

Ansible playbooks for provisioning Proxmox LXC containers with network services.

## Playbooks

### `deploy-dns.yml` — NextDNS + Dnsmasq

Deploys [NextDNS CLI](https://github.com/nextdns/nextdns) and [dnsmasq](https://thekelleys.org.uk/dnsmasq/doc.html) for network-wide DNS filtering with split DHCP configuration.

- NextDNS CLI as the upstream encrypted DNS resolver
- Dnsmasq for DHCP and local DNS caching
- Per-device DNS bypass (route specific devices to ISP DNS instead of NextDNS)
- Static DHCP reservations
- Tailscale MagicDNS forwarding
- Dnsmasq config validation before restart
- Stale config cleanup (removed local configs are removed from the server)
- Post-deploy DNS smoke test

### `deploy-tailscale.yml` — Tailscale Subnet Router

Deploys [Tailscale](https://tailscale.com) as a subnet router and exit node, bridging your local network into your tailnet.

- Tailscale CLI install and authentication
- IPv4 and IPv6 forwarding with SLAAC support (`accept_ra=2`)
- Static ULA address assignment (coexists with SLAAC GUA)
- Subnet route and exit node advertising
- Post-deploy connectivity check

### `deploy-docker.yml` — Docker + Dockge

Deploys [Docker CE](https://docs.docker.com/engine/) and [Dockge](https://github.com/louislam/dockge) with initial compose stacks for media and download services.

- Docker CE installed via official apt repository
- Dockge as compose stack management UI
- Emby media server with Intel Quick Sync hardware transcoding
- Speedtest Tracker for internet speed monitoring
- qBittorrent torrent client with split storage (incomplete on NVMe, complete on NFS)
- Dual-stack IPv4/IPv6 networking with SLAAC for incoming IPv6 peer connections
- Monthly cron to prune unused Docker images
- All service ports configurable via group_vars
- Post-deploy health checks

### `deploy-proxmox-host.yml` — Proxmox Host

Deploys host-level configuration to the Proxmox server.

- GPU device permissions for unprivileged LXC passthrough (udev rule)
- Weekly TRIM for the OS drive (`fstrim.timer`)
- ZFS autotrim for the tank pool

### `deploy-proxmox-firewall.yml` — Proxmox Firewall

Deploys firewall configuration to the Proxmox host, managing cluster-wide rules and per-container policies.

- Cluster firewall with security groups (DNS/DHCP, management, Tailscale, Docker)
- IPSet-based network aliases (local network, Tailscale network)
- Container-level firewall configs with IPv6 ipfilter for SLAAC

## Prerequisites

- Proxmox LXC containers running Debian/Ubuntu with systemd
- Ansible installed on your control machine
- SSH access to the containers and Proxmox host (key-based)
- A [NextDNS](https://nextdns.io) account and profile ID (for DNS playbook)
- A [Tailscale](https://login.tailscale.com/admin/settings/keys) auth key (for Tailscale playbook)

## Setup

1. Create and configure your Proxmox LXC containers following the [LXC setup guide](docs/proxmox-lxc-setup.md).

2. Copy the example files and fill in your values:

   ```bash
   cp inventory.ini.example inventory.ini
   cp group_vars/dns_servers.yml.example group_vars/dns_servers.yml
   cp group_vars/tailscale_nodes.yml.example group_vars/tailscale_nodes.yml
   cp group_vars/proxmox_hosts.yml.example group_vars/proxmox_hosts.yml
   cp group_vars/docker_hosts.yml.example group_vars/docker_hosts.yml
   ```

3. Edit `inventory.ini` with your server IPs and SSH settings.

4. Edit `group_vars/dns_servers.yml` with your network configuration:
   - `nextdns_id` — your NextDNS profile ID
   - `network_interface` — LXC network interface (e.g. `eth0`)
   - `domain` — local domain name (e.g. `home.arpa`)
   - `dns_server_ip`, `gateway_ip` — IPv4 network addresses
   - `ipv6_prefix` — ULA prefix for IPv6
   - `dhcp_range_start`, `dhcp_range_end`, `dhcp_lease_time`, `dhcp_lease_max` — DHCP pool settings
   - `nextdns_listen_port` — port NextDNS CLI listens on locally
   - `tailscale_domain` — your tailnet domain for MagicDNS forwarding
   - `bypass_devices` — devices that skip NextDNS filtering
   - `static_leases` — fixed DHCP reservations
   - `host_records` — DNS records for devices with static IPs that don't use DHCP

5. Edit `group_vars/tailscale_nodes.yml` with your Tailscale settings:
   - `tailscale_auth_key` — auth key from the Tailscale admin console
   - `ula_address` — static ULA address for the container (assigned alongside SLAAC)
   - `tailscale_args` — CLI flags for `tailscale up` (advertised routes, exit node, etc.)

6. Edit `group_vars/docker_hosts.yml` with your Docker settings:
   - `dockge_port` — Dockge web UI port (default: 5001)
   - `emby_port_http`, `emby_port_https` — Emby ports (default: 8096, 8920)
   - `emby_uid`, `emby_gid` — file ownership for Emby media access
   - `speedtest_port` — Speedtest Tracker port (default: 8088)
   - `speedtest_url` — IP or hostname for Speedtest Tracker's APP_URL
   - `speedtest_app_key` — application key ([generate here](https://speedtest-tracker.dev/))
   - `speedtest_schedule` — cron schedule for speed tests
   - `speedtest_servers` — comma-separated Ookla server IDs
   - `speedtest_mail_*` — SMTP settings for email notifications
   - `qbittorrent_port_webui`, `qbittorrent_port_torrent` — qBittorrent ports (default: 8080, 6881)
   - `qbittorrent_puid`, `qbittorrent_pgid` — file ownership for downloads
   - `nas_media_path` — bind mount path for Emby media library (Synology NFS)
   - `nas_complete_path` — bind mount path for qBittorrent completed downloads (Synology NFS)
   - `qbittorrent_incomplete_dir` — path for active downloads (ZFS bind mount)
   - `emby_transcode_dir` — path for Emby transcoding temp files (ZFS bind mount)
   - `docker_timezone` — timezone for all containers
   - `docker_ipv6_cidr` — ULA subnet for Docker's default bridge network
   - `docker_ipv6_pool` — ULA pool for Docker Compose networks

7. Edit `group_vars/proxmox_hosts.yml` with your Proxmox settings:
   - `dns_ctid`, `tailscale_ctid`, `docker_ctid` — container IDs
   - `local_ipv4_subnet` — your LAN subnet
   - `ipfilter_v6_prefixes` — IPv6 prefixes allowed in the Tailscale and Docker container ipfilters (must cover SLAAC addresses)
   - Docker service ports (must match `docker_hosts` group_vars for firewall rules)

## Deploy

```bash
ansible-playbook deploy-proxmox-host.yml
ansible-playbook deploy-dns.yml
ansible-playbook deploy-tailscale.yml
ansible-playbook deploy-docker.yml
ansible-playbook deploy-proxmox-firewall.yml
```

Update all LXC container packages:

```bash
ansible-playbook update-all.yml
```

Dry run (no changes):

```bash
ansible-playbook deploy-proxmox-host.yml --check
ansible-playbook deploy-dns.yml --check
ansible-playbook deploy-tailscale.yml --check
ansible-playbook deploy-docker.yml --check
ansible-playbook deploy-proxmox-firewall.yml --check
```

## LXC Notes

The container playbooks include workarounds for Proxmox LXC containers:

- Uses `systemctl` commands directly instead of the Ansible `systemd` module (which fails to enumerate services inside LXC)
- Pins `/etc/resolv.conf` to `127.0.0.1` and creates `.pve-ignore.resolv.conf` to prevent Proxmox from overwriting it (DNS playbook)
- Force-applies `sysctl` settings on every run and installs a `@reboot` cron job since LXC hosts can reset forwarding on container restart (Tailscale playbook)

## Project Structure

```
docs/
  proxmox-lxc-setup.md            # LXC container creation guide
group_vars/
  dns_servers.yml.example          # Example DNS variables
  docker_hosts.yml.example         # Example Docker variables
  proxmox_hosts.yml.example        # Example Proxmox variables
  tailscale_nodes.yml.example      # Example Tailscale variables
templates/
  dnsmasq.d/
    01-base.conf.j2                # Core dnsmasq settings
    02-dhcp.conf.j2                # DHCP, static leases, DNS bypass
    06-rfc6761.conf.j2             # RFC 6761 special domains
  docker/
    daemon.json.j2                 # Docker daemon config (IPv6, ip6tables)
    dockge/compose.yml.j2          # Dockge compose stack
    emby/compose.yml.j2            # Emby media server
    speedtest-tracker/compose.yml.j2  # Speedtest Tracker
    qbittorrent/compose.yml.j2     # qBittorrent torrent client
  proxmox/firewall/
    cluster.fw.j2                  # Datacenter firewall and security groups
    ct-dns.fw.j2                   # DNS container firewall
    ct-docker.fw.j2                # Docker container firewall
    ct-tailscale.fw.j2             # Tailscale container firewall + ipfilter
  nextdns.conf.j2                  # NextDNS CLI config
ansible.cfg                        # Ansible config (default inventory)
deploy-dns.yml                     # DNS playbook
deploy-docker.yml                  # Docker + Dockge playbook
deploy-proxmox-firewall.yml        # Proxmox firewall playbook
deploy-proxmox-host.yml            # Proxmox host config (GPU passthrough, TRIM)
deploy-tailscale.yml               # Tailscale playbook
update-all.yml                     # Update packages on all LXC containers
inventory.ini.example              # Example inventory
```

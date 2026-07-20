# Homelab Ansible

Ansible playbooks for provisioning Proxmox LXC containers with network services.

Playbooks are thin wrappers over roles in `roles/` — shared plumbing
(packages, locale, msmtp, sysctl, the LXC systemctl workaround) lives in the
`common` role, and each service has its own role. See
[docs/architecture.md](docs/architecture.md) for the LXC-specific design
decisions.

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
- IPv6 DNS reset detection with email alerts (ISP TR-069 monitoring, retry on mail failure)
- Post-deploy DNS smoke test

> Note: the NextDNS CLI config enables `log-queries` and `report-client-info`,
> so DNS queries and client names are visible in your NextDNS dashboard.
> Adjust `roles/dns_server/templates/nextdns.conf.j2` if you don't want that.

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
- Jellyfin media server with Intel Quick Sync hardware transcoding
- Speedtest Tracker for internet speed monitoring
- qBittorrent torrent client with split storage (incomplete on NVMe, complete on NFS)
- Dual-stack IPv4/IPv6 networking with SLAAC for incoming IPv6 peer connections
- NFS mount validation before starting stacks (fails if bind mounts are not active)
- Monthly cron to prune unused Docker images
- All service ports configurable via group_vars
- Post-deploy health checks

### `deploy-proxmox-host.yml` — Proxmox Host

Deploys host-level configuration to the Proxmox server.

- GPU device permissions for unprivileged LXC passthrough (udev rule)
- Weekly TRIM for the OS drive (`fstrim.timer`)
- Weekly TRIM for the ZFS tank pool (`zfs-trim-weekly@tank.timer`)

### `deploy-proxmox-firewall.yml` — Proxmox Firewall

Deploys firewall configuration to the Proxmox host, managing cluster-wide rules and per-container policies.

- Cluster firewall with security groups (DNS/DHCP, management, Tailscale, Docker)
- IPSet-based network aliases (local network, Tailscale network)
- Container-level firewall configs with IPv6 ipfilter for SLAAC

### `update-all.yml` — Package Updates

Runs `apt dist-upgrade` on all LXC containers and shows which packages were upgraded.
No reboot handling on purpose: LXC containers share the host kernel, so container
upgrades never require one.

## Prerequisites

- Proxmox LXC containers running Debian/Ubuntu with systemd
- `ansible-core` 2.15+ on your control machine
- SSH access to the containers and Proxmox host (key-based)
- Vault password file at `~/.ansible/vault_password` (see [Vault](#vault-encrypted-secrets) section)
- A [NextDNS](https://nextdns.io) account and profile ID (for DNS playbook)
- A [Tailscale](https://login.tailscale.com/admin/settings/keys) auth key (for Tailscale playbook)

## Setup

1. Install the required collections:

   ```bash
   ansible-galaxy collection install -r requirements.yml
   ```

   Create and configure your Proxmox LXC containers following the [LXC setup guide](docs/proxmox-lxc-setup.md).

2. Copy the example files and fill in your values:

   ```bash
   cp inventory.ini.example inventory.ini
   cp group_vars/dns_servers.yml.example group_vars/dns_servers.yml
   cp group_vars/tailscale_nodes.yml.example group_vars/tailscale_nodes.yml
   cp group_vars/proxmox_hosts.yml.example group_vars/proxmox_hosts.yml
   cp group_vars/docker_hosts.yml.example group_vars/docker_hosts.yml
   cp group_vars/all.yml.example group_vars/all.yml
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
   - `tailscale_auth_key` — auth key from the Tailscale admin console (🔒 vault-encrypt, see [Vault](#vault-encrypted-secrets))
   - `ula_address` — static ULA address for the container (assigned alongside SLAAC)
   - `tailscale_args` — CLI flags for `tailscale up` (advertised routes, exit node, etc.)

6. Edit `group_vars/all.yml` with shared settings:
   - `mail_host`, `mail_port`, `mail_username`, `mail_password`, `mail_from`, `mail_to` — SMTP settings for email notifications (DNS alerts + Speedtest); 🔒 vault-encrypt `mail_password`
   - `dockge_port` — Dockge web UI port (default: 5001)
   - `jellyfin_port_http`, `jellyfin_port_https` — Jellyfin ports (default: 8096, 8920)
   - `speedtest_port` — Speedtest Tracker port (default: 8088)
   - `qbittorrent_port_webui`, `qbittorrent_port_torrent` — qBittorrent ports (default: 8080, 6881)

7. Edit `group_vars/docker_hosts.yml` with your Docker settings:
   - `jellyfin_uid`, `jellyfin_gid` — container UID/GID for Jellyfin media access
   - `speedtest_url` — IP or hostname for Speedtest Tracker's APP_URL
   - `speedtest_app_key` — application key ([generate here](https://speedtest-tracker.dev/)); 🔒 vault-encrypt
   - `speedtest_schedule` — cron schedule for speed tests
   - `speedtest_servers` — comma-separated Ookla server IDs
   - `qbittorrent_puid`, `qbittorrent_pgid` — file ownership for downloads
   - `nas_media_path` — bind mount path for Jellyfin media libraries (Synology NFS)
   - `nas_complete_path` — bind mount path for qBittorrent completed downloads (Synology NFS)
   - `qbittorrent_incomplete_dir` — path for active downloads (ZFS bind mount)
   - `jellyfin_cache_dir` — path for Jellyfin cache data (ZFS bind mount)
   - `jellyfin_transcode_dir` — path for Jellyfin transcoding temp files (ZFS bind mount)
   - `docker_timezone` — timezone for all containers
   - `docker_ipv6_cidr` — ULA subnet for Docker's default bridge network
   - `docker_ipv6_pool` — ULA pool for Docker Compose networks

8. Edit `group_vars/proxmox_hosts.yml` with your Proxmox settings:
   - `dns_ctid`, `tailscale_ctid`, `docker_ctid` — container IDs
   - `local_ipv4_subnet` — your LAN subnet
   - `ipfilter_v6_prefixes` — IPv6 prefixes allowed in the Tailscale and Docker container ipfilters (must cover SLAAC addresses)
   - `lxc_pre_start_nfs_waits`, `lxc_pre_start_nfs_timeout` — per-CT list of NFS mount paths that must be mounted on the host before the LXC is allowed to start (eliminates bind-mount-captures-empty-dir race on Proxmox boot)

## Vault (Encrypted Secrets)

Sensitive values (`mail_password`, `speedtest_app_key`, `tailscale_auth_key`) are encrypted inline using `ansible-vault`. The vault password file at `~/.ansible/vault_password` is referenced in `ansible.cfg`, so playbooks decrypt automatically.

**View a decrypted value:**

```bash
ansible localhost -m debug -a "var=mail_password" -e @group_vars/all.yml
```

**Encrypt a new value:**

```bash
ansible-vault encrypt_string 'my-secret-value' --name 'variable_name'
```

Copy the output and replace the variable in the relevant `group_vars/*.yml` file.

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

Partial runs via tags — every task is tagged `install`, `config`, `service`
or `validate` (plus `firewall` on the firewall playbook):

```bash
# Re-render configs and restart what changed, skip package installs
ansible-playbook deploy-docker.yml --tags config,service

# Health checks only
ansible-playbook deploy-dns.yml --tags validate

# One-shot recursive reset of stack config ownership (off by default)
ansible-playbook deploy-docker.yml --tags fix-permissions
```

## Linting

```bash
pre-commit install        # once; runs yamllint + ansible-lint on commit
pre-commit run --all-files
```

## LXC Notes

The container roles include workarounds for Proxmox LXC containers —
`systemctl` via command instead of the `systemd` module, pinned
`resolv.conf`, sysctl via handler + `@reboot` cron, a pre-start NFS gate on
the host. The reasoning for each lives in
[docs/architecture.md](docs/architecture.md).

## Backup

`group_vars/*.yml` and `inventory.ini` are gitignored — the public repo only
carries `.example` templates, so **this working copy is the only place your
live config exists**. Secrets inside the real files are vault-encrypted, and
the vault password lives at `~/.ansible/vault_password` (outside the repo).
Make sure both are covered by a backup: either a private git remote for the
real files, or including this directory and the vault password file in your
machine backup (Time Machine, restic, etc.).

## Project Structure

```
docs/
  architecture.md                  # LXC-specific design decisions
  proxmox-lxc-setup.md             # LXC container creation guide
group_vars/
  all.yml.example                  # Shared settings (email, service ports)
  dns_servers.yml.example          # Example DNS variables
  docker_hosts.yml.example         # Example Docker variables
  proxmox_hosts.yml.example        # Example Proxmox variables
  tailscale_nodes.yml.example      # Example Tailscale variables
roles/
  common/                          # Base packages, locale, msmtp, shared LXC plumbing
    tasks/                         #   assert_debian, systemd_enable, sysctl_dropin, sysctl_reboot_cron
    templates/msmtprc.j2           #   SMTP client config for email alerts
  dns_server/                      # NextDNS + dnsmasq + IPv6 DNS reset monitoring
    templates/dnsmasq.d/           #   base config, DHCP, RFC 6761 special domains
    templates/nextdns.conf.j2      #   NextDNS CLI config
    templates/check-ipv6-dns.sh.j2 #   IPv6 DNS reset detection script
  tailscale_node/                  # Tailscale install, forwarding, ULA, auth
  docker_host/                     # Docker CE via deb822 repo, daemon.json, IPv6 RA
    templates/daemon.json.j2       #   Docker daemon config (IPv6, ip6tables)
  compose_stack/                   # Data-driven compose stacks (see defaults/main.yml)
    templates/                     #   dockge, jellyfin, speedtest-tracker, qbittorrent
  proxmox_host/                    # GPU passthrough, TRIM timers, NFS pre-start hook
    templates/wait-for-nfs.sh.j2   #   CT pre-start hook: block boot until NFS mounted
  proxmox_firewall/                # Cluster + per-CT firewall configs
    templates/                     #   cluster.fw, ct-dns.fw, ct-docker.fw, ct-tailscale.fw
ansible.cfg                        # Ansible config (inventory, fact cache, callbacks)
requirements.yml                   # Collection pins (community.docker, ansible.posix)
deploy-dns.yml                     # DNS playbook (common + dns_server)
deploy-docker.yml                  # Docker playbook (common + docker_host + compose_stack)
deploy-proxmox-firewall.yml        # Proxmox firewall playbook
deploy-proxmox-host.yml            # Proxmox host playbook
deploy-tailscale.yml               # Tailscale playbook (common + tailscale_node)
update-all.yml                     # Update packages on all LXC containers
inventory.ini.example              # Example inventory
```

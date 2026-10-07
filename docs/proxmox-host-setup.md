# Proxmox Host Setup

Manual steps on the Proxmox host: one-time setup before running the
playbooks (1–7), then routine host updates. Everything else — container
creation, host config (including the mail relay and SSH hardening), firewall
— is handled by `deploy-proxmox-lxcs.yml`, `deploy-proxmox-host.yml` and
`deploy-proxmox-firewall.yml`. Replace placeholder values (`<...>`) with your
own.

Requires Proxmox VE 8.2+ (native `devN` device passthrough).

## 1. Post-install script

Run the [community post-install script](https://community-scripts.org/scripts/post-pve-install)
from the Proxmox shell:

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/misc/post-pve-install.sh)"
```

It configures the no-subscription repository, removes the subscription nag,
and updates the system. Answer `yes` to all prompts and reboot when finished.

## 2. CPU governor — powersave

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

```bash
systemctl enable --now cpupower.service
```

## 3. ZFS pool `tank`

In the web UI: **Disks > ZFS > Create: ZFS**, select the NVMe drive, name
`tank`, RAID level Single Disk. All container root disks (`rootfs: tank:<size>`
in `proxmox_lxcs`) live here, and `deploy-proxmox-host.yml` enables its weekly
TRIM timer (`zfs_trim_pool`).

## 4. NFS storage (Synology)

In the web UI: **Datacenter > Storage > Add > NFS**, once per share. Proxmox
mounts each at `/mnt/pve/<storage-id>`. Those paths are the bind-mount sources
for the Docker container (`mp1`/`mp2` in `proxmox_lxcs`) and the paths listed
in `lxc_pre_start_nfs_waits`.

## 5. Docker scratch dataset

The Docker container's `mp0` is a separate ZFS dataset for qBittorrent
incomplete downloads and Jellyfin cache/transcode temp. Create it before
`deploy-proxmox-lxcs.yml` creates the container, and hand it to the
unprivileged container's root (UID 0 inside maps to 100000 on the host):

```bash
zfs create tank/subvol-<CTID>-tank
chown 100000:100000 /tank/subvol-<CTID>-tank
```

Then reference it in the container's `proxmox_lxcs` entry:

```yaml
mp0: tank:subvol-<CTID>-tank,mp=/mnt/tank
mp1: /mnt/pve/<media-storage-id>,mp=/mnt/nas/media
mp2: /mnt/pve/<complete-storage-id>,mp=/mnt/nas/complete
```

The mount points must match `docker_tank_mount`, `nas_media_path` and
`nas_complete_path` in `group_vars/docker_hosts.yml`.

## 6. Notification email

Set an email on **Datacenter > Permissions > Users > root@pam**. Proxmox's
default `mail-to-root` notification target sends there for backups, ZFS
(`zed`) and SMART (`smartd`) events. `deploy-proxmox-host.yml` configures
Postfix to relay through the `mail_*` SMTP account in `group_vars/all.yml`,
and limits the `default-matcher` to `warning,error` severity, so successful
backups don't send mail.

Check delivery after the first deploy:

```bash
echo test | mail -s "relay test" root
mailq    # empty once delivered
```

## 7. Backups

Create a vzdump job under **Datacenter > Backup** covering all guests. Current
job: daily at 03:00, mode `snapshot`, `zstd`, retention
`keep-daily=5,keep-weekly=3,keep-monthly=1`, stored on an NFS share of the
Synology.

- The backups sit on the same NAS as the media, so a NAS loss takes both.
  Copy them off the NAS (Synology Hyper Backup to USB or cloud) for a second copy.
- A successful job doesn't prove a restore works. Restore a backup to a spare
  CTID now and then and check that it boots. Before starting it, drop the
  network (it would clash with the live container's IP) and any bind mounts
  or devices it shares with the live container:
  ```bash
  pct restore 999 /mnt/pve/<backup-storage>/dump/vzdump-lxc-<CTID>-<date>.tar.zst --storage tank --unique 1
  pct set 999 --onboot 0 --delete net0,mp1,mp2,dev0
  pct start 999 && pct exec 999 -- systemctl --failed
  pct stop 999 && pct destroy 999
  ```
- The live `group_vars`, `inventory.ini` and vault password are not in these
  backups — see Backup in the [README](../README.md#backup).

## Host updates

`update-all.yml` and `unattended-upgrades` cover only the containers. Update
the host by hand, and reboot when a new `proxmox-kernel` is installed:

```bash
apt update && apt dist-upgrade
pveversion   # the running kernel shows in parentheses
reboot       # containers stop and start in their startup order
```

## Container specifics

These go in `proxmox_lxcs` (`group_vars/proxmox_hosts.yml`), not on the host
by hand:

- **Tailscale** — `/dev/net/tun` access via raw lines under `lxc:`:
  ```yaml
  lxc:
    - "lxc.cgroup2.devices.allow: c 10:200 rwm"
    - "lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file"
  ```
- **Docker** — `features: keyctl=1,nesting=1` for Docker-in-LXC, `ip6=auto`
  on `net0` for a SLAAC GUA (incoming IPv6 torrent peers), and the Intel
  Quick Sync render node:
  ```yaml
  dev0: /dev/dri/renderD128,gid=<render gid inside the CT>
  ```
  The gid is the container's `render` group (`getent group render`); set the
  same value as `jellyfin_render_gid` in `group_vars/docker_hosts.yml`.
- **Tailscale and Docker** — SLAAC addresses must be covered by
  `ipfilter_v6_prefixes`, or the Proxmox firewall's `ipfilter-net0` drops them.

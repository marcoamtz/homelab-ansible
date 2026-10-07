# Architecture Notes

Decisions in this repo that look odd without context. Most exist because the
targets are unprivileged Proxmox LXC containers, not VMs.

## Why `command: systemctl` instead of the `systemd` module

The Ansible `systemd` module fails to enumerate units inside Proxmox LXC
containers, so service enablement shells out to `systemctl enable --now`
directly. The pattern lives once in `roles/common/tasks/systemd_enable.yml`
(change detection: `'Created symlink' in stderr`) and is consumed via
`import_role` with a `systemd_unit` var. The matching ansible-lint rule
(`command-instead-of-module`) is skipped in `.ansible-lint` for this reason.

## Why sysctl is a copy + `sysctl -p` + boot unit, not `ansible.posix.sysctl`

The sysctl module's reload can silently fail inside Proxmox LXC containers.
Instead:

1. `roles/common/tasks/sysctl_dropin.yml` writes a drop-in under
   `/etc/sysctl.d/` (never `/etc/sysctl.conf`, to avoid clobbering unrelated
   tunables) and, when it changed, loads just that file with `sysctl -p`.
   Not `sysctl --system`: in an unprivileged LXC it always exits 1, because
   Debian's default drop-ins set host-only keys (`kernel.*`, `fs.protected_*`).
2. `roles/common/tasks/boot_oneshot.yml` installs a oneshot systemd unit
   (`After=network-online.target`) that re-runs `sysctl -p`, because LXC
   containers do not reliably re-apply sysctl.d on restart. Not a `@reboot`
   cron: cron can start before `eth0` is configured.

Do not "modernize" this to the module — it would re-introduce a bug this
repo already fixed.

## Why the Tailscale ULA is re-added at boot

Proxmox regenerates a container's network from its CT config on every
restart. CT 101 uses `ip6=auto` (SLAAC GUA for Tailscale IPv6), so the
static ULA is added at runtime with `ip -6 addr add` — and would vanish on
reboot, silently breaking the advertised IPv6 subnet route. The boot unit
re-adds it (tolerating "File exists") before re-soliciting RAs.

## Why resolv.conf is pinned on the DNS container

Proxmox rewrites `/etc/resolv.conf` in containers it manages. The DNS
container must resolve through itself (`127.0.0.1`), so the playbook pins
the file and creates `/etc/.pve-ignore.resolv.conf`, which tells Proxmox to
leave it alone.

## Why firewall files are touched before templating

pmxcfs (`/etc/pve`) does not support atomic renames, and Ansible's template
module writes to a temp file and renames it. Pre-creating empty `.fw` files
means template only ever overwrites in place. File ownership `root:www-data`
mode `0640` matches what pmxcfs enforces.

## Why LXC provisioning is create-only

`roles/proxmox_lxc` creates missing containers and only *reports* drift on
existing ones, so a playbook run never reconfigures a live container.
Drift is surfaced as an ignored failure; reconcile it deliberately, either in
`group_vars` or with `pct set`.

Raw `lxc.*` lines are appended with `printf >>` before first start, because
`pct` cannot set them and pmxcfs (see above) breaks `lineinfile`.

## Why the GPU uses `dev0` instead of a udev rule

Proxmox's native device passthrough (`devN`, PVE 8.2+) creates
`/dev/dri/renderD128` inside the container owned by the given gid, mode
`0660`. Jellyfin joins that group via `group_add: jellyfin_render_gid`. The
older approach — raw `lxc.cgroup2`/`lxc.mount.entry` lines plus a host udev
rule making the node `0666` — exposed the GPU to every host process;
`proxmox_host` removes that rule.

## Why third-party apt repos are deb822 with pinned keys

Docker, NextDNS and Tailscale install through one shared task
(`roles/common/tasks/apt_repository.yml`) using `deb822_repository` with
`signed_by`, so each repository is pinned to its own key under
`/etc/apt/keyrings/`. This replaces the vendors' `curl | sh` installers,
which ran unaudited code as root; the task also removes the `.list` files
those installers left behind.

## Why the Proxmox host relays mail

Proxmox sends backup, ZFS and SMART notifications through its local Postfix,
which by default delivers straight to the recipient's MX on port 25. ISPs
commonly block outbound port 25, and here every message sat deferred in the
queue — failures would have gone unseen. `proxmox_host` points Postfix at
the same authenticated SMTP account the containers use (`mail_*`), forcing
IPv4 because the host has no IPv6 route to the internet.

## Why the LXC pre-start NFS hook exists

On Proxmox boot, `pvestatd` (which mounts NFS storage) races
`pve-guests.service` (which starts containers). An LXC bind mount (`mp:`)
captures whatever is at the host path at start time — an empty directory if
NFS isn't mounted yet — which poisons qBittorrent state and Jellyfin libraries.
`wait-for-nfs.sh` is wired as a `pct` hookscript and blocks container start
until the listed NFS paths are mounted, failing the start (loudly) on
timeout instead of booting with empty mounts.

The same race is why `roles/docker_host` hard-fails when `findmnt` shows a
required bind mount missing inside the container.

## Why `flush_handlers` sits inside the docker_host role

A `daemon.json` change must restart Docker *before* compose stacks start,
or containers come up under the old daemon config. Handlers normally run at
end of play — after `compose_stack` — so `docker_host` flushes explicitly
right after deploying `daemon.json`. Keep that ordering if you reorganize
the roles.

## Compose stacks: data-driven, no restart handlers

`roles/compose_stack` loops over the `compose_stacks` list (defaults
reference the per-service variables in `group_vars`). Each stack is:
directory → leaf-ownership config dir → rendered template →
`community.docker.docker_compose_v2`. The module reconciles against the
rendered file, so containers are recreated only when the compose file
actually changed — no `--force-recreate` handlers, no stderr-regex change
detection. Compose startup waits for every service to be running or healthy;
host-side endpoint probes then verify that each published HTTP path is
reachable through Docker's port mapping. Jellyfin and Dockge inherit their
image healthchecks, while qBittorrent and Speedtest Tracker define explicit
checks in their Compose templates.

Docker health status is detection, not auto-healing: `restart: unless-stopped`
restarts exited containers but does not restart an `unhealthy` container.
Avoid blind auto-restart loops for NFS-backed services; runtime health alerts
belong in an external monitor that can also detect the Docker host being down.

Config-directory ownership is set on the leaf directory only; the container
manages files inside with its PUID/PGID. For a one-shot recursive reset
(e.g. after restoring a backup) run:

```bash
ansible-playbook deploy-docker.yml --tags fix-permissions
```

## Dockge and Ansible both write stack files

Dockge edits files under `/opt/stacks/<stack>/compose.yml`; so does Ansible.
**Ansible is the source of truth** — anything changed through the Dockge UI
is reverted on the next playbook run. Use Dockge for start/stop/logs, make
config changes here.

## Why update-all.yml has no reboot handling

LXC containers share the host kernel. `apt dist-upgrade` inside a container
never requires a container reboot; only host kernel updates matter, and the
Proxmox host is deliberately excluded from that playbook.

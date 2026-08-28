# Beszel — lab monitoring

Hub: **https://beszel.tedraykov.me** (private — resolves to 192.168.1.100, no DMZ hole).
Credentials on VM 100 at `/opt/stacks/beszel/.admin-credentials` (mode 600).

## Layout

| Piece | Where | How | Why there |
|---|---|---|---|
| Hub | VM 100 | Portainer stack `beszel` (id 10) | must reach into the trusted zone; DMZ `OUT DROP` makes VM 200 blind |
| Agent | VM 100 | same stack, unix socket to the hub | no TCP port opened at all |
| Agent | Proxmox host | **systemd binary** `/opt/beszel-agent` | no Docker on the hypervisor; `EXTRA_FILESYSTEMS` is binary-only |
| Agent | VM 200 | plain compose, `/opt/stacks/beszel` | Dokploy host, staged like its other stacks |

All pinned to **0.18.8**. The host agent was installed with `--auto-update=false`
so it does not drift out of lockstep with the containers.

## The VM 200 direction choice

Beszel's default is WebSocket: the agent dials *out* to the hub. For VM 200 that
would mean a DMZ -> trusted outbound hole, which is the one thing the two-zone
design exists to prevent. So that agent runs in **SSH mode** (no `HUB_URL`) and
the hub connects *in* to 45876 — trusted -> DMZ, a direction already allowed.

One rule in `/etc/pve/firewall/200.fw` (backup `/root/200.fw.bak-prebeszel`):

    IN ACCEPT -source 192.168.1.100 -p tcp -dport 45876

Safe after the `GROUP` line because `dmz-guest` carries no `IN DROP` — same
reasoning as the existing Dokploy `:3000` rule.

## Host agent config

`/etc/systemd/system/beszel-agent.service.d/override.conf`:

- `FILESYSTEM=nvme0n1` — the agent otherwise picks the *busiest* device for root
  I/O and had chosen `zd32`, which is VM 200's zvol.
- `EXTRA_FILESYSTEMS=/srv/immich,/srv/media,/mnt/archive` — these live on `tank`,
  **outside every VM**, so `vzdump` does not cover them. Capacity here is the
  number that matters most.
- `AmbientCapabilities=CAP_SYS_RAWIO` for SMART.

## Known gaps

- **SATA SMART does not report.** The `tank` disks sit behind a SAT layer and need
  `smartctl -d sat`; the agent has no way to pass a device type. NVMe SMART works.
  Watch mirror health with `zpool status` instead — that is the better signal anyway.
- **No notification channel is configured.** The 12 alerts fire in the UI only
  until SMTP or a webhook is set under Settings -> Notifications.
- ZFS datasets are not block devices, so the extra filesystems report **capacity
  but not I/O** ("Device not found in diskstats" is expected, not a fault).

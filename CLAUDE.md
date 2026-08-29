# Home lab — Proxmox host, and this repo

This repository holds the Docker Compose definitions for the home lab. The lab
runs on a single Proxmox VE 9 host; the migration off the old `sof1` box is
**complete** and the public cutover is **done**.

**Full original plan:** https://claude.ai/code/artifact/4c133e3e-75df-408b-910a-5cca9872d9d0
(fetch with WebFetch if you need detail). Much of it is now historical — where
the plan and this file disagree, **this file is what was actually built.**

Last verified against live infrastructure: **2026-08-29**.

## Host access

| | |
|---|---|
| Proxmox host | `root@192.168.1.26` |
| Web UI | `https://192.168.1.26:8006` |
| VM 100 (homelab) | `teo@192.168.1.100` |
| VM 200 (dokploy) | `teo@192.168.1.200` |
| LXC 300 (dns) | `192.168.1.53` |
| SSH | key-based everywhere, no password prompt |
| Display attached | **none, ever** — network access only |

Run host commands with `ssh root@192.168.1.26 '<command>'`. There is no local
console, so a change that breaks boot is a change that cannot be diagnosed.
Treat bootloader, initramfs and network edits accordingly.

Docker on VM 200 needs `sudo` (the `teo` user is not in the `docker` group
there, unlike VM 100).

## Hardware

- Ryzen 5 5600 — 6c/12t, **no integrated graphics**
- 32 GB DDR4-3000 — the binding constraint on everything
- RTX 3060 12 GB — passed through whole to VM 100
- 1 × 1 TB NVMe (`S7HDNS0L300359N`) — `rpool`, single vdev, no redundancy
- 2 × 2 TB SATA HDD — `tank`, **ZFS mirror**
  (`ST2000DM008 WFLA253K` + `ST2000LM015 ZDZQGL87`)

## Guests

| Guest | Spec | Zone | Runs |
|---|---|---|---|
| VM 100 `homelab` | 6 vCPU, **10 GB pinned**, GPU | trusted | Docker: Jellyfin, Immich, Home Assistant + Zigbee, the \*arr stack, Traefik, Portainer, Beszel |
| VM 200 `dokploy` | 4 vCPU, **12 GB max / 6 GB balloon floor** | DMZ | Dokploy PaaS — Plane, MinIO, excalidash, plus tenants. The only VM the router forwards to. |
| LXC 300 `dns` | 1 vCPU, 512 MB | trusted | AdGuard Home — split-horizon DNS + ad blocking |

## Memory — rebalanced 2026-08-29

VM 100 was over-provisioned at 16 GB while using under 5, which starved VM 200:
its balloon was pinned at its 4 GB floor with **953 MB swapped out**, before any
tenant had deployed anything. Reallocated:

| | Before | After |
|---|---|---|
| VM 100 | 16384, balloon 0 | **10240**, balloon 0 |
| VM 200 | 8192, balloon 4096 | **12288**, balloon **6144** |

Worst-case commit: ARC 3.1 + host ~2 + LXC 0.5 + VM 100 10 + VM 200 12 =
**27.6 of 31 GiB**, ~3.4 GiB slack. After the change the host went from 4.9 GiB
free to ~10 GiB, VM 200 ballooned to 7.5 GiB on its own, and **swap use dropped
to zero**.

**VFIO pins VM 100's full allocation — ballooning must stay off there**, so its
memory is consumed whether used or not. That is why over-provisioning it costs
VM 200 directly. Changing VM 100's memory requires a **shutdown**, not a reboot.

Do not add a guest without taking RAM from another.

**LXC 300 is DNS, not Proxmox Backup Server.** PBS was never built — there was
no RAM for it, and `vzdump` needs none. Older notes calling 300 "PBS" are wrong.

---

# This repository

One directory per stack, each holding a compose file. **`.env` files are
gitignored** (`**/.env`) — secrets never land here.

## Path convention — the rule that matters

| What | Where | Why |
|---|---|---|
| App config, databases, small state | **`/opt/stacks/<stack>/<subdir>`** | VM 100's own zvol (ext4 on NVMe). Proper block semantics for SQLite/Postgres; covered by `vzdump`. |
| Bulk media, downloads | **`/srv/media`** | ZFS `tank/media`, virtiofs into the VM |
| Photo library | **`/srv/immich`** | ZFS `tank/immich`, virtiofs into the VM |

The old `/mnt/drive1/{config,data}/*` paths are **dead** and were removed from
every compose file on 2026-08-29. That drive is now inside this host and its
contents live at `/mnt/archive` (see Storage).

**Never put a database on virtiofs.** SQLite and Postgres over virtiofs risk
locking corruption. That is the whole reason configs live on `/opt/stacks` and
not `/srv/appdata`. This applies to `rpool/postgres` too — it exists but is
deliberately not shared into any VM.

**`/srv/*` paths are pool-neutral on purpose.** The pool name must never leak
into a bind mount or directory mapping. That is exactly what made swapping the
backing store from an ext4 disk to a ZFS mirror a two-line change with no
consumer edits.

## Stacks in this repo

| Stack | Status | Notes |
|---|---|---|
| `homeassistant` | **live on VM 100** | HA, mosquitto, zigbee2mqtt, hass-configurator. `nodered` is behind `profiles: ["optional"]` — defined on `sof1` but never ran. |
| `jellyfin` | **live on VM 100** | NVENC verified |
| `immich` | **live on VM 100** | v1.120.2 pinned |
| `servarr` | **live on VM 100** | sonarr, radarr, prowlarr, bazarr, qbittorrent, overseerr, flaresolverr |
| `traefik` | **live on VM 100** | v3.7, wildcard cert |
| `portainer` | **live on VM 100** | deployed from `/opt/stacks/portainer/compose.yaml`, not via Portainer itself |
| `excalidraw` | live on **VM 200** | container/data are named `excalidash` |
| `n8n` | **not running anywhere** | slated for VM 100 |
| `minecraft` `monitoring` `plex` | **dormant** | not deployed. Data still at `/mnt/archive/config/*` and `/mnt/archive/data/*` if wanted. |

**`minio` runs on VM 200 but is no longer in this repo** — it was pruned here
on 2026-08-29 while still live. Its definition survives only in VM 200's own
`/opt/stacks/minio/`. Recover from this repo's git history if it moves to
VM 100.

Removed 2026-08-29 as unwanted: `nextcloud`, `roberts50`, `arbitrage`, `zrok`,
`pelias`, `wordpress`, `dnsmasq`, then `kafka`, `minio`, `prefect`,
`structurizr`, `uptime`, `utorrent`, `perfume-app`. All recoverable from git
history. (`dnsmasq` is genuinely obsolete — AdGuard in LXC 300 does that job
now; `plex` and `utorrent` were superseded by `jellyfin` and `qbittorrent`.)

Several dormant stacks still carry `*.treble.bg` Traefik labels and
`certresolver=tlsresolver` from the `sof1` era. The live domain is
`tedraykov.me` and the VM 100 resolver is named `letsencrypt`. **Fix the labels
before deploying any of them.**

## How this repo reaches the running containers

Not GitOps — there is no webhook and no remote pull. **Portainer stacks 7, 8
and 9 are uploaded copies of this entire repository**, and each deploys one
subdirectory's compose file:

| Portainer stack | Deploys | Config file |
|---|---|---|
| 7 | jellyfin | `/data/compose/7/jellyfin/docker-compose.yml` |
| 8 | servarr | `/data/compose/8/servarr/docker-compose.yml` |
| 9 | homeassistant | `/data/compose/9/homeassistant/docker-compose.yaml` |
| 1 | immich | standalone editor stack |
| 6 | traefik | standalone editor stack |
| 10 | beszel | standalone editor stack |

So **editing a file here changes nothing until the stack is re-uploaded or
edited in Portainer.** Stacks 1, 6 and 10 have no copy here at all — their
definitions live only in Portainer's own volume.

VM 200 keeps its own separate git repo at `/opt/stacks/` on that VM
(`excalidash`, `minio`, `n8n`, `beszel`), with data under
`/opt/appdata/<svc>` and secrets in `/opt/appdata/env/*.env` (mode 600).

---

# Storage — as built

## The `tank` mirror (built 2026-08-27)

The second HDD arrived and `tank` is a **true two-disk mirror**, resilvered
726 G with no errors. This resolved the long-standing "photos exist in exactly
one place" risk — they are now on redundant disks.

| Dataset | Mountpoint | Holds |
|---|---|---|
| `tank/media` | `/srv/media` | 588 G — media + torrents |
| `tank/immich` | `/srv/immich` | 112 G — photo library |
| `tank/archive` | `/mnt/archive` | 80 G — **the old `sof1` drive contents, verbatim** |
| `rpool/appdata` | `/srv/appdata` | scratch/staging only |
| `rpool/ROOT/pve-1` | `/` | host OS, `reservation=32G` |

`rpool/media` and `rpool/immich` were set `mountpoint=none` — they were empty
placeholders. `rpool/postgres` exists and is deliberately unused.

`/etc/fstab` is back to stock (just `proc`) — the ext4 binds are gone, ZFS
mounts these itself. Old backup: `/root/fstab.bak-predrive`.

## `/mnt/archive` — the rollback and the dormant-service data

This is the entire former `/mnt/drive1` tree from `sof1`, copied onto the
mirror before the source disk was wiped and added to it. It holds:

- `config/` — plex (119 M), prefect3 (276 M), prometheus (65 M), grafana, loki
  (824 M), kafka, zookeeper, structurizr, utorrent, promtail, and the old
  jellyfin/hass/\*arr configs that were already migrated
- `data/` — minecraft (264 M), uptime-kuma (6.5 M), minio, n8n, excalidash,
  whisper (805 M), piper (708 M), pelias (3.8 G)
- `backups/proxmox/` — the vzdump target
- `migration/` — `immich_pgdata.tar`, `portainer_data.tar`

**This is where a dormant stack's data comes from if you deploy it.** It is
also the migration rollback. Nothing here is in active use.

## ARC

Capped at 3357540352 (3.13 GiB) in `/etc/modprobe.d/zfs.conf`, set by the PVE 9
installer at 10% of RAM. `zfs_arc_min` was never set.

## Backups

- Storage **`hdd-backup`** → `/mnt/archive/backups/proxmox`, with
  `is_mountpoint /mnt/archive` so a missing pool can never silently fill root.
- Job `backup-all`: **twice daily, 02:30 and 14:30**, all guests, `mode
  snapshot`, zstd, retention `keep-daily=7,keep-weekly=4,keep-monthly=3`.
- Throughput ~250–350 MB/s. VM 200 dumps ~4–5 G each.

### ⚠ The remaining backup gap

VM backups cover the **VMs**. Media and photos live on `tank`, outside any VM,
so `vzdump` does not touch them. The mirror protects against a *drive* failure;
it does **not** protect against fire, theft, ransomware, or `zfs destroy`.

**There is still no off-site copy of the photo library.** This is now the
single largest risk in the build. Options: `zfs send` to an external disk kept
elsewhere, or a cloud target (restic/rclone to B2 or S3).

---

# Network

Everything is one flat `192.168.1.0/24`. Gateway `192.168.1.1`.

**VLANs are off the table.** The router is a Huawei **HG8145X6-10**, an
ISP-supplied GPON ONT: DHCP, port forwarding and WiFi only — no 802.1Q per
port, no inter-VLAN ACLs, and no second switch. `vmbr0` is a plain bridge and
should stay one; editing `/etc/network/interfaces` on a console-less host is
the riskiest routine change available.

**Why the router's firewall is irrelevant:** two VMs on the same subnet are
switched by `vmbr0` *inside the host*. That traffic never reaches the cable, so
the router cannot filter it. Isolation happens at the hypervisor or not at all.

## Fixed addresses

| Host | IP |
|---|---|
| Proxmox host | `192.168.1.26` |
| VM 100 homelab | `192.168.1.100` |
| VM 200 dokploy | `192.168.1.200` |
| LXC 300 dns | `192.168.1.53` |
| Public IPv4 | `83.228.116.18` |

Matching the VMID is deliberate — the firewall IPSets depend on these.

## Router forwards — **already flipped**

- TCP **80** and **443** → `192.168.1.200` ✅ done
- UDP **51820** → WireGuard — **not yet set up**

## Split-horizon DNS — LXC 300, AdGuard Home

This is what makes the lab reachable by name from inside the house without
relying on NAT hairpin (which the Huawei ONT was never confirmed to do).

- Upstream: `https://dns10.quad9.net/dns-query` (DoH), bootstrap `9.9.9.10`
- **`*.tedraykov.me` → `192.168.1.100`** (wildcard, VM 100's Traefik)
- Per-name overrides → `192.168.1.200` for the DMZ apps:
  `n8n`, `minio`, `minio-console`, `excalidraw`, `plane`
  (backup before edits: `/opt/AdGuardHome/AdGuardHome.yaml.bak-preperfume`)

So a LAN client goes straight to the serving VM; the public internet goes
through the router forward to VM 200. **When a service moves between VMs, the
AdGuard rewrite must move with it** — otherwise LAN clients hit the wrong box.

For this to apply to a device, the router's DHCP must hand out `192.168.1.53`
as the resolver.

## Proxmox firewall — the enforcement point

`/etc/pve/firewall/cluster.fw`, enabled cluster-wide with `policy_in: ACCEPT` /
`policy_out: ACCEPT`. The **host is deliberately left open** — all isolation is
per-guest, which is what keeps a firewall mistake from locking out a box with
no console.

| IPSet | Members |
|---|---|
| `lan` | `192.168.1.0/24` |
| `dmz` | `192.168.1.200` |
| `trusted` | `192.168.1.100` |

| Security group | Rules |
|---|---|
| `dmz-guest` | IN accept 80, 443 from anywhere; IN accept 22 + icmp from `+dc/lan`; **OUT DROP to `+dc/lan`, logged** |
| `trusted-guest` | IN DROP from `+dc/dmz`, logged |

`OUT DROP -dest +dc/lan` replaces the VLAN deny rule. It stops VM 200 reaching
VM 100 *and* your laptop, TV and IoT gear in one rule, at the tap device,
before the bridge can switch the packet.

Inbound SSH from the LAN still works — Proxmox accepts ESTABLISHED/RELATED
before the guest chains, so the outbound deny does not break return traffic on
sessions the LAN opened.

### Attaching the policy to a new guest

Two things, and the group does nothing without both:

```
# 1. /etc/pve/firewall/<vmid>.fw
[OPTIONS]
enable: 1
policy_in: DROP
policy_out: ACCEPT

[RULES]
GROUP dmz-guest

# 2. arm the firewall on the NIC itself
qm set <vmid> -net0 virtio,bridge=vmbr0,firewall=1
```

---

# Public routing — how a service becomes public

80/443 forward to **VM 200**, whose Dokploy Traefik reverse-proxies back to
VM 100. Forwarding straight to VM 100 was rejected: this way, if Traefik is
ever compromised, the attacker lands on the **disposable** VM rather than the
one holding the photo library and Home Assistant.

Making a VM 100 service public takes **three** pieces. Any one missing and it
silently fails:

1. `/etc/pve/firewall/200.fw` — `OUT ACCEPT -dest 192.168.1.100 -p tcp -dport <port>`
2. `/etc/pve/firewall/100.fw` — `IN ACCEPT -source +dc/dmz -p tcp -dport <port>`
3. A router pair in `/etc/dokploy/traefik/dynamic/vm100.yml` on VM 200

**Both firewall rules must precede the `GROUP` line.** Proxmox evaluates rules
in order and the group expands inline, so a rule after it is never reached.

`vm100.yml` defines **two routers per service**: `<name>-http` on `web` (plain
HTTP — used for Host-header verification and as the ACME HTTP-01 path) and
`<name>` on `websecure` with `certResolver: letsencrypt`. **No `tls.domains`
anywhere** — that would make Traefik attempt issuance eagerly at startup.

Backups of the firewall files: `/root/{100,200}.fw.bak*` on the host.

## Live public services — all verified 2026-08-29

Certificates issued via **HTTP-01** on the DMZ Traefik, `ssl_verify_result=0`
on every one:

| Name | Port on VM 100 | Name | Port |
|---|---|---|---|
| `jellyfin` | 8096 | `sonarr` | 8989 |
| `photos` (immich) | 2283 | `radarr` | 7878 |
| `ha` | 8123 | `prowlarr` | 9696 |
| `hass-configurator` | 3218 | `bazarr` | 6767 |
| `zigbee2mqtt` | 8080 | `qbittorrent` | 8084 |
| `overseerr` | 5055 | `portainer` | 9443 (https, insecureSkipVerify) |

Plus **`plane`**, which runs on VM 200 itself via Dokploy.

The DMZ→VM 100 holes are an allowlist of exactly the ports that are published
to the internet anyway — coherent, and **nothing else**. Verified from inside
VM 200: SSH, the Proxmox UI, the router, and every other port on VM 100 are
blocked.

## Deliberately NOT public

- **Dokploy's control plane** (VM 200 `:3000`) — LAN/VPN only, via a per-guest
  rule. It can deploy arbitrary code and read every stored secret.
  **Never port-forward 3000.**
- **Both Traefik dashboards.** `sof1` ran `--api.insecure=true` with
  `traefik.tedraykov.me` public and unauthenticated. Not restoring that.

## Staged but not public

`n8n`, `minio`, `minio-console`, `excalidraw` have DNS pointing at
the public IP but **no certificate and no `websecure` router** — their Traefik
labels are still `entrypoints=web` only. HTTPS to them fails with TLS error 18.
They resolve correctly on the LAN via AdGuard.

## ⚠ Home Assistant behind a proxy — `.storage/http` overrides YAML

HA returned **400 Bad Request** to proxied requests. Adding an `http:` block to
`configuration.yaml` did **nothing** — HA (2026.8.3) migrated http config into
**`/config/.storage/http`**, marked `"yaml_migration_done": true`, and that
file wins.

`trusted_proxies` must list **both** proxy paths:
`["172.16.0.0/12", "192.168.1.200", "127.0.0.1", "::1"]`

- `172.16.0.0/12` — VM 100's own Traefik, seen as a Docker gateway
- `192.168.1.200` — the DMZ Traefik, seen as a real LAN address, because
  Docker's DNAT preserves the source for connections from another host

**If HA ignores a YAML setting, check `.storage/` before debugging the YAML.**
Backup: `.storage/http.bak-sof1`.

---

# VM 100 — homelab

Debian 13 trixie. `qm` id 100, starts on boot.

| Setting | Value | Why |
|---|---|---|
| machine / bios | `q35` / `ovmf` | required for PCIe passthrough |
| efidisk0 | `efitype=4m,pre-enrolled-keys=0` | **guest Secure Boot OFF** — NVIDIA DKMS is self-signed with an unenrolled MOK |
| cpu / cores | `host` / 6 | |
| memory / balloon | 10240 / **0** | VFIO pins the full allocation |
| hostpci0 | `0000:05:00,pcie=1` | whole card, both functions. No `x-vga=1`, so noVNC still works |
| scsi0 | `local-zfs` 100G, discard, iothread, ssd | |
| net0 | `virtio,bridge=vmbr0,firewall=1` | `firewall=1` required or the group is ignored |
| usb0 | `host=10c4:ea60` | Sonoff ZBDongle-P (Silicon Labs CP210x) |
| virtiofs0/1/2 | `dirid=immich` / `media` / `appdata` | |

Firewall: `GROUP trusted-guest`, `policy_in: ACCEPT` — reachable from the LAN
by design; only the DMZ is restricted.

## virtiofs

Host `/srv/{immich,media,appdata}` appear at the **same paths** in the guest via
`/etc/fstab` lines like `immich /srv/immich virtiofs defaults 0 0`. The tag is
the mapping id from `/cluster/mapping/dir`.

**UID behaviour, tested not assumed:** host dirs are `1000:1000` and guest
`teo` is uid 1000, so writes pass through cleanly. A container running as
**root** writes through and the file lands root-owned on the host — so Immich
runs as root exactly as it did on `sof1`, no `user:` override needed. Match
`1000:1000` only when writing into existing 1000-owned trees.

**Hardlinks work through virtiofs** — verified same-inode, `links=2` between
`/data/torrents` and `/data/media`. The whole \*arr import pipeline depends on
this. **Do not split those two across devices.**

## GPU

Driver **`nvidia-open` 610.57.04** from **NVIDIA's CUDA repo**
(`developer.download.nvidia.com/compute/cuda/repos/debian13`), DKMS, open
modules, signed via MOK.

**Debian's `nvidia-driver` was purged** — it caps at 550 even in backports, and
Sunshine's ffmpeg needs NVENC API 13 / driver ≥570. Consequences:

- `apt upgrade` now pulls NVIDIA-repo driver updates. Watch for DKMS rebuilds
  against new kernels; `apt-mark hold` if you want it frozen.
- `nvidia-container-toolkit` was collaterally removed by the `libnvidia*` purge
  and reinstalled + `nvidia-ctk runtime configure` re-run. **If Docker GPU
  breaks after a driver change, reinstall it first.**

Jellyfin NVENC verified end to end: full NVDEC→NVENC on 10-bit HEVC → 720p
H.264, both engines engaged, 1.79× realtime. `sof1` was CPU-only. GPU reaches
the container via `deploy.resources.reservations.devices` +
`NVIDIA_DRIVER_CAPABILITIES=compute,video,utility`. No `/dev/dri` needed.

## Generic kernel, not cloud — and the trap that cost a rebuild

VM 100 runs the **generic** kernel (`linux-image-amd64`). The Debian **cloud**
kernel is stripped and **has no `uinput`**, so Sunshine cannot inject input and
Moonlight shows an uncontrollable screen.

**Do NOT `apt purge` the cloud kernel while installing generic.** GRUB's
top-level "simple" entry kept pointing at the cloud kernel; deleting it left
GRUB loading a missing kernel → **unbootable, no serial, no console** (the GPU
has the only display). Recovery was a snapshot rollback.

The safe setup, now in place:
- Keep **both** kernels installed; cloud stays as fallback.
- `GRUB_DEFAULT` in `/etc/default/grub` is the **explicit generic entry**
  (`gnulinux-advanced-<uuid>>gnulinux-6.12.101+deb13-amd64-advanced-<uuid>`),
  then `update-grub`. `grub-reboot` one-shots did **not** stick here.

## Remote desktop — Sunshine + XFCE

GPU-streamed desktop reached with a **Moonlight** client: NvFBC capture +
NVENC. Web UI / pairing at https://192.168.1.100:47990. Ports 47984/47989/47990/
48010 TCP + 47998-48000 UDP, LAN-only.

Non-obvious wiring:
- **No display manager.** `nvidia_drm.ko` has no `modeset` parameter, so KMS
  can't be enabled → logind reports `CanGraphical=no` → lightdm/GDM won't start
  X, and `/dev/dri` is absent. So DRM is bypassed and Xorg runs directly.
- **`desktop-x.service`** (`/usr/local/bin/homelab-desktop.sh`) runs **as
  root** — the setuid Xorg wrapper forbids absolute `-config`/`-logfile` when
  elevated. It starts `Xorg :0`, then `xfce4-session` + Sunshine **as teo**.
- **Virtual display** in `/etc/X11/xorg.conf`: nvidia driver, `BusID PCI:1:0:0`,
  `ConnectedMonitor DFP-0`, relaxed `ModeValidation`, CVT ModeLine for
  **3456x2234@60** (MacBook Pro 16" M4 panel), plus 1728x1117 and 1920x1080.
  XFCE at 2x HiDPI (`Gdk/WindowScalingFactor=2`, `Xft/DPI=192`).
  `xorg.conf.bak-1080p` is the old config.
- **Input injection needs two things**, both initially missing: `uinput` loaded
  via `/etc/modules-load.d/uinput.conf` (teo in the `input` group), and
  **`AutoAddDevices "true"`** in a `ServerFlags` section — an explicit
  ServerLayout otherwise disables hotplug so Sunshine's virtual devices never
  attach. Working = `xinput list` shows "Mouse/Keyboard passthrough".
- **Audio:** null-sink in `/etc/pulse/default.pa.d/sunshine-sink.pa`.

## Installed

Docker CE 29.7.2 + Compose v5.5.0, `nvidia-container-toolkit` 1.20.0,
`qemu-guest-agent`, `rsync`. `teo` is in the `docker` group.

## Portainer

**2.39.6** at https://192.168.1.100:9443, endpoint id 1 (`local`). Admin
credentials in `/opt/stacks/portainer/.admin-credentials`, mode 600.

Gotcha for a fresh install: initialisation **times out** if no admin account is
created within 5 minutes of first start. Fix is to restart and POST to
`/api/users/admin/init` with the `X-Setup-Token` from the logs — the token is
printed with ANSI colour codes around it, so strip them or the grep silently
returns nothing.

## Snapshots

`pre-desktop`, `pre-driver-upgrade`, `desktop-working`. The last is the good
checkpoint (generic kernel, driver 610, Sunshine + input working). Delete the
first two once the desktop is confirmed stable.

---

# VM 200 — Dokploy DMZ

**Ubuntu 24.04 LTS**, `qm` id 200. Ubuntu rather than Debian 13 deliberately —
Dokploy's supported list stops at Debian 12, and the internet-facing VM is the
wrong place to find out whether 13 works.

| Setting | Value |
|---|---|
| bios / machine | seabios / i440fx — no passthrough, no reason to add OVMF |
| cores / memory | 4 / 12288, **balloon 6144** — nothing is pinned here |
| scsi0 | `local-zfs` 150G |
| ipconfig0 | `192.168.1.200/24`, gw `192.168.1.1` |
| nameserver | **`1.1.1.1`** — the router's resolver is unreachable by design |

Swap 8 GB, `vm.swappiness=10` — Docker builds are memory-spiky and the OOM
killer taking out Traefik is a bad afternoon. `unattended-upgrades` on, SSH
keys-only.

## Deliberately NOT done: ufw inside the guest

- Docker and Swarm write their own iptables rules and publish ports **past**
  ufw via `DOCKER-USER`. ufw on a Docker host reads as protection it isn't.
- The Proxmox firewall is strictly stronger: it filters at the **tap device,
  outside the guest**, so nothing Docker does can punch through.

If it's ever wanted, it must be configured against `DOCKER-USER`.

## Dokploy

**v0.30.2**, Docker Swarm single node. Traefik **v3.6.7** (VM 100 runs v3.7).
UI at http://192.168.1.200:3000 — **admin account exists**, Plane is deployed
through it. ACME email `business@tedraykov.me` in
`/etc/dokploy/traefik/traefik.yml`; resolver `letsencrypt`, **HTTP-01** on the
`web` entrypoint.

## Running on VM 200

| Service | State |
|---|---|
| Plane v1.3.1 (13 containers) | **up, public at `plane.tedraykov.me`** |
| minio | up, HTTP-only, not public |
| excalidash frontend/backend | up, HTTP-only, not public |
| n8n | **not deployed** |
| beszel-agent | up |

`excalidash-backend` reports **unhealthy** — pre-existing, and true on `sof1`
too. Its healthcheck curls `/health` over localhost HTTP but `TRUST_PROXY=1` +
`FRONTEND_URL` makes the app 302 it to HTTPS, so it can never return 200. The
app serves correctly. Not worth fixing.

## Parallel-run safety

`n8n` is **cutover-only** — its workflows are *active* (`Домоуправител`,
`Домоуправител — разнасяне на вноски`, `OLX Watcher`) and would fire twice if
two instances ran. Start it in exactly one place.

## Dokploy multi-tenancy — verified against the v0.30.2 source

**Supported:** Organization → Project → Environment → Service. Per-member flags
in `packages/server/src/db/schema/user.ts`: `canCreateProjects`,
`canCreateServices`, `canDeleteProjects`, `canDeleteServices`,
`canAccessToGitProviders`, `canAccessToDocker`, `canAccessToTraefikFiles`,
`canAccessToAPI`, `canAccessToSSHKeys`. Members can be scoped to specific
Projects/Environments and can connect their own GitHub.

**Not supported:**
- **No self-registration.** `register.tsx` redirects once an admin exists — on
  self-hosted it is one-time server setup. **The owner must invite everyone.**
- **No user-created organizations** — `/organization/create` is in
  `disabledPaths` in `packages/server/src/lib/auth.ts`. One org, forever.
- **Custom roles are Enterprise (paid).** Self-hosted gets Owner/Admin/Member.
- Social login needs `GITHUB_CLIENT_ID`/`SECRET`, not set here.

### The security reality — accepted, with a condition

**Anyone who can deploy code on Dokploy effectively has root on VM 200.**
Builds run as root; a compose with `privileged: true` or a `docker.sock` mount
is a full takeover. The permission flags gate the **UI**, not the code. There
is no sandbox between tenants — one daemon, one Swarm.

The DMZ bounds the blast radius, and the plan always labelled VM 200 *"untrusted
code, treat as hostile"*. **But VM 200 still stores** MinIO root credentials, excalidash's JWT/CSRF
secrets, and n8n's staged credential store at `/opt/appdata/n8n` — data left
behind is as readable as data in use. perfume-app's Anthropic key and Turso
token were **removed on 2026-08-29** along with its orphaned containers; both
should still be rotated at source, since deletion does not undo prior exposure.

**Decision: accepted — invite only people you actually know.** If that stops
being true, move your own apps to VM 100 behind the same per-port allowlist.

Tenant onboarding: invite by email → they set a password → grant
`canCreateProjects`, `canCreateServices`, `canAccessToGitProviders` → scope to
their Project. **Never grant** `canAccessToDocker`, `canAccessToTraefikFiles`
or `canAccessToSSHKeys` — those hand over the rest of the VM.

---

# Boot & passthrough facts (verified on the host)

- **UEFI + GRUB**, managed by `proxmox-boot-tool` (ESP `2E2A-7A9C`). Kernel
  cmdline edits go in `/etc/default/grub` then `proxmox-boot-tool refresh`.
  **`/etc/kernel/cmdline` is a no-op here** — it only drives systemd-boot.
  Editing the wrong one is a silent failure on a box with no console.
- **Secure Boot is ENABLED** — `shim → GRUB` (`\EFI\PROXMOX\SHIMX64.EFI`). That
  is *why* the installer chose GRUB. Host passthrough is unaffected: `vfio-pci`
  and `vfio_iommu_type1` are in-tree and signed.
- **GRUB never reads ZFS** — `proxmox-boot-tool` copies kernel + initrd to the
  FAT32 ESP, so pool feature flags are irrelevant to booting.
- **IOMMU is ON** (BIOS: ASRock B550M Pro4 → Advanced → AMD CBS → NBIO Common
  Options → IOMMU = Enabled, plus Above 4G Decoding). 16 groups, **interrupt
  remapping enabled**, so no `allow_unsafe_interrupts`.
- **No kernel cmdline change was needed.** AMD-Vi came up once the firmware
  published IVRS. `/etc/default/grub` is untouched, deliberately.
- **The GPU is alone in IOMMU group 2** with only its own bridges. **No ACS
  override.** Group 0 fuses NVMe + SATA + USB + NIC — unusable, but not needed.
- **vfio config**, all baked into the initramfs:
  - `/etc/modules-load.d/vfio.conf` — `vfio`, `vfio_iommu_type1`, `vfio_pci`
    (`/etc/modules` is deprecated on Debian 13)
  - `/etc/initramfs-tools/modules` — same three, so binding beats userspace
  - `/etc/modprobe.d/vfio.conf` — `options vfio-pci ids=10de:2504,10de:228e disable_vga=1`
  - `/etc/modprobe.d/blacklist-nvidia.conf` — blacklists `nouveau`, `nvidia`,
    `nvidiafb`, `nvidia_drm`, plus `softdep snd_hda_intel pre: vfio-pci`.
    **Deliberately not** a blanket `blacklist snd_hda_intel` — that would kill
    onboard audio (`07:00.4`); the softdep is surgical.
  - `vfio_virqfd` intentionally omitted — merged into `vfio` in kernel 6.2.
  - Rollback snapshot: `rpool/ROOT/pve-1@pre-vfio`.
- **The host has no display output any more.** Console goes black right after
  GRUB. That is correct, not a fault — judge boot success by SSH.
- PVE 9.2.2, kernel 7.0.2-6-pve.

---

# Certificates

## VM 100 — wildcard via Cloudflare DNS-01

Traefik v3.7 serves a real **`*.tedraykov.me` wildcard** from Let's Encrypt via
**Cloudflare DNS-01**. No inbound connection involved — which is exactly why it
works for services that are not internet-reachable. **Adding a subdomain needs
no new issuance.**

Routes live in `/opt/stacks/traefik/dynamic/homelab.yml` as a **file provider**
pointing at `http://192.168.1.100:<port>` — chosen over Docker labels so the
existing stacks did not have to be redeployed onto a shared network. To add a
service: publish its port, add a router+service pair.

## The Cloudflare token — three things that cost time

1. Cloudflare now issues prefixed "scannable" tokens: `cfk_` (Global Key),
   `cfut_` (User token), `cfat_` (**Account** token) — prefix + 40 chars +
   checksum, ~53 chars. **Ours is `cfat_`.** Old unprefixed tokens still work.
2. **`/user/tokens/verify` returns "Invalid API Token" for an ACCOUNT token.**
   That is the wrong endpoint, not a broken token. **Verify by doing the real
   operations**: `GET /zones?name=...` and a TXT create+delete. That is what
   lego does.
3. Required: **`Zone:Zone:Read` AND `Zone:DNS:Edit`**, scoped to the one zone.
   Cloudflare's "Edit zone DNS" template grants only DNS:Edit — **Zone:Read
   must be added by hand** or lego cannot resolve the zone ID.

## How the token is stored — and why not `env_file`

- `/opt/stacks/traefik/.env` and a bare-value copy at
  `/opt/stacks/traefik/cf_token`, both **mode 600**, both gitignored.
- Compose passes **`CF_DNS_API_TOKEN_FILE=/run/secrets/cf_token`** with the
  file bind-mounted in.
- **`env_file` with a host path does NOT work in a Portainer-deployed stack** —
  Portainer runs compose *inside its own container* and cannot read host paths.
  Bind mounts work (they go to the Docker daemon); `env_file` does not. A
  Portainer stack variable would have written the secret into Portainer's DB.
  The `_FILE` route avoids both problems.
- **The token is on VM 100 only. It must never go on VM 200** — DMZ tenants
  deploy arbitrary code there, and this token can rewrite any record in the
  zone, including MX.

## VM 200 — per-name via HTTP-01

13 certs issued in `/etc/dokploy/traefik/dynamic/acme.json`. Requires the port
forward, which is why nothing could issue before the cutover.

---

# Decisions — don't re-litigate without new information

- **Both workloads are VMs**, not bare metal. Rollback matters most on the
  irreplaceable half.
- **Docker in LXC was rejected** — nesting + keyctl + AppArmor concessions
  erode the isolation that was the reason to containerise.
- **Bulk data lives on host ZFS datasets**, virtiofs'd into VM 100, not inside
  the VM disk. Keeps VM backups small; ZFS snapshots protect data.
- **Databases never on virtiofs.** Configs go on VM 100's own zvol.
- **Home Assistant is a container in VM 100**, not its own HAOS VM — there is
  no spare 4 GB at 32 GB of RAM.
- **Everything that had a domain on `sof1` is public again** (2026-08-23). This
  superseded an earlier "split by sensitivity" plan. The DMZ→VM 100 holes are
  exactly the ports that are public anyway.
- **The proxy stays in the DMZ.** Forwarding 80/443 straight to VM 100 was
  rejected — a Traefik compromise should land on the disposable VM.
- **VM 200 keeps its own `/opt/stacks` repo and `/opt/appdata`**, deliberately
  *not* virtiofs from `/srv`: virtiofs is not covered by the Proxmox firewall,
  so mapping host datasets into the DMZ would open the exact path the two-zone
  design exists to prevent.
- **No VLAN-aware bridge.** Revisit only if the ISP router is replaced with
  something that does 802.1Q.

---

# Open items

1. **Off-site backup of the photo library.** The mirror is not a backup. This
   is the largest remaining risk.
2. **Move `n8n`, `minio` and `excalidraw` to VM 100.** Per service: stop on
   VM 200 → copy `/opt/appdata/<svc>` → `/opt/stacks/<svc>/…` on VM 100 → start
   → add a route to VM 100's `homelab.yml` (wildcard cert already covers it) →
   **update the AdGuard rewrite to drop the `.200` override** → add DMZ→VM 100
   firewall holes if they should stay public. n8n is the delicate one: its
   workflows are active, so it must run in exactly one place.
3. **WireGuard** — UDP 51820 forward not yet configured. No remote access
   except through the public services.
5. **Dormant stacks** carry stale `*.treble.bg` labels and `tlsresolver`. Fix
   before deploying any.
6. Dokploy cleanup schedule, per-app memory/CPU limits, S3 database backups.
7. `rpool/immich` holds ~2.2 G of orphaned data from the aborted WiFi rsync.
8. Later: second node + QDevice for quorum. Note the mirror protects against a
   *drive* failure, not a *host* failure.

# Working agreements

- Show me the command and what it will do before running anything destructive
  — `zpool`, `zfs destroy`, `rm`, partitioning, `/etc/network/interfaces`,
  `/etc/kernel/cmdline`, `update-initramfs`.
- Snapshot before risky changes: `zfs snapshot rpool/ROOT/pve-1@pre-<change>`.
- Before any reboot, say what should come back up and how I verify it.
- Prefer reading state (`zpool status`, `pvesh get`, `docker ps`, `ip a`) over
  assuming it. **Read the running container's labels rather than trusting a
  compose file** — that is what caught the stale-stack-93 trap on `sof1`.
- This machine has no console. If a change could affect boot, say so first.

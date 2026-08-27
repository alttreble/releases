# Home lab — Proxmox build & migration

Working context for building out a Proxmox VE 9 host and migrating a home lab
onto it. The full plan lives at the artifact URL below; this file is the
condensed version so a fresh session starts oriented.

**Full plan:** https://claude.ai/code/artifact/4c133e3e-75df-408b-910a-5cca9872d9d0
(fetch it with WebFetch if you need detail beyond what's here — it is readable
with the user's claude.ai login)

## Host access

| | |
|---|---|
| Proxmox host | `root@192.168.1.26` |
| Web UI | `https://192.168.1.26:8006` |
| SSH | key-based, no password prompt |
| Display attached | **none, ever** — network access only |

Run commands with `ssh root@192.168.1.26 '<command>'`. There is no local
console on this machine, so a change that breaks boot is a change that cannot
be diagnosed. Treat bootloader, initramfs and network edits accordingly.

## Hardware

- Ryzen 5 5600 — 6c/12t, **no integrated graphics**
- 32 GB DDR4-3000 — the binding constraint on everything
- RTX 3060 12 GB — passed through whole to the home lab VM
- 1 × 4 TB NVMe — `rpool`, single vdev, **no redundancy** — currently holds everything
- **No HDDs.** They were never purchased. `tank` does not exist and is not
  planned. NVMe-only is the design, not a temporary state. All 6 SATA ports
  are free if that ever changes.

## Target layout

| Guest | Spec | Zone | Runs |
|---|---|---|---|
| VM 100 | 6 vCPU, 16 GB pinned, GPU | trusted | Docker: Home Assistant, Immich, Jellyfin, Whisper, Piper |
| VM 200 | 4 vCPU, 8 GB balloon | DMZ | Dokploy — public PaaS, the only thing with forwarded ports |
| LXC 300 | 2 vCPU, 2 GB | trusted | Proxmox Backup Server |

Memory budget is exact and has ~1 GB of slack: ARC 3 + host 2 + VM100 16 +
VM200 8 + PBS 2 = 31 of 32 GB. Do not add a guest without taking RAM from
another. VFIO pins VM 100's full 16 GB — ballooning must stay off there.

## Decisions already made — don't re-litigate without new information

- **Both workloads are VMs**, not bare metal. Rollback matters most on the
  irreplaceable half (HA config, Immich), and bare metal would have given it
  to the disposable half instead. Revisit only if 16 GB proves too tight or
  the user wants a local LLM.
- **Docker in LXC was rejected** — nesting + keyctl + AppArmor concessions
  erode the isolation that was the reason to containerise.
- **Bulk data lives on host ZFS datasets**, shared into VM 100 over virtiofs,
  not inside the VM disk. Keeps VM backups small; ZFS snapshots protect data.
- **Host paths are `/srv/*`, never `/tank/*` or `/rpool/*`.** The pool name
  must not leak into directory mappings, bind mounts or rsync targets — that
  is what makes moving a dataset between pools a one-command job.
- **Redundancy follows replaceability.** Photos and PBS belong on the HDD
  mirror. **Currently suspended** — see "Storage" below; with no `tank`,
  nothing is redundant and off-site backup is the only protection.
- **Home Assistant runs as a container in VM 100**, not its own HAOS VM —
  there is no spare 4 GB at 32 GB of RAM.
- **Public exposure is port-forward + Traefik.** The deny rule between zones
  is the point of the whole design — but it is now enforced by the **Proxmox
  firewall**, not by VLANs. See "Network" below.

## Storage — as actually built (NVMe-only, 2026-08-22)

The plan's two-tier `rpool` + `tank` split is **not happening** — the HDDs were
never bought. Everything lives on `rpool`. The layout is still pool-neutral, so
adopting a second tier later stays a one-command job per dataset.

| Dataset | Mountpoint | recordsize | compression |
|---|---|---|---|
| `rpool/immich` | `/srv/immich` | 1M | lz4 |
| `rpool/media` | `/srv/media` | 1M | lz4 |
| `rpool/pbs` | `/srv/pbs` | 1M | off |
| `rpool/appdata` | `/srv/appdata` | 128K | lz4 |
| `rpool/postgres` | `/srv/postgres` | 16K | lz4 |

`atime=off` on all five. `reservation=32G` on `rpool/ROOT` so a runaway import
cannot fill the pool and wedge the host — this guard exists only because bulk
data and the OS now share one pool.

**Mountpoints are deliberately pool-neutral.** Configure every consumer against
`/srv/...`. The plan text says `/tank/media` and `/tank/immich` in Phases 3 and
5 — translate those to `/srv/media` and `/srv/immich`.

ARC is capped at 3357540352 (3.13 GiB) in `/etc/modprobe.d/zfs.conf`, set by
the PVE 9 installer at 10% of RAM. Close enough to the planned 3 GiB; left
alone. `zfs_arc_min` was never set.

### Two consequences of having no redundant tier — these are permanent now

- **PBS backs up to the same disk it backs up.** A dead NVMe loses the VMs
  and their backups together. **Off-site is the only real backup that exists.**
  It is required before cutover, not a later phase. This is the single largest
  risk in the build.
- **Nothing survives a drive failure.** The two-week rule on the old host
  matters more than ever: do not delete the source.

### If disks are ever added later

Connect both drives, confirm they appear, then:

```
zpool create -o ashift=12 tank mirror /dev/disk/by-id/ata-XXX /dev/disk/by-id/ata-YYY
systemctl stop <consumers>              # nothing may be writing
zfs snapshot rpool/immich@move
zfs send -Rp rpool/immich@move | zfs recv -F tank/immich   # -p carries mountpoint
zfs set mountpoint=none rpool/immich    # keep the source until verified
zfs mount tank/immich                   # reappears at /srv/immich
```

Repeat for `media` and `pbs`. Because `-p` carries the `mountpoint` property,
each dataset comes back at the same `/srv` path — directory mappings, bind
mounts and the PBS datastore path all keep working untouched. Destroy the
`rpool` copies only after verifying, then drop the `rpool/ROOT` reservation if
you want the space back.

## Boot & passthrough facts (verified on the host)

- **UEFI + GRUB**, managed by `proxmox-boot-tool` (ESP `2E2A-7A9C`). Kernel
  cmdline edits go in `/etc/default/grub` then `proxmox-boot-tool refresh`.
  **`/etc/kernel/cmdline` is a no-op here** — it only drives systemd-boot.
  Editing the wrong one is a silent failure on a box with no console.
  The plan text (Phase 2) wrongly asserts systemd-boot. It is GRUB. Verified.
- **Secure Boot is ENABLED** — chain is `shim → GRUB`
  (`\EFI\PROXMOX\SHIMX64.EFI`). This is *why* the installer chose GRUB over
  systemd-boot; nothing is misconfigured. Host-side passthrough is unaffected:
  `vfio-pci` and `vfio_iommu_type1` are in-tree and signed.
- **GRUB never reads ZFS.** `proxmox-boot-tool` copies `vmlinuz` + `initrd` to
  the FAT32 ESP, so the usual "GRUB lags OpenZFS feature flags" hazard does not
  apply. Pool feature flags are irrelevant to booting.
- **Phase 3 trap — guest Secure Boot.** The NVIDIA driver in VM 100 is an
  out-of-tree DKMS module. Proxmox's default EFI disk has Secure Boot on with
  pre-enrolled keys, which will silently block it. Create VM 100's EFI disk as
  `efitype=4m,pre-enrolled-keys=0`. Guest Secure Boot is independent of the
  host's — disabling it in the guest does not weaken the host.
- **IOMMU is ON** (BIOS fixed 2026-08-22 on the ASRock B550M Pro4: Advanced →
  AMD CBS → NBIO Common Options → IOMMU = Enabled, plus Above 4G Decoding =
  Enabled). 16 groups, **interrupt remapping enabled** — so no
  `allow_unsafe_interrupts` needed.
- **No kernel cmdline change was required.** AMD-Vi came up by itself once the
  firmware published IVRS. `/etc/default/grub` is untouched — deliberately, to
  keep the bootloader out of the risk surface on a console-less box.
- **The GPU is alone in IOMMU group 2** with only its own bridges
  (`00:03.0`, `00:03.1`). No other endpoint shares it, so **no ACS override**.
  Group 0 by contrast fuses NVMe + SATA + USB + NIC — unusable, but not needed.
- **vfio config, as written** (all baked into the initramfs):
  - `/etc/modules-load.d/vfio.conf` — `vfio`, `vfio_iommu_type1`, `vfio_pci`
    (`/etc/modules` is deprecated on Debian 13; do not use it)
  - `/etc/initramfs-tools/modules` — same three, so binding beats userspace
  - `/etc/modprobe.d/vfio.conf` — `options vfio-pci ids=10de:2504,10de:228e disable_vga=1`
  - `/etc/modprobe.d/blacklist-nvidia.conf` — blacklists `nouveau`, `nvidia`,
    `nvidiafb`, `nvidia_drm`, plus `softdep snd_hda_intel pre: vfio-pci`.
    **Deliberately not** a blanket `blacklist snd_hda_intel` as the plan says —
    that would also kill onboard audio (`07:00.4`); the softdep is surgical.
  - `vfio_virqfd` intentionally omitted — merged into `vfio` in kernel 6.2.
  - Rollback snapshot: `rpool/ROOT/pve-1@pre-vfio`.
- **The host has no display output any more.** Console goes black right after
  GRUB. That is correct behaviour, not a fault — judge boot success by SSH.
- PVE 9.2.2, kernel 7.0.2-6-pve.

## Network — as actually built (flat + hypervisor firewall, 2026-08-22)

**VLANs are off the table.** The router is a Huawei **HG8145X6-10** — an
ISP-supplied GPON ONT. It does WAN-side service VLANs, but the LAN side offers
only DHCP, port forwarding and WiFi: no 802.1Q tagging per port, no inter-VLAN
ACLs. There is no second switch. So the planned VLAN 10 / VLAN 20 split cannot
be enforced upstream.

Everything is one flat `192.168.1.0/24`. Gateway and DNS are `192.168.1.1`.

**Why the router's own firewall is irrelevant here:** two VMs on the same
subnet are switched by `vmbr0` *inside the host*. That traffic never travels
down the cable, so the router never sees it and cannot filter it. Isolation has
to happen at the hypervisor or not at all.

### The enforcement point is `/etc/pve/firewall/cluster.fw`

Enabled cluster-wide with `policy_in: ACCEPT` / `policy_out: ACCEPT` — the
**host is deliberately left open**, exactly as before. All isolation is
per-guest. This is what keeps a firewall mistake from locking out a box with
no console.

| IPSet | Members |
|---|---|
| `lan` | `192.168.1.0/24` |
| `dmz` | `192.168.1.200` (VM 200) |
| `trusted` | `192.168.1.100` (VM 100) |

| Security group | Rules |
|---|---|
| `dmz-guest` | IN accept 80, 443 from anywhere; IN accept 22 + icmp from `+dc/lan`; **OUT DROP to `+dc/lan`, logged** |
| `trusted-guest` | IN DROP from `+dc/dmz`, logged |

`OUT DROP -dest +dc/lan` is the replacement for the VLAN deny rule. It stops
VM 200 reaching VM 100 *and* your laptop, TV and IoT gear in one rule, at the
tap device, before the bridge can switch the packet anywhere.

### Fixed addresses (assign statically inside each guest)

- **VM 100** home lab → `192.168.1.100`
- **VM 200** Dokploy → `192.168.1.200`

Matching the VMID is deliberate. Set them inside the guest or as DHCP
reservations; the IPSets above depend on them.

### Attaching the policy when a VM is created

Two things are required — the group does nothing without both:

```
# 1. per-guest policy, e.g. /etc/pve/firewall/200.fw
[OPTIONS]
enable: 1
policy_in: DROP
policy_out: ACCEPT

[RULES]
GROUP dmz-guest

# 2. arm the firewall on the NIC itself
qm set 200 -net0 virtio,bridge=vmbr0,firewall=1
```

For VM 100 use `GROUP trusted-guest` with `policy_in: ACCEPT` (it is reachable
from the LAN by design; only the DMZ is blocked).

**VM 200 must use public DNS** (`1.1.1.1`) and a static IP. The outbound deny
covers `192.168.1.1`, so the router's resolver and DHCP are unreachable from
the DMZ — that is intended, not a bug.

### Deliberately not done: VLAN-aware bridge

`vmbr0` was left as a plain bridge. With a router that cannot terminate VLANs
it would buy nothing, and editing `/etc/network/interfaces` on a console-less
host is the riskiest routine change available. Revisit only if the ISP router
is replaced with something that does 802.1Q.

### What this does not cover

The rest of the house is still one flat network — laptops, phones and IoT can
all reach each other. Segmenting that needs a better router and is a separate
project, not a prerequisite for this build.

### Still needed from the router

- Forward TCP **80** and **443** → `192.168.1.200`
- Forward UDP **51820** → wherever WireGuard lands, for remote access

## VM 100 — home lab (built 2026-08-22)

Reachable at **`teo@192.168.1.100`** with the same key as the host. Debian 13
trixie, kernel 6.12 cloud. `qm` id 100, name `homelab`, starts on boot.

**Built from the Debian genericcloud qcow2 + cloud-init, not the ISO
installer** — no interactive install step, so the whole VM is reproducible from
the `qm create` in the git history of this file.

| Setting | Value | Why |
|---|---|---|
| machine / bios | `q35` / `ovmf` | required for PCIe passthrough |
| efidisk0 | `efitype=4m,pre-enrolled-keys=0` | **guest Secure Boot OFF** — NVIDIA DKMS is self-signed with an unenrolled MOK and would be refused otherwise |
| cpu / cores | `host` / 6 | |
| memory / balloon | 16384 / **0** | VFIO pins the full 16 GB; ballooning must stay off |
| hostpci0 | `0000:05:00,pcie=1` | whole card, both functions. No `x-vga=1`, so noVNC still works |
| scsi0 | `local-zfs` 100G, discard, iothread, ssd | grew automatically via cloud-init |
| net0 | `virtio,bridge=vmbr0,firewall=1` | firewall=1 is required or the group is ignored |
| virtiofs0/1/2 | `dirid=immich` / `media` / `appdata` | |

Firewall: `/etc/pve/firewall/100.fw` → `GROUP trusted-guest`,
`policy_in: ACCEPT` (reachable from the LAN by design; only the DMZ is blocked).

### virtiofs

Host `/srv/{immich,media,appdata}` appear at the **same paths** inside the
guest via `/etc/fstab` entries of the form `immich /srv/immich virtiofs
defaults 0 0`. The tag is the mapping id from `/cluster/mapping/dir`.

**UID mapping is aligned and must stay that way.** Host dirs are `1000:1000`
and the guest user `teo` is uid 1000, so writes pass through cleanly (verified
both directions). Run containers as `1000:1000` or you get permission errors
that look like corruption.

### GPU — verified working end to end

`nvidia-smi` in the guest and inside a CUDA container both list the card:
driver **550.163.01**, CUDA 12.4, 12288 MiB. Debian's packaged `nvidia-driver`
from `non-free` (components were added to `/etc/apt/sources.list.d/debian.sources`).

### Installed

Docker CE 29.7.2 + Compose v5.5.0 (official Docker repo, `trixie`),
`nvidia-container-toolkit` 1.20.0 with `nvidia-ctk runtime configure`,
`qemu-guest-agent`. `teo` is in the `docker` group.
`/opt/stacks` exists as a git repo (`main`, `.env` gitignored) — one directory
per service, per the plan.

### Memory is now fully committed

With VM 100 up the host shows **20 GB used, ~10 GB free**. VM 200 (8) + PBS (2)
consume exactly that. **There is no slack left** — do not add a guest without
taking RAM from another.

### Still open on VM 100

- **Services not yet deployed**: Home Assistant, Immich, Jellyfin, Whisper,
  Piper. See the plan's Phase 3 for compose fragments and the Bulgarian TTS
  research (`bg_BG-dimitar-medium` is the practical Piper voice).
- **`rpool/postgres` is unused.** It was created per the plan but is *not*
  shared into the VM — Postgres over virtiofs is a bad idea (fsync/locking
  semantics). Decide at Immich time: either keep the DB on the VM's own zvol
  (simplest, same NVMe, proper block semantics) or attach a dedicated
  `volblocksize=16K` zvol as a second disk. Do not virtiofs it.
- `/opt/stacks` has no remote yet — the plan wants it in a private Git repo.

## Remote desktop — Sunshine + XFCE on VM 100 (2026-08-22)

GPU-streamed desktop for gaming/3D/video, reached with a **Moonlight** client.
Near-native: **NvFBC capture + NVENC** on the 3060.

- **Web UI / pairing:** https://192.168.1.100:47990 (set admin user/pass on
  first load). Moonlight ports 47984/47989/47990/48010 TCP + 47998-48000 UDP;
  reachable on the LAN because VM 100's firewall is `policy_in: ACCEPT`.
- **Driver was upgraded 550 → 610** (see below) — Sunshine's bundled ffmpeg
  needs NVENC API 13 / driver ≥570; Debian ships only 550.

### NVIDIA driver is now NVIDIA-repo, NOT Debian

**Critical for future maintenance.** Debian's `nvidia-driver` (max 550, even in
backports) was **purged** and replaced with the **open** kernel modules from
NVIDIA's CUDA repo (`developer.download.nvidia.com/compute/cuda/repos/debian13`).
Installed: **`nvidia-open` 610.57.04** (DKMS builds for the generic kernel) (DKMS, open modules, signed via MOK).
- `apt upgrade` now pulls NVIDIA-repo driver updates — watch for DKMS rebuilds
  against new kernels. Consider `apt-mark hold` if you want to freeze it.
- Jellyfin NVENC re-verified working on 610 (its own jellyfin-ffmpeg).
- `nvidia-container-toolkit` was collaterally removed by the `libnvidia*` purge
  and **reinstalled** + `nvidia-ctk runtime configure` re-run. If Docker GPU
  breaks after a driver change, reinstall it.

### VM 100 runs the GENERIC kernel, not cloud (required for Sunshine input)

The Debian **cloud** kernel (`*-cloud-amd64`) is stripped and **has no `uinput`**
module — so Sunshine cannot inject mouse/keyboard and Moonlight shows an
uncontrollable screen. VM 100 was switched to the **generic** kernel
(`linux-image-amd64`, `6.12.101+deb13-amd64`), which ships `uinput`.

**Kernel-switch gotcha that cost a rebuild:** do NOT `apt purge` the cloud
kernel in the same step as installing generic. Its default GRUB entry (the
top-level "simple" entry) kept pointing at the cloud kernel; deleting that
kernel's file left GRUB loading a missing kernel → **unbootable, no serial, no
console** (GPU has the only display). Recovery was a snapshot rollback.
The safe way, now in place:
- Keep BOTH kernels installed (cloud stays as fallback).
- `GRUB_DEFAULT` in `/etc/default/grub` is set to the **explicit generic entry**
  `gnulinux-advanced-<uuid>>gnulinux-6.12.101+deb13-amd64-advanced-<uuid>`,
  then `update-grub`. (`grub-reboot` one-shots did NOT stick here.)
- Generic-kernel boot + networking verified fine; the earlier hang was only the
  deleted-kernel/GRUB problem, not the kernel.

### Input injection (Moonlight interaction)

Two things are required and both were missing initially:
- **`uinput` loaded** — `/etc/modules-load.d/uinput.conf`. teo is in the
  `input` group; the sunshine udev rule gives uaccess. (`/dev/uinput` only
  exists on the generic kernel.)
- **`AutoAddDevices "true"`** in a `ServerFlags` section of `/etc/X11/xorg.conf`
  — an explicit ServerLayout otherwise disables input hotplug, so Sunshine's
  virtual devices never attach. When working, `xinput list` shows
  "Mouse passthrough" / "Keyboard passthrough".

### How the headless desktop is wired (non-obvious)

- **No display manager.** Debian's `nvidia_drm.ko` was built with **no
  parameters** (no `modeset`), so KMS can't be enabled → logind reports
  `CanGraphical=no` → lightdm/GDM won't start X. Even on open 610, `/dev/dri`
  is absent here. So we **bypass DRM entirely** and run Xorg directly.
- **`desktop-x.service`** (`/usr/local/bin/homelab-desktop.sh`) runs **as root**
  (the setuid Xorg wrapper forbids absolute `-config`/`-logfile` when elevated,
  so root is required), starts `Xorg :0` on the GPU, then launches
  `xfce4-session` + Sunshine **as teo**. lightdm is disabled.
- **Virtual display** via `/etc/X11/xorg.conf`: nvidia driver, `BusID PCI:1:0:0`,
  `ConnectedMonitor DFP-0`, relaxed `ModeValidation`, a 1920x1080\@60 ModeLine.
  Shows up as `HDMI-0 1920x1080`. Add ModeLines for more res/refresh if wanted.
- **Display resolution is 3456x2234\@60** (MacBook Pro 16" M4 native panel),
  set via a CVT ModeLine in `/etc/X11/xorg.conf` (`Virtual 3456 2234`;
  1728x1117 and 1920x1080 also offered). XFCE runs at **2x HiDPI**
  (`Gdk/WindowScalingFactor=2`, `Xft/DPI=192` in xsettings.xml) → logical
  1728x1117, matching macOS default. `xorg.conf.bak-1080p` is the old config.
  In Moonlight: set a custom/native 3456x2234 @60 for 1:1 pixel mapping.
- **Audio:** null-sink in `/etc/pulse/default.pa.d/sunshine-sink.pa` (no real
  card to capture).
- Sunshine user service `app-dev.lizardbyte.app.Sunshine.service` is started
  from `.xinitrc` after X is up (env imported into the user manager first).

### Snapshots (rollback points on VM 100)

- `pre-desktop` — clean server + working Jellyfin, before any desktop.
- `pre-driver-upgrade` — desktop installed, driver 550, cloud kernel. (Was
  rolled back to once during the kernel/driver saga.)
- `desktop-working` — **the good checkpoint**: generic kernel, driver 610,
  Sunshine NVENC + input working, Jellyfin OK.
Delete the first two once the desktop is confirmed stable over a few days.

## Phases

- [x] Phase 0 — downloads, BIOS, memtest
- [x] Phase 1 — datasets, ARC cap, zone firewall. `tank` dropped (no HDDs);
      VLAN-aware bridge dropped (router can't do VLANs)
- [x] Phase 2 — host side done: IOMMU on, both GPU functions bound to
      `vfio-pci`, `/dev/vfio/2` present. Attaching to VM 100 happens in Phase 3
- [~] Phase 3 — VM 100 built: GPU verified in a CUDA container, Docker +
      virtiofs + firewall done. **Services not yet deployed.**
- [~] Phase 4 — VM 200 built (Ubuntu 24.04), DMZ isolation verified,
      Dokploy v0.30.2 installed. **Admin account, certs, app migration and the
      router port-forward flip all still pending.**
- [~] Phase 5 — Jellyfin cut over (config migrated, media served from `sof1`
      over NFS ro; bulk media relocation still pending). Other services pending.
- [ ] Backups: PBS nightly + off-site
- [ ] Later: second node, QDevice for quorum

## Migration source — old host `sof1` @ 192.168.1.79 (inspected 2026-08-22)

User `tedraykov`, key-based SSH (key enrolled). Ubuntu 22.04, Docker 29.5.3.
**Drops ICMP** — a ping sweep misses it; it is up. Media drive `/mnt/drive1`
(1.8 T, ~714 G used). Portainer at `:9000`, GitOps stacks live in the portainer
volume under `.../compose/93/<service>/` and mirror `/home/tedraykov/releases/`.

**Nothing is deleted from it** until the new host is verified and one restore is
proven. Two-week rule after cutover. It is still serving live — leave it up.

Workloads seen (12+ day uptime): jellyfin, immich (server+ml+pg pgvecto-rs
v0.2.0+redis, **v1.120.2**), home-assistant + zigbee2mqtt + mosquitto +
hass-configurator, the *arr stack (radarr/sonarr/prowlarr/bazarr/qbittorrent/
overseerr/flaresolverr), traefik v3.7 (public TLS via `*.tedraykov.me`,
`tlsresolver`), n8n, minio, perfume-app (+worker+redis), excalidash.

### Migration mechanics that worked for Jellyfin (reuse these)

- **Config vs bulk split.** App config/DB is small (Jellyfin: 383 M) — copy it.
  Media is huge (589 G) — leave it on `sof1`, reach it over the network now,
  relocate in Phase 5.
- **Network media access = NFS ro.** `sof1` now runs `nfs-kernel-server`
  exporting `/mnt/drive1/data/plex` **ro to 192.168.1.100 only** (`/etc/exports`).
  VM 100 mounts it at `/mnt/oldmedia` via fstab
  (`ro,soft,timeo=30,_netdev,x-systemd.automount`). SMB "Home Server" share is
  `/mnt/drive1/data/samba` — does NOT cover media, don't use it for that.
- **Keep the in-container path identical** so the copied library DB needs no
  rescan: old container had `plex:/data:ro`; new container has
  `/mnt/oldmedia:/data:ro`. Same `/data`, paths still resolve.
- **Consistent DB copy:** tar-over-ssh live for bulk, then `docker stop` the old
  container, re-copy, `docker start` it again (~15 s old-host downtime), so the
  SQLite/DB snapshot is clean. Old service keeps running afterwards.
- **App config goes on the VM's own ext4 disk** (`/opt/stacks/<svc>/config`),
  **never virtiofs** — SQLite over virtiofs risks locking corruption. Same
  reasoning as `rpool/postgres`.

## Jellyfin — migrated to VM 100 (2026-08-22, first service moved)

Running at **http://192.168.1.100:8096** (`TedServer`, Jellyfin 10.11.11),
`/opt/stacks/jellyfin/compose.yaml`, official `jellyfin/jellyfin:latest`.
Users, library and watch state migrated intact (`config/data/jellyfin.db`).

- **NVENC verified end to end on the 3060.** Full NVDEC→NVENC pipeline on a
  10-bit HEVC source → 720p H.264, both GPU engines engaged (dec 47 / enc 58 %),
  1.79× realtime. The old host was CPU-only (`HardwareAccelerationType=none`).
  Set `nvenc` in `config/config/encoding.xml` (backup: `encoding.xml.pre-nvenc.bak`),
  enabled HEVC encode, left AV1 encode off (Ampere can't). Broadened hw-decode
  codec list.
- GPU reaches the container via `deploy.resources.reservations.devices` +
  `NVIDIA_DRIVER_CAPABILITIES=compute,video,utility`. No `/dev/dri` needed.
- **Dropped the Traefik labels + `proxy` network** from the old compose — public
  routing (`jellyfin.tedraykov.me`) is Phase 4. LAN/`:8096` only for now.
- **Still live on `sof1` too** — both read the media ro, no conflict. Decommission
  old Jellyfin only after the two-week rule.

## Portainer

- **VM 100:** `portainer/portainer-ce:lts` at **https://192.168.1.100:9443**
  (`/opt/stacks/portainer/`). Set the admin password before it locks.
- **Old host:** `:9000`, holds the source-of-truth GitOps stack definitions.
## VM 200 — Dokploy DMZ (built 2026-08-22)

Reachable at **`teo@192.168.1.200`** with the same key. **Ubuntu 24.04 LTS**
(noble cloud image + cloud-init), `qm` id 200, name `dokploy`, starts on boot.

**Ubuntu, not Debian 13, deliberately** — Dokploy's supported list stops at
Debian 12. Debian 13 would probably work; the internet-facing VM is the wrong
place to find out.

| Setting | Value | Why |
|---|---|---|
| bios / machine | seabios / i440fx (default) | no passthrough here — no reason to add OVMF to the risk surface |
| cores / cpu | 4 / `host` | |
| memory / balloon | 8192 / **4096** | ballooning ON — unlike VM 100, nothing is pinned |
| scsi0 | `local-zfs` 150G, discard, iothread, ssd | grew via cloud-init |
| net0 | `virtio,bridge=vmbr0,firewall=1` | firewall=1 required or the group is ignored |
| ipconfig0 | `192.168.1.200/24`, gw `192.168.1.1` | |
| nameserver | **`1.1.1.1`** | the router's resolver is unreachable by design |

Swap: **8 GB** `/swapfile`, `vm.swappiness=10` — Docker image builds are
memory-spiky and the OOM killer taking out Traefik is a bad afternoon.
`unattended-upgrades` on. SSH is keys-only (`passwordauthentication no`).

### Firewall — isolation verified, not assumed

`/etc/pve/firewall/200.fw` → `GROUP dmz-guest`, `policy_in: DROP`, plus one
per-guest rule the shared group does not carry:

```
IN ACCEPT -source +dc/lan -p tcp -dport 3000   # Dokploy UI, LAN/VPN only
```

Port **3000 must never be port-forwarded** — the control plane can deploy
arbitrary code and read every stored secret.

Tested from inside VM 200 after boot:

| Target | Result |
|---|---|
| `https://cloudflare.com`, DNS via 1.1.1.1 | reachable ✅ |
| `192.168.1.100:22`, `:8096` (VM 100) | **blocked** ✅ |
| `192.168.1.26:8006` (Proxmox host) | **blocked** ✅ |
| `192.168.1.79:22` (old host) | **blocked** ✅ |
| `192.168.1.1:80` and its DNS (router) | **blocked** ✅ |

Inbound SSH from the LAN still works — Proxmox accepts ESTABLISHED/RELATED
before the guest chains, so `OUT DROP -dest +dc/lan` does not break return
traffic on sessions the LAN opened. **This was the main thing to prove**; the
whole two-zone design rests on it.

### Deliberately NOT done: ufw inside the guest

The plan calls for `ufw` allowing 22/80/443/3000. Skipped on purpose:

- Docker (and Swarm, which Dokploy uses) writes its own iptables rules and
  publishes ports **past** ufw via the `DOCKER-USER` chain. ufw on a Docker
  host reads as protection it does not actually provide.
- The Proxmox firewall is strictly stronger here: it filters at the **tap
  device, outside the guest**, so nothing Docker does can punch through it.

Adding ufw would create a maintenance trap for no gain. If it is ever wanted
anyway, it must be configured against `DOCKER-USER`, not the default chains.

### Dokploy

**v0.30.2**, installed via `curl -sSL https://dokploy.com/install.sh | sh`.
Docker Swarm single node. Services: `dokploy`, `dokploy-postgres` (postgres:16),
`dokploy-traefik` (**traefik v3.6.7** — note the old host runs v3.7).

- UI: **http://192.168.1.200:3000** — **admin account not yet created**;
  `/register` is open to anyone on the LAN until it is. Do this first, enable 2FA.
- Traefik answers on :80 with 404 (no routes defined yet) — correct.

Snapshot: **`dokploy-fresh`** — clean install, before any app or admin account.

### Still open on VM 200

- **Admin account + 2FA** — not created yet. Everything below is deployed with
  plain `docker compose`, NOT yet adopted into Dokploy's UI, because creating
  apps in Dokploy requires the account first.
- ~~Dokploy's ACME email~~ **fixed 2026-08-22**: set to `business@tedraykov.me`
  in `/etc/dokploy/traefik/traefik.yml` (backup `.bak`), matching the ACME
  account `sof1` already uses. Resolver is named `letsencrypt` and uses an
  **HTTP-01** challenge on the `web` entrypoint — precisely what is unavailable
  until the port-forward cutover.
- Dokploy cleanup schedule, per-app memory/CPU limits, S3 database backups.
- Registry: GHCR already in use (`ghcr.io/tedraykov/perfumes-app`).

## Public apps — staged on VM 200 (2026-08-22)

**The GitOps definitions in Portainer stack `93` are STALE** — they still say
`treble.bg` and `traefik:v2.10`. Do not use them. The live definitions are in
`/var/lib/docker/volumes/portainer_portainer_data/_data/compose/<id>/`:

| Service | Stack id | Public host |
|---|---|---|
| traefik v3.7 | `89` | `traefik.tedraykov.me` |
| n8n 2.34.4 | `92` | `n8n.tedraykov.me` |
| minio | `71` | `minio.tedraykov.me`, `minio-console.tedraykov.me` |
| excalidash | `24` | `excalidraw.tedraykov.me` |
| perfume-app | `87` | `perfumes.tedraykov.me` |

**Always read the running container's labels rather than trusting a compose
file on that host** — that is what caught the stale-93 trap.

### Where things live on VM 200

- **`/opt/stacks/<svc>/compose.yaml`** — git repo, `.env` gitignored, mirrors
  VM 100's layout. Committed.
- **`/opt/appdata/<svc>/`** — application data, on the **VM's own 150 G disk**.
  Deliberately *not* virtiofs from `/srv`: virtiofs is not covered by the
  Proxmox firewall, so mapping host datasets into the DMZ would open a path
  the whole two-zone design exists to prevent. PBS backs the VM up whole.
- **`/opt/appdata/env/*.env`**, mode 600 — secrets pulled straight out of the
  old host's running containers (MinIO root creds, excalidash JWT/CSRF, the
  perfume-app Turso + Anthropic keys).

Data copied so far (all tiny — ~103 MB total): n8n 12 M, minio 91 M,
excalidash 904 K, perfume redis 28 K. UIDs preserved with `--numeric-owner`
(n8n 1000, excalidash 1001, redis 999, minio root) — they matter.

### HTTP-only until cutover

Every router is on **`entrypoints=web`** with **no** `tls.certresolver`, and
joins the external **`dokploy-network`**. Verified by Host-header curl against
`192.168.1.200` — no DNS or port-forward involved:

| Host | Result |
|---|---|
| `n8n.tedraykov.me` | 200 — 3 workflows loaded, encryption key decrypted them |
| `minio.tedraykov.me` | 403 — correct, that is MinIO's unauthenticated S3 root |
| `minio-console.tedraykov.me` | 200 |
| `excalidraw.tedraykov.me` | 200 |
| `perfumes.tedraykov.me` | 200 |

At cutover, each router flips to `entrypoints=websecure` +
`traefik.http.routers.<n>.tls.certresolver=letsencrypt`.

### Parallel-run safety — the classification that matters

Jellyfin could run on both hosts because it only *reads*. These cannot:

| Service | Parallel-run? | Why |
|---|---|---|
| n8n | **NO — stopped** | Its workflows are *active*: `Домоуправител`, `Домоуправител — разнасяне на вноски`, `OLX Watcher`. Two instances fire the same automations twice. |
| perfume-app + worker | **NO — stopped** | `DATABASE_URL` is an **external Turso DB shared with `sof1`**. Two workers write to the same live database. |
| minio | yes (running) | Separate copy of the data, no external side effects |
| excalidash | yes (running) | Separate SQLite copy, no external side effects |
| flaresolverr / redis | yes (running) | stateless / local |

**n8n and perfume-app are cutover-only services** — start them on VM 200 only
in the same maintenance window in which they are stopped on `sof1`.

For minio and excalidash the copies **diverge from the moment they start**, so
re-copy them at cutover with the source stopped (the Jellyfin pattern).

### Two gotchas found while staging

- **`:latest` drift.** excalidash and minio are pinned to `:latest`, so the new
  host pulled newer images than `sof1` runs. excalidash-backend immediately
  applied migration `20260415122313_add_drawing_snapshots` to the copied DB
  (new `DrawingSnapshot` table). Harmless here, but it means the copied DB is
  **schema-upgraded and no longer loadable by the old container** — the old
  host keeps its own untouched copy, so rollback is still fine. n8n was pinned
  to `2.34.4` on purpose to avoid exactly this.
- **excalidash-backend reports `unhealthy` — it is pre-existing**, and true on
  `sof1` too (13 days). Its healthcheck curls `/health` over localhost HTTP,
  but `TRUST_PROXY=1` + `FRONTEND_URL` makes the app 302 it to HTTPS, so it can
  never return 200. The app itself serves correctly. Not worth fixing.

## Public cutover — the sequencing constraint

There is **one** port-forward pair (80/443) and it currently points at `sof1`.
DNS for `tedraykov.me` is on **Cloudflare**; public IPv4 is **83.228.116.18**
(no wildcard record — individual A records per subdomain). Flipping the forward
to `.200` moves **all** public services at once, so it must be the *last* step,
not a test step.

The way to build and validate Dokploy without touching the live forward:

1. Give Traefik a **DNS-01 (Cloudflare) resolver** — issues real certificates
   with no inbound connection at all. Needs a Cloudflare API token scoped
   `Zone:DNS:Edit` on `tedraykov.me`.
2. Migrate the public apps into Dokploy and verify each one with
   `curl --resolve <host>:443:192.168.1.200`, bypassing DNS entirely.
3. Flip the router forward to `192.168.1.200` **once**, when everything is
   already serving correctly. Roll back by pointing it at `.79` again.

Public apps: perfume-app (+worker+redis), n8n, minio, excalidash — **all now
staged and routing on VM 200 over HTTP** (see above). Still to add: the Traefik
routes for jellyfin/immich once those are ready.

Remaining cutover steps, in order:
1. Create the Dokploy admin account + 2FA; adopt the four stacks into its UI.
2. Fix the ACME email; add `websecure` + `certresolver=letsencrypt` to routers.
3. Stop n8n, perfume-app and perfume-app-worker on `sof1`; re-copy n8n, minio
   and excalidash data with the sources stopped; start them on VM 200.
4. Flip the router forward 80/443 → `192.168.1.200`. Certificates issue on
   first request. Roll back by pointing it at `.79`.

## Subdomain strategy — split by sensitivity (decided 2026-08-22)

### The exposure audit that drove the decision

`sof1` currently routes **18 subdomains straight from the internet with ZERO
Traefik middlewares** — no basic-auth, no forward-auth, nothing. Each app's own
login is the only barrier:

`ha`, `zigbee2mqtt`, `qbittorrent`, `portainer`, `hass-configurator`, `traefik`
(running `--api.insecure=true`, so the dashboard/API is public and unauthenticated),
`radarr`, `sonarr`, `bazarr`, `prowlarr`, `overseerr`, `photos`, `jellyfin`,
`plex`, `nodered`, `n8n`, `minio`, `minio-console`, `excalidraw`, `perfumes`.

There is no split-horizon DNS today: the `dnsmasq` stack exists in Portainer
but **is not running**, and `/mnt/drive1/config/certs` is empty. Everything is
public DNS → public IP → `sof1` Traefik, with per-name certs via TLS-ALPN.

**Reducing this is a goal of the migration, not a side effect.**

### The constraint

One public IP, one 80/443 forward, pointing at exactly one VM. VM 200 owns it.
So a VM 100 app can get a subdomain in only two ways.

### Path A — public, proxied through the DMZ (narrow allowlist)

For services that genuinely need public reach: **Jellyfin, Overseerr.**

Requires **three** things; any one missing and it silently fails:

1. `/etc/pve/firewall/200.fw` — `OUT ACCEPT -dest 192.168.1.100 -p tcp -dport <port>`
2. `/etc/pve/firewall/100.fw` — `IN ACCEPT -source +dc/dmz -p tcp -dport <port>`
3. A router in `/etc/dokploy/traefik/dynamic/vm100.yml` on VM 200

**Both rules must precede the `GROUP` line** — Proxmox evaluates rules in order
and the group expands inline, so a rule after it never gets reached.

Backups of the originals: `/root/100.fw.bak`, `/root/200.fw.bak` on the host.

**Built and verified for Jellyfin 2026-08-22:**

| Check | Result |
|---|---|
| `Host: jellyfin.tedraykov.me` → `192.168.1.200` | 302 → `/web/`, `/System/Info/Public` returns `TedServer` 10.11.11 ✅ |
| VM 200 → `192.168.1.100:8096` | open (intended) |
| VM 200 → `.100` ports 22, 80, 443, 9443, 8000, 47990 | **all blocked** ✅ |
| VM 200 → host `:8006`, `sof1:22`, router `:80` | **all blocked** ✅ |

The deny rule is now an allowlist, but a one-port-wide one. A compromised DMZ
tenant reaches Jellyfin's HTTP port and nothing else — not SSH, not Proxmox,
not the rest of the LAN.

### Path B — private, direct, no hole at all

For everything sensitive: **Home Assistant, Immich, Portainer, Zigbee2MQTT,
the \*arr stack.** These stop being internet-facing entirely.

- A **second Traefik on VM 100** (ports 80/443 are free there).
- DNS A records → `192.168.1.100` (Cloudflare, DNS-only/grey cloud). Resolves
  everywhere, usable only on the LAN or over the VPN.
- **TLS needs DNS-01 via Cloudflare** — VM 100 is not internet-reachable, so
  HTTP-01 is impossible. DNS-01 also yields a `*.tedraykov.me` **wildcard**, so
  new subdomains work instantly with no per-name issuance.
  **Requires a Cloudflare API token scoped `Zone:DNS:Edit` on `tedraykov.me`.**
- Remote access via **WireGuard** (UDP 51820 forward — a separate forward from
  80/443, so no conflict with the DMZ).

**Not yet built — blocked on the Cloudflare API token.**

## Dokploy multi-tenancy — what it actually supports

Verified against the v0.30.2 source, not docs prose.

**Supported:** hierarchy is Organization → Project → Environment → Service.
Per-member permission flags exist (`packages/server/src/db/schema/user.ts`):
`canCreateProjects`, `canCreateServices`, `canDeleteProjects`,
`canDeleteServices`, `canAccessToGitProviders`, `canAccessToDocker`,
`canAccessToTraefikFiles`, `canAccessToAPI`, `canAccessToSSHKeys`. Members can
be scoped to specific Projects/Environments, and one with
`canAccessToGitProviders` can connect **their own GitHub** and deploy their repos.

**NOT supported — the requirement that does not survive contact:**

- **Users cannot self-register.** `apps/dokploy/pages/register.tsx` does
  `const hasAdmin = await isAdminPresent(); if (hasAdmin) { redirect }`. On
  self-hosted the page is titled *"Setup the server"* — it is one-time server
  setup, not open signup. **The owner must invite every user.**
- **Users cannot create their own organizations** — `/organization/create` is
  in `disabledPaths` in `packages/server/src/lib/auth.ts`. One org, the one
  made at install.
- **Custom/fine-grained roles are Enterprise (paid).** Self-hosted gets
  Owner / Admin / Member.
- GitHub/Google social login exists but needs `GITHUB_CLIENT_ID` /
  `GITHUB_CLIENT_SECRET` env vars, which are **not set** on this instance.

### The security reality of tenant deploys

**Anyone who can deploy arbitrary code on Dokploy effectively has root on
VM 200.** Docker builds run as root; a compose with `privileged: true` or a
`/var/run/docker.sock` bind mount is a full host takeover. The permission flags
gate the **UI**, not the code being deployed. There is no sandbox between
tenants — one Docker daemon, one Swarm.

The DMZ bounds the blast radius (verified: a tenant cannot reach VM 100 beyond
Jellyfin's port, the host, `sof1`, or the router), and the plan always intended
this — its diagram labels VM 200 *"untrusted code, treat as hostile."*

**But the plan assumed the DMZ held only disposable things, and it no longer
does.** VM 200 now also stores: MinIO root credentials, the perfume-app's
Anthropic API key and Turso token, excalidash's JWT/CSRF secrets, and n8n's
entire credential store.

**Decision (2026-08-22): accepted.** Tenants will be people the user actually
knows, and your apps stay co-located with theirs on VM 200. The consequence to
keep in mind: **invite only trusted people** — any Dokploy member who can
deploy code can in principle read every secret above. If that ever stops being
true, the fix is to move your own apps to VM 100 behind the same narrow
per-port allowlist as Jellyfin, leaving VM 200 purely untrusted.

### Tenant onboarding workflow (invite-driven, not self-serve)

1. Owner invites the user by email from Dokploy → Settings → Users.
2. User accepts and sets a password (no self-registration exists).
3. Grant `canCreateProjects`, `canCreateServices`, `canAccessToGitProviders`.
4. Scope them to their own Project/Environment.
5. They connect **their own GitHub** under Git Sources and deploy their repos.

Do **not** grant `canAccessToDocker`, `canAccessToTraefikFiles` or
`canAccessToSSHKeys` to a tenant — those hand over the rest of the VM directly.

## THE NETWORK FINDING that reshaped the migration (2026-08-23)

**`sof1` is on WiFi.** Its ethernet `eno1` is `NO-CARRIER`; `192.168.1.79` lives
on `wlo1`. Measured VM 100 ↔ `sof1` throughput: **1.2 MB/s**.

Measured at the same time: virtiofs write to `/srv/immich` = **4.4 GB/s**. The
storage side was never the problem — every slow copy was the WiFi link.

At 1.2 MB/s the remaining bulk (Immich 112 G + media 589 G ≈ 700 G) would take
**~6.7 days**. That is why network migration was abandoned.

**Consequence to remember: Jellyfin on VM 100 currently streams over NFS from a
WiFi host at ~10 Mbps.** It is degraded until the media is local — that is
expected, not a Jellyfin fault.

### The source drive

`sof1:/dev/sdb1` — **Seagate ST2000LM015, serial ZDZQGL87**, 2.5" **SMR** laptop
drive, 1.8 T ext4, UUID `9b60cd50-64dd-48f9-ac90-32c00023e2f0`, mounted
`/mnt/drive1`, 714 G used. Reads ~100–140 MB/s.

Because that SMR disk is the real limit, **an ethernet cable and physically
moving the drive land at the same ~1.7 h** for 700 G. Speed was not the reason
to choose one over the other.

**Decision: move the drive** (and the Zigbee dongle at the same time).

### What lives where — verified

**Everything homelab is on `/mnt/drive1` (sdb1)**: all app configs
(`/mnt/drive1/config/*`), the media tree (`/mnt/drive1/data/plex`, 589 G =
551 G media + 39 G torrents), and the Immich library
(`/mnt/drive1/data/immich/library`, 112 G).

**The only exception was the `immich_pgdata` Docker volume** (1 GB) on `sof1`'s
root disk `sda2`. **Staged onto the drive** so it travels with it:

| `/mnt/drive1/migration/` | Size | Notes |
|---|---|---|
| `immich_pgdata.tar` | 1.0 G | copied with Postgres **stopped** — consistent |
| `portainer_data.tar` | 4.4 M | Portainer stack definitions |

`immich_model-cache` (1.7 G) deliberately **not** copied — ML models
re-download, and VM 100 has real bandwidth.

### Pulling the drive is a FULL cutover, not just the homelab

Traefik's ACME storage (`/mnt/drive1/config/letsencrypt`), minio's data and
n8n's data are all on that drive too. When it leaves, **every public site on
`sof1` stops**. Public app data is already staged on VM 200, so the answer is
to complete the VM 200 cutover in the same window.

### The low-downtime sequence

Do **not** wait for the ZFS copy before restoring service:

1. Stage volumes onto the drive ✅ done.
2. Stop containers on `sof1`, unmount `/mnt/drive1`, power off.
3. Shut down VM 100 + VM 200, then the Proxmox host (a SATA disk needs the
   case open, so the host must be off).
4. Move the drive to a free SATA port; move the **Sonoff ZBDongle-P** to a host
   USB port. All 6 SATA ports are free; controller is AMD 500-series.
5. Boot; mount the ext4 drive **read-only** at `/srv/olddrive`; add a Proxmox
   directory mapping and virtiofs it into VM 100.
6. Deploy the homelab stacks **against `/srv/olddrive/...`** — service is back
   within minutes, before any bulk copy has happened.
7. **Background:** rsync `/srv/olddrive/data/plex` → `/srv/media` and the Immich
   library → `/srv/immich`, locally (~1.7 h).
8. Switch the stacks to `/srv/media` and `/srv/immich`.
9. **Leave the ext4 drive untouched for two weeks** — it is the rollback.

USB passthrough for the dongle once it is on the host:
`qm set 100 -usb0 host=10c4:ea60` (Silicon Labs CP210x; it is the only one).

## Portainer on VM 100 — initialised (2026-08-23)

**2.39.6** at https://192.168.1.100:9443. It had **timed out** — no admin
account was created within 5 minutes of first start, which disables
initialisation until a restart. Fixed by restarting and posting to
`/api/users/admin/init` with the `X-Setup-Token` from the logs (the token is
printed with ANSI colour codes between `setup_token=` and the value — strip
them or the grep silently returns nothing).

- Admin credentials: **`/opt/stacks/portainer/.admin-credentials`** on VM 100,
  mode 600. Change the password when convenient.
- Local Docker environment registered as **endpoint id 1** (`local`).
- **Stacks are to be deployed through Portainer**, matching the `sof1` workflow.

A migration SSH key now exists: `teo@vm100` → `tedraykov@sof1`
(`/home/teo/.ssh/id_ed25519`, installed in `sof1`'s `authorized_keys`).

`rsync` was missing on VM 100 and is now installed.

### virtiofs UID behaviour — tested, not assumed

A container running as **root** in VM 100 writes through virtiofs and the file
lands **root-owned on the host**. So Immich can run as root exactly as it does
on `sof1`; no `user:` override and no chown are needed. The earlier
"run containers as 1000:1000" note applies to keeping *existing* 1000-owned
trees consistent, not to root-owned ones like Immich's.

## STORAGE ARCHITECTURE — as built after the drive move (2026-08-23)

The drive was physically moved from `sof1` into the Proxmox host and is now
`/dev/sda` (Seagate ST2000LM015, serial ZDZQGL87, ext4, 1.8 T, 715 G used).

**The tiering rule: bulk media on the HDD, everything else on the NVMe.**

| Path | Device | Holds |
|---|---|---|
| `/mnt/hdd` | HDD ext4 | the raw drive as it came out of `sof1` |
| `/srv/media` | HDD (bind of `/mnt/hdd/data/plex`) | 551 G media + 39 G torrents |
| `/srv/immich` | HDD (bind of `/mnt/hdd/data/immich/library`) | the photo library |
| `/srv/appdata` | **NVMe** (`rpool/appdata`) | scratch/staging only |
| `/opt/stacks/*` on VM 100 | **NVMe** (VM 100's own zvol) | **all app config + databases** |
| `/mnt/hdd/backups/proxmox` | HDD | Proxmox VM backups |

`/etc/fstab` on the host (backup: `/root/fstab.bak-predrive`):

```
UUID=9b60cd50-64dd-48f9-ac90-32c00023e2f0  /mnt/hdd  ext4  defaults,nofail  0  2
/mnt/hdd/data/plex           /srv/media   none  bind,nofail  0  0
/mnt/hdd/data/immich/library /srv/immich  none  bind,nofail  0  0
```

**`nofail` is mandatory on all three** — this host has no console, so a failing
or absent drive must never wedge boot.

`rpool/media` and `rpool/immich` were set `mountpoint=none` (they were empty
placeholders). **The `/srv/*` paths are unchanged for every consumer** — that
pool-neutral rule is exactly what made swapping the backing device a two-line
job. `rpool/immich` still holds **2.22 G of orphaned data** from the aborted
WiFi rsync; reclaim it when convenient.

### Why configs are on VM 100's disk and not `/srv/appdata`

VM 100's disk is a zvol on the NVMe, so `/opt/stacks/*` satisfies "configs on
the SSD" **and** keeps SQLite/Postgres off virtiofs — the project's standing
rule. `/srv/appdata` is virtiofs and is only for bulk/scratch.

### Verified after the move

| Check | Result |
|---|---|
| Immich Postgres restored from `immich_pgdata.tar` | **8,526 assets, 5 users** ✅ |
| Immich server | v1.120.2 listening on 2283 ✅ |
| Zigbee dongle at the **same `/dev/serial/by-id` path** as `sof1` | ✅ device line unchanged |
| Zigbee network | coordinator EmberZNet 8.0.3, **3 devices still joined — no re-pairing** ✅ |
| Home Assistant | HTTP 200 on :8123 ✅ |
| Jellyfin `/data` | `media/`, `torrents/` visible ✅ |
| **Hardlink `/data/torrents` → `/data/media` through virtiofs** | **same inode, links=2** ✅ |

That last one matters most: the whole \*arr import pipeline depends on
hardlinking, and it works **through virtiofs** because both directories are on
the one ext4 filesystem. Do not split them across devices.

## Homelab stacks — deployed via PORTAINER (2026-08-23)

Deployed through the Portainer API on VM 100 (endpoint id 1), not raw compose:

| Stack | id | Services |
|---|---|---|
| `immich` | 1 | server, machine-learning, redis, postgres (pgvecto-rs pg14-v0.2.0) |
| `homeassistant` | 2 | homeassistant, mosquitto, zigbee2mqtt, hass-configurator |
| `servarr` | 3 | sonarr, radarr, prowlarr, bazarr, qbittorrent, overseerr, flaresolverr |
| `jellyfin` | 4 | jellyfin (moved off raw compose into Portainer) |

**All versions pinned to what `sof1` ran** (Immich v1.120.2 especially) — the
excalidash `:latest` drift earlier is why.

`/data` is the **same in-container path** as on `sof1`, so every path stored in
the \*arr and Jellyfin databases still resolves. No rescan was needed.

**Secrets recovery:** Immich's `DB_PASSWORD` lived only in Portainer's stack env
on the dead `sof1`. It was recovered with `strings` on `portainer.db` from
`portainer_data.tar`. That file also revealed the old GitOps repo:
**`github.com/alttreble/releases`, branch `trunk`** — this working directory.
The extracted copy was deleted afterwards (it contained every stack secret).

`nodered` was defined in the old HA stack but **was not running** on `sof1`; it
was not redeployed. Its data is at `/opt/stacks/homeassistant/` if wanted.

## VM backups (2026-08-23)

- Storage **`hdd-backup`** → `/mnt/hdd/backups/proxmox`, 1.07 T free, with
  `is_mountpoint /mnt/hdd` so a missing drive can never silently fill the root fs.
- Nightly **02:30**, all guests, `mode snapshot`, zstd, retention
  `keep-daily=7,keep-weekly=4,keep-monthly=3`.
- Observed throughput ~250–350 MB/s.
- PBS in LXC 300 was **not** built — only ~4 G RAM is free. `vzdump` needs none.
  Revisit if dedup/incremental is wanted.

### ⚠ The gap this leaves — photos have NO backup

VM backups cover the **VMs**. The Immich library and the media tree live on the
**HDD, outside any VM**, so `vzdump` does not touch them.

- Movies/TV (551 G) — replaceable, acceptable.
- **The Immich photo library is irreplaceable and now exists in exactly one
  place, on a single non-redundant SMR drive.** `sof1` no longer has a copy —
  that drive *is* the former copy.

This is the single largest risk in the build right now. It resolves when the
second HDD arrives and the pair becomes a mirror, but until then a drive
failure loses the photos permanently. Options meanwhile: keep the Immich
library on the NVMe (112 G on a 3.6 T pool) and back it up to the HDD, or add
an off-site target.

## Planned: second HDD → ZFS mirror

Do **not** reformat the current HDD to ZFS first — its data is the only copy.
The clean path when the second disk arrives:

1. `zpool create tank <new-disk>` (single disk, no redundancy yet).
2. Create `tank/media` and `tank/immich`, copy from the ext4 drive.
3. Repoint `/srv/media` and `/srv/immich` at the ZFS datasets (consumers see no
   change — that is the point of the pool-neutral paths).
4. Verify, then wipe the ext4 drive and **`zpool attach tank <new-disk> <old-disk>`**
   to make it a true mirror.

That reaches RAID1 with the data copied **once**. Note this protects against a
*drive* failure, not a *host* failure — actual high availability needs the
second node and QDevice already in the plan.

## Private subdomains — BUILT (2026-08-23)

Traefik on VM 100 (`/opt/stacks/traefik`, Portainer stack id 6, traefik v3.7)
serves every private service over HTTPS with a real **`*.tedraykov.me` wildcard**
from Let's Encrypt via **Cloudflare DNS-01**. No inbound connection is involved,
which is exactly why it works for services that are not internet-exposed.

All verified working end-to-end with real DNS and `ssl_verify_result=0`:

`ha` · `photos` · `zigbee2mqtt` · `hass-configurator` · `sonarr` · `radarr` ·
`prowlarr` · `bazarr` · `qbittorrent` · `overseerr` · `portainer` · `jellyfin`

Their Cloudflare A records now point at **192.168.1.100** (were the public IP).
Nothing was lost by this: they previously resolved to the public IP, which the
router still forwards to the powered-off `sof1`, so they were dead either way.

`jellyfin` deliberately still points at **83.228.116.18** — it is the one public
service, routed via the DMZ Traefik on VM 200. It stays dead until the router
forward is moved to `192.168.1.200`.

Routes live in `/opt/stacks/traefik/dynamic/homelab.yml` as a **file provider**
pointing at `http://192.168.1.100:<port>` — chosen over Docker labels so the
four existing stacks did not have to be redeployed onto a shared network. To add
a service: publish its port, add a router+service pair, point the name at
192.168.1.100. **The wildcard already covers it — no per-name issuance.**

### Cloudflare token — three things that cost time

1. **Cloudflare now issues prefixed "scannable" tokens**: `cfk_` (Global Key),
   `cfut_` (User API Token), `cfat_` (**Account** API Token) — prefix + 40 chars
   + checksum, so ~53 chars. Old unprefixed 40-char tokens still work.
   Ours is **`cfat_`**.
2. **`/user/tokens/verify` returns "Invalid API Token" for an ACCOUNT token.**
   That is not a broken token — it is the wrong endpoint. **Verify by doing the
   real operations instead**: `GET /zones?name=...` (Zone:Read) and a TXT
   create+delete (DNS:Edit). That is precisely what lego does.
3. Required permissions: **`Zone:Zone:Read` AND `Zone:DNS:Edit`**, scoped to the
   single zone. Cloudflare's "Edit zone DNS" template grants only DNS:Edit —
   Zone:Read must be added by hand or lego cannot resolve the zone ID.

### How the token is stored — and why not env_file

- `/opt/stacks/traefik/.env` (`CF_DNS_API_TOKEN=…`) and a bare-value copy at
  `/opt/stacks/traefik/cf_token`, both **mode 600**, both gitignored.
- The compose passes **`CF_DNS_API_TOKEN_FILE=/run/secrets/cf_token`** with the
  file bind-mounted in.
- **`env_file` with a host path does NOT work in a Portainer-deployed stack** —
  Portainer runs compose *inside its own container*, so it cannot read host
  paths. Bind mounts work (they go to the Docker daemon); env_file does not.
  A Portainer stack variable would also have written the secret into Portainer's
  database. The `_FILE` route avoids both problems.
- The token is on **VM 100 only**. It must never go on VM 200 — DMZ tenants
  deploy arbitrary code there, and this token can rewrite any record in the zone
  (including MX).

### ⚠ Home Assistant behind a proxy — `.storage/http` overrides YAML

HA returned **400 Bad Request** to every proxied request:
`Received X-Forwarded-For header from an untrusted proxy 172.20.0.1`.

Adding an `http:` block to `configuration.yaml` **did nothing** — not with the
CIDR, not with the exact IP, not even with `use_x_forwarded_for: false`. The
YAML parsed correctly and `check_config` was clean.

**Cause:** HA (2026.8.3) has migrated http config into
**`/config/.storage/http`**, which is marked `"yaml_migration_done": true` and
**takes precedence over configuration.yaml**. That file still carried `sof1`'s
Traefik network: `trusted_proxies: ["172.29.0.0/16"]`.

**Fix:** stop HA, edit `.storage/http` (backup: `.storage/http.bak-sof1`), set
`trusted_proxies` to `["172.16.0.0/12","127.0.0.1","::1"]`, start HA → 200.

**If HA ignores a YAML setting, check `.storage/` before debugging the YAML.**
The `configuration.yaml` block was left in place and set back to `true` so a
future migration would land on the right value, but it is *not* what is in
effect today.

## Immich made public (2026-08-23)

`photos.tedraykov.me` is being moved from private to **public via the DMZ**, at
the user's request. Same three-piece pattern as Jellyfin:

1. `200.fw` — `OUT ACCEPT -dest 192.168.1.100 -p tcp -dport 2283`
2. `100.fw` — `IN ACCEPT -source +dc/dmz -p tcp -dport 2283`
3. `photos` router in `/etc/dokploy/traefik/dynamic/vm100.yml` on VM 200

(backups: `/root/{100,200}.fw.bak-prephotos`)

**Verified**: `Host: photos.tedraykov.me` → `192.168.1.200` returns 200, and the
API correctly returns 401. Re-checked that the hole did not widen — from the
DMZ, only **2283 and 8096** are open on VM 100; 22, 80, 443, 9443, 8123, 8989
all still blocked.

`vm100.yml` now defines **two routers per service**: `<name>-http` on `web`
(plain HTTP — used for Host-header verification and as the ACME HTTP-01 path)
and `<name>` on `websecure` with `certResolver: letsencrypt`. **No `tls.domains`
is set anywhere** — that would make Traefik attempt issuance eagerly at startup
and burn Let's Encrypt rate limits while the port forward still points elsewhere.

### Two things gate public access — in this order

1. **Move the router forward**: 80/443 → `192.168.1.200`. Certificates cannot
   issue before this (Dokploy's Traefik uses HTTP-01), and nothing public works.
2. **Only then** repoint `photos.tedraykov.me` A → `83.228.116.18`.
   **Do not flip DNS first** — it currently resolves to `192.168.1.100` and
   works on the LAN; flipping early points it at a forward that still targets
   the powered-off `sof1`, breaking what works today.

### The LAN-access catch — NAT hairpin

Once `photos` resolves to the public IP, LAN clients reach it only if the Huawei
HG8145X6 does **NAT hairpin** (loopback). Many ISP ONTs do not. If it does not,
`photos.tedraykov.me` stops working *inside the house* — which matters most for
a phone photo app on home WiFi.

Both routes already exist (VM 100 Traefik privately, VM 200 Traefik publicly);
the only question is what DNS answers. Options if hairpin fails:

- **Local DNS (best)** — dnsmasq/AdGuard on VM 100 answering `photos` →
  `192.168.1.100` for LAN clients, public DNS answering the public IP for
  everyone else. True split-horizon, and it fixes this for every future service.
  Needs the router's DHCP to hand out the local resolver.
- Per-device `hosts` entries (fine for a laptop, useless for phones).
- Cloudflare proxy (orange cloud) — note their ToS discourages proxying large
  media, so not a great fit for a photo/video library.

### Risk note

Immich holds **the irreplaceable data**, it is now internet-facing, and it still
has **no backup** (the library sits on the HDD outside any VM, so `vzdump` does
not cover it). Public exposure and zero backup compound. Worth resolving the
backup gap before or soon after the forward is moved.

## DECISION REVERSED: everything public again (2026-08-23)

The user asked for **`sof1`'s behaviour back — every service that had a domain
is public**. This supersedes the earlier "split by sensitivity" choice. Recorded
as a deliberate decision, not drift.

(My own inconsistency that surfaced it: Overseerr was in the *public* list of the
split option the user originally accepted, but I built it as private along with
the rest of the \*arr stack.)

### How it is wired — and why the proxy stays in the DMZ

80/443 forward to **VM 200**, whose Traefik reverse-proxies to VM 100. The
alternative — forwarding straight to VM 100 — was rejected: if Traefik is ever
compromised, this way the attacker lands on the **disposable** VM rather than the
one holding the photo library and Home Assistant.

The firewall holes add little marginal exposure: a hole to `sonarr:8989` is
exactly the path that makes Sonarr public in the first place. The rule is now
"the DMZ may reach precisely the ports that are published to the internet
anyway", which is coherent — and **nothing else**. Re-verified from inside
VM 200: SSH, the Proxmox UI, the router and Portainer's HTTP port are all
still blocked.

Ports opened DMZ → VM 100 (backups `/root/{100,200}.fw.bak-preallpublic`):

| Port | Service | Port | Service |
|---|---|---|---|
| 8096 | jellyfin | 8989 | sonarr |
| 2283 | photos (immich) | 7878 | radarr |
| 8123 | ha | 9696 | prowlarr |
| 3218 | hass-configurator | 6767 | bazarr |
| 8080 | zigbee2mqtt | 8084 | qbittorrent |
| 5055 | overseerr | 9443 | portainer (https, insecureSkipVerify) |

**Deliberately still NOT public:** Dokploy's control plane (VM 200 :3000, LAN
only) and both Traefik dashboards. Exposing an unauthenticated Traefik dashboard
— which is what `traefik.tedraykov.me` was on `sof1`, with `--api.insecure=true`
— is not something to restore.

All 12 verified through the DMZ Traefik with Host-header curls.

### Home Assistant needed a SECOND trusted proxy

Via VM 100's own Traefik, HA sees the peer as a Docker gateway (`172.20.0.1`).
Via the DMZ Traefik it sees **`192.168.1.200`** — the real LAN address, because
Docker's DNAT preserves the source for connections arriving from another host.
`.storage/http` `trusted_proxies` is now:
`["172.16.0.0/12","192.168.1.200","127.0.0.1","::1"]` — both paths return 200.

### Remaining, in order

1. **Move the router forward: 80/443 → `192.168.1.200`.** Nothing public works
   before this, and HTTP-01 cannot issue certificates.
2. **Test NAT hairpin** from a LAN client before touching DNS.
3. **Then** repoint the 11 A records from `192.168.1.100` back to
   `83.228.116.18`.

**Do not do 3 before 1** — those names currently resolve to VM 100 and work on
the LAN; flipping early aims them at a forward still pointing at the dead host.

### ⚠ Hairpin now affects EVERYTHING, not just photos

Once all names resolve to the public IP, LAN clients reach them only if the
Huawei HG8145X6 does NAT loopback. If it does not, **the entire lab becomes
unreachable by name from inside the house** — a much bigger deal than when it
was only `photos`.

The robust fix is **split-horizon DNS**: dnsmasq or AdGuard on VM 100 answering
`*.tedraykov.me` → `192.168.1.100` for LAN clients while public DNS answers the
public IP. It needs the router's DHCP to hand out the local resolver. Worth
building **before** flipping DNS if hairpin turns out not to work — the VM 100
Traefik and its wildcard certificate already serve every one of these names, so
split-horizon requires no new routing at all, only DNS.

## Working agreements

- Show me the command and what it will do before running anything destructive
  — `zpool`, `zfs destroy`, `rm`, partitioning, `/etc/network/interfaces`,
  `/etc/kernel/cmdline`, `update-initramfs`.
- Snapshot before risky changes: `zfs snapshot rpool/ROOT/pve-1@pre-<change>`.
- Before any reboot, say what should come back up and how I verify it.
- Prefer reading state (`zpool status`, `pvesh get`, `ip a`) over assuming it.
- This machine has no console. If a change could affect boot, say so first.

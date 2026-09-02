# Home lab — Proxmox host, and this repo

This repository holds the Docker Compose definitions for the home lab. The lab
runs on a single Proxmox VE 9 host; the migration off the old `sof1` box is
**complete** and the public cutover is **done**.

**Full original plan:** https://claude.ai/code/artifact/4c133e3e-75df-408b-910a-5cca9872d9d0
(fetch with WebFetch if you need detail). Much of it is now historical — where
the plan and this file disagree, **this file is what was actually built.**

Last verified against live infrastructure: **2026-08-31**.

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
| Photo library | **`/srv/immich`** | ZFS `tank/immich`, virtiofs into the VM. Mounted at **`/data`** in the container since 2026-08-31, not `/usr/src/app/upload`. |

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
| `immich` | **live on VM 100** | v3.1.0. Git-backed Portainer stack **13**. Library at `/data`, CUDA ML + NVENC. |
| `servarr` | **live on VM 100** | sonarr, radarr, prowlarr, bazarr, qbittorrent, overseerr, flaresolverr |
| `traefik` | **live on VM 100** | v3.7, wildcard cert |
| `portainer` | **live on VM 100** | deployed from `/opt/stacks/portainer/compose.yaml`, not via Portainer itself |
| `minio` | **live on VM 100** | S3 object storage. Git-backed Portainer stack 11. |
| `n8n` | **live on VM 100** | Git-backed Portainer stack 12. |
| `esphome` | **live on VM 100** | Device builder for the voice satellite. Git-backed Portainer stack 14. Host networking, LAN only. |
| `excalidraw` | live on **VM 200** | **Dokploy-managed, git-sourced from this repo.** Container/data are named `excalidash`. |
| `minecraft` `monitoring` `plex` | **dormant** | not deployed. Data still at `/mnt/archive/config/*` and `/mnt/archive/data/*` if wanted. |

Removed 2026-08-29 as unwanted: `nextcloud`, `roberts50`, `arbitrage`, `zrok`,
`pelias`, `wordpress`, `dnsmasq`, then `kafka`, `minio`, `prefect`,
`structurizr`, `uptime`, `utorrent`, `perfume-app`. All recoverable from git
history. (`dnsmasq` is genuinely obsolete — AdGuard in LXC 300 does that job
now; `plex` and `utorrent` were superseded by `jellyfin` and `qbittorrent`.)

Several dormant stacks still carry `*.treble.bg` Traefik labels and
`certresolver=tlsresolver` from the `sof1` era. The live domain is
`tedraykov.me`; **VM 100's resolver is named `cloudflare`** (DNS-01) and
VM 200's is `letsencrypt` (HTTP-01). On VM 100 a label only needs `tls=true` —
`tls.yml` sets a default generated wildcard cert, so no resolver is named per
router. **Fix the labels before deploying any of them.**

## How this repo reaches the running containers

**Mostly git-backed now.** Seven of the nine stacks pull from
`https://github.com/alttreble/releases` (`refs/heads/trunk`) — edit here, push,
then redeploy and the change lands. Verified against the Portainer API on
2026-08-31; an earlier version of this file undercounted them.

| Portainer stack | Deploys | Config file |
|---|---|---|
| **7** | **jellyfin** | **git-backed** → `jellyfin/docker-compose.yml` |
| **8** | **servarr** | **git-backed** → `servarr/docker-compose.yml` |
| **9** | **homeassistant** | **git-backed** → `homeassistant/docker-compose.yaml` |
| **11** | **minio** | **git-backed** → `minio/docker-compose.yaml` |
| **12** | **n8n** | **git-backed** → `n8n/docker-compose.yml` |
| **13** | **immich** | **git-backed** → `immich/docker-compose.yml` |
| **14** | **esphome** | **git-backed** → `esphome/docker-compose.yml` |
| 6 | traefik | standalone editor stack |
| 10 | beszel | standalone editor stack |

Only **6 and 10** are left as editor stacks. Stack 6 (traefik) has no copy here
at all — its definition lives only in Portainer's own volume. Stack 10 (beszel)
now has one at `beszel/`, but the stack itself is still an editor stack, so
**editing this repo does not change what beszel runs** until it is converted to
git-backed.

**Stack 1 no longer exists.** Immich was an editor stack until 2026-08-31.
Portainer cannot convert one in place, so it was deleted and recreated as a
git-backed stack, which allocated a **new id, 13**. The containers came back
untouched: `docker compose down` without `-v` cannot touch a bind mount, and
Portainer does not pass `-v`, so `/opt/stacks/immich/pgdata`, `/srv/immich` and
the `immich_model-cache` volume all survived. Expect the same id change if 6 or
10 are converted.

The repo is **public**, so Portainer needs no credentials to pull it. That also
means everything committed here is internet-readable — keep secrets in
gitignored `.env` files and bind-mounted secret files, never in a compose.

Redeploy a git-backed stack:
`PUT /api/stacks/<id>/git/redeploy?endpointId=1` with `{"pullImage":false}`.

VM 200's `/opt/stacks/` repo now holds only `beszel` — `excalidash` became a
Dokploy-managed deployment on 2026-08-29. Data still lives under
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
- Per-name overrides → `192.168.1.200` for the DMZ apps: `excalidraw`, `plane`
  (backups: `/opt/AdGuardHome/AdGuardHome.yaml.bak-*`)

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

Plus **`plane`** and **`excalidraw`**, which run on VM 200 itself via Dokploy.

## DNS is now a single wildcard record

Cloudflare holds **exactly one** record: `*.tedraykov.me A → 83.228.116.18`.
Every per-name record was removed, including `beszel`, which had pointed at
`192.168.1.100` — a private address, so Let's Encrypt could not validate it and
its certificate failed until the record was deleted. **Never point a public A
record at a 192.168.x.x address for a name that needs HTTP-01.**

### The wildcard DNS record and the wildcard certificate are different things

A frequent source of confusion. `*.tedraykov.me A → 83.228.116.18` is
**routing** — it says where to send traffic. The Cloudflare **API token** on
VM 100 is for the ACME **DNS-01 challenge**: Traefik writes a temporary
`_acme-challenge` TXT record to prove domain control, then deletes it. That is
the only way to obtain a **wildcard TLS certificate**, and the only way to get
any certificate for a host that is not internet-reachable.

So the token is not what makes the names resolve. It is what lets **VM 100**
serve valid HTTPS on the LAN fast path without being internet-reachable.
Remove the token and VM 100's Traefik has no way to obtain a certificate →
you would have to drop it and route all LAN traffic through the DMZ.

### ⚠ IPv6 breaks split-horizon DNS

The Huawei ONT advertises **itself** as a DNS server over IPv6 router
advertisement, and that entry sorts *ahead* of AdGuard on clients:

```
nameserver[0] : fe80::1%en0      <- the router, via IPv6 RA
nameserver[1] : 192.168.1.53     <- AdGuard
```

So a LAN client often resolves via the router, gets the public IP, and reaches
VM 200 instead of VM 100. While every name was public this was invisible; it
surfaced as "n8n has no certificate" when n8n was VM-100-only. Now that all
names work through both paths it is cosmetic — but it means **AdGuard's
wildcard rewrite cannot be relied on** as the only route to anything.

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

Nothing. As of 2026-08-29 **every service is public over HTTPS** — `minio`,
`minio-console`, `n8n` and `beszel` were added to the DMZ path with the same
three-piece pattern, and all hold valid certificates.

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

## Voice assistant — reSpeaker XVF3800 satellite (started 2026-09-02)

A **reSpeaker XVF3800 with an onboard XIAO ESP32-S3**, wake word **"Patrick"**.
It is an ESPHome device on WiFi, *not* a USB microphone — nothing is passed
through from the host, and it never touches virtiofs or the Proxmox USB map.

**The pipeline is ElevenLabs for both halves.** The official HA integration does
STT *and* TTS, so there is no `wyoming-whisper` and no `wyoming-piper` here —
nothing local, no GPU reservation, no RAM on a VM with ~3.7 GiB free. Wake word
detection is the one local piece and runs on the ESP32-S3 itself, so no
`openWakeWord` either. The cost is that every utterance leaves the house and
draws ElevenLabs quota; if that bites, move STT to a local `faster-whisper` on
the 3060 and keep ElevenLabs for TTS — STT is the expensive half by volume.

**The ESPHome Device Builder had to become a container** (stack 14). Every guide
for this board says "Settings → Add-ons → ESPHome Device Builder", and **this HA
install has no add-on store** — it is the plain image, not HAOS or Supervised.
The *integration* that talks to the finished device is in HA core and needs none
of it; the builder is only a compiler and can be stopped once a device is
flashed.

- **https://esphome.tedraykov.me — private, and authenticated.** Host networking
  means Traefik's docker provider cannot see the container, so the route is a
  file-provider entry in `homelab.yml`, not labels. **No DNS record was needed**
  — the zone is a single wildcard and AdGuard's `*.tedraykov.me → 192.168.1.100`
  rewrite already covers the name. There is **no DMZ hole for 6052**, so it is
  not internet-reachable, unlike the rest of the lab.
- **The builder ships with no authentication whatsoever** — that is why it is
  the one exception to "everything public". `ESPHOME_USERNAME`/`ESPHOME_PASSWORD`
  are **Portainer stack variables** (this repo is public), and the compose uses a
  `:?` guard so a missing value fails the deploy rather than silently starting an
  open dashboard that can flash arbitrary firmware. Credentials:
  `/opt/stacks/esphome/.admin-credentials`, mode 600.
  `ESPHOME_TRUSTED_DOMAINS` pins the websocket Origin/Host allowlist the UI rides
  on; `normalize_host()` strips the port, so the name plus `192.168.1.100` covers
  both paths. Unset disables the check.
- Route backup: `/opt/stacks/traefik/dynamic/homelab.yml.bak-preesphome`.
- **The device YAML is not deployed by the stack.** Portainer clones this repo
  to its own directory while the builder reads `/opt/stacks/esphome/config`.
  `esphome/respeaker-patrick.yaml` is copied there by hand, next to a
  gitignored `secrets.yaml`.
- The board needs formatBCE's ESPHome packages — the **stock `i2s_audio`
  component does not work** with it. The array needs a 12.288 MHz MCLK that
  ESPHome cannot generate, so the package firmware makes the XVF3800 the I2S
  master and flashes itself over I2C on first boot. There is no laptop-side DFU
  step, and GPIO9 MCLK must stay disabled.

**⚠ Not finished: the "Patrick" wake word does not exist yet.** The stock
micro_wake_word catalogue is `okay_nabu`, `hey_jarvis`, `hey_mycroft`, `stop` —
that is the whole list. The model must be trained (microwakeword.com) and
dropped at `/opt/stacks/esphome/config/models/patrick.json`, or the build fails.
`esphome/README.md` carries the procedure and the fallback.

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

## Immich — the 1.120.2 → 3.1.0 upgrade, and why it took four deploys

Done 2026-08-31. **Immich cannot be upgraded from 1.120.2 to v3 in one jump**,
and both gates fail closed — the server refuses to start rather than corrupting
anything, but you find out only after the pull.

| Hop | Why it exists |
|---|---|
| → **1.132.3** | v1.137.0 removed TypeORM and refuses to migrate unless Immich has started at least once on **1.132.0–1.136.0**. Skipping it gives `Migrations failed: Error: Invalid upgrade path`. 1.132.3 is what `docs.immich.app/errors#typeorm-upgrade` names; **1.136.0 has a blocking bug** when coming from ≤1.131. |
| → **2.7.5** | v1.133.0 moved off pgvecto.rs to VectorChord, and **v3 deleted the migration code**. v3 cannot convert pgvecto.rs data, so the conversion must run on a 1.133–2.x server. 2.7.5 is the last of those. |
| → **3.1.0** | Ordinary bump once VectorChord is in place. |
| → **/data** | Optional media-location move, kept separate on purpose. |

Immich also refuses to upgrade **from below 1.107.2**, so a stale instance may
need more hops than this.

### The VectorChord migration is automatic here, and a one-way door

`ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0` carries
`vchord`, `vector` **and** `vectors` simultaneously, with
`shared_preload_libraries = 'vchord.so, vectors.so'`. That is the precondition
the migration guide asks for, and because the container connects as superuser
Immich does the whole `CREATE EXTENSION` → reindex → `DROP EXTENSION` sequence
itself. Watch for `Reindexed face_index` / `Reindexed clip_index`. **Do not
restart during it.** At 8622 assets it took under two seconds; large libraries
take tens of minutes and look hung.

The image generates `/etc/postgresql/postgresql.conf` from a template at
startup and passes it with `-c config_file=`, so **the `postgresql.conf` inside
PGDATA is bypassed, not merged.** That is why the tensorchord leftovers there
are harmless. Its entrypoint also reads `POSTGRES_PASSWORD_FILE` in
`set-env.sh` **as root, before `gosu`**, so the secret file's ownership only has
to satisfy root.

Immich leaves the empty `vectors` **schema** behind after dropping the
extension; `DROP SCHEMA vectors RESTRICT;` finishes the job. The template's
`search_path` still names it, which is harmless — Postgres ignores missing
schemas in `search_path`.

**After this you can never run Immich below 1.133.0 against this database.**

### `/data`, not `/usr/src/app/upload`

v1.137.0 changed the default media location. It is genuinely optional — Immich
prefers the legacy path whenever it exists — but this stack moved anyway.
Immich detects the change and rewrites the stored paths itself:

```
Media location changed (from=/usr/src/app/upload, to=/data)
```

It rewrote **27456** absolute paths across `asset.originalPath` (8622),
`asset_file.path` (18617) and `person.thumbnailPath` (217). Reverting is the
same mechanism in reverse, not a restore. This was safe here because the stack
has a **single** mount; installs that split out `ENCODED_VIDEO_LOCATION` and
moved only one of them ended up with broken transcoded video.

Note the paths were *relative* (`upload/library/…`) on 1.120.2 and *absolute*
by 3.1.0 — something in between rewrote them, so do not assume the shape.

### GPU — new, and it never worked before

Immich had **never** used the RTX 3060. The compose file in this repo carried an
nvidia device reservation, but that file was never deployed (stack 1 was an
editor stack), and the plain `immich-machine-learning` image is CPU-only ONNX
anyway — the reservation would have handed it a device it had no runtime for.

Now `immich-machine-learning:<ver>-cuda` with a gpu reservation, and NVENC on
the server via `capabilities: [gpu, compute, video]`, both inlined from
upstream's `hwaccel.*.yml` so Portainer still deploys a single file. Verified
inside the containers: `CUDAExecutionProvider` + `TensorrtExecutionProvider`
available, and `h264_nvenc`/`hevc_nvenc`/`av1_nvenc` present. **No `/dev/dri`**,
same as Jellyfin — NVENC goes through the driver, not DRM, which matters here
because `nvidia_drm` has no modeset and `/dev/dri` does not exist in VM 100.

**The compose only grants the device.** Hardware transcoding still has to be
switched to NVENC in Immich's admin settings.

### Backups taken, and what they are good for

In `/opt/stacks/immich/backup/`, mode 600, `pg_dumpall --clean --if-exists`:
`immich-pre-v3.sql` (1.120.2), `immich-1.132.3-pre-vectorchord.sql`,
`immich-3.1.0-pre-datamove.sql`. The first two are **unrestorable in practice** —
no server that can read them will ever run against this database again — so they
are evidence, not rollback. Delete them once v3 is trusted.

Proxmox snapshot `pre-immich-v3` on VM 100, **disk-only** (`hostpci0` rules out
vmstate). It covers `/opt/stacks/immich/pgdata` and nothing under `/srv/immich`,
because the library is virtiofs from `tank` and lives outside the VM disk. That
is the right shape for a DB rollback and useless as a library backup — see open
item 1.

### Not applicable to this install, but check on the next major

v3's other breaking changes: removed API endpoints and renamed DTO fields (no
third-party API consumers here), `MACHINE_LEARNING_PRELOAD__*` and
`IMMICH_MACHINE_LEARNING_PING_TIMEOUT` removal (never set), OAuth insecure-HTTP
now blocked (no OAuth), and an x86-64-v2 CPU floor (met by a Ryzen 5 5600 as
`cpu: host`).

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

## Declaring Dokploy domains in the compose file

**Yes — and this repo does it.** Dokploy's Traefik runs a **docker provider**
(`exposedByDefault: false`, `network: dokploy-network`), so ordinary Traefik
container labels are read directly. Dokploy clones the repo to
`/etc/dokploy/compose/<appName>/code/` and **does not rewrite the compose
file**, so labels pass through verbatim.

There are two ways a domain can reach Traefik, and they should not be mixed:

| | UI-managed domain | Labels in compose |
|---|---|---|
| Stored in | Dokploy's Postgres (`domain` table) | **this repo** |
| Router names | generated, e.g. `alt-treble-plane-gxu4db-2-web` | whatever you name them |
| Shows in Dokploy UI | yes | no |
| Survives a rebuild from git alone | no — needs the DB | **yes** |
| Used by | `plane` | `excalidash` |

The declarative version is the better default here: the domain lives in git next
to the service, and a rebuild from a clean Dokploy needs nothing but this repo.
The cost is that Dokploy's Domains tab shows nothing, so the UI is not the
source of truth — the file is.

### The template to copy

```yaml
labels:
  - traefik.enable=true
  - traefik.docker.network=dokploy-network
  # HTTPS
  - traefik.http.routers.<name>.rule=Host(`<name>.tedraykov.me`)
  - traefik.http.routers.<name>.entrypoints=websecure
  - traefik.http.routers.<name>.tls=true
  - traefik.http.routers.<name>.tls.certresolver=letsencrypt
  - traefik.http.services.<name>.loadbalancer.server.port=<container port>
  # HTTP -> HTTPS
  - traefik.http.routers.<name>-http.rule=Host(`<name>.tedraykov.me`)
  - traefik.http.routers.<name>-http.entrypoints=web
  - traefik.http.routers.<name>-http.service=<name>
  - traefik.http.routers.<name>-http.middlewares=redirect-to-https@file
```

Four things that are easy to get wrong:

1. **`redirect-to-https@file` must be added by hand.** Dokploy attaches it
   automatically to UI-managed domains; hand-written labels do not get it, so
   plain HTTP is served instead of redirecting. The middleware is defined in
   `/etc/dokploy/traefik/dynamic/middlewares.yml`.
2. **The service must join `dokploy-network`** (external) or the docker
   provider ignores it entirely.
3. `certresolver=letsencrypt` is technically redundant — the `websecure`
   entrypoint already defaults to it — but state it anyway; it documents intent
   and survives a change to the static config.
4. **Do not also add a UI domain for the same host.** You get two router sets
   for one hostname and the winner is arbitrary.

The ACME HTTP-01 challenge still works with the redirect in place: Traefik
answers `/.well-known/acme-challenge/` on `:80` at a higher priority than user
routers. Verified on excalidash — cert issued *after* the redirect was added.

**On VM 100 this does not apply.** That Traefik has *only* a file provider, so
labels there are inert and routes go in `/opt/stacks/traefik/dynamic/homelab.yml`.

## Deploying to Dokploy from this repo

Use the **generic Git provider** (`sourceType: "git"`), not the GitHub one.
The GitHub integration lists only repositories the Dokploy GitHub App can see,
and `alttreble/releases` belongs to an **organization**, not a personal account
— so it does not appear unless the App is installed org-wide. The repo is
public, so generic Git needs no credentials at all and sidesteps this entirely.

Dokploy's REST API is `POST /api/<router>.<procedure>`, authenticated with an
**`x-api-key`** header (create one at Settings → API/Swagger). There is no
`swagger.json` served in v0.30.2 — the input schemas are readable inside the
container at
`/app/node_modules/.pnpm/@dokploy+server@*/node_modules/@dokploy/server/dist/db/schema/<router>.js`.

The working sequence — `compose.create` accepts only a subset of fields, so the
git source has to be set in a second call:

```
POST /api/compose.create   {name, description, environmentId, composeType:"docker-compose", appName, sourceType:"git"}
POST /api/compose.update   {composeId, sourceType:"git", customGitUrl, customGitBranch,
                            composePath:"./<dir>/docker-compose.yml", env:"KEY=value\n...", autoDeploy:true}
POST /api/compose.deploy   {composeId}
```

`env` is a single newline-delimited string, and it is stored in **Dokploy's own
database** — acceptable for services you own, but it is why the Cloudflare
token must never be handed to Dokploy.

**Remove any hand-rolled stack first.** A compose with an explicit
`container_name` will collide with the Dokploy deployment and the deploy fails.

| Service | composeId | Source |
|---|---|---|
| `plane` | `RwDSRUW6bYlF8ZP3IZTQH` | `raw` — compose pasted into the UI |
| `excalidash` | `dVMNVxK1ljWL5o2nhWloP` | **`git`** — `alttreble/releases` @ `trunk`, `./excalidraw/docker-compose.yml` |

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
| excalidash frontend/backend | up, HTTP-only, not public |
| beszel-agent | up |

`minio` and `n8n` **left VM 200 on 2026-08-29** — data migrated to VM 100 and
the DMZ copies deleted, MinIO's root credentials shredded. VM 200's only
remaining secret is `excalidash.env`.

`excalidash-backend` reports **unhealthy** — pre-existing, and true on `sof1`
too. Its healthcheck curls `/health` over localhost HTTP but `TRUST_PROXY=1` +
`FRONTEND_URL` makes the app 302 it to HTTPS, so it can never return 200. The
app serves correctly. Not worth fixing.

## Parallel-run safety

`n8n` is **cutover-only** — its workflows are *active* (`Домоуправител`,
`Домоуправител — разнасяне на вноски`, `OLX Watcher`) and would fire twice if
two instances ran. It now runs **only on VM 100**. Never start a second copy.

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
code, treat as hostile"*. **Cleaned up 2026-08-29**: perfume-app's Anthropic key and Turso token were
shredded with its orphaned containers, and MinIO's root credentials left with
MinIO itself. Both sets should still be **rotated at source** — deletion does
not undo prior exposure. VM 200's only remaining secret is excalidash's
JWT/CSRF pair, plus whatever Dokploy's own database holds.

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
existing stacks did not have to be redeployed onto a shared network.
**Traefik on VM 100 has no Docker provider at all**, so container labels there
are inert; they are kept only for parity and future use. To add a service:
publish its port, add a router+service pair, restart traefik.

`tls.yml` sets a `defaultGeneratedCert` for `tedraykov.me` + `*.tedraykov.me`
via the `cloudflare` resolver, so a router only needs `tls: {}` — never name a
resolver per router.

**Gotcha:** `homelab.yml` ends with a `serversTransports:` section *after*
`services:`. Appending a service at EOF silently nests it under
`serversTransports` and Traefik fails with `field not found, node: loadBalancer`.
Insert before that section. Backups: `homelab.yml.bak-*`.

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
2. **Train the "Patrick" wake word model.** No such model ships with
   micro_wake_word, so the satellite runs `hey_jarvis` meanwhile. Train it (see
   `esphome/README.md`), drop `patrick.json` + `patrick.tflite` in
   `/opt/stacks/esphome/config/models/`, repoint `wake_word_model`, and push it
   over OTA. `secrets.yaml` is complete and the config validates; what remains
   is the first USB flash and adoption in HA.
3. **Immich post-upgrade, in the web UI** — the compose file cannot do either:
   - Admin → Video Transcoding → **Hardware Acceleration = NVENC**. The
     container has the device; nothing uses it until this is set.
   - Re-run the **Metadata Extraction** job over pre-v3 assets, per the v3
     migration guide.
4. **`excalidraw` is the last of yours still in the DMZ.** Same recipe as the
   minio/n8n move: stop on VM 200 → copy `/opt/appdata/excalidash` →
   `/opt/stacks/excalidraw/prisma` on VM 100 → add a route to `homelab.yml` →
   drop the AdGuard `.200` override. Low urgency: it holds only drawings.
5. **Rotate** the Anthropic key, Turso token and MinIO root credentials — they
   sat on the DMZ VM for a week.
6. **Migrate stacks 6 (traefik) and 10 (beszel) to git-backed** Portainer
   stacks — the last two that are not. `beszel/` now has a compose file here,
   so it only needs converting; traefik still needs one written out. Editing
   this repo changes nothing for either until they are converted. Note that
   traefik's stack carries the
   Cloudflare token via a bind-mounted secret file, which is what makes it
   safe to move into a public repo; use the same `_FILE` pattern immich now
   uses. Converting reallocates the stack id (immich went 1 → 13).
7. **WireGuard** — UDP 51820 forward not yet configured. No remote access
   except through the public services.
8. **Dormant stacks** carry stale `*.treble.bg` labels and `tlsresolver`. Fix
   before deploying any.
9. Dokploy cleanup schedule, per-app memory/CPU limits, S3 database backups.
10. `rpool/immich` holds ~2.2 G of orphaned data from the aborted WiFi rsync.
11. Delete Proxmox snapshot `pre-immich-v3` on VM 100 and the two unrestorable
    dumps in `/opt/stacks/immich/backup/` once v3.1.0 is trusted.
12. Later: second node + QDevice for quorum. Note the mirror protects against a
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

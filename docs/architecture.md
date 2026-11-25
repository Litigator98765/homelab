# Architecture — Pi Homelab

**Last updated:** 2026-11-24 · **Host:** `raspberrypi` · **Repo:** `Litigator98765/homelab` (private)

How the homelab is put together. For what was done and the commands used, see
[`setup.md`](setup.md). To rebuild from scratch, see [`rebuild.md`](rebuild.md).

---

## 1. Overall architecture

A single Raspberry Pi 4 runs two kinds of software:

- **Host services** (managed by systemd, outside Docker): `sshd`, `tailscaled`, `smbd`
  (Samba file share), the Docker Engine + containerd, and the Claude Code CLI.
- **Docker containers** (defined in `/mnt/ssd/compose/*/docker-compose.yml`): Portainer,
  Jellyfin, Beszel (hub + agent) and the four-container Immich stack.

Storage is split across two drives:

- **microSD (64 GB):** the OS only. The Pi always boots, even without the SSD.
- **SanDisk USB SSD (1 TB) at `/mnt/ssd`:** Docker and containerd data, the compose
  projects (this repo), media and the Samba share.

Devices reach the Pi two ways: **home Wi-Fi** (`raspberrypi.local`) when at home, and
**Tailscale** (`raspberrypi` / `100.x.x.x`) from anywhere. No ports are opened on the
router, so nothing is exposed to the public internet.

```mermaid
flowchart TB
    %% ── Devices ─────────────────────────────────────────
    Phone["📱 Phone<br/>Immich app · Tailscale app"]
    Laptop["💻 Laptop<br/>Browser · SSH · File Explorer"]
    Roku["📺 Roku<br/>Jellyfin app"]

    %% ── Network paths ───────────────────────────────────
    subgraph NET["🌐 Network paths"]
        direction LR
        WIFI["🏠 Home Wi-Fi<br/>raspberrypi.local"]
        TS["🔒 Tailscale VPN<br/>raspberrypi · 100.x.x.x<br/>no open router ports"]
    end

    %% ── Raspberry Pi ────────────────────────────────────
    subgraph PI["🍓 Raspberry Pi 4 · 4 GB RAM · Raspberry Pi OS 64-bit"]
        direction TB

        subgraph HOST["Host services (systemd, outside Docker)"]
            direction LR
            SSHD["sshd<br/>:22"]
            TSD["tailscaled"]
            SMB["📂 smbd (Samba)<br/>:445"]
            DOCKERD["🐳 Docker Engine<br/>+ containerd"]
            CLAUDE["🤖 Claude Code CLI"]
        end

        subgraph DOCKER["Docker containers (/mnt/ssd/compose)"]
            direction LR
            PORT["📊 Portainer CE<br/>:9443"]
            JF["🎬 Jellyfin<br/>:8096 (host network)"]
            subgraph BESZEL["📈 Beszel"]
                direction LR
                BHUB["beszel hub<br/>:8090"]
                BAG["beszel-agent<br/>(host network)"]
            end
            subgraph IMMICH["📷 Immich stack"]
                direction LR
                SRV["immich_server<br/>:2283"]
                ML["immich_machine_learning<br/>(paused)"]
                PG[("immich_postgres")]
                RD[["immich_redis"]]
            end
        end

        subgraph STORAGE["Storage"]
            direction LR
            SD[("💾 microSD 64 GB<br/>OS only")]
            SSD[("⚡ SSD 1 TB · /mnt/ssd<br/>docker · containerd · compose<br/>media · share")]
        end
    end

    %% ── Off the Pi ──────────────────────────────────────
    subgraph OFF["☁️ Off the Pi"]
        direction LR
        GH["GitHub (private)<br/>homelab repo · config only"]
        BK["AWS S3 Glacier Deep Archive<br/>photo backup (to do)"]
    end

    %% ── Access ──────────────────────────────────────────
    Phone -->|"at home"| WIFI
    Phone -->|"away"| TS
    Laptop --> WIFI
    Laptop -.->|"away"| TS
    Roku -->|"home only"| WIFI
    TS --> TSD
    WIFI -->|":2283 :8096 :8090 :9443"| DOCKER
    WIFI -->|"SMB share"| SMB
    TSD -->|":2283 / :9443"| DOCKER
    TSD -->|":22"| SSHD

    %% ── Inside the Pi ───────────────────────────────────
    DOCKERD -->|"runs"| DOCKER
    PORT -.->|"docker.sock"| DOCKERD
    BAG -.->|"docker.sock (ro)"| DOCKERD
    BAG -->|"unix socket"| BHUB
    SRV --> PG
    SRV --> RD
    SRV -.-> ML
    SRV -->|"photos"| SSD
    PG -->|"database files"| SSD
    JF -->|"/mnt/ssd/media (ro)"| SSD
    SMB -->|"/mnt/ssd/share"| SSD
    DOCKERD -->|"images · volumes"| SSD

    %% ── Off-Pi flows ────────────────────────────────────
    SSD -.->|"git push"| GH
    SSD -.->|"planned"| BK

    classDef main fill:#fff3b0,stroke:#333,stroke-width:2px
    classDef planned fill:#ffffff,stroke:#888,stroke-dasharray:5 5
    class SRV,JF main
    class ML,BK planned
```

**Legend:** yellow = main apps · dashed box = planned or paused · dashed arrow = occasional or planned flow

---

## 2. What each component does

| Component | Type | Role | Address | Restart policy |
|---|---|---|---|---|
| `immich_server` | Docker | Web UI, API, uploads, thumbnails, video transcoding, job workers | `:2283` | `always` |
| `immich_machine_learning` | Docker | Face recognition + smart search (CLIP). **Paused** to save RAM | internal | `always` |
| `immich_postgres` | Docker | Photo index: assets, albums, people, search vectors | internal | `always` |
| `immich_redis` | Docker | Background job queue (Valkey) | internal | `always` |
| `jellyfin` | Docker | Media server for movies and shows (Roku, browser). Host network so Roku discovery works | `:8096` | `unless-stopped` |
| `beszel` | Docker | Monitoring dashboard: history for CPU, RAM, temp, disks, network, per-container stats | `:8090` | `unless-stopped` |
| `beszel-agent` | Docker | Collects stats for the hub; talks to it over a unix socket | none (socket) | `unless-stopped` |
| Portainer CE 2.45 | Docker | Dashboard for containers: status, stats, logs, console | `:9443` (HTTPS) | `unless-stopped` |
| Docker Engine + containerd | Host | Run all containers. Data on the SSD; wait for `/mnt/ssd` to mount | — | systemd, on boot |
| smbd (Samba) | Host | Windows file share of `/mnt/ssd/share`. Waits for `/mnt/ssd` | `:445` | systemd, on boot |
| tailscaled | Host | Private encrypted network to phone + laptop | `raspberrypi` / `100.x.x.x` | systemd, on boot |
| sshd | Host | Remote terminal | `:22` | systemd, on boot |
| Claude Code | Host CLI | AI assistant in the terminal | `claude` | n/a |

---

## 3. Sequence — phone photo backup

What happens when the Immich app uploads a photo (shown via Tailscale; at home the
Wi-Fi path replaces the Tailscale hop).

```mermaid
sequenceDiagram
    participant P as 📱 Immich app
    participant T as 🔒 Tailscale
    participant S as immich_server
    participant R as immich_redis
    participant DB as immich_postgres
    participant D as ⚡ library/ on SSD
    participant ML as immich_machine_learning

    P->>T: upload new photo (away from home)
    T->>S: HTTP POST :2283 over encrypted tunnel
    S->>D: write original to library/YYYY/YYYY-MM/
    S->>DB: save asset record + metadata
    S->>R: queue jobs (thumbnail, EXIF, ML)
    S-->>P: upload accepted
    loop background workers, concurrency 1
        S->>R: take next job
        S->>D: write thumbnails / previews
        S->>DB: store EXIF, date, location
    end
    alt ML enabled
        S->>ML: detect faces + build search embedding
        ML-->>S: faces, vectors
        S->>DB: save faces + vectors
    else ML paused (current setting)
        S->>S: skip, run later via Jobs → Missing
    end
```

---

## 4. Sequence — what happens at boot

Everything comes back on its own after a reboot or power loss. Anything whose data is on the
SSD waits for it to mount (`RequiresMountsFor=/mnt/ssd`). If the SSD is missing, the Pi still
boots (`nofail`), SSH and Tailscale work, and Docker + Samba stay stopped instead of writing
to the SD card.

```mermaid
sequenceDiagram
    participant HW as 🍓 Pi hardware
    participant SYS as systemd
    participant SSD as ⚡ /mnt/ssd
    participant TSD as tailscaled
    participant DK as 🐳 containerd + Docker
    participant SMB as 📂 smbd
    participant C as Containers

    HW->>SYS: power on, kernel boots from microSD (cgroup_enable=memory)
    SYS->>TSD: start tailscaled (doesn't need the SSD)
    SYS->>SSD: mount by UUID (nofail, 10 s timeout)
    alt SSD mounted
        SYS->>DK: start containerd, then docker
        SYS->>SMB: start smbd
        DK->>C: start portainer, jellyfin, beszel (unless-stopped)
        DK->>C: start 4 Immich containers (always)
        C-->>HW: Immich ready on :2283 after 1-2 min
    else SSD missing
        SYS--xDK: not started (RequiresMountsFor)
        SYS--xSMB: not started
        Note over SYS: Pi still reachable over SSH to fix it
    end
```

---

## 5. Folder layout and what git tracks

Green = committed to GitHub · red dashed = ignored (secrets or data).

```mermaid
flowchart LR
    SSDROOT["⚡ /mnt/ssd"] --> DOCK["📁 compose/<br/>git repo → GitHub"]
    SSDROOT --> MEDIA["📁 media/<br/>Movies · Shows"]
    SSDROOT --> SHARE["📁 share/<br/>Samba share"]
    SSDROOT --> ENG["📁 docker/ · containerd/<br/>images, volumes"]

    DOCK --> PORT["📁 portainer/"]
    DOCK --> IMM["📁 immich/"]
    DOCK --> JF["📁 jellyfin/"]
    DOCK --> BZ["📁 beszel/"]
    DOCK --> DOCS["📁 docs/<br/>setup.md · architecture.md · rebuild.md"]

    PORT --> P1["docker-compose.yml"]
    PORT --> P2["data/"]
    IMM --> I1["docker-compose.yml<br/>.env.example"]
    IMM --> I2[".env"]
    IMM --> I3["library/<br/>postgres/"]
    JF --> J1["docker-compose.yml<br/>.env.example"]
    JF --> J2[".env<br/>config/ · cache/"]
    BZ --> B1["docker-compose.yml<br/>.env.example"]
    BZ --> B2[".env (key + token)<br/>beszel_data/ · beszel_agent_data/<br/>beszel_socket/"]

    classDef tracked fill:#e6f4ea,stroke:#2e7d32
    classDef ignored fill:#fdecea,stroke:#c62828,stroke-dasharray:4 3
    class P1,I1,J1,B1,DOCS tracked
    class P2,I2,I3,J2,B2,MEDIA,SHARE,ENG ignored
```

Not in git and not on the SSD: `/etc/fstab`, `/etc/docker/daemon.json`,
`/etc/containerd/config.toml`, the `RequiresMountsFor` drop-ins in
`/etc/systemd/system/*.service.d/`, `/etc/samba/smb.conf`, `/var/lib/tailscale` (login keys —
secret) and `/boot/firmware/cmdline.txt`. These are all on the SD card; `rebuild.md` has the
commands to recreate them.

---

## 6. Key design decisions

- **OS on the SD card, everything else on the SSD** — the Pi always boots and stays reachable
  over SSH even if the SSD is unplugged or dies. `nofail` + `RequiresMountsFor` stop Docker and
  Samba from silently writing to the SD card when the SSD is missing, and the empty mount
  point is made immutable (`chattr +i`) as a second guard.
- **Both Docker and containerd moved** — Docker 29 stores images in containerd, so moving
  only Docker's `data-root` would leave images on the SD card.
- **Docker Compose files in a git repo, not Portainer stacks** — config stays in plain files
  that git can track and that move as one folder; Portainer only monitors.
- **Tailscale and Samba on the host, not in containers** — remote access, SSH and the file
  share keep working even if Docker breaks; Samba writes files as the real `litigator` user.
- **Beszel for monitoring** — light enough to run next to Immich + Jellyfin on 4 GB, keeps
  history, and the hub ↔ agent link is a unix socket, so the agent opens no network port.
- **No public exposure** — no router port-forwarding; all remote access is over Tailscale.
  Jellyfin is home-only (Roku).
- **Jellyfin on the host network** — Roku finds servers with a UDP broadcast that Docker's
  bridge network doesn't forward.
- **Immich ML paused** — the Pi has 4 GB of RAM (soldered, not upgradeable). Re-enable and run
  Jobs → Missing now that data is on the SSD.
- **Immich storage template `{{y}}/{{y}}-{{MM}}/{{filename}}`** — photos on disk stay
  human-readable, so there is no lock-in if Immich is ever replaced.
- **Secrets never in git** — `.env` files (Immich DB password, Beszel key + token),
  `portainer/data/` and Tailscale keys are ignored; `.env.example` files document the settings.

---

## 7. Planned changes

| Change | Effect on this diagram |
|---|---|
| Re-enable Immich ML | `ML` no longer dashed |
| Photo backup to AWS S3 Glacier Deep Archive | `BK` becomes solid with a scheduled arrow |
| Samba `[media]` share | Second arrow from `smbd` to `/mnt/ssd/media` |
| Tailscale on the laptop | Laptop's dashed Tailscale arrow becomes solid |

# Architecture — Pi Homelab

**Last updated:** 2026-11-22 · **Host:** `raspberrypi` · **Repo:** `Litigator98765/homelab` (private)

How the homelab is put together. For what was done and the commands used, see
[`setup.md`](setup.md). To rebuild from scratch, see [`rebuild.md`](rebuild.md).

---

## 1. Overall architecture

A single Raspberry Pi 4 runs two kinds of software:

- **Host services** (managed by systemd, outside Docker): `sshd`, `tailscaled`, the
  Docker Engine, and the Claude Code CLI.
- **Docker containers** (defined in `~/docker/*/docker-compose.yml`): Portainer and
  the four-container Immich stack.

Devices reach the Pi two ways: **home Wi-Fi** (`raspberrypi.local`) when at home, and
**Tailscale** (`raspberrypi` / `100.x.x.x`) from anywhere. No ports are opened on the
router, so nothing is exposed to the public internet.

```mermaid
flowchart TB
    %% ── Devices ─────────────────────────────────────────
    Phone["📱 Phone<br/>Immich app · Tailscale app"]
    Laptop["💻 Laptop<br/>Browser · SSH · Tailscale"]

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
            DOCKERD["🐳 Docker Engine<br/>v29.10 · Compose v5.8"]
            CLAUDE["🤖 Claude Code CLI<br/>~/.local/bin/claude"]
        end

        subgraph DOCKER["Docker containers (~/docker)"]
            direction LR
            PORT["📊 Portainer CE<br/>:9443"]
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
            SD[("💾 microSD 64 GB<br/>OS · Docker · photos")]
            SSD[("USB SSD<br/>(coming)")]
        end
    end

    %% ── Off the Pi ──────────────────────────────────────
    subgraph OFF["☁️ Off the Pi"]
        direction LR
        GH["GitHub (private)<br/>homelab repo · config only"]
        BK["Photo backup<br/>(to do)"]
    end

    %% ── Access ──────────────────────────────────────────
    Phone -->|"at home"| WIFI
    Phone -->|"away"| TS
    Laptop -->|"anywhere"| TS
    TS --> TSD
    WIFI -->|":2283 / :9443"| DOCKER
    TSD -->|":2283 / :9443"| DOCKER
    TSD -->|":22"| SSHD

    %% ── Inside the Pi ───────────────────────────────────
    DOCKERD -->|"runs"| DOCKER
    PORT -.->|"docker.sock"| DOCKERD
    SRV --> PG
    SRV --> RD
    SRV -.-> ML
    SRV -->|"photos"| SD
    PG -->|"database files"| SD

    %% ── Off-Pi flows ────────────────────────────────────
    SD -.->|"rpi-clone"| SSD
    SD -.->|"git push"| GH
    SD -.->|"planned"| BK

    classDef main fill:#fff3b0,stroke:#333,stroke-width:2px
    classDef planned fill:#ffffff,stroke:#888,stroke-dasharray:5 5
    class SRV main
    class ML,SSD,BK planned
```

**Legend:** yellow = main app · dashed box = planned or paused · dashed arrow = occasional or planned flow

---

## 2. What each component does

| Component | Type | Role | Address | Restart policy |
|---|---|---|---|---|
| `immich_server` | Docker | Web UI, API, uploads, thumbnails, video transcoding, job workers | `:2283` | `always` |
| `immich_machine_learning` | Docker | Face recognition + smart search (CLIP). **Paused** to save RAM | internal | `always` |
| `immich_postgres` | Docker | Photo index: assets, albums, people, search vectors | internal | `always` |
| `immich_redis` | Docker | Background job queue (Valkey) | internal | `always` |
| Portainer CE 2.45 | Docker | Dashboard for containers: status, stats, logs, console | `:9443` (HTTPS) | `unless-stopped` |
| Docker Engine | Host | Runs all containers | — | systemd, on boot |
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
    participant D as 💾 library/ on disk
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

Everything comes back on its own after a reboot or power loss.

```mermaid
sequenceDiagram
    participant HW as 🍓 Pi hardware
    participant SYS as systemd
    participant TSD as tailscaled
    participant DK as 🐳 Docker Engine
    participant C as Containers

    HW->>SYS: power on, kernel boots with cgroup_enable=memory
    SYS->>TSD: start tailscaled
    TSD-->>TSD: rejoin tailnet as raspberrypi
    SYS->>DK: start docker.service
    DK->>C: start portainer (restart unless-stopped)
    DK->>C: start 4 Immich containers (restart always)
    C-->>HW: Immich ready on :2283 after 1-2 min
```

---

## 5. Folder layout and what git tracks

Green = committed to GitHub · red dashed = ignored (secrets or data).

```mermaid
flowchart LR
    HOME["🏠 ~ (/home/litigator)"] --> DOCK["📁 docker/<br/>git repo → GitHub"]
    DOCK --> PORT["📁 portainer/"]
    DOCK --> IMM["📁 immich/"]
    DOCK --> JF["📁 jellyfin/ (empty)"]
    DOCK --> ROOT["📄 .gitignore<br/>📄 setup-host.sh (planned)"]
    DOCK --> DOCS["📁 docs/<br/>setup.md · architecture.md · rebuild.md"]

    PORT --> P1["docker-compose.yml"]
    PORT --> P2["data/"]
    IMM --> I1["docker-compose.yml<br/>.env.example"]
    IMM --> I2[".env"]
    IMM --> I3["library/<br/>postgres/"]

    classDef tracked fill:#e6f4ea,stroke:#2e7d32
    classDef ignored fill:#fdecea,stroke:#c62828,stroke-dasharray:4 3
    class P1,I1,ROOT,DOCS tracked
    class P2,I2,I3 ignored
```

Data that lives **outside** `~/docker` and is not in git: `/var/lib/docker` (images,
volumes), `/var/lib/tailscale` (login keys — secret), `/boot/firmware/cmdline.txt`
(+ `.bak`).

---

## 6. Key design decisions

- **Docker Compose files in `~/docker`, not Portainer stacks** — config stays in plain
  files that git can track and that move with the SSD; Portainer only monitors.
- **Tailscale on the host, not in a container** — remote access (including SSH) keeps
  working even if Docker breaks.
- **No public exposure** — no router port-forwarding; all remote access is over
  Tailscale.
- **Immich ML paused on the SD card** — the Pi has 4 GB of RAM (soldered, not
  upgradeable). Re-enable after moving to the SSD and run Jobs → Missing.
- **Immich storage template `{{y}}/{{y}}-{{MM}}/{{filename}}`** — photos on disk stay
  human-readable, so there is no lock-in if Immich is ever replaced.
- **Secrets never in git** — `.env`, `portainer/data/`, and Tailscale keys are ignored;
  `.env.example` documents the settings with a placeholder password.

---

## 7. Planned changes

| Change | Effect on this diagram |
|---|---|
| Clone SD → USB SSD and boot from it | `SD` replaced by `SSD` as the storage node |
| Re-enable Immich ML | `ML` no longer dashed |
| Photo backup (2nd drive / Backblaze B2) | `BK` becomes solid with a scheduled arrow |
| Jellyfin + powered external HDD | New container under Docker; HDD node under Storage |

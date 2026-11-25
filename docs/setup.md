# Raspberry Pi Homelab — Setup Log

**Device:** Raspberry Pi 4 Model B Rev 1.5 · 4 GB RAM · Raspberry Pi OS 64-bit (aarch64)
**Hostname / user:** `raspberrypi` / `litigator`
**Home LAN IP:** `192.168.1.247` (reserve in router — Roku connects by IP)
**Storage:** 64 GB microSD (OS only) + 1 TB SanDisk USB SSD at `/mnt/ssd` (Docker, compose projects, media, file share)
**Repo:** github.com/Litigator98765/homelab (private) — config only, no secrets or data
**Architecture:** [`architecture.md`](architecture.md) · **Rebuild:** [`rebuild.md`](rebuild.md)

---

## 2026-11-22

### System checks
```bash
cat /proc/device-tree/model     # Raspberry Pi 4 Model B Rev 1.5
free -h                         # 3.7 GiB total, 2 GiB swap
uname -m                        # aarch64 (64-bit)
df -h /                         # 59 GB card, 47 GB free
```

### Docker (already installed)
```bash
docker --version                # 29.10.0
docker compose version          # v5.8.0
```
Starts on boot via systemd. No reinstall needed.

### Portainer — rebuilt (forgot old password)
Old container: created ~5 months earlier, port 9000, settings in volume `portainer_data`.
```bash
docker stop portainer && docker rm portainer
docker volume rm portainer_data
mkdir -p ~/docker/portainer ~/docker/immich ~/docker/jellyfin
# wrote ~/docker/portainer/docker-compose.yml (port 9443, ./data:/data, restart: unless-stopped)
cd ~/docker/portainer
docker compose pull && docker compose up -d
docker image prune -f           # reclaimed 229 MB
docker logs portainer 2>&1 | grep setup_token   # one-time setup token for first login
```
- URL: `https://raspberrypi:9443` (Portainer CE 2.45.4 LTS)
- Edge Compute: not used (single host)

### Immich — installed
```bash
cd ~/docker/immich
wget -O docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env
sed -i "s/^DB_PASSWORD=.*/DB_PASSWORD=$(openssl rand -hex 16)/" .env
sed -i "s|^# TZ=.*|TZ=America/New_York|" .env
docker compose up -d
```
- Containers: `immich_server`, `immich_machine_learning`, `immich_postgres`, `immich_redis` — all healthy, `restart: always`
- `UPLOAD_LOCATION=./library`, `DB_DATA_LOCATION=./postgres` (on SD card for now)
- URL: `http://raspberrypi:2283` (Tailscale) / `http://raspberrypi.local:2283` (home)
- Onboarding: Map **on**, Version check **on**, Storage template **year/month** (`{{y}}/{{y}}-{{MM}}/{{filename}}`)
- **Machine learning turned OFF** in Admin → Settings (RAM; re-enable after SSD move)
- Phone app connected; test batch only — full camera-roll backup waits for SSD

### Tailscale — installed on host (not Docker)
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --qr          # browser login failed with ERR_SSL_PROTOCOL_ERROR; QR on phone worked
sudo tailscale logout           # first login used wrong email
sudo tailscale up --qr          # re-logged with correct account
tailscale status
```
- Phone connected and verified on cellular

### Git repo for config
```bash
cd ~/docker
git init
# .gitignore: .env, immich/library/, immich/postgres/, portainer/data/, jellyfin/config/, jellyfin/cache/
cp immich/.env immich/.env.example   # password replaced with "changeme"
git config --global user.name "Max Miller"
git config --global user.email "<github email>"
git add . && git commit -m "Initial homelab config: Portainer + Immich"
git branch -M main
git remote add origin https://github.com/Litigator98765/homelab.git
git push -u -f origin main     # overwrote GitHub's starter commit
```

### Claude Code — installed
```bash
curl -fsSL https://claude.ai/install.sh | bash     # v2.1.330 → ~/.local/bin/claude
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc
```

### Container memory stats enabled
`docker stats` showed `0B / 0B` (memory cgroup disabled by default on Pi OS).
```bash
sudo cp /boot/firmware/cmdline.txt /boot/firmware/cmdline.txt.bak
sudo sed -i '$ s/$/ cgroup_enable=memory cgroup_memory=1/' /boot/firmware/cmdline.txt
sudo reboot
```
Rollback if the Pi won't boot: put the SD card in another computer and copy `cmdline.txt.bak` over `cmdline.txt`.

---

## 2026-11-23

### Jellyfin — installed (chosen over Plex)
Why Jellyfin: free with no account or cloud dependency; free hardware transcoding if it later moves to the old desktop PC. Media streaming is home-only (Roku), so Plex's remote paywall was not a factor.

```bash
cd ~/docker/jellyfin
mkdir -p config cache ~/media/Movies ~/media/Shows
echo 'MEDIA_DIR=/home/litigator/media' > .env
cp .env .env.example
docker compose up -d
```

`~/docker/jellyfin/docker-compose.yml` (final):
```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    user: 1000:1000
    network_mode: host
    volumes:
      - ./config:/config
      - ./cache:/cache
      - ${MEDIA_DIR}:/media:ro
    environment:
      - TZ=America/New_York
    mem_limit: 1g
    restart: unless-stopped
```
- URL: `http://192.168.1.247:8096` / `http://raspberrypi.local:8096`
- Libraries: Movies → `/media/Movies`, Shows → `/media/Shows` (container paths)
- Trickplay + chapter image extraction **off**; hardware acceleration **None** (Pi 4)
- Media on SD card for testing only; change `MEDIA_DIR` in `.env` when the media drive is mounted

**Issue — Roku couldn't find the server (laptop worked):**
- Cause: Jellyfin clients find servers with a UDP **broadcast** on port 7359. With Docker's default bridge network, published ports (`ports:`) only forward traffic addressed to the Pi, so broadcasts never reached the container. Adding `7359:7359/udp` did not help.
- Fix: replaced `ports:` with `network_mode: host` — the container shares the Pi's network directly and hears the broadcast. `docker ps` shows an empty PORTS column in this mode (expected). No conflicts: Immich 2283, Portainer 9443.

---


## 2026-11-24

### Storage plan changed: OS stays on the SD card, data moves to the SSD
The original plan was to `rpi-clone` the SD card to the SSD and boot from it. Instead, the
OS stays on the SD card so **the Pi always boots, even if the SSD is unplugged or fails**,
and everything else lives on the SSD at `/mnt/ssd`.

### SSD — formatted and mounted at `/mnt/ssd`
- Reformatted as a single GPT partition `/dev/sda1`, **ext4**, label `ssd`.
- Mount point protected so nothing can be written to the SD card by mistake when the SSD
  isn't mounted:
  ```bash
  sudo mkdir -p /mnt/ssd
  sudo chattr +i /mnt/ssd          # immutable while EMPTY → writes fail instead of filling the SD card
  ```
- Mounted by UUID in `/etc/fstab` (backup at `/etc/fstab.bak`):
  ```
  UUID=<ssd-uuid>  /mnt/ssd  ext4  defaults,noatime,nofail,x-systemd.device-timeout=10s  0  2
  ```
  `noatime` = don't write "last read" times · `nofail` = boot continues without the SSD ·
  `x-systemd.device-timeout=10s` = only wait 10 s for it · `2` = fsck it after the root FS.
- After mounting: `sudo chown litigator:litigator /mnt/ssd`.

### Docker + containerd data moved to the SSD
Docker 29 uses the **containerd image store** (`docker info` → driver `overlayfs`), so images
live under containerd, not Docker — **both** had to move.

```bash
# Docker
echo '{ "data-root": "/mnt/ssd/docker" }' | sudo tee /etc/docker/daemon.json

# containerd (backup: /etc/containerd/config.toml.bak) — inserted as line 1:
#   root = "/mnt/ssd/containerd"

# Don't start either service until the SSD is mounted
for s in docker containerd; do
  sudo mkdir -p /etc/systemd/system/$s.service.d
  printf '[Unit]\nRequiresMountsFor=/mnt/ssd\n' | sudo tee /etc/systemd/system/$s.service.d/ssd.conf
done
sudo systemctl daemon-reload
```
- Verified: `docker info` → Docker Root Dir `/mnt/ssd/docker`; containerd root `/mnt/ssd/containerd`;
  images, containers and Portainer all OK.
- containerd logs a harmless "migrated from version 0" config warning.
- Swap is zram (in RAM) — nothing to move.

### Compose projects + media moved to the SSD
This repo moved from `~/docker` to **`/mnt/ssd/compose`**, and media from `~/media` to
**`/mnt/ssd/media`**.

```bash
# 1. Stop everything (from the old folders)
cd ~/docker/portainer && docker compose down
cd ~/docker/jellyfin  && docker compose down
cd ~/docker/immich    && docker compose down

# 2. Copy, keeping owners, permissions, hard links, ACLs and xattrs
sudo rsync -aHAX --info=progress2 /home/litigator/docker/ /mnt/ssd/compose/
sudo rsync -aHAX --info=progress2 /home/litigator/media/  /mnt/ssd/media/

# 3. Jellyfin was the only project with an absolute path
sed -i 's|^MEDIA_DIR=.*|MEDIA_DIR=/mnt/ssd/media|' /mnt/ssd/compose/jellyfin/.env

# 4. Start from the new location (same folder names → same compose project names)
cd /mnt/ssd/compose/portainer && docker compose up -d
cd /mnt/ssd/compose/jellyfin  && docker compose up -d
cd /mnt/ssd/compose/immich    && docker compose up -d
```
- Immich (`./library`, `./postgres`) and Portainer (`./data`) use relative paths → no edits.
- Jellyfin's container-side path is still `/media`, so its libraries didn't change.
- Check every container runs from the SSD:
  ```bash
  docker ps -a --format 'table {{.Names}}\t{{.Status}}\t{{.Label "com.docker.compose.project.working_dir"}}'
  ```
- **Gotchas:** files owned by container users need `sudo` for `rsync`/`du`. If a container
  gets started from the old folder by mistake: stop it, re-sync with `rsync -aHAX --delete`,
  then start it from `/mnt/ssd/compose`.

### Beszel — system monitoring
Lightweight dashboard with history for the whole Pi (CPU, RAM, temperature, network, SD card
and SSD) and per-container stats. Chosen over Grafana + Prometheus (too heavy next to
Immich + Jellyfin on 4 GB) and Glances (no history).

```bash
mkdir -p /mnt/ssd/compose/beszel /mnt/ssd/.beszel
cd /mnt/ssd/compose/beszel
# wrote docker-compose.yml (see beszel/docker-compose.yml in this repo)
docker compose up -d beszel       # hub only first — the agent needs a key + token from it
```
- Hub: `http://raspberrypi.local:8090` → created admin account → **Add System**:
  name `raspberrypi`, Host/IP `/beszel_socket/beszel.sock`; copied the Public Key + Token
  into the agent's `KEY` / `TOKEN` (values keep their quotes), then `docker compose up -d`.
- Hub and agent talk over a **unix socket** in `./beszel_socket` — no agent port on the network.
- Agent runs `network_mode: host` (sees the Pi's real network traffic) and reads
  `docker.sock` read-only for container stats.
- SSD monitoring: an empty marker folder on the SSD is mounted at
  `/extra-filesystems/sda1__SSD` (`sda1` = device for I/O stats, `SSD` = chart label).
  The SD card is the root filesystem and is monitored automatically.

### Samba — file share from Windows
Installed **on the host**, not in Docker: keeps working if Docker breaks, and files are owned
by the real `litigator` user (same UID 1000 that Jellyfin runs as).

```bash
sudo apt install samba -y
mkdir /mnt/ssd/share
sudo smbpasswd -a litigator       # Samba has its own password list
```
Share block at the bottom of `/etc/samba/smb.conf` (replaced the Ubuntu guide's
`[sambashare]`, whose `path = /home/username/sambashare` placeholder was never filled in):
```ini
[share]
   comment = Pi SSD share
   path = /mnt/ssd/share
   read only = no
   valid users = litigator
```
```bash
testparm -s                       # check config
sudo systemctl restart smbd
systemctl is-enabled smbd         # enabled → starts at boot

# Wait for the SSD before starting
sudo mkdir -p /etc/systemd/system/smbd.service.d
printf '[Unit]\nRequiresMountsFor=/mnt/ssd\n' | sudo tee /etc/systemd/system/smbd.service.d/ssd.conf
sudo systemctl daemon-reload
```
- Windows: `\\raspberrypi.local\share`, user `litigator` + Samba password.
- To drop movies in from Windows, add a second `[media]` block with `path = /mnt/ssd/media`.
- **Never share** `/mnt/ssd/docker`, `/mnt/ssd/containerd` or `/mnt/ssd/compose`
  (Docker data, Immich's database, `.env` passwords).

### Issue — SSH from the laptop stopped working
- Cause: home Wi-Fi dropped and the laptop silently fell back to the iPhone hotspot
  (`172.20.10.x`), a different network from the Pi.
- Fix: reconnect the laptop to home Wi-Fi. Longer term: DHCP reservation, Tailscale on the
  laptop (work laptop may block it), or a lower-priority fallback Wi-Fi profile on the Pi
  via `nmcli`.

---

## Open items
- [ ] Delete old copies after a few days of everything working:
      `/var/lib/docker.old`, `/var/lib/containerd.old`, `~/docker.old`, `~/media.old`
      (confirm the renames were done first). Optional: `ln -s /mnt/ssd/compose ~/docker`
- [ ] Journald size cap: `SystemMaxUse=50M` in `/etc/systemd/journald.conf` (unconfirmed)
- [ ] Reboot test: all containers + `smbd` come back with no manual steps
- [ ] Immich: re-enable Machine Learning (now on the SSD), run Smart Search + Face Detection on "Missing"
- [ ] Immich: set job concurrency to 1 (Thumbnails, Smart Search, Face Detection, Video Conversion)
- [ ] Immich app: Settings → Networking → automatic URL switching (local + Tailscale)
- [ ] Beszel: set up alerts (disk space, temperature, system down)
- [ ] Tailscale admin console: **Disable key expiry** on the Pi; remove the Pi from the wrong-email account
- [ ] Tailscale on phone: VPN On Demand (iOS) / Always-on VPN (Android)
- [ ] Router: DHCP reservation for the Pi at 192.168.1.247
- [ ] Optional: boot to console instead of desktop (`raspi-config`) to free RAM

## Next up
1. Full phone photo backup into Immich
2. Off-site photo backup: AWS S3 Glacier Deep Archive (bucket created, `us-east-2`)
3. Samba `[media]` share if copying movies in from Windows
4. Optional: Tailscale on the laptop, hotspot fallback Wi-Fi on the Pi

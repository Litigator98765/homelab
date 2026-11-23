# Raspberry Pi Homelab — Setup Log

**Device:** Raspberry Pi 4 Model B Rev 1.5 · 4 GB RAM · Raspberry Pi OS 64-bit (aarch64)
**Hostname / user:** `raspberrypi` / `litigator`
**Home LAN IP:** `192.168.1.247` (reserve in router — Roku connects by IP)
**Storage:** 64 GB microSD (temporary) → USB SSD (SanDisk ~600 MB/s, purchased, not yet installed)
**Repo:** github.com/Litigator98765/homelab (private) — config only, no secrets or data
**Architecture diagram:** https://claude.ai/artifact/AEgivDoV5oMG81MUFcTyKf

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

## Open items
- [ ] Confirm `docker stats` shows real memory after reboot
- [ ] Immich: set job concurrency to 1 (Thumbnails, Smart Search, Face Detection, Video Conversion)
- [ ] Immich app: Settings → Networking → automatic URL switching (local + Tailscale)
- [ ] Tailscale admin console: **Disable key expiry** on the Pi
- [ ] Tailscale on phone: VPN On Demand (iOS) / Always-on VPN (Android)
- [ ] Add `setup-host.sh` + README to repo; commit this log
- [ ] Optional: boot to console instead of desktop (`raspi-config`) to free RAM
- [ ] Remove the Pi from the wrong-email Tailscale account
- [ ] Router: DHCP reservation for the Pi at 192.168.1.247
- [ ] Commit Jellyfin config: `git add jellyfin && git commit -m "Add Jellyfin" && git push`

## Next up
1. **SSD arrives:** `lsblk` → `rpi-clone sda` → remove SD → boot from SSD
2. Re-enable Immich ML, run Smart Search + Face Detection jobs on "Missing"; turn on full phone backup
3. Photo backup: second drive and/or Backblaze B2
4. Media drive for Jellyfin (powered external HDD or SSD); point `MEDIA_DIR` at it

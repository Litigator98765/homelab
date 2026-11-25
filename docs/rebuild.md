# Rebuild Runbook — Pi Homelab

How to set this homelab up again from scratch on a new machine (or a fresh SD card).
Run the steps in order. Commands come from the original setup (2026-11-22 to 2026-11-24),
adapted for a clean machine.

**Target:** Raspberry Pi 4 (4 GB) · Raspberry Pi OS 64-bit (aarch64) · user `litigator` · hostname `raspberrypi`
**Layout:** OS on the microSD card · everything else on a USB SSD mounted at `/mnt/ssd`
**Repo:** `github.com/Litigator98765/homelab` (private) — config only, no secrets or data

> **Only the SD card died?** The SSD still holds all the data and this repo. Flash a new card,
> then do steps 1–4 (skipping the formatting in step 3 — just mount the existing SSD by its
> UUID), 6, 9 and 10. Every compose project starts from `/mnt/ssd/compose` exactly as before.

---

## 0. Before you start: what is *not* in git

These are needed for a full restore and must come from the SSD or a backup, not the repo:

| What | Where it lives | If lost |
|---|---|---|
| Immich photos | `/mnt/ssd/compose/immich/library/` | Photos gone unless backed up elsewhere |
| Immich database | `/mnt/ssd/compose/immich/postgres/` | Albums, faces, metadata gone (Immich also writes nightly DB dumps to `library/backups/`) |
| Immich DB password | `/mnt/ssd/compose/immich/.env` | Generate a new one (only matters if restoring the old DB) |
| Jellyfin config | `/mnt/ssd/compose/jellyfin/config/` | Re-create user + libraries; it rescans media |
| Media | `/mnt/ssd/media/` | Re-copy from wherever it came from |
| Samba share files | `/mnt/ssd/share/` | Gone unless backed up |
| Beszel history + key/token | `/mnt/ssd/compose/beszel/` (`beszel_data/`, `.env`) | Re-create admin, re-add the system for a new key + token |
| Portainer settings | `/mnt/ssd/compose/portainer/data/` | Just re-create the admin user |
| Tailscale login | `/var/lib/tailscale/` (SD card) | Just log in again |

> ⚠️ As of 2026-11-24 there is **no off-site photo backup** yet (planned: AWS S3 Glacier
> Deep Archive) — see open items in `setup.md`.

---

## 1. Flash the OS and check the hardware

Flash **Raspberry Pi OS (64-bit)** to the microSD card with Raspberry Pi Imager. In the
Imager settings set hostname `raspberrypi`, user `litigator`, Wi-Fi, and enable SSH.

After first boot:

```bash
cat /proc/device-tree/model     # expect: Raspberry Pi 4 Model B
uname -m                        # expect: aarch64
free -h
sudo apt update && sudo apt full-upgrade -y
```

## 2. Enable container memory stats

Pi OS disables the memory cgroup by default, so `docker stats` shows `0B / 0B`.

```bash
sudo cp /boot/firmware/cmdline.txt /boot/firmware/cmdline.txt.bak
sudo sed -i '$ s/$/ cgroup_enable=memory cgroup_memory=1/' /boot/firmware/cmdline.txt
cat /boot/firmware/cmdline.txt  # must still be ONE line, ending in cgroup_memory=1
sudo reboot
```

If the Pi won't boot: put the card in another computer and copy `cmdline.txt.bak`
over `cmdline.txt` on the boot partition.

## 3. Prepare and mount the SSD

Plug the SSD into a **blue USB 3 port**.

```bash
lsblk -f                        # find the SSD — usually sda, ~931G. CHECK THE NAME.
```

**New/empty SSD only** — this erases it:

```bash
sudo wipefs -a /dev/sda
sudo parted -s /dev/sda mklabel gpt mkpart ssd ext4 0% 100%
sudo mkfs.ext4 -L ssd /dev/sda1
```

Mount it at `/mnt/ssd` (both new and existing SSDs):

```bash
sudo mkdir -p /mnt/ssd
sudo chattr +i /mnt/ssd         # immutable while EMPTY → nothing lands on the SD card if the SSD is missing
sudo blkid /dev/sda1            # copy the UUID
sudo cp /etc/fstab /etc/fstab.bak
echo 'UUID=<ssd-uuid>  /mnt/ssd  ext4  defaults,noatime,nofail,x-systemd.device-timeout=10s  0  2' | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
df -h /mnt/ssd                  # should show ~916G
sudo chown litigator:litigator /mnt/ssd   # new SSD only
```

Optional: cap the system log so it doesn't wear the SD card — set `SystemMaxUse=50M` in
`/etc/systemd/journald.conf`, then `sudo systemctl restart systemd-journald`.

## 4. Install Docker and point it at the SSD

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker                   # or log out and back in
docker --version                # original setup: 29.10.x
docker compose version          # original setup: v5.8.0
```

Move **both** Docker and containerd to the SSD (Docker 29 keeps images in containerd):

```bash
sudo systemctl stop docker docker.socket containerd

echo '{ "data-root": "/mnt/ssd/docker" }' | sudo tee /etc/docker/daemon.json
sudo cp /etc/containerd/config.toml /etc/containerd/config.toml.bak
sudo sed -i '1i root = "/mnt/ssd/containerd"' /etc/containerd/config.toml

for s in docker containerd; do
  sudo mkdir -p /etc/systemd/system/$s.service.d
  printf '[Unit]\nRequiresMountsFor=/mnt/ssd\n' | sudo tee /etc/systemd/system/$s.service.d/ssd.conf
done
sudo systemctl daemon-reload
sudo systemctl start containerd docker

docker info | grep -i 'root dir'          # Docker Root Dir: /mnt/ssd/docker
sudo containerd config dump | grep '^root' # root = "/mnt/ssd/containerd"
```

On an existing SSD, Docker picks up its old images and volumes from these folders.

## 5. Install Tailscale (on the host, not in Docker)

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --qr          # scan the QR on your phone — browser login hit ERR_SSL_PROTOCOL_ERROR last time
tailscale status
```

Then in the Tailscale admin console:
- Remove the **old** `raspberrypi` machine if it's still listed (otherwise the new one becomes `raspberrypi-1`)
- **Disable key expiry** on the new Pi

Check you're logged in with the right account — `tailscale status` shows the owner.
If wrong: `sudo tailscale logout` and run `sudo tailscale up --qr` again.

## 6. Clone the config repo onto the SSD

Skip the clone if the SSD already has `/mnt/ssd/compose`.

```bash
sudo apt install -y git
git config --global user.name "<your name>"
git config --global user.email "<your github email>"
git clone https://github.com/Litigator98765/homelab.git /mnt/ssd/compose   # private repo: use a GitHub personal access token as the password
ln -s /mnt/ssd/compose ~/docker       # optional shortcut
mkdir -p /mnt/ssd/media/Movies /mnt/ssd/media/Shows /mnt/ssd/share /mnt/ssd/.beszel
```

## 7. Portainer

```bash
cd /mnt/ssd/compose/portainer
docker compose pull && docker compose up -d
docker logs portainer 2>&1 | grep setup_token   # one-time token for first login
```

Open `https://raspberrypi:9443` (accept the self-signed cert), paste the setup token,
create the admin user. Skip Edge Compute (single host).

> Portainer only gives you a few minutes to create the admin user after first start.
> If it times out: `docker restart portainer` and try again.

## 8. Immich

Create `.env` from the template with a fresh database password:

```bash
cd /mnt/ssd/compose/immich
cp .env.example .env
sed -i "s/^DB_PASSWORD=.*/DB_PASSWORD=$(openssl rand -hex 16)/" .env
grep -E 'TZ|UPLOAD_LOCATION|DB_DATA_LOCATION|IMMICH_VERSION' .env   # TZ=America/New_York, ./library, ./postgres
```

**If restoring data:** copy the old `library/` and `postgres/` folders into
`/mnt/ssd/compose/immich/` *before* starting, and use the **old** `DB_PASSWORD` in `.env`
instead of a new one. (Or start empty and restore a dump from `library/backups/`
using Immich's backup-and-restore docs.)

```bash
docker compose pull && docker compose up -d
docker compose ps               # wait until all 4 show (healthy), ~1–2 min
```

Open `http://raspberrypi:2283` and run onboarding with the same settings as before:
- Map: **on** · Version check: **on**
- Storage template: **on**, `{{y}}/{{y}}-{{MM}}/{{filename}}`
- Admin → Settings → Job settings: concurrency **1** for Thumbnails, Smart Search, Face Detection, Video Conversion
- Machine Learning: on, but watch RAM (it was paused to save memory)

Phone app: server URL `http://raspberrypi:2283`. In Settings → Networking, turn on
automatic URL switching (`http://raspberrypi.local:2283` at home, Tailscale URL away).

## 9. Jellyfin

```bash
cd /mnt/ssd/compose/jellyfin
cp .env.example .env            # MEDIA_DIR=/mnt/ssd/media
mkdir -p config cache
docker compose pull && docker compose up -d
```

Open `http://raspberrypi.local:8096`, create the user, and add libraries:
Movies → `/media/Movies`, Shows → `/media/Shows` (container paths). Turn **off** trickplay
and chapter image extraction; hardware acceleration **None** (Pi 4). It runs with
`network_mode: host` so the Roku can discover it.

## 10. Beszel (monitoring)

```bash
cd /mnt/ssd/compose/beszel
cp .env.example .env
docker compose up -d beszel     # hub only first
```

Open `http://raspberrypi.local:8090`, create the admin account, then **Add System**:
name `raspberrypi`, Host/IP `/beszel_socket/beszel.sock`. Put the Public Key and Token it
shows into `.env` (`BESZEL_KEY="ssh-ed25519 …"`, `BESZEL_TOKEN="…"`), then:

```bash
docker compose up -d            # starts the agent
```

The Pi turns green on the dashboard within about 30 seconds.

## 11. Samba (file share)

```bash
sudo apt install samba -y       # answer "No" to the WINS/DHCP question
sudo smbpasswd -a litigator
```

Add to the bottom of `/etc/samba/smb.conf` (`sudo nano /etc/samba/smb.conf`):

```ini
[share]
   comment = Pi SSD share
   path = /mnt/ssd/share
   read only = no
   valid users = litigator
```

```bash
sudo mkdir -p /etc/systemd/system/smbd.service.d
printf '[Unit]\nRequiresMountsFor=/mnt/ssd\n' | sudo tee /etc/systemd/system/smbd.service.d/ssd.conf
sudo systemctl daemon-reload
testparm -s
sudo systemctl enable --now smbd
sudo systemctl restart smbd
```

Windows: `\\raspberrypi.local\share`, user `litigator` + the Samba password.

## 12. Claude Code (optional)

```bash
curl -fsSL https://claude.ai/install.sh | bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc
claude --version
```

## 13. Verify everything

```bash
sudo reboot
# after it comes back:
findmnt /mnt/ssd                                # mounted
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Label "com.docker.compose.project.working_dir"}}'
                                                # 8 containers Up, all from /mnt/ssd/compose
docker stats --no-stream                        # memory column shows real numbers, not 0B
systemctl is-active smbd tailscaled             # active, active
tailscale status                                # raspberrypi online
```

From the phone on **cellular** (Wi-Fi off): open the Immich app and confirm it connects.
From the laptop: open `\\raspberrypi.local\share` and the Beszel dashboard.

---

## Quick reference

| Service | URL | Compose file |
|---|---|---|
| Immich | `http://raspberrypi:2283` (Tailscale) · `http://raspberrypi.local:2283` (home) | `immich/docker-compose.yml` |
| Jellyfin | `http://raspberrypi.local:8096` · `http://192.168.1.247:8096` (Roku) | `jellyfin/docker-compose.yml` |
| Beszel | `http://raspberrypi.local:8090` | `beszel/docker-compose.yml` |
| Portainer | `https://raspberrypi:9443` | `portainer/docker-compose.yml` |
| Samba | `\\raspberrypi.local\share` | `/etc/samba/smb.conf` (not in git) |
| SSH | `ssh litigator@raspberrypi` | — |

Update a stack: `cd /mnt/ssd/compose/<service> && docker compose pull && docker compose up -d && docker image prune -f`

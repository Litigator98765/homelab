# Rebuild Runbook — Pi Homelab

How to set this homelab up again from scratch on a new machine (or a fresh SD/SSD).
Run the steps in order. Every command here was used in the original setup on
2026-11-22, adapted for a clean machine.

**Target:** Raspberry Pi 4 (4 GB) · Raspberry Pi OS 64-bit (aarch64) · user `litigator` · hostname `raspberrypi`
**Repo:** `github.com/Litigator98765/homelab` (private) — config only, no secrets or data

> **Migrating an existing Pi to an SSD?** Use `rpi-clone` instead of this runbook
> (see [Moving to a new disk](#moving-to-a-new-disk-instead-of-rebuilding)).
> This runbook is for when you're starting from an empty OS.

---

## 0. Before you start: what is *not* in git

These are needed for a full restore and must come from a backup, not the repo:

| What | Where it lives | If lost |
|---|---|---|
| Immich photos | `~/docker/immich/library/` | Photos gone unless backed up elsewhere |
| Immich database | `~/docker/immich/postgres/` | Albums, faces, metadata gone (Immich also writes nightly DB dumps to `library/backups/`) |
| Immich DB password | `~/docker/immich/.env` | Generate a new one (only matters if restoring the old DB) |
| Portainer settings | `~/docker/portainer/data/` | Just re-create the admin user |
| Tailscale login | `/var/lib/tailscale/` | Just log in again |

> ⚠️ As of 2026-11-22 there is **no photo backup** yet — see open items in `setup.md`.

---

## 1. Flash the OS and check the hardware

Flash **Raspberry Pi OS (64-bit)** with Raspberry Pi Imager. In the Imager settings set
hostname `raspberrypi`, user `litigator`, Wi-Fi, and enable SSH.

After first boot:

```bash
cat /proc/device-tree/model     # expect: Raspberry Pi 4 Model B
uname -m                        # expect: aarch64
free -h
df -h /
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

## 3. Install Docker

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker                   # or log out and back in
docker --version                # original setup: 29.10.0
docker compose version          # original setup: v5.8.0
sudo systemctl enable docker    # start on boot (usually already enabled)
```

## 4. Install Tailscale (on the host, not in Docker)

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

## 5. Clone the config repo

```bash
sudo apt install -y git
git config --global user.name "<your name>"
git config --global user.email "<your github email>"
cd ~
git clone https://github.com/Litigator98765/homelab.git docker   # private repo: use a GitHub personal access token as the password
cd ~/docker
mkdir -p portainer/data immich/library immich/postgres jellyfin
```

## 6. Portainer

```bash
cd ~/docker/portainer
docker compose pull
docker compose up -d
docker logs portainer 2>&1 | grep setup_token   # one-time token for first login
```

Open `https://raspberrypi:9443` (accept the self-signed cert), paste the setup token,
create the admin user. Skip Edge Compute (single host).

> Portainer only gives you a few minutes to create the admin user after first start.
> If it times out: `docker restart portainer` and try again.

## 7. Immich

Create `.env` from the template with a fresh database password:

```bash
cd ~/docker/immich
cp .env.example .env
sed -i "s/^DB_PASSWORD=.*/DB_PASSWORD=$(openssl rand -hex 16)/" .env
grep -E 'TZ|UPLOAD_LOCATION|DB_DATA_LOCATION|IMMICH_VERSION' .env   # TZ=America/New_York, ./library, ./postgres
```

**If restoring data:** copy the old `library/` and `postgres/` folders into
`~/docker/immich/` *before* starting, and use the **old** `DB_PASSWORD` in `.env`
instead of a new one. (Or start empty and restore a dump from `library/backups/`
using Immich's backup-and-restore docs.)

```bash
docker compose pull
docker compose up -d
docker compose ps               # wait until all 4 show (healthy), ~1–2 min
```

Open `http://raspberrypi:2283` and run onboarding with the same settings as before:
- Map: **on** · Version check: **on**
- Storage template: **on**, `{{y}}/{{y}}-{{MM}}/{{filename}}`
- Admin → Settings → Machine Learning: **off** while on the SD card (re-enable on SSD)
- Admin → Settings → Job settings: concurrency **1** for Thumbnails, Smart Search, Face Detection, Video Conversion

Phone app: server URL `http://raspberrypi:2283`. In Settings → Networking, turn on
automatic URL switching (`http://raspberrypi.local:2283` at home, Tailscale URL away).

## 8. Claude Code (optional)

```bash
curl -fsSL https://claude.ai/install.sh | bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc
claude --version
```

## 9. Verify everything

```bash
sudo reboot
# after it comes back:
docker ps --format '{{.Names}}\t{{.Status}}'   # portainer + 4 immich containers, all Up / healthy
docker stats --no-stream                       # memory column shows real numbers, not 0B
tailscale status                               # raspberrypi online
```

From the phone on **cellular** (Wi-Fi off): open the Immich app and confirm it connects.

---

## Moving to a new disk instead of rebuilding

When the existing Pi is healthy and you just want it on the SSD:

```bash
lsblk                           # find the SSD, usually sda
sudo rpi-clone sda              # copies the whole running system
sudo shutdown -h now
# remove the SD card, boot from the SSD
```

Everything — Docker, Tailscale login, Immich data, `.env` — comes along, so none of
the steps above are needed.

---

## Quick reference

| Service | URL | Compose file |
|---|---|---|
| Immich | `http://raspberrypi:2283` (Tailscale) · `http://raspberrypi.local:2283` (home) | `immich/docker-compose.yml` |
| Portainer | `https://raspberrypi:9443` | `portainer/docker-compose.yml` |
| SSH | `ssh litigator@raspberrypi` | — |

Update a stack: `cd ~/docker/<service> && docker compose pull && docker compose up -d && docker image prune -f`

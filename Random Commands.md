### Price comparision of Microcenter bundle
```
AMD Ryzen 7 9700X - 33359
32gb ddr5 rAM - 45000
Gigabyte B650 Aorus Elite AX Ice - 19320

9060xt - 51k
```
 
### Command to turn on phone camera to use in OBS in Linux after running modprobe registering v412-loopback devices as kernel modules 

```
scrcpy \
  --video-source=camera \
  --camera-id=0 \
  --camera-size=1920x1080 \
  --camera-fps=30 \
  --video-bit-rate=8M \
  --no-audio-playback \
  --v4l2-sink=/dev/video0
```

### Running immich temporarily on my bazzite system
```
sudo env PATH=$PATH /home/linuxbrew/.linuxbrew/opt/docker-engine/bin/dockerd
```

```
cd /run/media/switchblade/Seagate2TB/immich && sudo docker-compose up -d
```

### Backup and snapshot commands of my server and tie in with Kopia

```
sudo rsync -aHAXv --numeric-ids --info=progress2 --partial --append-verify ./fladder ./forgejo ./searxng ./thought-train.github.io ./viewer ./yams switchblade@bazzite:/run/media/switchblade/Seagate2TB/ServerBackup/backup_20260530192130/home/
```

```
sudo rsync -aHAXv --numeric-ids --info=progress2 --partial --append-verify --dry-run --delete /mnt/hdd/500G-9VMSAPAK/backup-server-contents/ switchblade@bazzite:/run/media/switchblade/Seagate2TB/ServerBackup/backup_20260530192130/mount/
```

### Commands for cloudflare tunnels in my server1

- /home/switchblade-user/.cloudflared/cert.pem

- /home/switchblade-user/.cloudflared/3f829e72-bf3f-4415-a862-2c8c29b2b852.json

- To add new service to cloudflare tunnel: 

	- `sudo nano /etc/cloudflared/config.yml`
	- cloudflared tunnel route dns homelab fladder.thought-train.dev
	- sudo systemctl restart cloudflared.service
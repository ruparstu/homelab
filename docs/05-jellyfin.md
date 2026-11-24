# 05 – Jellyfin Media Server

*Date: November 23, 2026*

## Goal

I wanted my own streaming server: my movies and shows on my own hardware, playable on any device in the house, with no subscription and no one else's servers in the middle.

**Jellyfin** is a free, open-source media server. It scans a folder of video files, pulls posters and descriptions from online databases, and streams to a web browser, phone, or TV app. When a device can't play a file as-is (wrong format, or too high a bitrate for the connection), Jellyfin **transcodes** it on the fly into something the device can handle.

By the end of the day I had:

- Jellyfin running in its own container at `192.168.1.13`, reachable as `jellyfin.home.lan`.
- My 5 TB Seagate drive moved to the Dell and formatted as media storage.
- **Hardware transcoding** on the Dell's Intel GPU, verified, not just switched on.
- A known-good backup of the working container.

## Design decisions

### A container, a bind mount, and the GPU

| Piece | Choice | Why |
|---|---|---|
| Where Jellyfin runs | **Unprivileged LXC container** (CT 102) | Same reasons as Pi-hole ([doc 04](04-pihole.md)): light, quick to build, and root inside it isn't root on the host |
| Where the media lives | On the **host**, on the Seagate, shared into the container with a **bind mount** | The media isn't part of the container. I can rebuild or replace the container without touching a single file, and later share the same folder with other services (file shares in Phase 7) |
| How it transcodes | **Pass the Intel GPU's render device** into the container | Software transcoding on the CPU would work, but it would pin all four cores for one stream. The i7-6700's built-in GPU has **Intel Quick Sync**, a dedicated video engine that does this with very little CPU |

A full VM would also have worked, but it would need the GPU passed through as a whole PCI device, which on a single-GPU machine like the Dell means the host loses it. A container shares the host's kernel and drivers, so I can hand it just the one device file it needs.

### The Seagate is media only

The Seagate is a USB drive and it's the only copy of the media for now. I decided it's **only for Jellyfin**, not for backups of other machines. Backups belong on separate hardware (the Lenovo, Phase 6), so one failed drive can't take out both the data and its backup.

## 1. Prepare the Seagate

The Seagate came from my tower and was formatted **NTFS** (Windows). Linux can read NTFS, but ext4 is the native Linux filesystem: it handles Linux file ownership and permissions properly, which matters for the ID mapping in step 3.

I plugged it into the Dell. In the web UI under **Node → Disks**, it showed up as `/dev/sdb` (model `BUP_Portable`).

**Check the device name before wiping anything.** `/dev/sda` is the Dell's SSD with Proxmox on it. Wiping the wrong one would erase the server.

| Step | Where | Why |
|---|---|---|
| Wipe | Disks → select `/dev/sdb` → **Wipe Disk** | Removes the old NTFS partition table so Proxmox sees a blank disk |
| Format and add | Disks → **Directory → Create**: disk `/dev/sdb`, filesystem **ext4**, name `media`, **Add Storage** checked | Creates the partition, formats it, mounts it, and registers it as Proxmox storage in one step |

Proxmox mounted it at **`/mnt/pve/media`** and set it to mount at boot **by UUID**, not by device name. That's important for a USB drive: `/dev/sdb` can become `/dev/sdc` if I plug in something else, but the filesystem's UUID never changes.

### This needed `root@pam`, not my admin account

My daily admin account (`rsp@pve`, from [doc 03](03-hardening-proxmox.md)) got a **permission error** on Wipe Disk. Wiping and formatting physical disks is restricted to the real Linux `root` account in Proxmox, no matter what permissions another user has. That's deliberate: these actions can destroy the host.

My first root login also failed. The login screen's **Realm** was still set to **Proxmox VE authentication server**, which is where `rsp@pve` lives. `root` is a Linux account, so it needs the **Linux PAM standard authentication** realm. Same password, different realm, and then it worked.

## 2. Build the container

| Setting | Value | Why |
|---|---|---|
| CT ID / Hostname | `102` / `jellyfin` | |
| Template | `debian-13-standard` | Same base as my other containers, and Jellyfin's installer supports it |
| Unprivileged | Checked | Isolation from the host |
| Nesting | On | Lets newer Debian's systemd run cleanly inside the container |
| CPU / RAM / Swap | 2 cores / 2048 MB / 512 MB | Jellyfin is light when the GPU does the transcoding. Easy to raise later |
| Disk | 16 GB on `local-lvm` | Only the app, its database, and image cache. The media is on the Seagate |
| IPv4 | Static `192.168.1.13/24`, gateway `192.168.1.254` | A server needs an address that doesn't move |
| DNS | Server `192.168.1.11` (Pi-hole), search domain `home.lan` | So the container uses my own DNS and short names work |
| SSH public key | My tower's public key | Key login from day one |
| Firewall | On (on `net0`) | So I can add per-container rules later without rebuilding |

The rest I did as root over SSH on the Proxmox host:

```bash
pct set 102 -mp0 /mnt/pve/media,mp=/media   # bind mount: host folder -> /media in the container
pct set 102 -onboot 1                        # start automatically when the Dell boots
pct start 102
pct enter 102                                # open a root shell inside the container
```

- **`pct`** is Proxmox's container tool. It only exists **on the host**.
- **`mp0`** is "mount point 0". The left side is the path on the host, `mp=` is where it appears inside the container.
- **Start at boot** matters here because if the Dell reboots, the media server should come back without me.

Then inside the container:

```bash
apt update && apt -y full-upgrade
apt install -y curl
```

## 3. Unprivileged ID mapping (the part that took real thought)

An unprivileged container doesn't use the host's real user IDs. Proxmox **shifts every ID by 100000**: user 0 (root) inside the container is user 100000 on the host, user 102 inside is 100102 on the host, and so on. That's what makes root in the container powerless on the host.

The catch shows up with a bind mount. The files on the Seagate are owned by **host** IDs. If the host owns them as `root`, the container sees an unknown owner (`nobody:nogroup`) and Jellyfin can't write to them.

So I needed Jellyfin's IDs inside the container. (I ran this after the install in step 5, since the installer creates the `jellyfin` user.)

```bash
id jellyfin
# uid=102(jellyfin) gid=105(jellyfin) groups=105(jellyfin),44(video),992(render)
```

Then on the **host**, I created the media folders and gave them to the shifted IDs:

```bash
mkdir -p /mnt/pve/media/{movies,shows,videos}
chown -R 100102:100105 /mnt/pve/media/{movies,shows,videos}
```

- **`{movies,shows,videos}`** is shell brace expansion: one command, three folders.
- **`100102:100105`** is `100000 + 102` and `100000 + 105`, Jellyfin's user and group as the host sees them.

Inside the container, those folders now show as owned by `jellyfin`. The other folders Proxmox created on the storage (`dump`, `images`, `template`, `private`, `snippets`) still show as `nobody:nogroup`, which is fine: Jellyfin has no business touching them.

## 4. Pass the GPU into the container

The Intel GPU's video engine is exposed on the host as the device file **`/dev/dri/renderD128`**. The container needs that file, and Jellyfin's user needs permission to use it.

On Debian, access to render devices is controlled by the **`render`** group. I checked its group ID inside the container from the host:

```bash
pct exec 102 -- getent group render
# render:x:992:jellyfin
```

(Like step 3, I ran this after the install in step 5, since the installer creates the `jellyfin` user.) That told me two things: the group is **992**, and the `jellyfin` user is already a member. Then I passed the device in with that group:

```bash
pct set 102 -dev0 /dev/dri/renderD128,gid=992
pct reboot 102
```

Inside the container, the device was now there with the right group:

```bash
ls -l /dev/dri
# crw-rw---- 1 root render ... renderD128
```

`crw-rw----` means owner and group can read and write, everyone else gets nothing. Jellyfin gets in through the `render` group, and nothing else in the container does.

The first time, I added the device through the GUI **without** a GID. It showed up in the container owned by `root`, so Jellyfin couldn't use it. Setting `gid=992` fixed that.

## 5. Install Jellyfin

Jellyfin's official installer for Debian and Ubuntu:

```bash
curl -fsSL https://repo.jellyfin.org/install-debuntu.sh | bash
```

Same pattern I picked apart in doc 04, with one new flag: **`-f`** makes curl **fail on an HTTP error** instead of passing the error page along. That's exactly the protection I was missing when I piped HTML into `bash` during the Pi-hole install. The script also printed the OS it detected and asked me to confirm before it changed anything, which I appreciated.

It added Jellyfin's package repository (so `apt upgrade` will keep Jellyfin updated along with everything else) and installed **Jellyfin 12.4**, listening on port **8096**.

### Setup wizard

At `http://192.168.1.13:8096`:

| Step | Choice | Why |
|---|---|---|
| Admin account | Created, saved in **Bitwarden by hand** | The browser didn't prompt to save it, so I added it myself |
| Library | **Movies** → `/media/movies`, English, United States | `/media/movies` is the bind-mounted Seagate folder from step 3 |
| Remote connections | Allowed | Needed later for Tailscale. I have **no port forwarding**, so nothing on the internet can reach it |

Jellyfin 12.4 has **no UPnP / automatic port mapping** option, and the public port fields in Networking are informational only. Good: I don't want any app opening ports on my router by itself. Remote access will come through **Tailscale** in Phase 5.

### Transcoding settings

**Dashboard → Playback → Transcoding:**

| Setting | Value | Why |
|---|---|---|
| Hardware acceleration | **Intel QuickSync (QSV)** | Uses the GPU's video engine |
| Device | `/dev/dri/renderD128` | The device passed in during step 4 |
| Hardware decoding | **H264, HEVC, MPEG2, VC1, VP8** only | See below |
| Low-power encoders | Off | Older Intel generations don't support them well |
| HEVC encoding | Off | Clients get H.264, which every device can play |

The i7-6700 is a **Skylake** chip with **HD Graphics 530**. Its video engine can decode H.264, 8-bit HEVC, MPEG-2, VC-1, and VP8, but **not 10-bit HEVC, VP9, or AV1**. If I tick a codec the hardware can't do, Jellyfin hands it to the GPU anyway and playback fails. Unticked codecs fall back to the CPU, which is slow but works. So the settings have to match the actual chip, not the newest options on the page.

## 6. Add a test movie

I copied one movie from my tower with **`scp`** (copy over SSH), from PowerShell:

```powershell
scp ".\<movie file>.mp4" root@192.168.1.10:/mnt/pve/media/movies/patlabor.mp4
```

The destination name renamed it to drop the spaces in transit, because spaces in paths need quoting on both ends. Then on the host, I gave it Jellyfin's naming format and ownership:

```bash
cd /mnt/pve/media/movies
mkdir "Patlabor The Movie (1989)"
mv patlabor.mp4 "Patlabor The Movie (1989)/Patlabor The Movie (1989).mp4"
chown -R 100102:100105 "Patlabor The Movie (1989)"
```

Jellyfin matches metadata by **name and year** in a folder per movie. After a library scan, it found the right poster and description.

## 7. Verify hardware transcoding

Turning QSV on doesn't prove it works. I wanted evidence.

### Forcing a transcode

If the player can handle the file directly, Jellyfin **direct plays** it (sends the file untouched, no transcoding at all). To force a transcode, I lowered the quality setting in the player.

My first try was **720 kbps**, and it still direct played. The file itself was only **568 kbps**, so it already fit under the limit. The limit has to be **below the source bitrate** to force a transcode. At **420 kbps**, the stream type changed to **HLS**, meaning Jellyfin was transcoding.

### Proving it's the GPU

The player's Playback Info overlay didn't show the transcode details I wanted, so I went to the source and looked at the `ffmpeg` process Jellyfin starts for a transcode. From the host:

```bash
pct exec 102 -- pgrep -a ffmpeg
```

- **`pgrep -a`** finds processes by name and prints their **full command line**.

The relevant parts:

| Argument | What it proves |
|---|---|
| `-init_hw_device vaapi=va:/dev/dri/renderD128,driver=iHD` | ffmpeg opened the passed-in GPU device with Intel's media driver |
| `-hwaccel vaapi` | Decoding is on the GPU |
| `scale_vaapi=w=960:h=520` | Resizing is on the GPU |
| `-codec:v:0 h264_qsv` | Encoding is on Quick Sync |

The whole pipeline (decode, scale, encode) was on the GPU. Verified.

## 8. A friendly name

I added a Local DNS Record in Pi-hole: **`jellyfin.home.lan` → `192.168.1.13`**. Now it's `http://jellyfin.home.lan:8096`.

## 9. Back it up

I tried to take a snapshot of the working setup and the button was greyed out: *"current guest configuration does not support taking new snapshots."*

The bind mount is the reason. A snapshot captures the container's disks at one instant, but `/media` isn't a container disk. It's a folder on the host that Proxmox can't snapshot. So Proxmox refuses rather than take a snapshot that would only be partly true.

Instead I used **Backup now** (to `local` storage, snapshot mode, ZSTD compression, note `hw-transcode-working`). A backup is a full copy of the container in a file. **Bind mounts are excluded from backups**, which is what I want here: the backup is the app and its config, not 5 TB of media.

That leaves the media itself with **no backup yet**. That's a known gap for Phase 6.

## Problems & fixes

### 1. Permission error wiping the disk

Disk wipe, format, bind mounts, and device passthrough are `root@pam` only in Proxmox. Logged in as root instead of my admin account.

### 2. Root login failed

The realm was **Proxmox VE authentication server**. `root` needs **Linux PAM standard authentication**.

### 3. `apt update && -y full-upgrade`

I left out the second `apt`. `&&` runs a **whole new command** if the first succeeds, so the right side needs its own program name: `apt update && apt -y full-upgrade`.

### 4. `pct: command not found`

I ran `pct` inside the container. `pct` only exists on the Proxmox host. Now I check the prompt first: **`root@pve`** is the host, **`root@jellyfin`** is the container.

### 5. GPU device without a group

Added through the GUI without a GID, it showed up owned by `root` and Jellyfin couldn't use it. Fixed with `gid=992`.

### 6. `mv` failed: no such file

I ran the `mv` before the `scp` had actually run, so there was nothing to move. Copy first, then rename.

### 7. `scp`: Could not resolve hostname `pve.home.lan`

My tower couldn't resolve any `home.lan` names. `nslookup` showed the answering server was **`103.86.96.100`**, not Pi-hole. That's **NordVPN's DNS server**.

The VPN client adds its own network adapter and ranks it **above** Wi-Fi, so Windows asked the VPN's DNS server first, and the VPN's DNS has never heard of `home.lan`. Disconnecting the VPN wasn't enough; the adapter's DNS setting stayed. **Fully quitting** NordVPN fixed it, and `nslookup` showed `Server: pi.hole` again. I used the IP address to finish the copy, and now I quit the VPN client when I'm working on the lab at home.

**Lesson:** a VPN client's DNS can silently override my own DNS server. When local names stop resolving, the first question is "who is actually answering?"

### 8. Transcode wouldn't start

A 720 kbps limit on a 568 kbps file still direct played. The limit has to be under the source bitrate.

### 9. Snapshot greyed out

Bind mounts block snapshots. Used a vzdump backup instead.

## How this maps to work

| What I did | The enterprise version |
|---|---|
| Media on separate storage from the app | **Separating data from compute**: app servers are disposable, data lives on a SAN/NAS or object storage |
| Unprivileged container + ID mapping | **Least privilege** and **user namespaces**, the same isolation behind Docker and Kubernetes security |
| Passing one GPU device to one container | **GPU passthrough / device plugins** for VDI, video, and AI workloads |
| Codec settings matched to the actual chip | Checking a **hardware compatibility list** instead of trusting that a feature checkbox means it works |
| Proving transcoding with `pgrep` | **Verifying a change with evidence**, not with "the setting is on" |
| VPN DNS overriding local DNS | **Split DNS** on corporate VPNs: internal names resolve only when the client asks the right server |
| No UPnP, no port forwarding | **Change control on the perimeter**: apps don't get to open firewall ports on their own |
| Backup without the media | **Backup scope**: knowing exactly what each backup covers, and what it doesn't |

## Lessons learned

- **Keep data out of the container.** The container is replaceable; the media isn't.
- **Unprivileged means shifted IDs.** Host-side ownership is `100000 +` the container's ID.
- **Check which machine you're on.** The prompt tells you: host or container.
- **Match settings to the hardware.** Skylake can't decode 10-bit HEVC no matter what the checkbox says.
- **"Enabled" isn't "working."** I proved the GPU was doing the work by reading the actual process.
- **Know who answers your DNS.** A VPN client can quietly take over.
- **Know what your backup doesn't cover.** Mine covers the app, not the media.

## Next

- Decide what media to put on it, and add a Shows library.
- **Phase 5: Tailscale**, so I can watch from outside the house without opening a single port.
- **Phase 6:** a real backup plan, including what to do about the media on the Seagate.

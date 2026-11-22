# 02 – My First LXC Container

*Date: November 21, 2026*

## Goal

With Proxmox installed, I wanted to actually run something on it. I built a throwaway **LXC container** as a sandbox: a place to learn how creating, networking, breaking, and restoring a guest works before I set up real services. Jellyfin, Pi-hole, and the rest will each get their own clean container later.

## VM vs. container

Proxmox can run two kinds of guests, and it helps to know which is which before creating one.

| | Virtual machine (VM) | LXC container |
|---|---|---|
| What it is | A full virtual computer with its own kernel | An isolated Linux system that shares the host's kernel |
| Operating systems | Anything (Linux, Windows, BSD) | Linux only |
| Startup time | Like a normal PC boot | A few seconds |
| Resources | Heavier: needs its own RAM for a whole OS | Tiny: 512 MB of RAM is plenty for many services |

Most home lab services are Linux apps, so they run happily in containers. I'll use VMs when I need a different OS or full isolation (for example, a Windows test machine or a firewall later).

## Download a template

Containers are built from **templates**, which are small pre-made OS images.

In the web UI: **local → CT Templates → Templates**, then I downloaded:

```
debian-13-standard (13.6-1)
```

## Create the container

I clicked **Create CT** and used these settings:

| Tab | Setting | Value | Why |
|---|---|---|---|
| General | CT ID | `100` | Proxmox numbers guests starting at 100 |
| General | Hostname | `test01` | Clearly marked as a test box |
| General | Unprivileged container | Checked | Root inside the container is **not** root on the host, so a compromised container can't easily take over the server |
| General | Nesting | On (default) | Newer Debian releases need it for some system services |
| General | Password | Set | Root password for the console |
| General | SSH public key | Left blank | I'll switch to SSH keys soon (see below) |
| Template | Template | `debian-13-standard` | |
| Disks | Storage | `local-lvm` | |
| Disks | Root disk size | 8 GB | Plenty for a test box |
| CPU | Cores | 1 | |
| Memory | RAM / Swap | 512 MB / 512 MB | |
| Network | Bridge | `vmbr0` | |
| Network | Firewall | On | |
| Network | IPv4 | Static `192.168.1.20/24`, gateway `192.168.1.254` | Fixed address, outside the DHCP range |
| Network | IPv6 | Left blank | (This turned out to matter, see below) |
| DNS | DNS | Use host settings | |

Then I checked **Start after created** and finished.

A few terms that were new to me:

- **rootfs** is the container's root filesystem, the disk the OS lives on (like `C:\` on Windows). Extra disks get added as **mount points** (`mp0`, `mp1`, ...).
- **vmbr0** is a **Linux bridge**: a virtual switch inside Proxmox that's tied to the server's physical network card. Guests plug into it and appear on my home network like any other device.
- **SSH keys** replace passwords for logins. I keep the **private key** on my tower and put the **public key** on each server. I'll generate a pair with `ssh-keygen` and start using it on new containers.

## Test the network

From the container's console:

```bash
ping -c 3 google.com
```

`-c 3` means "send 3 pings and stop." On Linux, `ping` runs forever otherwise (unlike Windows, which stops after 4).

It worked, but the replies came from an **IPv6** address (`2607:f8b0:...`), even though I left IPv6 blank. My AT&T gateway advertises IPv6 on the network (**SLAAC**), so the container configured itself an IPv6 address automatically.

That explains the IPv6 trouble from [doc 01](01-proxmox-install.md): the Proxmox host has a static network config and didn't pick up IPv6, so `apt` failed when it tried IPv6 there. The container did pick it up, so IPv6 just worked.

To test IPv4 specifically:

```bash
ping -4 -c 3 google.com
```

That replied from an IPv4 address (`172.253.x.x`). Both paths work.

## Snapshots: break it on purpose

A **snapshot** saves the exact state of a guest so I can roll back to it later. I wanted to see that work before I ever need it for real.

1. In the container's **Snapshots** tab, I took a snapshot named `clean`.
2. Then I broke the container on purpose by deleting `apt`:

   ```bash
   rm -rf /usr/bin/apt*
   ```

3. Confirmed it was broken:

   ```bash
   apt update
   # -bash: apt: command not found
   ```

4. Back in **Snapshots**, I selected `clean` and clicked **Rollback**. Rollback stops the container and restores it.
5. Started the container, logged in, and tested again:

   ```bash
   apt update
   ```

   It worked and reported 59 packages that could be upgraded, so I upgraded them:

   ```bash
   apt full-upgrade -y
   ```

## Problems & fixes

### 1. The GUI lagged after the rollback

Right after the rollback, the **Start** button still looked clickable, and clicking it gave a `CT 100 already running` error. The console was also blank for a moment. Nothing was actually wrong: the web UI just hadn't caught up yet, and the console stayed blank until the container finished booting. Waiting a few seconds fixed it.

**Lesson:** the web UI can lag behind what's really happening. Check the task log at the bottom of the screen before clicking again.

### 2. IPv6 showed up when I didn't configure it

Covered above. Not a problem this time, but it's a good reminder that "blank" doesn't mean "off" when the router hands out IPv6 automatically.

## Lessons learned

- **Containers first.** For Linux services, an LXC container is faster and lighter than a VM.
- **Unprivileged by default.** It limits the damage if a container is ever compromised.
- **Snapshot before risky changes.** A rollback took seconds and undid something that would otherwise have meant rebuilding.
- **Containers are cheap and disposable.** If one gets messy, delete it and build a new one.
- **Watch for IPv6 surprises.** My network has IPv6 whether I configure it or not.

## Next

- **Phase 3:** Pi-hole at `192.168.1.11` for network-wide DNS filtering and ad blocking.
- Generate an SSH key pair on my tower and use it for new containers.
- Finish the rest of the security baseline on the Proxmox host.

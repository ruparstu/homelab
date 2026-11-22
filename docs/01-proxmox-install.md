# 01 – Installing Proxmox VE

*Date: November 21, 2026*

## Goal

I want hands-on practice with the things IT teams actually run: virtual machines, containers, networking, backups, and updates. Instead of installing one operating system on one PC, I installed a **hypervisor**, which lets a single machine run many isolated VMs and containers. If I break one, I delete it and try again without touching anything else.

I chose **Proxmox VE** because:

- It's free and open source (based on Debian Linux).
- It runs both full VMs (KVM) and lightweight Linux containers (LXC).
- It's managed from a web browser, so the server can sit headless in a corner.
- It's widely used in home labs and small businesses, so there's a lot of documentation and community help.

## Hardware

| Device | Role | Key specs |
|---|---|---|
| Dell OptiPlex 5050 Micro | Proxmox server | Intel i7-6700 (4 cores / 8 threads), 16 GB DDR4-2133 (2 × 8 GB, dual channel), 512 GB SATA SSD, empty M.2 NVMe slot |
| Lenovo ThinkCentre M625q Tiny | Future backup server (Proxmox Backup Server) | AMD dual-core, up to 8 GB RAM |
| Seagate 5 TB USB HDD | Future media / file storage | 2.5" portable drive |
| Dell XPS 13 | Test client | Wi-Fi only |
| Gaming tower | Management workstation | Where I use the web UI and SSH from |

I didn't know the password for the old OS on the OptiPlex, so I read the specs straight from the BIOS (F2). While I was there I confirmed these settings:

- **SATA mode:** AHCI
- **Virtualization (VT-x):** Enabled (required to run VMs)
- **VT for Direct I/O (VT-d):** Enabled (needed later if I want to pass hardware directly to a VM)

The Micro form factor only has one 2.5" bay, which the SSD uses. The empty M.2 slot is my upgrade path for dedicated VM storage.

## Network plan

My home network runs on an AT&T fiber gateway.

| Setting | Value |
|---|---|
| LAN subnet | 192.168.1.0/24 |
| Gateway and DNS | 192.168.1.254 |
| DHCP range | starts around .64 (my phone got .80) |
| Proxmox server | **192.168.1.10/24** (static) |
| Hostname | `pve.home.lan` |

Decisions and why:

- **Static IP outside the DHCP range.** A server needs an address that never changes, otherwise I'd be hunting for it after every reboot. Choosing .10, which is below where DHCP starts handing out addresses, means the router will never give that address to another device.
- **`home.lan` instead of `.local`.** `.local` is reserved for mDNS (used by things like Apple Bonjour), and using it for normal DNS can cause odd name-resolution problems.
- **Wired Intel NIC for management.** The OptiPlex also had internal Wi-Fi and I had a USB Wi-Fi dongle, but a hypervisor's management interface should be wired: it's faster, more stable, and Linux bridges (which VMs use for networking) don't work well over Wi-Fi.
- **Kept "Pin network interface names" on.** This gives the NICs stable names (`nic0`, `nic1`, ...) so the network config doesn't break if a kernel update or a new NIC changes how interfaces are detected.

## Install steps

1. **Installer USB.** I already had a USB stick with the Proxmox VE 9.1 ISO, written earlier with balenaEtcher.
2. **Boot it.** Plugged the stick into the OptiPlex, pressed **F12** at power-on, and picked the USB under **UEFI** boot.
3. **Target disk.** Selected the 512 GB SSD with the **ext4** filesystem. (ZFS has more features, but with a single disk and 16 GB of RAM, ext4 is simpler and lighter.)
4. **Location and time zone.** Set to America/Chicago.
5. **Root password and admin email.** Set a strong root password.
6. **Network.** Management interface `nic0` (the wired Intel NIC), hostname `pve.home.lan`, IP `192.168.1.10/24`, gateway `192.168.1.254`, DNS `192.168.1.254`.
7. **Install and reboot.** Removed the USB stick when prompted.
8. **Log in from my tower.** Browsed to:

   ```
   https://192.168.1.10:8006
   ```

   The browser warns about the certificate. That's expected: Proxmox uses a self-signed certificate out of the box. I logged in as `root` with the **Linux PAM** realm.

### Post-install: repositories and updates

Proxmox ships pointing at its **enterprise** update repositories, which need a paid subscription. Without one, updates fail. For a home lab, the free **no-subscription** repository is the right choice.

In the web UI under **Node → Updates → Repositories**, I:

1. Disabled the `pve-enterprise` repository.
2. Disabled the Ceph enterprise repository.
3. Added the `pve-no-subscription` repository.

Then, from the node's shell:

```bash
apt update
apt full-upgrade
reboot
```

This upgraded 289 packages and took me from Proxmox VE 9.1 to **9.2**, including a new 7.0 kernel, which is why the reboot was needed.

## Problems & fixes

Most of the learning happened here.

### 1. An old Proxmox install I couldn't get into

The OptiPlex already had an old Proxmox install on it, with a root password I'd forgotten and a static IP (10.0.0.100) from a different network. Password recovery is possible, but there was nothing on it I needed, so a **clean reinstall** was faster and gave me a known-good starting point.

### 2. Windows said the USB stick was "write protected" and wanted to format it

Windows also showed the stick as two drives. This is normal for a **hybrid ISO** image: the stick holds partitions Windows can't read. **Don't format it.** The real test is whether it boots, and it did.

### 3. The installer showed the network interface as DOWN

I had plugged the Ethernet cable in after the installer had already started. With a static IP, the installer doesn't need an active link; it just writes the config. Once the cable was in and the server rebooted, the link came up.

### 4. I typed my password into the username field

At the console login, I typed the root password where it asked for the username, so it showed on screen in plain text. I treated it as exposed: logged in, changed it right away with `passwd`, and cleared the screen.

**Lesson:** a password that's been visible anywhere should be treated as compromised and changed.

### 5. Upgrade failed: "Network is unreachable" (IPv6)

`apt` tried to reach the update server over an IPv6 address, but my server is only configured for IPv4, so it has no route there. I told apt to stick to IPv4:

```bash
echo 'Acquire::ForceIPv4 "true";' > /etc/apt/apt.conf.d/99force-ipv4
```

After that, `apt update` worked. Setting up IPv6 properly is on my list for later.

### 6. Ctrl+C in the changelog viewer aborted the upgrade

During the upgrade, `apt-listchanges` opened a pager showing package changelogs. I pressed **Ctrl+C** to leave it, which killed the whole upgrade partway through. The correct key is **q**.

To recover, I skipped the changelog viewer, let `dpkg` finish configuring the half-installed packages, and re-ran the upgrade:

```bash
export APT_LISTCHANGES_FRONTEND=none
dpkg --configure -a
apt full-upgrade
```

It finished cleanly.

### 7. The web shell pop-up stopped taking input

The pop-up console window froze and stopped accepting keystrokes. I switched to the built-in shell on the node page (**Node → >_ Shell**) and also started using SSH from my tower, which is more comfortable for longer work anyway:

```bash
ssh root@192.168.1.10
```

## Lessons learned

- **Check the BIOS first.** Virtualization has to be enabled before a hypervisor is any use, and the BIOS is a reliable place to read specs when you can't log in to the OS.
- **Plan the IP before installing.** Knowing the subnet, gateway, and DHCP range ahead of time made the network step easy and avoided address conflicts.
- **A clean reinstall is a valid fix.** When there's no data to save, it's often faster than troubleshooting an unknown setup.
- **Read the error message.** "Network is unreachable" plus an IPv6 address pointed straight at the cause.
- **Know your tools' exit keys.** `q` closes a pager. Ctrl+C stops the whole process.
- **`dpkg --configure -a` is the recovery command** for an interrupted package install.
- **Treat exposed passwords as compromised.** Change them immediately.

## Next

- **Phase 2:** Security baseline, then my first VM and my first LXC container.
- Configure IPv6 properly instead of forcing IPv4.
- Add an NVMe drive in the empty M.2 slot for dedicated VM storage.

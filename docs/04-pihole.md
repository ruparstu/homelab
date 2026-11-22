# 04 – Pi-hole and Local DNS Names

*Date: November 22, 2026*

## Goal

Every device on my network looks up names (`google.com`, `doubleclick.net`, `pve.home.lan`) through DNS before it connects to anything. Whoever answers those lookups gets to decide what resolves and what doesn't. I wanted that to be a server I run.

**Pi-hole** is a DNS server that acts as a **sinkhole**: when a device asks for a known ad, tracking, or telemetry domain, Pi-hole answers with a dead address (`0.0.0.0`) instead of the real one, so the request never leaves the device. Everything else gets forwarded to a normal upstream DNS provider.

By the end of the day I had:

- Pi-hole running in its own container at `192.168.1.11`.
- My tower and my iPhone using it, with blocking verified on both.
- **Local DNS names** for the lab, so I can type `pve.home.lan` instead of `192.168.1.10`.

## Design decisions

### Container on the Dell, not a Raspberry Pi

Pi-hole is famous for running on a Raspberry Pi, but I don't have one right now and the Dell has plenty of headroom. Pi-hole needs very little: one core and 512 MB of RAM is more than enough for a home network. Running it as an LXC container also means I get snapshots and the Proxmox firewall for free.

The trade-off: if the Dell goes down, DNS goes down for every device using it. I'll fix that later with a **second Pi-hole** on separate hardware (a Raspberry Pi or the Lenovo).

### Start small: only my own devices

The normal way to roll out Pi-hole is to change the router's DHCP settings so it hands out Pi-hole as the DNS server to every device. My AT&T BGW320 gateway **doesn't let me change the DNS server it hands out**, so that wasn't an option anyway.

That turned out to be a good constraint. Instead of switching the whole household at once, I pointed **only my own devices** at Pi-hole by hand. If something broke, the blast radius was me, not everyone in the house. Once I'm confident it's stable (and has a backup), I'll widen it.

## Create the container

Same process as [doc 02](02-first-container.md), with production settings this time:

| Setting | Value | Why |
|---|---|---|
| CT ID / Hostname | `101` / `pihole` | |
| Template | `debian-13-standard` | |
| Unprivileged | Checked | Root in the container isn't root on the host |
| CPU / RAM / Disk | 1 core / 512 MB / 8 GB | Pi-hole is tiny |
| IPv4 | Static `192.168.1.11/24`, gateway `192.168.1.254` | A DNS server needs an address that never changes |
| SSH public key | My tower's public key | So I can log in by key from day one ([doc 03](03-hardening-proxmox.md)) |

Then from PowerShell on my tower:

```powershell
ssh root@192.168.1.11
```

It logged me in with my key, no container password needed.

First, bring the container up to date and install `curl`, which the Debian template doesn't include:

```bash
apt update && apt full-upgrade -y
apt install -y curl
```

## Install Pi-hole

Pi-hole's official one-line installer:

```bash
curl -sSL https://install.pi-hole.net | bash
```

This was the first time I'd really looked at what a command like this does:

- **`curl`** downloads a URL and prints it to the screen.
  - `-s` silent (no progress bar)
  - `-S` but still show errors
  - `-L` follow redirects
- **`|` (pipe)** sends the output of the left command into the right command, instead of to the screen.
- **`bash`** runs whatever it receives as a script.

So the whole line means "download this script and run it immediately, as root." That's convenient, and it's also a real risk: I'm running code I haven't read, and if the download were tampered with or I typed the URL wrong, `bash` would run whatever came back. The safer pattern is to download first, read it, then run it:

```bash
curl -sSL https://install.pi-hole.net -o install-pihole.sh
less install-pihole.sh
bash install-pihole.sh
```

For a well-known project over HTTPS I accepted the risk, but now I know the review-first option exists. (I also found out firsthand what happens when you get the command wrong. See Problems & fixes.)

### Installer choices

| Prompt | My choice | Why |
|---|---|---|
| Static IP | Acknowledged | I'd already set `192.168.1.11` in Proxmox |
| Upstream DNS | **Quad9 (filtered, DNSSEC)** | Privacy-focused, blocks known-malicious domains, and validates DNSSEC signatures |
| Blocklist | Default list | A sensible starting point. I can add more later. |
| Query logging | On | I need to see what's being asked and blocked, especially while testing |
| Privacy level | **0** (show everything) | It's my own devices on my own network, and full detail makes troubleshooting much easier. I can raise it if other people's devices start using it. |

### Change the admin password

At the end, the installer printed a random admin password on screen. Anything shown on a screen should be treated as exposed (it's in my scrollback, maybe in a screenshot), so I changed it right away and saved the new one in Bitwarden:

```bash
pihole setpassword
```

The admin page is at `http://192.168.1.11/admin`.

## Point my tower at Pi-hole

My tower is on Wi-Fi, so I set DNS on the Wi-Fi network's settings in Windows (**Settings → Network & internet → Wi-Fi → my network → DNS server assignment → Edit → Manual**):

| | Preferred DNS | Alternate DNS |
|---|---|---|
| IPv4 | `192.168.1.11` | *(blank)* |
| IPv6 | The container's public IPv6 address | *(blank)* |

Two things I learned here:

1. **IPv6 is a bypass if I ignore it.** At first I only set IPv4. But my AT&T gateway advertises **its own IPv6 DNS server** to every device, and Windows will happily use it. Lookups that went over IPv6 skipped Pi-hole entirely. Setting the IPv6 DNS to Pi-hole's IPv6 address closed that gap. (The container picked up its IPv6 address automatically from the gateway, the same way `test01` did in doc 02.)
2. **No alternate DNS while testing.** If I listed the gateway as a backup, Windows could quietly fall back to it and I'd never know whether blocking was really working. During testing, I wanted failures to be obvious. The real backup will be a second Pi-hole.

Then I cleared Windows' DNS cache, which **needs an admin terminal**:

```powershell
ipconfig /flushdns
```

### Test it

```powershell
nslookup doubleclick.net
```

The answer came back as `0.0.0.0` (IPv4) and `::` (IPv6), with `pihole` / `192.168.1.11` listed as the server. Blocked. A normal site like `google.com` still returned real addresses.

## Point my iPhone at Pi-hole

**Settings → Wi-Fi → (i) next to my network:**

- **Configure IP:** leave on **Automatic**. The phone still gets its address from DHCP; I'm only changing DNS.
- **Configure DNS → Manual:** removed the gateway's entries and added Pi-hole's IPv4 and IPv6 addresses.
- **Limit IP Address Tracking:** turned **off** for this network. When it's on, iCloud Private Relay can route some lookups around my DNS server.

I also turned off the iOS setting that falls back to cellular when Wi-Fi seems to be failing (**Settings → Cellular → Wi-Fi Assist**). A blocked lookup can look like a failure, and the phone would then just do it over cellular, where Pi-hole can't see it.

## Verify from Pi-hole's side

Testing from the client only proves half of it. Pi-hole's **Network** page shows every device that has sent it queries, colored by how recently:

- My **tower** and **phone** were both green, so both were really using it.
- **Pi-hole's own row was red.** That's normal: Pi-hole doesn't use itself for DNS, it goes straight to Quad9.

The phone made about **twice as many queries as the PC**. Phones are chatty: every app checks in, phones home, and refreshes in the background.

## What DNS blocking can't do

Pi-hole is great, but I wanted to be realistic about it:

- **YouTube, Facebook, Reddit, and most streaming services serve ads from the same domains as the content.** If Pi-hole blocked those domains, the content would break too. DNS can only block or allow a whole name, so these ads get through.
- For those, I use a **browser-based blocker**: **uBlock Origin** in Firefox (or Brave's built-in blocker), which can filter individual requests inside a page.
- What Pi-hole **does** kill well is **trackers and telemetry** across every app and device, including ones I can't install a blocker on.

When a site breaks, the fix is the **Query Log**: find the blocked domain the site needed and **allowlist** it.

## Operations

| Task | How | Why |
|---|---|---|
| Start at boot | Container → Options → **Start at boot: Yes** | If the Dell reboots, DNS has to come back without me |
| Snapshot | Snapshot named `clean-install` | A known-good point to roll back to before changing blocklists or upgrading |
| Turn off Pi-hole's NTP sync | See below | Fixes a false warning without weakening the container |

### The NTP warning

Pi-hole's diagnosis page showed an error from its time sync: **insufficient permissions**. Pi-hole v6 includes an NTP client that tries to set the system clock, but an **unprivileged container isn't allowed to change the clock**. The clock belongs to the host kernel, which the container shares.

I could have given the container more privileges, but that would weaken the isolation I picked unprivileged containers for in the first place. The Proxmox host already keeps the time correct, and the container inherits it. So I turned off Pi-hole's clock sync instead:

```bash
pihole-FTL --config ntp.sync.active false
```

## Local DNS names

Since Pi-hole is now my DNS server, it can also answer for names that only exist in my lab. **Where:** Pi-hole admin → **Settings → Local DNS Records**:

| Name | Address |
|---|---|
| `pve.home.lan` | `192.168.1.10` |
| `pihole.home.lan` | `192.168.1.11` |
| `test01.home.lan` | `192.168.1.20` |

### Ping timed out: the firewall was doing its job

`ping pve.home.lan` resolved to `192.168.1.10` correctly, so DNS worked, but every ping **timed out**.

The cause was my own hardening from [doc 03](03-hardening-proxmox.md). The datacenter firewall's input policy is **DROP**, and the `management` IPSet only opens the web UI, SSH, and consoles. **Ping (ICMP) wasn't on the list**, so the host silently dropped it.

I added a rule at **Datacenter → Firewall → Add**:

| Field | Value |
|---|---|
| Direction | in |
| Action | ACCEPT |
| Macro | **Ping** |
| Source | `+management` (the IPSet) |
| Enable | **On** |

The first time, the rule didn't work because **new rules are created disabled** unless you tick **Enable**. Once it was on:

- `ping pve.home.lan` replied.
- `https://pve.home.lan:8006` opened the web UI (with the usual self-signed certificate warning).
- `ssh root@pve.home.lan` worked, after asking me to accept the host's fingerprint again. That's expected: SSH remembers fingerprints **by name**, and `pve.home.lan` was a new name to it, even though it's the same server.

### A note on `.home.lan`

`.lan` is a common home choice, but the name actually reserved for home networks is **`.home.arpa`** (RFC 8375). Nothing breaks with `.lan` today, but I may switch when I set up my own domain and HTTPS in Phase 8.

## Problems & fixes

### 1. `sSL` without the dash

I typed `curl sSL https://install.pi-hole.net | bash`. Without the `-`, curl treated `sSL` as a **hostname** to download from, which failed. It then fetched the real URL, but without `-L` it didn't follow the redirect, so it got back a short **HTML "302 Moved" page** and piped that into `bash`, which threw errors trying to run HTML.

### 2. `- sSL` with a space

Next try, I put a space between the dash and the letters. A lone `-` isn't the same as `-sSL`, so curl misread it again.

### 3. `http://` instead of `https://`

I tried plain `http://`. The server returned a **404 page**, and once again that got piped straight into `bash`.

**Lesson:** when piping anything into a shell, always use `https://`. Plain HTTP can be read or changed by anything between me and the server, and whatever comes back gets **executed as root**. These three mistakes were harmless because the responses were error pages, but they showed me exactly how blindly `| bash` runs whatever it's given.

### 4. The noVNC console can't paste

The Proxmox web **Console** (noVNC) doesn't support pasting text, so I couldn't paste the long installer command. I switched to **SSH from Windows Terminal**, which is better anyway. (The **xterm.js** console option in Proxmox also supports paste.)

### 5. `ipconfig /flushdns` failed

It needs an **administrator** terminal. Right-click Windows Terminal → Run as administrator.

### 6. Interactive `nslookup`

Running `nslookup` with no arguments drops into its own interactive prompt (`>`), where it expects just a name, not a full command. That confused me for a minute. Typing `exit` and running `nslookup doubleclick.net` as one line was what I wanted.

### 7. Blocking "didn't work" on IPv6

Covered above: the gateway's IPv6 DNS was a bypass until I set Pi-hole's IPv6 address on the client too.

### 8. Ping blocked by my own firewall, and a disabled rule

Covered above: DROP policy blocked ICMP, and the new rule was created disabled.

## How this maps to work

| What I did | The enterprise version |
|---|---|
| Pi-hole blocking ads and trackers | **Protective DNS** / DNS filtering (Cisco Umbrella, Cloudflare Gateway, Infoblox) blocking malware and phishing domains company-wide |
| Rolling out to my own devices first | A **pilot group** before a full rollout, to limit the blast radius |
| Plan for a second Pi-hole | **Redundant DNS servers**: every client gets at least two, because DNS down means everything is down |
| Local DNS records | **Internal DNS zones**, usually on Active Directory DNS |
| Clients bypassing DNS via IPv6 / Private Relay | **DNS bypass** is a real security problem; companies block outbound DNS to anything but their own resolvers |
| Allowlisting from the Query Log | The **exception request** process when a filter blocks something a business app needs |

## Lessons learned

- **DNS is a control point.** Whoever answers DNS decides what a device can reach.
- **Pilot before rollout.** Starting with only my devices meant nobody else noticed my mistakes.
- **IPv6 is part of the network whether I configure it or not.** Any DNS change has to cover both.
- **Read what you pipe into `bash`.** Use HTTPS, and download-then-review when it matters.
- **Verify from both ends.** `nslookup` on the client, and the Network page on the server.
- **Don't trade isolation for convenience.** I fixed the NTP warning by turning off a feature, not by giving the container more privileges.
- **A firewall that blocks something you forgot about is working correctly.** Allow what you need on purpose.

## Next

- Run Pi-hole for a week and see what breaks.
- **Second Pi-hole** on separate hardware (Raspberry Pi or the Lenovo) for redundancy, then list both on each device.
- Add more of my devices once it's proven stable.
- **Phase 4:** Jellyfin media server.

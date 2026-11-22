# 03 – Hardening Proxmox

*Date: November 22, 2026*

## Goal

Out of the box, my Proxmox server had one account (`root`), protected by one password, reachable over SSH and the web UI from anything that could reach it. That's fine for an install, but this server is going to run everything else in the lab, so I wanted a real security baseline before I put any services on it.

By the end of the day the server had:

- **Two-factor authentication** on every admin login, with recovery keys.
- **Key-only SSH.** Password logins are turned off.
- **A firewall** that only lets my home network reach the management ports.
- **A named admin account** for daily work, with `root` kept for emergencies.
- **A weekly update routine.**

## Threat model

Before changing anything, I asked who could actually reach this server and what they could do with it.

| Question | Answer for my lab |
|---|---|
| What's exposed? | The web UI (port **8006**) and SSH (port **22**) on `192.168.1.10` |
| Who can reach them? | Only devices on my home LAN. I have **no port forwarding** on the AT&T gateway, so nothing on the internet can connect. |
| What's the realistic risk? | A compromised device on my LAN (a laptop, a smart TV, a guest's phone), a stolen or reused password, or me making a mistake. |
| What's the impact? | Total. Whoever has root on the hypervisor controls every VM and container on it. |

That shaped the rules I'm keeping:

- **Never port-forward 8006 or 22.** When I want remote access, it'll go through a VPN (Tailscale or WireGuard, Phase 5), so the management ports never face the internet.
- **Assume a password alone isn't enough.** Every admin login needs a second factor.
- **Defense in depth.** No single control has to be perfect, because each layer covers for the others.

## Password manager first

Every step below creates a secret: recovery keys, a key passphrase, a new account's password. I needed somewhere safe to put them before I started, so I set up **Bitwarden** (the cloud version) as my password manager.

Self-hosting it with **Vaultwarden** is on the roadmap (Phase 8), but I didn't want the vault that holds my lab's recovery keys to live *inside* the lab I'm still learning to run.

## 1. Two-factor authentication on root

**Where:** Datacenter → Permissions → Two Factor → Add → **TOTP**

I scanned the QR code with an **authenticator app on my phone**, not the password manager. If the password and the second factor sit in the same vault, one compromise gets both, and it's not really two factors anymore.

Then I added **Recovery Keys** (Add → Recovery Keys). These are one-time codes that get me in if I lose my phone. I saved them in Bitwarden.

**Testing it:** I logged in from a **private browser window** while staying logged in to my normal window. If something had gone wrong, I still had a working session to fix it from.

## 2. SSH keys instead of passwords

A password can be guessed, phished, or reused somewhere else. An SSH key pair is much stronger: the **private key** stays on my tower and never leaves it, and the server only holds the **public key**, which is useless to an attacker.

### Generate the key (PowerShell on my tower)

```powershell
ssh-keygen -t ed25519
```

- **ed25519** is the modern default. It's shorter and faster than RSA, and just as strong or stronger.
- I accepted the default file location (`C:\Users\<me>\.ssh\id_ed25519`).
- I set a **passphrase** and saved it in Bitwarden. If someone ever copies the private key file off my tower, it's still useless without the passphrase.

### Copy the public key to the server

Windows doesn't have `ssh-copy-id`, so I piped the public key over SSH and appended it to root's authorized keys:

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh root@192.168.1.10 "cat >> /root/.ssh/authorized_keys"
```

`>>` appends instead of overwriting, so any keys already in the file stay put.

Then I opened a **new** terminal and ran `ssh root@192.168.1.10`. It asked for the key passphrase instead of the root password, which proved the key worked.

### Turn off password logins

Only after the key worked did I disable passwords. I kept my existing SSH session open the whole time as a **lifeline**: if I locked myself out, that session would still be connected and I could undo the change.

Rather than editing the main `sshd_config`, I added a small drop-in file. It's easy to find later, easy to remove, and package updates won't overwrite it.

```bash
cat > /etc/ssh/sshd_config.d/10-keys-only.conf <<'EOF'
PasswordAuthentication no
KbdInteractiveAuthentication no
EOF

sshd -t && systemctl restart ssh
```

- `KbdInteractiveAuthentication` is the other way SSH can ask for a password, so both need to be off.
- `sshd -t` tests the config first. The `&&` means SSH only restarts if the test passes, so a typo can't take SSH down.

### Prove it

```powershell
ssh -o PubkeyAuthentication=no root@192.168.1.10
```

This forces SSH to skip my key and try a password. The answer was **`Permission denied`**, which is exactly what I wanted.

## 3. The Proxmox firewall

Even on my own LAN, there's no reason for every device to be able to reach the management ports. Proxmox has a built-in firewall at the datacenter, node, and guest levels. I set it up at the datacenter level.

### The `management` IPSet

**Where:** Datacenter → Firewall → IPSet

An **IPSet** is a named list of addresses. Proxmox treats an IPSet named **`management`** specially: addresses in it are allowed to reach the web UI, SSH, and the consoles. I created it with one entry:

```
192.168.1.0/24
```

For now that's my whole home network. Once I have VLANs (Phase 10), I'll narrow it to a management network.

### Turn it on

**Where:** Datacenter → Firewall → Options

| Setting | Value | Why |
|---|---|---|
| Firewall | Yes | Turns the firewall on for the host |
| Input Policy | **DROP** | Anything not explicitly allowed is silently dropped |

I created the IPSet **before** turning the firewall on. In the other order, a DROP policy with nothing allowed would have locked me out of the web UI.

Guests have their own firewall settings, so turning this on didn't affect `test01`.

**Testing it:** I opened a fresh SSH session and refreshed the web UI. Both still worked from my tower.

## 4. Least privilege: a named admin, root as break-glass

Logging in as `root` every day is a habit I'd never accept at work. Root has no name attached to it, so the logs can't tell me who did something. It's also the first username anyone would try to guess.

**Where:** Datacenter → Permissions

1. **Users → Add.** I created a named account in the **Proxmox VE authentication server** (`pve`) realm. That realm lives inside Proxmox, not in Linux, so this account can't log in to the host's shell.
2. **Permissions → Add → User Permission.** Path `/`, role **Administrator**, **Propagate** checked, so the role applies to everything under the root of the tree.
3. **Two Factor.** Added TOTP and recovery keys for this account too.

Now I log in with the named account every day. **`root` is break-glass only**: a strong password, 2FA, recovery keys in Bitwarden, and only used if the named account breaks.

One trade-off: the node **Shell** button in the web UI only works for `root@pam`. So for command-line work I use SSH from my tower, which I prefer anyway.

## 5. Weekly update routine

Security patches only help if they get installed. Once a week:

1. **Snapshot** any container before a big upgrade (I practiced rollbacks in [doc 02](02-first-container.md)).
2. **Update the host** over SSH:

   ```bash
   apt update && apt full-upgrade
   ```

3. **Update each container** the same way, from inside it.
4. **Reboot** if a new kernel came in.

Later I'll look at `unattended-upgrades` for automatic security patches in the containers. I want to keep host upgrades hands-on for now, because a bad hypervisor update takes everything down with it.

## Problems & fixes

None of these broke anything, but every one taught me something.

### 1. I pasted the web URL into the ssh-keygen prompt

`ssh-keygen` asked *"Enter file in which to save the key"* and I pasted `https://192.168.1.10:8006`, thinking it wanted the server. It didn't make a usable key. I ran it again and pressed Enter for the default location.

**Lesson:** read the prompt. `ssh-keygen` is asking where to save the file, not which server it's for.

### 2. PowerShell said "Unexpected token"

I dropped `type` from the start of the copy command, so PowerShell tried to run the key file's path as a command. Putting `type` back (PowerShell's alias for `Get-Content`) fixed it.

### 3. My new admin account had no permissions

I logged in with the named account and couldn't see or change anything. The permission grant hadn't actually saved. I confirmed it with the user's **Permissions** button (Datacenter → Permissions → Users → select the user → Permissions), which shows what a user *actually* has. Then I added the grant again.

**Lesson:** verify access with the effective-permissions view instead of assuming the grant worked. It's the same habit as checking a user's effective rights in any access-management system.

### 4. I ran a command on the wrong machine

I was in an SSH session on the Dell and ran `ssh root@192.168.1.10`, which just connected the server to itself. The commands I meant for my tower never ran there.

**Lesson:** check the prompt before every command.

| Prompt | Where I am |
|---|---|
| `PS C:\Users\...>` | PowerShell on my tower |
| `root@pve:~#` | The Proxmox host |
| `root@test01:~#` | The test container |

### 5. Bitwarden's "Add SSH key" makes a new key

I clicked **Add SSH key** in Bitwarden, expecting to import my existing key. It generates a brand-new key pair instead. I cancelled. Using Bitwarden's SSH agent to hold my key is a possible upgrade later, but I'd rather do it on purpose than by accident.

## How this maps to enterprise practice

My lab is small, but each control has a bigger version that IT teams run:

| What I did | What a company would use |
|---|---|
| Firewall allows only my LAN | A dedicated management network or **bastion host** (jump box) |
| A named Proxmox account | **SSO** (Active Directory, Entra ID, Okta) so accounts are created and removed in one place |
| SSH keys on each server | **Short-lived SSH certificates** from a central CA |
| Root as break-glass | **Privileged Access Management (PAM)** with checkout, approval, and session recording |

I plan to build lab versions of some of these in Phase 11: a bastion container, SSO, and an SSH certificate authority.

## Lessons learned

- **Keep a lifeline.** Leave a working session open whenever a change could lock you out.
- **Test before you enforce.** Prove the key works before disabling passwords, and create the allow list before setting the firewall to DROP.
- **Verify from the outside.** A fresh session, a private window, and a forced password login proved the controls actually work.
- **Second factor means a separate device.** Don't keep the TOTP code in the same vault as the password.
- **Use a named account daily.** Root is for emergencies.
- **Read the prompt, both kinds.** The one asking you a question, and the one telling you which machine you're on.

## Next

- **Phase 3:** Pi-hole at `192.168.1.11` for network-wide DNS filtering and local DNS names.
- Optional: build my first VM to compare it with a container.
- Later: remote access over Tailscale instead of port forwarding (Phase 5).

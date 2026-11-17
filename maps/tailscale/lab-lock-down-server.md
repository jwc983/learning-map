# Lab: Lock Down a Cloud Server with Tailscale

**Date:** 2026-11-16 · **Map:** [Tailscale](README.md)

A record of a real lock-down: what I did, what broke, how I fixed it, and what I learned. The [map](README.md) holds the ideas; this file holds the hands-on steps.

## The setup

| Device | Tailnet address | Role |
| ------ | --------------- | ---- |
| Mac (`jwc-mac`) | `100.121.243.64` | My laptop. Runs Claude Code. |
| Server (Alibaba Cloud, Ubuntu 24.04) | `100.90.238.88` | Tagged `tag:server`. Runs SSH, the Claude Code service, a proxy, and a reading page. |
| iPhone | `100.74.164.55` | Should **not** reach the server. |

What runs on the server (found with `ss -tlnp`):

| Port | Program | Who can reach it | Why |
| ---- | ------- | ---------------- | --- |
| 22 | sshd | Tailnet only (public rule disabled) | Admin login |
| 8787 | clay | Tailnet only (listens on the `100.x` address) | Claude Code on the Mac uses it as `ANTHROPIC_BASE_URL` |
| 443 | xray | Public | Proxy server; the Mac's local proxy (`127.0.0.1:1082`) connects to it |
| 8080 | caddy | Public, password protected | learn-to-read reading page (I use it) |
| 2019, 53 | caddy admin, DNS | Server only | Local services |

## Final access policy

The `grants` part of the policy in the Tailscale admin console:

```json
"grants": [
    { "src": ["autogroup:member"], "dst": ["autogroup:self"],     "ip": ["*"] },
    { "src": ["autogroup:member"], "dst": ["autogroup:internet"], "ip": ["*"] },
    { "src": ["100.121.243.64"],   "dst": ["tag:server"],         "ip": ["22", "8787"] },
],
```

1. My devices can reach each other on any port. `autogroup:self` means "devices owned by the same person". The server belongs to `tag:server` now, not to me, so this rule does **not** cover it.
2. My devices may use an exit node. I don't use one now, so this rule can be removed.
3. Only my Mac can reach the server, and only for SSH (22) and Claude Code (8787).

The `tests` part runs every time I click Save. If a change would break a test, Save is refused:

```json
"tests": [
    {
        "src":    "100.121.243.64",
        "accept": ["tag:server:22", "tag:server:8787"],
    },
    {
        "src":  "100.74.164.55",
        "deny": ["tag:server:22", "tag:server:8787"],
    },
],
```

## Steps I took

1. **Replaced the "allow all" rule** (`*` → `*`) with narrow rules.
2. **Added `tests`** to the policy, then broke a rule on purpose to see Save refuse it.
3. **Tested the emergency way in**: Alibaba Cloud console → Remote Connection → VNC → logged in as `root` with the password (not an SSH key).
4. **Disabled the public port 22 rule** in the Alibaba Cloud firewall. Kept 443 (proxy) and 8080 (reading page).
5. **Tested with new connections**: SSH and Claude Code over Tailscale both still worked.
6. **Turned off key expiry for the server** in the admin console: Machines → `...` → Disable key expiry.
7. **Turned on Tailscale SSH** (see the section below).
8. **Restarted the server** to finish an update and test that everything comes back (see "Restart test").

## Tailscale SSH

Before, sshd checked my SSH key file. Now Tailscale checks my Tailscale identity, but only on the Tailscale road (`100.90.238.88:22`). The public road is still closed by the Alibaba firewall, and sshd keeps running behind it as a backup.

The `ssh` part of the policy. The first rule was already there; I added the second:

```json
"ssh": [
    {
        "action": "check",
        "src":    ["autogroup:member"],
        "dst":    ["autogroup:self"],
        "users":  ["autogroup:nonroot", "root"],
    },
    {
        "action": "accept",
        "src":    ["autogroup:member"],
        "dst":    ["tag:server"],
        "users":  ["root"],
    },
],
```

- `autogroup:self` does not cover tagged servers, so the server needed its own rule.
- `accept` logs in at once. `check` asks me to confirm in a browser every 12 hours, which also breaks scripts and AI tools that use SSH. `accept` is fine here because the grants already limit port 22 to my Mac.

Then, **on the server**:

```
tailscale set --ssh
```

It warned that my SSH session would disconnect, because Tailscale takes over all SSH traffic on the Tailscale road. I typed `y`, the session dropped, and a new login worked.

Test, with key files and passwords forbidden. Plain sshd would refuse this; only Tailscale SSH lets it in:

```
ssh -o PubkeyAuthentication=no -o PasswordAuthentication=no root@100.90.238.88
```

Recovery options, easiest first:

1. **VNC** — always works.
2. **From VNC, run `tailscale set --ssh=false`** — sshd and my key file handle the Tailscale road again.
3. **Enable the Alibaba port 22 rule** — sshd and my key file work through the public IP.

Options 2 and 3 need my SSH key file and the server's `authorized_keys`, so I keep them.

## Restart test

The login message said `*** System restart required ***`. The reason was a `libc6` update (`/var/run/reboot-required.pkgs`), the basic library almost every program uses. A restart is also the real test of the boot-order risk: will everything come back without me?

Before the restart, I checked that each service starts on boot:

| Service | Starts on boot? |
| ------- | --------------- |
| `tailscaled` | Yes, and Tailscale SSH stays on (`RunSSH: true`) |
| `ssh.socket` | Yes. Ubuntu 24.04 starts sshd when a connection arrives, so `ssh` showing "disabled" is normal |
| `caddy`, `xray` | Yes |
| `clay` | Yes. Its unit waits for Tailscale (`After=tailscaled.service`) and retries every 3 seconds forever (`Restart=always`, `StartLimitIntervalSec=0`), so the boot-order risk is handled |

Then, from a normal terminal (not Claude Code, because Claude Code goes through the server):

```
ssh root@100.90.238.88
reboot
```

After about 2 minutes I logged in again and checked:

```
systemctl is-active tailscaled caddy xray clay ssh.socket
cat /var/run/reboot-required
```

Result: all five `active`, and `No such file or directory` (no restart needed anymore). Ports 22, 443, 8080 and 8787 listened on the same addresses as before, and Claude Code worked again.

**Typo trap:** I first typed `tailscailed`, and it said `inactive`. `systemctl is-active` does not say "no such service"; it just says `inactive`. When one result looks wrong, check the spelling first, or use `systemctl status <name>`, which says `could not be found`.

## What broke and how I fixed it

| Problem | Cause | Fix |
| ------- | ----- | --- |
| New Claude Code windows could not connect; the old window still worked | Claude Code reaches the server on port **8787**, and the new rules only allowed 22. The old window's connection was opened before the change. | Added `"8787"` to the Mac → server rule |
| Save failed after editing the first rule | `"ip": [""]`: an empty string instead of `"*"` | Changed it to `"ip": ["*"]` |
| Save failed: `invalid character '{' at start of object name` | The line `// "tests": [` was still a comment, so Tailscale never saw the opening of the list | Removed the `//` |
| The server still had a key expiry date even though it was tagged | It was tagged **after** joining, so it kept its old expiry | Disabled key expiry by hand |
| `tailscale set --ssh` said: "does not run in sandboxed Tailscale GUI builds" | I ran it on my **Mac**, not the server. The Mac's App Store Tailscale can't be an SSH server, and doesn't need to be. | Logged in to the server first. Check the prompt: `❯` with `~/projects/...` is the Mac; `root@iZ...#` is the server. |
| `tailscale set --ssh` printed "aborted, no changes made" | The question was `[y/N]`; pressing Enter means **No** | Ran it again and typed `y` |

## Things that surprised me

- **Port tests from my Mac lie.** My Mac's proxy answers every connection, so `nc` said "open" even for ports with nothing behind them. To test a public port, use another network, for example a phone on mobile data.
- **No UDP 41641 rule was needed.** Tailscale still connects directly, because the server sends packets out first and the firewall lets the replies back in (NAT traversal).
- **The Alibaba firewall warning** ("disabled the default port 22 allow rule") is expected. The console's Workbench login uses public SSH; VNC does not.

## Lessons

1. **Default deny**: once "allow all" is gone, list every connection you need. Tools like Claude Code use connections you never see.
2. **Test with a new connection**: old connections hide changes.
3. **Make the emergency way in before you close the door**, and test it.
4. **Servers need identities that don't expire**: tags plus disabled key expiry. People's devices should still expire.
5. **Check, don't assume**: inspect key expiry and listening ports yourself.
6. **Read the warning before you answer**: Tailscale said exactly what would happen, and it happened.
7. **One file controls everything**: `grants` decide who can reach the server; `ssh` rules decide who can log in. Removing a device in the admin console ends its SSH access at once.

## Reminders

- My Mac's key expires on **2027-01-18**. If Claude Code suddenly can't connect around then, log in to Tailscale again on the Mac.
- To undo the lock-down in an emergency: log in through VNC, or enable the port 22 rule again in the Alibaba Cloud firewall.

## Optional next steps

- Disable the public **TCP 80** firewall rule. Nothing listens on port 80.
- Remove the unused `autogroup:internet` rule, if I never use an exit node.
- Add `sshTests` to the policy, so a bad edit to the `ssh` rules is refused on Save.

# Portunia Gate

**English** · [Français](README.fr.md)

**The SSH and Remote Desktop gateway of [Portunia](https://github.com/jpiperault/Portunia-releases).**
Users go through the Gate to reach servers: the Gate checks who they are, decides where they may go, supplies the server account when needed without ever showing it, records sessions and keeps a tamper-evident audit log.

> Portunia Gate was called **Heimdia** until version 0.12. Commands, package, service and variables are now named `portunia-gate` / `PORTUNIA_GATE_*`.

## What the Gate does

| | |
|---|---|
| **Three modes per server** | **Jump**: end-to-end relay; the Gate logs the connection without seeing its content. **Recorded**: the Gate relays and records the session; users type their own credentials. **Vault**: like Recorded, and the Gate supplies the server account, which the user never knows. |
| **Two-factor sign-in** | Password, SSH key and TOTP code (Google Authenticator, Microsoft Authenticator…), in any combination. A code works only once. |
| **Permissions** | Rules link a group to servers, with days, hours, an expiry date, and SFTP and tunnels allowed or not. Anything not allowed is denied. |
| **Recording and replay** | Every session in Vault or Recorded mode is recorded and signed. In Portunia, replay it like a video: keystrokes as subtitles, idle time skipped, sensitive commands highlighted, search across all sessions. Typed passwords are masked; every replay is logged. |
| **Audit log** | Each event contains the fingerprint of the previous one: a deleted or edited line shows. |
| **Remote Desktop** | Windows servers (RDP) behind the Gate, from Portunia or the browser. |
| **Web portal** | For people without Portunia at hand: sign-in, terminal and Remote Desktop in the browser. Published through a Cloudflare tunnel run by the Gate, a dedicated relay in a DMZ, a proxy or WAF, or directly over HTTPS, with your PKI certificate or Let's Encrypt. Everything is set in Portunia. |
| **Remote assistance** | Help someone on their Windows PC: they run a small program, read you a code, and choose view only or view and control. End-to-end encrypted, relayed by the Gate. |
| **Portunia** | The Gate appears in Portunia's sidebar; its servers open with one click; administrators and auditors manage everything from Portunia Teams. |
| **Other clients** | Any SSH client: OpenSSH, PuTTY, MobaXterm, WinSCP (SFTP). |

## Documentation

- **[Administration guide](docs/guide-administration.md)** (French): install, accounts, servers and vault, rules, web portal, assistance, audit, backups, updates, troubleshooting, command line.
- **[User guide](https://github.com/jpiperault/Portunia-releases/blob/main/docs/guide-utilisation.md)** (French): Portunia, connecting through a Gate, web portal, assistance.
- Operations guide installed with the package: `/usr/share/doc/portunia-gate/operations.md`.

## Install

**Server**: Debian 12 or 13, Ubuntu 22.04 or later, x86-64 processor. 1 CPU, 2 GB RAM and 10 GB of disk are enough. A virtual machine or an LXC container is fine (nothing to enable).

```bash
curl -fsSLO https://github.com/jpiperault/Portunia-Gate-releases/releases/latest/download/installer.sh
sudo bash installer.sh
```

The wizard downloads the latest version, checks its SHA-256 fingerprint, installs the package and PostgreSQL, then asks five questions (Enter keeps the suggested value):

| Question | Suggested value |
|---|---|
| Gate name | Portunia Gate |
| Administrator login | admin |
| SSH port | 2222 |
| API port (used by Portunia) | 8443 |
| Time zone | the server's |

It then creates the keys, the certificate, the database and the administrator account, starts the Gate and shows what to enter in Portunia: **address**, **certificate fingerprint**, **login** and **temporary password**. The Gate runs under its own unprivileged account, restarts with the server, and restarts by itself if it ever stops responding.

### Back up the vault key, right away

The vault key encrypts server passwords and TOTP secrets: without it, a restore loses all those secrets. The wizard puts a copy in your home folder (`~/portunia-gate-vault.key`) and shows the commands to copy it to your computer and delete it from the server. Keep it in a password manager or the company vault.

### Running the wizard again

| | |
|---|---|
| `sudo bash installer.sh` | Menu: information, update, backup administrator, web portal, hardening |
| `sudo bash installer.sh info` | Address and fingerprint to give to users |
| `sudo bash installer.sh update` | Installs the latest version and restarts (**open sessions are cut**) |
| `sudo bash installer.sh admin julien` | Creates a backup administrator with a temporary password |
| `sudo bash installer.sh harden` | Hardens a server dedicated to the Gate (`plan` shows what would change, `undo` reverts) |
| `sudo bash installer.sh relay CODE` | Makes this machine a dedicated web portal relay (code given by the Gate) |
| `sudo bash installer.sh --package portunia-gate_X.Y.Z_amd64.deb` | Installs a package downloaded beforehand (server without Internet) |

Without a terminal (automation), the wizard uses the suggested values or those of the `PORTUNIA_GATE_NOM`, `PORTUNIA_GATE_ADMIN`, `PORTUNIA_GATE_SSH_PORT`, `PORTUNIA_GATE_API_PORT` and `TZ` variables.

The wizard, the Gate and the web portal speak English, or French when the system or the browser is in French.

## Adding the Gate to Portunia

1. Portunia › **Extensions** › Portunia Gate › **Enable**: the **Gates** section appears in the sidebar.
2. Gates › **+** › type the address given by the wizard.
3. Compare the fingerprint shown by Portunia with the wizard's (or `sudo portunia-gate info` on the server). If it differs, stop.
4. Login and temporary password; **leave the Code field empty**.
5. Choose your password, scan the QR code with your authenticator app and type the code it shows.
6. Allowed servers appear under the Gate; one click opens the session.
7. Administration (Portunia Teams): Gate **⋯** menu › **Administer**: servers and vault, accounts, groups, rules, web portal, assistance, settings. Audit: Gate **⋯** menu › **Audit**: sessions and log.

## Web portal

In Portunia: Administer › **Web portal**, choose how the portal is published, then the public address, the allowed group and the switch.

- **Cloudflare Tunnel** (no open port): in Cloudflare, Networking › Tunnels › **Create Tunnel**, copy the install command shown and paste it in Portunia: the Gate runs the tunnel itself. Then, in the tunnel, Routes › Add route › **Published application**, Service URL `http://localhost:8445`.
- **Internal / VPN**, **direct HTTPS** or **proxy / WAF**: open the HTTPS listener from the same screen (port 8446 by default; redirect 443 to it on the firewall), then import your PKI certificate (PEM or PFX) or let the Gate get one from **Let's Encrypt** and renew it.
- **Dedicated relay**: a machine in the DMZ serves the portal; the Gate connects to it, never the other way round.

Details in the [administration guide](docs/guide-administration.md), chapter 10.

## Connecting without Portunia

```bash
ssh -p 2222 julien@gate              # menu of allowed servers
ssh -p 2222 julien%web-01@gate       # straight to web-01
ssh -J julien@gate:2222 admin@fw-01  # end-to-end jump
sftp -P 2222 julien%web-01@gate      # files (Vault mode, if the rule allows it)
```

## Reporting a problem

Open an [issue](https://github.com/jpiperault/Portunia-Gate-releases/issues) with the version (`sudo portunia-gate info`), what you were doing and the message shown (`sudo journalctl -u portunia-gate -n 50` for the service log).

---

© Portunia. All rights reserved. Portunia Gate relies on open-source software (russh, IronRDP, xterm.js…), under MIT and Apache 2.0 licences.

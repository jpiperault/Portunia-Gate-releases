# Heimdia

**English** · [Français](README.fr.md)

**The SSH and Remote Desktop bastion of [Portunia](https://github.com/jpiperault/Portunia-releases).**
Users go through Heimdia to reach servers: Heimdia checks who they are, decides where they may go, supplies the server account when needed without ever showing it, records sessions and keeps a tamper-evident audit log.

## What Heimdia does

| | |
|---|---|
| **Three modes per server** | **Jump**: end-to-end relay; the bastion logs the connection without seeing its content. **Recorded**: the bastion relays and records the session; users type their own credentials. **Vault**: like Recorded, and the bastion supplies the server account, which the user never knows. |
| **Two-factor sign-in** | Password, SSH key and TOTP code (Google Authenticator, Microsoft Authenticator…), in any combination. A code works only once. |
| **Permissions** | Rules link a group to servers, with days, hours, an expiry date, and SFTP and tunnels allowed or not. Anything not allowed is denied. |
| **Recording and replay** | Every session in Vault or Recorded mode is recorded and signed. In Portunia, replay it like a video: keystrokes as subtitles, idle time skipped, sensitive commands highlighted, search across all sessions. Typed passwords are masked; every replay is logged. |
| **Audit log** | Each event contains the fingerprint of the previous one: a deleted or edited line shows. |
| **Remote Desktop** | Windows servers (RDP) behind the bastion, from Portunia or the browser. |
| **Web portal** | For people without Portunia at hand: sign-in, terminal and Remote Desktop in the browser, behind Cloudflare, with no open port. Turned on or off with one click in Portunia. |
| **Portunia** | The bastion appears in Portunia's sidebar; its servers open with one click; administrators and auditors manage everything from Portunia Teams. |
| **Other clients** | Any SSH client: OpenSSH, PuTTY, MobaXterm, WinSCP (SFTP). |

## Install

**Server**: Debian 12 or 13, Ubuntu 22.04 or later, x86-64 processor. 1 CPU, 2 GB RAM and 10 GB of disk are enough. A virtual machine or an LXC container is fine (nothing to enable).

```bash
curl -fsSLO https://github.com/jpiperault/Heimdia-releases/releases/latest/download/installer.sh
sudo bash installer.sh
```

The wizard downloads the latest version, checks its SHA-256 fingerprint, installs the package and PostgreSQL, then asks five questions (Enter keeps the suggested value):

| Question | Suggested value |
|---|---|
| Bastion name | Heimdia |
| Administrator login | admin |
| SSH port | 2222 |
| API port (used by Portunia) | 8443 |
| Time zone | the server's |

It then creates the keys, the certificate, the database and the administrator account, starts the bastion and shows what to enter in Portunia: **address**, **certificate fingerprint**, **login** and **temporary password**. The bastion restarts on its own with the server, and restarts by itself if it ever stops responding.

### Back up the vault key, right away

The vault key encrypts server passwords and TOTP secrets: without it, a restore loses all those secrets. The wizard puts a copy in your home folder (`~/heimdia-coffre.key`) and shows the commands to copy it to your computer and delete it from the server. Keep it in a password manager or the company vault.

### Running the wizard again

| | |
|---|---|
| `sudo bash installer.sh` | Menu: information, update, backup administrator, web portal |
| `sudo bash installer.sh info` | Address and fingerprint to give to users |
| `sudo bash installer.sh update` | Installs the latest version and restarts (**open sessions are cut**) |
| `sudo bash installer.sh admin julien` | Creates a backup administrator with a temporary password |
| `sudo bash installer.sh portal` | Installs Cloudflare Tunnel for the web portal |
| `sudo bash installer.sh --package heimdia_X.Y.Z_amd64.deb` | Installs a package downloaded beforehand (server without Internet) |

Without a terminal (automation), the wizard uses the suggested values or those of the `HEIMDIA_NOM`, `HEIMDIA_ADMIN`, `HEIMDIA_SSH_PORT`, `HEIMDIA_API_PORT` and `TZ` variables.

The wizard, the bastion and the web portal speak English, or French when the system or the browser is in French.

The operations guide (backups, logs, troubleshooting) is installed with the package: `/usr/share/doc/heimdia/operations.md`.

## Adding the bastion to Portunia

1. Portunia › **Extensions** › Heimdia › **Enable**: the **Bastions** section appears in the sidebar.
2. Bastions › **+** › type the address given by the wizard.
3. Compare the fingerprint shown by Portunia with the wizard's (or `sudo heimdia info` on the server). If it differs, stop.
4. Login and temporary password; **leave the Code field empty**.
5. Choose your password, scan the QR code with your authenticator app and type the code it shows.
6. Allowed servers appear under the bastion; one click opens the session.
7. Administration (Portunia Teams): bastion **⋯** menu › **Administer**: servers and vault, accounts, groups, rules, web portal, settings. Audit: bastion **⋯** menu › **Audit**: sessions and log.

## Web portal

1. In Cloudflare Zero Trust › Networks › Tunnels, create a tunnel (Cloudflared) and copy its token.
2. On the bastion: `sudo bash installer.sh portal`, then paste the token.
3. In the tunnel's *Public Hostname* tab: your address, type **HTTP**, URL `localhost:8445`.
4. In Portunia: Administer › **Web portal**: public address, allowed group, then the switch. Cloudflare Access in front of the address is recommended.

## Connecting without Portunia

```bash
ssh -p 2222 julien@bastion              # menu of allowed servers
ssh -p 2222 julien%web-01@bastion       # straight to web-01
ssh -J julien@bastion:2222 admin@fw-01  # end-to-end jump
sftp -P 2222 julien%web-01@bastion      # files (Vault mode, if the rule allows it)
```

## Reporting a problem

Open an [issue](https://github.com/jpiperault/Heimdia-releases/issues) with the version (`sudo heimdia info`), what you were doing and the message shown (`sudo journalctl -u heimdia -n 50` for the service log).

---

© Heimdia. All rights reserved. Heimdia relies on open-source software (russh, IronRDP, xterm.js…), under MIT and Apache 2.0 licences.

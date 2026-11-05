# Administration guide

*Portunia Gate 0.30.1 · Portunia 0.70.1 · Updated: 5 November 2026*

> The Portunia user guide is in [Portunia-releases](https://github.com/jpiperault/Portunia-releases/blob/main/docs/user-guide.md). Version française : [guide d'administration](guide-administration.md).

Install, configure and run the Gate, and prepare Portunia on workstations: accounts, servers, rules, web portal, assistance, audit, backups, updates and troubleshooting. Each screen is described with its fields, default values and the equivalent commands on the server.

<a id="avant"></a>

## 1\. Before you start

Portunia Gate is the mandatory crossing point between users and servers. It checks **who** connects (password, SSH key, TOTP code), decides **where** each person may go (rules), supplies the server account when needed **without showing it** (vault), **records** sessions and writes everything to a **tamper-evident log**.

Administration is done from Portunia (the Gate's **Administer** tab) or on the command line on the server (`sudo portunia-gate …`). Both do the same things and are logged the same way.

### Roles

| Role | Can | Cannot |
| --- | --- | --- |
| **User** (`user`) | Open the servers their rules allow. Replay their own sessions if the setting allows it ("My sessions"). | See the administration or other people's audit. |
| **Auditor** (`auditor`) | Replay all sessions, search their content, view and export the log. | Change anything; reveal masked input. |
| **Administrator** (`admin`) | Everything: servers and vault, accounts, groups, rules, settings, portal, assistance, audit, revealing masked input. | Remove their own administrator rights or disable their own account (the Gate refuses). |

### Vocabulary

| Term | Meaning |
| --- | --- |
| Server (target) | A machine behind the Gate: SSH (Linux, network devices) or Remote Desktop (Windows). |
| Mode | How the Gate relays: **jump**, **recorded** or **vault** (see [§7](#modes)). |
| Group | A set of accounts. Rights are always given to a group, never to a single account. |
| Rule | Links a group to servers, with days, hours, expiry, SFTP, tunnels, recorded keystrokes. |
| Vault | The servers' passwords and keys, encrypted with the **vault key** (`vault.key`). |
| Fingerprint | The identity of a key or certificate. Portunia pins the Gate's; the Gate pins each server's. |
| Token | A one-time pass (60 s) that Portunia requests at each opening: no need to type the code again. |

### What you need

| Item | Details |
| --- | --- |
| Server | Debian 12 or 13, Ubuntu 22.04 or later; virtual machine or LXC container. Package: 1 CPU, 2 GB of memory, 10 GB of disk. Docker: 2 CPUs, 4 GB, 20 GB. |
| Network | Ports **2222** (SSH) and **8443** (API) reachable from workstations; the Gate must reach the servers (22, 3389…). Details in [appendix C](#flux). |
| Time | Server and phones on time (NTP): TOTP codes depend on it. |
| Workstations | Portunia 0.48 or later. The **Administer** tab requires the **Teams** edition (during the preview, every feature is open). |
| Phone | An authenticator app: Google Authenticator, Microsoft Authenticator, FreeOTP, 1Password… |
| Web portal and assistance (optional) | A domain name managed by Cloudflare and a (free) Cloudflare account for the tunnel. |

<a id="installer"></a>

## 2\. Install the Gate

|  | Debian package (recommended) | Docker |
| --- | --- | --- |
| Duration | 1 to 2 minutes (package already built) | 10 to 30 minutes (built on the server) |
| How it runs | Hardened systemd service, the distribution's PostgreSQL | Portunia Gate and PostgreSQL 17 containers (and cloudflared as an option) |
| LXC container | Nothing to enable (unless the service does not start: *nesting*) | *nesting* and *keyctl* options required |
| Processors | x86-64 | x86-64 and ARM |

### Debian package

```
curl -fsSLO https://github.com/jpiperault/Portunia-Gate-releases/releases/latest/download/installer.sh
sudo bash installer.sh
```

The assistant downloads the latest version, **checks its SHA-256 fingerprint**, installs the package and PostgreSQL, creates the `portunia-gate` database (local socket access, no password, owned by `_portunia_gate`) and the `portunia-gate` service. The service runs under the `_portunia_gate` system account and only writes to `/var/lib/portunia-gate`.

Without Internet access, or for a specific version: copy the `.deb` to the server, then `sudo bash installer.sh --package portunia-gate_0.11.0_amd64.deb`.

### Docker

```
git clone https://github.com/jpiperault/Portunia-Gate.git
cd Portunia-Gate
sudo ./installer.sh --docker
```

The repository is private: `git clone` asks for a GitHub login and, as the password, a read access token. The assistant installs Docker if it is missing and, with less than 4 GB of memory, offers a temporary swap file for the build. All `installer.sh` and `docker compose` commands are then run from this folder.

### The five questions

| Question | Suggested | Advice |
| --- | --- | --- |
| Name of the Gate | Portunia Gate | A meaningful name ("ACME Gate"): it appears in Portunia and in everyone's authenticator app. |
| Administrator login | admin | Preferably your name (`julien`): the log will say who did what. Lowercase letters, digits, dot, hyphen, underscore; 32 characters at most. |
| SSH port | 2222 | Keep 2222 unless there is a conflict. Port 22 stays with the server's own administration SSH. |
| API port | 8443 | If you change it, users will add `address:port` in Portunia. |
| Time zone | the server's | The users' time zone: it is used for the rules' time ranges. |

The assistant then creates the keys, the certificate, the database and the administrator account, starts the Gate and shows a box with the **address**, the certificate's **fingerprint**, the **login** and a **temporary password** (three groups of four characters). Write them down for the first sign-in. An interrupted installation resumes when you run the assistant again: the answers already given are offered again.

> [!CAUTION]
> **Right away: the vault key** — The assistant leaves a copy of `vault.key` in your home folder (`~/portunia-gate-vault.key`). Copy it off the server (password manager, company vault), then delete the copy: `scp me@gate:~/portunia-gate-vault.key .` from your workstation, then `rm ~/portunia-gate-vault.key` on the server. Without this key, a restore loses every vault secret and every TOTP code.

### Unattended installation

Outside a terminal (automation tool), the assistant takes the suggested values or those of the variables `PORTUNIA_GATE_NAME`, `PORTUNIA_GATE_ADMIN`, `PORTUNIA_GATE_SSH_PORT`, `PORTUNIA_GATE_API_PORT`, `TZ`:

```
sudo PORTUNIA_GATE_ADMIN=julien PORTUNIA_GATE_NAME="ACME Gate" bash installer.sh </dev/null
```

### Running the assistant again

| Command | Effect |
| --- | --- |
| `sudo bash installer.sh` | On an installed Gate: menu of the choices below. |
| `sudo bash installer.sh info` | Address, ports and fingerprint to give to users. |
| `sudo bash installer.sh update` | Latest version, restart. **Cuts open sessions.** |
| `sudo bash installer.sh admin rescue` | Creates an administrator with a temporary password (last administrator locked out, lost phone). |
| `sudo bash installer.sh portal` | Hands the Cloudflare tunnel to the Gate (command or token to paste), like Administer › Web portal. |
| `sudo bash installer.sh harden` | Hardens the server (see [§17](#securite)); `harden plan`, `harden status`, `harden undo`. |
| `sudo bash installer.sh relay CODE NAME` | On **another** machine: installs a dedicated relay with the code given by the Gate (see Appendix A). |

With Docker, replace `sudo bash installer.sh` with `sudo ./installer.sh` in the repository folder.

### Company certificate (optional)

The API certificate is self-signed: Portunia recognises it by its fingerprint, which is enough. The one browsers see on the portal is separate and is set in Administer › Web portal (see the web portal). To use a company certificate for the API (PEM, certificate followed by its chain):

```
sudo install -o _portunia_gate -g _portunia_gate -m 600 cert.pem /var/lib/portunia-gate/entreprise.crt
sudo install -o _portunia_gate -g _portunia_gate -m 600 key.pem  /var/lib/portunia-gate/entreprise.key
echo 'PORTUNIA_GATE_TLS_CERT=/var/lib/portunia-gate/entreprise.crt' | sudo tee -a /etc/portunia-gate/portunia-gate.env
echo 'PORTUNIA_GATE_TLS_KEY=/var/lib/portunia-gate/entreprise.key'  | sudo tee -a /etc/portunia-gate/portunia-gate.env
sudo systemctl restart portunia-gate
```

> [!WARNING]
> **The fingerprint changes** — Warn users beforehand: Portunia will show them the new fingerprint, to compare with `sudo portunia-gate info`. Same thing when the certificate is renewed.

<a id="jour1"></a>

## 3\. Day one: the checklist

1.  **Put the vault key somewhere safe** and delete the copy in your home folder (see above).
2.  **Add the Gate in Portunia**: Extensions › Portunia Gate › Enable; Gates › **+** › address. **Compare the fingerprint** shown with the assistant's; if it differs, stop.
3.  **First sign-in**: login and temporary password, **Code field empty**. Choose a password (at least 10 characters), scan the QR code, type the 6-digit code.
4.  **Create a second administrator** (a colleague, or a backup account whose phone is kept elsewhere): if you lose your phone, it will give you access back.
5.  **Settings**: name, legal banner, default sign-in, session duration (see [§9](#reglages)).
6.  **Groups** by job: "Linux admins", "Windows admins", "Network", "Contractors"…
7.  **Servers**: start with a test server in vault mode with "Generate a key" (see [§7](#serveurs)).
8.  **A rule** linking a group to this server. Open it from Portunia, then replay the session in Audit: the whole chain is checked end to end.
9.  **User accounts**: create them, send the temporary passwords through a separate channel (see [§5](#comptes)).
10.  **Schedule the backup** of the database and the data folder (see [§14](#sauvegarde)).

> [!NOTE]
> **Nothing shows under the Gate?** — That is normal as long as no rule gives access to a server: you need a server, a group containing the account and an active rule linking them (day, hour and date included).

<a id="portunia"></a>

## 4\. Portunia on workstations

| Topic | Good to know |
| --- | --- |
| Installation | Windows installer (NSIS): **for the user** (no administrator rights) or **for all users of the computer** (Program Files, administrator rights, recommended on shared workstations and jump servers). Updates keep the chosen mode; in "all users" mode, Windows asks for permission at each update. Data stays specific to each Windows account. Download: `github.com/jpiperault/Portunia-releases`, latest version. |
| Updates | Settings › Updates: check at startup (on by default), a banner offers the new version, nothing is installed without consent. The package is signed; Portunia checks the signature before installing. |
| Extensions to enable | **Portunia Gate** (Gates section), **Remote Desktop** (otherwise RDP servers stay greyed out), **Remote assistance** for support staff. Extensions menu in the sidebar. |
| Licence | Settings › Licence: paste the text received by e-mail. Checked offline, without an online account. The edition shows at the bottom of the sidebar. Administer and Audit require **Teams**. |
| Locking | Recommended on administration workstations: Settings › Security, master password (local data is then encrypted), Windows Hello as an option, lock after inactivity, Ctrl+Shift+L to lock. Master password forgotten without Windows Hello = local data lost, by design. |
| Gate token | Stored in the Windows Credential Manager. The Gate password is stored there only if the user asks **and** the Gate allows it ("remember password" setting); the code is still asked. |
| Session expired | Beyond the duration set on the Gate (8 h by default), Portunia offers to sign in again, then resumes the current action. |

> [!WARNING]
> **Before licences go on sale** — Portunia is currently in preview: all Pro and Teams features are open, without a licence. Once licence checking is in service, administration and audit workstations will need a Teams licence.

<a id="comptes"></a>

## 5\. Accounts

Portunia › the Gate's **⋯** menu › **Administer** › **Accounts**. The table shows the login, role, groups, how the account signs in and its state: *temporary password*, *code enrolled*, *at first sign-in*, *disabled*.

### Create an account

1.  **Create an account**: login, role, sign-in (or "General setting"), groups.
2.  Portunia shows a temporary password **only once**. Send it through a channel other than the Gate's address (phone, text message, in person).
3.  The user adds the Gate in Portunia, signs in with this password and an empty Code field, chooses a password and enrols their code.

On the command line: `sudo portunia-gate user add marc --provisoire` (same steps), or `sudo portunia-gate user add marc` to type the password and enrol the code on the spot (QR code shown in the terminal; an abandoned enrolment deletes the account instead of leaving it half-created).

### Ways to sign in

| Choice in Portunia | Value | Use |
| --- | --- | --- |
| General setting | (empty) | Follows the Gate's setting — `password+totp,key+totp` by default. |
| Password + code | `password+totp` | The usual case. |
| Password or key, + code | `password+totp,key+totp` | Portunia with a password, SSH client with a key; the code always. |
| SSH key + code | `key+totp` | Contractors, sensitive access. This account does not use Portunia to sign in to the Gate (see limits). |
| Password only | `password` | Avoid; for tests only. On the web portal, this account's own setting exempts it from the code (see [§10](#portail)). |
| SSH key only | `key` | Technical accounts (scripts); standard SSH client only. |

A code can be used only once: the same 6-digit code is refused if it has already been used; wait for the next one (30 s). Five failures in ten minutes on an account impose a wait.

### Actions on an account

| Button | Effect | Command |
| --- | --- | --- |
| Edit | Role, sign-in, groups, disabling. | `user role`, `user policy`, `user disable` |
| Temporary password | New password to send; **closes all the account's sessions**: Portunia, web portal and open sessions. For a forgotten or stolen password. | `user passwd` |
| Reset the code | Lost or changed phone: the user will enrol a new code at their next sign-in. Also closes all their sessions. | `user totp` |
| SSH key | Adds a public key (`ssh-ed25519 AAAA…`); Portunia shows its fingerprint. | `user key` |
| Delete | Erases the account; its session history and the log are kept. | `user delete` |

### When someone leaves

1.  **Disable** the account (Edit › Account disabled) rather than deleting it right away: their Portunia tokens are erased at once and they can no longer sign in; their name stays readable in the audit filters.
2.  If they had **recorded** or **jump** access with their own credentials on servers, have those accounts disabled too: the Gate does not know them.
3.  For servers in the **vault**, nothing to change: they never knew the passwords. That is the advantage of the vault.
4.  Delete the account later, once the audits of the departure are done.

> [!WARNING]
> **Session already open** — Disabling prevents any new sign-in; an SSH session already open lasts until it closes. In an emergency, restarting the service (`sudo systemctl restart portunia-gate`) cuts every session, including other people's.

<a id="groupes"></a>

## 6\. Groups

Administer › **Groups**: create, delete. **Members are chosen in each account's form** (Accounts › Edit › Groups), or on the command line: `sudo portunia-gate group member "Linux admins" marc julien` (`group remove` to take them out).

Deleting a group **also deletes its rules**. Two groups have a special role:

| Group (default name) | Role | Where to choose it |
| --- | --- | --- |
| Accès web | Only its members can sign in to the web portal. | Administer › Web portal |
| Dépanneurs | Only its members can start remote assistance. | Administer › Assistance |

> [!NOTE]
> **These groups are not created automatically** — If the screen says "the group … does not exist yet", create it in Groups with exactly that name (or choose another one in the list), then add the accounts to it.

<a id="serveurs"></a>

## 7\. Servers and vault

Administer › **Servers and vault** › **Add a server**. Every change is logged.

### Fields

| Field | Details |
| --- | --- |
| Protocol | SSH, Remote Desktop (RDP), or Web interface (HTTPS or HTTP, Gate 0.30). The port switches to 22, 3389, 443 or 80 as long as you have not changed it. |
| Name | No spaces (`web-01`): it is also the name to type with an SSH client (`julien%web-01@gate`). |
| Folder | Sorts the servers under the Gate in Portunia (Production, Staging, Network…). |
| Address, Port | As the Gate reaches them (internal address). Changing the address of a server that has a secret in the vault requires **typing the secret again**: a stolen administration token is not enough to send it to another machine. The log records the old and new address. Cloud providers' metadata addresses (`169.254.169.254`…) are refused. |
| Mode | Jump, recorded or vault (below). |
| Account on the server | Required in vault mode. For Remote Desktop: `DOMAIN\account`, offered at opening. |
| Vault secret | Vault mode only: *Generate a key and install it*, *Account password*, *Existing private key* (without passphrase); for Remote Desktop, the account password only. When editing: *Keep the current secret*. |

<a id="modes"></a>

### Choosing the mode

|  | Jump | Recorded | Vault |
| --- | --- | --- | --- |
| Server credentials | Typed by the user in Portunia | Typed by the user in the terminal (3 attempts) | Supplied by the Gate, never shown |
| The Gate sees the content | No (end-to-end encrypted) | Yes | Yes |
| Recording | No: connection, duration, bytes | Yes, signed | Yes, signed |
| SFTP, command, tunnel | — | — | Yes, depending on the rule |
| Web portal | RDP only, relayed without recording (SSH: Portunia only) | SSH and RDP | SSH and RDP |
| Remote Desktop | Yes | Yes | Yes (Gate 0.29): Windows session opened by the Gate |
| When to choose it | Devices whose content must not pass in clear through the Gate; minimal traceability needed. | Everyone keeps their personal accounts on the servers, but you want recording. | **The recommended case**: shared accounts, contractors, departures without changing passwords. |

### Putting a server in the vault

**Generate a key and install it** (recommended for Linux): give the account's current password once. The Gate creates a key, puts it in the account's `~/.ssh/authorized_keys`, checks that it works, then forgets the password (`cible.cle_installee`). You can then change or disable the account's password on the server.

A secret **can never be read back**, neither in Portunia nor on the command line. If the password changes on the server, the connection fails with "The vault account was refused" (`cible.coffre_refuse`): replace the secret (Edit › Vault secret).

### Server fingerprint

On the first connection, the Gate remembers the server's fingerprint (Fingerprint column: *known*). If it changes later, the connection is refused (`cible.empreinte_differente`): either a reinstallation, or a machine pretending to be the server. Check with the team concerned, then **Forget the fingerprint**; the next one will be remembered at the next connection. Same principle for the certificate of Windows servers in recorded and vault modes.

### Windows servers (Remote Desktop)

-   **Jump** and **recorded** modes: the user types their Windows credentials at each opening; neither Portunia nor the Gate keeps them.
-   In recorded mode, Windows authentication (NLA) happens between the workstation and the server: the Gate records the image, the clicks, the copy-pastes (masked until revealed) and the keystrokes if the rule asks for it, without seeing the password.
-   **Vault** (Gate 0.29): the Gate keeps the password of the Windows account (local or `DOMAIN\account`) and opens the session itself: NLA authentication happens between the Gate and the server, then the Gate relays and records as in recorded mode. The user types nothing and never sees the password; Portunia and the web portal authenticate to the Gate with their one-time token. The server must offer NLA (Windows' default setting); otherwise the connection is refused (`cible.coffre_refuse`, reason "serveur sans NLA"). The RDP vault keeps a password, not a key.
-   **Jump from the web portal** (Gate 0.29): a browser cannot build an end-to-end tunnel, so the Gate relays the session without recording it (`rebond.ouvert`, `rebond.ferme`: duration and bytes). From Portunia, jump stays end to end. If the content must never pass in clear through the Gate, do not put these servers in the portal's group.
-   The Gate must reach port 3389. RDP servers do not appear in the Gate's SSH menu.
-   At opening, the user chooses the **display quality**, in Portunia as in the web portal: *Balanced* (default), *Fast* (no wallpaper, themes or animations, for slow links) or *Best* (all visual effects). This choice changes nothing in the recording or its replay.
-   **File copy** (Gate 0.28): allowed only by a rule with the **Files** option; otherwise the Gate blocks it, even with a client that would try. Each copied file is logged with its SHA-256 fingerprint; `sudo portunia-gate setting file_copies_mb 200` also keeps a copy of the files (up to 200 MB per session) next to the recording. In jump mode from Portunia, the Gate does not see the stream: block copying with a Windows group policy instead.

**On the command line**, a Windows server is declared with `--protocol rdp` and a `--mode`: without it, the command uses vault mode, which then asks for the account and its password. Access is then given as for SSH, through a group and a rule:

```
sudo portunia-gate target add srv-accounts --host 10.0.5.12 --protocol rdp --mode enregistre --account "ACME\accounts" --folder Windows
sudo portunia-gate target add srv-payroll --host 10.0.5.13 --protocol rdp --mode coffre --account "ACME\svc-payroll" --password --folder Windows
sudo portunia-gate group add Accounting
sudo portunia-gate group member Accounting alice bob
sudo portunia-gate rule add accounting-rdp --group Accounting --targets srv-accounts --days mon-fri --hours 7-20
sudo portunia-gate target list        # protocol, mode, fingerprint ("to remember" before the first connection)
```

`--port` if the server does not listen on 3389; in jump and recorded modes, `--account` is optional (account offered at opening, never a password); in vault mode, it is required with `--password`. A rule's `--sftp` and `--tunnels` options only concern SSH. Then: `target edit` (address, mode, account…), `target forget` after reinstalling Windows, `target delete` (Gate 0.25 and later, see Appendix A).

<a id="web"></a>

### Web interfaces (Gate 0.30)

-   For administration consoles: vCenter, iDRAC, iLO, firewalls, NAS… Protocol **Web interface (HTTPS)** or **(HTTP)**, the server's address and port; recorded mode only. The application's credentials are typed in its page, as usual.
-   The Gate relays every request: the rule is checked again at most every 30 seconds, each **page opened** is logged (`web.page`: server, path), and so is the opening (`web.ouvert`). The application's images, scripts and calls are not logged. Neither the image nor the keystrokes are recorded.
-   The certificate of an HTTPS server is checked by its fingerprint, kept on the first connection, as for Remote Desktop.
-   **From Portunia**: the server opens in a dedicated window, through a local relay to the Gate; the web session lasts 8 hours.
-   **From the web portal**: in a tab, at `/web/<name>/`. A browser has **one web interface at a time**: opening a second one replaces the first. The application shares the portal's address: only publish trusted applications on the portal.
-   The Gate's cookies are never sent to the server; neither is the visitor's address (the server only sees the Gate). The application's WebSockets (consoles, live updates) go through.

```
sudo portunia-gate target add vcenter --host 10.0.4.10 --protocol https --folder Infrastructure
sudo portunia-gate rule add vsphere --group "VMware admins" --targets vcenter --days mon-fri --hours 7-20
```

### Deleting a server

Its secret is erased; the session history and recordings are kept.

<a id="regles"></a>

## 8\. Access rules

Administer › **Rules** › **New rule**. Principle: **whatever no rule allows is refused**. Several rules touching the same server for an account **add up**. A change applies at the **next session opening**; a session already open continues.

| Field | Details |
| --- | --- |
| Name, Group | One group per rule. Create the group first. |
| Servers | One or more. |
| Days | Boxes M T W T F S S; all ticked = every day. |
| From / To (hour) | From 0 to 23, to 1 to 24; empty = all day. A range may cross midnight (22 → 6). The Gate's time zone (`TZ`). |
| File transfers | SFTP, `scp`, `rsync` (servers in the vault). |
| Tunnels | Port forwarding `ssh -L` through the server (vault). Remote forwardings `-R` are always refused. |
| Files (Remote Desktop) | File copy through the clipboard, in both directions, on Windows servers in recorded mode (Gate 0.28) or vault mode (Gate 0.29). **Without this option, the Gate blocks it**; copied text always goes through. With it, each file is logged (`fichier.copie`: direction, name, size, SHA-256 fingerprint) and appears in the recording. On the command line: `--files`; in Portunia (0.69): the rule's « Remote Desktop: file copy » box, and the Options column shows « RDP files ». Remote Desktop tabs of Portunia 0.69 send and retrieve files; in jump mode, where the Gate sees nothing, Portunia applies the option. |
| Also record keystrokes | Subtitles and search in what is typed. **Also captures typed passwords** (masked on replay, revealable by an administrator). |
| Expires on | Empty = never. After the date, the rule becomes "expired". |

The State column shows *active*, *outside range* (outside the days or hours, right now) or *expired*.

### Examples

| Need | Rule |
| --- | --- |
| Linux team during business hours | Group "Linux admins", production servers, Mon–Fri, 7 am–8 pm, SFTP.   `sudo portunia-gate rule add prod --group "Linux admins" --targets web-01,db-01 --sftp --days mon-fri --hours 7-20` |
| Night on-call | Group "On-call", same servers, every day, 8 pm → 7 am (crosses midnight). |
| Contractor for an assignment | Group "Contractors", one server, keystrokes recorded, expiry at the end of the contract.   `sudo portunia-gate rule add contractor-db --group Contractors --targets db-01 --record-input --expires 2026-12-31` |
| File exchanges with a Windows server | Group "Accounting", recorded Windows server, file copy allowed and traced.   `sudo portunia-gate rule add accounting-files --group Accounting --targets srv-accounts --files` |
| Database access from a workstation | Group "DBA", SSH jump server in the vault, **Tunnels** ticked: `ssh -L 5432:db-internal:5432`. |

> [!NOTE]
> **A user sees nothing?** — Check in order: the account is in the group (account form); the rule targets this server; it is *active* now (day, hour in the Gate's time zone, date); the account is not disabled. A refusal is logged as `acces.refuse`.

<a id="reglages"></a>

## 9\. General settings

Administer › **Settings**.

| Setting | Default | Details |
| --- | --- | --- |
| Name of the Gate | the one from installation | Shown in Portunia and in the authenticator app (for codes enrolled afterwards). |
| Sign-in banner | — | Legal warning before SSH identification, e.g. "Restricted access. Sessions are recorded." Recommended: state that sessions are recorded. |
| Default sign-in for accounts | Password or key, + code | Applies to accounts set to "General setting". |
| Session duration in Portunia | 8 h | From 1 to 168 h. Beyond it, the user signs in again (password and code). |
| Language of the Gate | system language | SSH menu and messages, default banner, service logs. Portunia and the portal follow each person's language. |
| Remember the password | no | Allows Portunia to keep the Gate password; the code is still asked. |
| Replay one's own sessions | no | Adds "My sessions" to the Gate's menu for each account, without being able to reveal masked input. |
| Sensitive commands | `rm -rf`, `rm -fr`, `mkfs`, `dd if=`, `chmod -R 777`, `shutdown`, `reboot`, `iptables -F`, `userdel`, `DROP TABLE`, `DROP DATABASE` | One per line; marked in red on the player's timeline. Add those of your trade (`systemctl stop`, `kubectl delete`…). |
| Copies of copied files (Gate 0.28) | 0 MB | Size kept per session of the files copied in Remote Desktop, next to the recording (0 to 10240 MB; 0: only name, size and SHA-256 are logged). In Portunia 0.69: Settings › Remote Desktop file copy; on the command line: `setting file_copies_mb`. |

On the command line: `sudo portunia-gate setting auth_policy "password+totp,key+totp"`, `setting banner "…"`, `setting name "…"`, `setting language en` (without a value: shows the current value).

<a id="portail"></a>

## 10\. Web portal

To open one's servers from a browser, without Portunia: sign-in, list of allowed servers, SSH terminal or Remote Desktop in a tab. No administration. Same rules, same vault, same recording, same log; a web session appears in the audit with "(web)" after the address.

Web interfaces (Gate 0.30): the server's button, then the application in a new tab (see [§7](#web)). Remote Desktop in the browser, in all three modes (Gate 0.29): the user types their Windows credentials (nothing in vault mode, where the Gate opens the session) and chooses the **display quality** (Balanced, Fast or Best, remembered by the browser); the size of the Windows desktop follows the window's. The client loads once, then stays cached until the Gate's next version.

**Files** (Gate 0.28, rule with the Files option, portal copy-paste allowed): **Send files** button or files dropped on the desktop, then pasted into Windows (Ctrl+V); the other way round, files copied in Windows (Ctrl+C) appear in a panel, with a **Download** button. Each file is logged with its fingerprint.

### Setting up

1.  **Create the tunnel** in Cloudflare: **Networking › Tunnels › Create Tunnel**, a name (e.g. `portunia-gate`), then copy the installation command shown, for any system. It contains the token; do not run it.
2.  **Give it to the Gate**: Administer › **Web portal** › **Cloudflare** mode, paste the command, **Save the token**. The Gate downloads and starts `cloudflared` itself; the tunnel becomes **Healthy** in Cloudflare. Without Portunia: `sudo bash installer.sh portal`.
3.  **Publish the address**: in the tunnel, **Routes › Add route › Published application** tab. **Subdomain** and **Domain** (the *Full hostname*, e.g. `access.your-domain.com`, is the public address), **Path** empty, **Service URL** `http://localhost:8445`, then **Add route**. Never add an `ssh://` or `rdp://` route to your servers: it would bypass the Gate.
4.  **Check**: public address `https://` + Full hostname, **Save**, then **Test the public address**.
5.  **Recommended**: a **Cloudflare Access** application on this address (company account required before the page even shows), and **Turnstile** (anti-bot).
6.  **In Portunia**: Administer › **Web portal**: group, durations, copy-paste, Turnstile, Save, then the switch. On the command line: `sudo portunia-gate portal set --url https://access.your-domain.com` (and `--group`, `--duration`, `--idle`, `--clipboard`, `--code`), then `sudo portunia-gate portal on`.
7.  **Create the group** ("Accès web" by default) and add the accounts concerned.

### The Web portal screen

| Item | Details |
| --- | --- |
| Switch | Off: the page shows "Portal closed" and open web sessions close. |
| Counters | Open web sessions, sign-ins and failures of the day, tunnel state: *connected*, *not connected*, *not set up*. In Cloudflare mode, the detailed state (download, start, connections, cloudflared error) shows above the steps. |
| Close all web sessions | Without turning the portal off; users will have to sign in again. |
| Public address | The one Cloudflare publishes; checked at every connection (a request coming from elsewhere is refused). |
| Allowed group | Only its members sign in. |
| Web session duration | 1, 4 or 8 h (1 h by default; from 1 to 12 h). |
| Close after inactivity | 15 min, 30 min or 1 h (15 min by default; from 5 to 240 min). Warning 90 s before. |
| Copy-paste | Allows text copy-paste in the web terminal. |
| Require the authenticator code | Ticked by default. Unticked: no more code on the portal for accounts that follow the general setting. An account with its own sign-in combinations follows them: `password` exempts it from the code even with the box ticked (exception), `password+totp` requires it even with the box unticked. On the command line: `sudo portunia-gate portal set --code off`; for a single account: `sudo portunia-gate user policy marc password`. |
| Turnstile | Site key and secret key (kept encrypted by the Gate; leave empty to keep it). |

### What is always enforced

-   Password, and code unless exempted (setting above or the account's own combinations). On the Internet, an account without a code is protected by its password only: keep Turnstile.
-   Five attempts per account and twenty per address in ten minutes.
-   First sign-in and lost code: not on the portal, but in Portunia or with an administrator.
-   Browser session checked again at every call: an account disabled or removed from the group is cut off at once.

**HTTPS listener** (Internal, Direct HTTPS, Proxy modes): Administer › Web portal › **HTTPS listener**, a port or address:port (e.g. `8446`), **Open**; applied within a few seconds, without restarting the Gate. Ports below 1024 are refused by the system: forward 443 to 8446 on the router or firewall; the hardened firewall allows 8446. On the command line: `sudo portunia-gate portal https 8446` (`--off` closes it).

**Portal certificate** (Internal, Direct HTTPS, Proxy modes), separate from the API's: changing it does not affect Portunia consoles. Administer › Web portal › **Portal certificate**:

-   **Import a certificate** from an internal or public PKI: PEM with its chain and unencrypted key, or PFX (.pfx, .p12) and its password; applied within a few seconds. On the command line: `sudo portunia-gate portal cert --import cert.pem --key key.pem`, `--remove` to go back to the self-signed one.
-   **Let's Encrypt** (Pro): domain names, e-mail, *TLS-ALPN-01* validation (HTTPS listener open, public 443 forwarded to it) or *DNS-01 through Cloudflare* (API token Zone › DNS › Edit; nothing to open, wildcard names allowed). Renews by itself 30 days before expiry; after a failure, new attempt every 6 hours or **Renew now**. A staging authority is available for tests.
-   An alert shows 21 days before an imported certificate expires.

**Cloudflare proxy instead of the tunnel**: `sudo portunia-gate portal https 8446` (Docker: `PORTUNIA_GATE_PORTAL_PORT=0.0.0.0:8446` in `.env`, then restart); the proxied address points to this port over HTTPS (SSL mode "Full"). The Gate only accepts Cloudflare's addresses there.

<a id="assistance"></a>

## 11\. Remote assistance

A helper takes control of a relative's or a customer's PC. The person goes to `https://portal-address/assistance`, downloads **Portunia Assistance** (a program, no installation or account), and reads a **9-digit code** to the helper, who types it in Portunia (**Remote assistance** in the sidebar). The person sees the helper's name and chooses *view only* or *view and control*. The link is end-to-end encrypted from the code: the Gate sees neither the screen nor the keystrokes and does not record them.

### Setting up

1.  Have the portal's public address (§10, steps 1 to 3). The portal itself may stay off.
2.  In Portunia, enable the **Remote assistance** extension on the helpers' workstations.
3.  Administer › **Assistance**: helpers' group, maximum duration, **Save**, then the switch. Or `sudo portunia-gate assist set --group Dépanneurs --duration 4` then `sudo portunia-gate assist on`.
4.  Create the group ("Dépanneurs" by default) and add the accounts to it.
5.  Check that the **Portunia Assistance** box switches to *ready*: the Gate downloads the program from Portunia's releases (outbound Internet access needed), then once a day, and checks its SHA-256 fingerprint.

### The Assistance screen

| Item | Details |
| --- | --- |
| Switch | Off by default. Turning it off ends ongoing assistance. |
| Counters | People waiting, assistance sessions of the day, total; program state (*ready* / *missing*). |
| Page of the person helped | The address to give, with a Copy button. |
| Helpers' group | Only its members, signed in to the Gate from Portunia, can start an assistance session. |
| Maximum duration | 1, 4 or 8 h (4 h by default; from 1 to 12 h). The assistance then stops by itself. |

### What to tell the people helped

-   Windows SmartScreen warns at first launch (program not signed yet): "More info" › "Run anyway".
-   A frame and a **Stop** bar stay on the screen; pressing Esc three times also stops.
-   A wrong code ends the attempt; the person gets a new one. A slot waits 30 minutes at most.

Log: `assistance.ouverte`, `assistance.fermee`, `assistance.reglage`, `assistance.place_inconnue`, `assistance.programme`. The same address cannot open more than 20 connections in 10 minutes.

<a id="audit"></a>

## 12\. Audit and session replay

The Gate's **⋯** menu › **Audit** › **Sessions** (administrators and auditors). Filters by account, server, period (7 days, 30 days, 12 months), "With keystrokes" and "With file copies" boxes (Portunia 0.69; the session then shows how many files, and the player places them on the timeline). The search covers the keystrokes and the displayed text of all sessions: the "Found" column gives the moment and the excerpt; a click opens the replay at that point.

### The player

| Function | Details |
| --- | --- |
| Playback | Space play/pause, arrows ±5 s, buttons ±10 s, speed 1× to 16×, full screen; click on the timeline or a line of the panel to jump there. |
| Pauses | Silences longer than 5 s are hatched and skipped ("Skip pauses"). |
| Subtitles | **CC Keystrokes**: what the person types, special keys included. |
| Markers | Each command; in orange sudo, Ctrl+C and masked input; in red sensitive commands; at the bottom the errors displayed. |
| Signature | Checked at every opening; a file modified since the session is flagged in red. |
| Exports | `.cast` recording (readable by asciinema) and timestamped transcript. |

### Masked input and revealing

Keystrokes that are not displayed on screen (sudo, su, ssh passwords…) are replaced by dots, in the player as in the search. They stay in the signed file: an **administrator** can **reveal** them; this replay is logged (`enregistrement.revele`). Every replay is logged (`enregistrement.consulte`) and the player shows the viewing account as a watermark.

> [!WARNING]
> **Frame the revealing** — Decide in writing who may reveal and in which cases (incident, investigation). The log will prove it, but the rule must exist beforehand.

### Remote Desktop

Recorded RDP sessions (recorded and vault modes) are replayed like SSH sessions (Portunia 0.47 and later): image, clicks, copy-pastes masked until revealed, keystrokes if the rule records them.

### Disk space

Recordings are stored in `/var/lib/portunia-gate/enregistrements/YYYY/MM/`; an RDP session weighs much more than an SSH session. There is no automatic retention period yet: watch the space (`sudo du -sh /var/lib/portunia-gate/enregistrements/*`) and archive old months off the server according to your retention policy. A moved file can no longer be replayed from Portunia.

<a id="journal"></a>

## 13\. Audit log

Audit › **Log**: every event (sign-ins, failures, sessions, administration, replays), filterable by account, server or event, with CSV export. Each event contains the fingerprint of the previous one: a deleted or modified line breaks the chain.

**Check the chain**: button in Portunia, or `sudo portunia-gate audit --verify`, which shows the first missing or modified event. Do it regularly and before handing over any evidence.

```
sudo portunia-gate audit --last 100          # latest events
sudo portunia-gate audit --verify --last 50  # checks the whole chain, shows the last 50
sudo portunia-gate sessions --last 20        # latest sessions
```

Useful reading: `auth.echec` gives the exact reason for a refusal (unknown account, password, code…) that the user does not see; `auth.reussie` gives the factors used (the `via` field is `portunia` for an API sign-in). The full list is in [appendix E](#evenements).

<a id="sauvegarde"></a>

## 14\. Back up and restore

### What to back up

| Item | Where | If it is lost |
| --- | --- | --- |
| **Vault key** | `/var/lib/portunia-gate/vault.key` | **Server secrets and TOTP codes lost.** Keep it off the server, separate from the database. |
| SSH host key | `/var/lib/portunia-gate/ssh_host_ed25519_key` | New SSH fingerprint for everyone to accept. |
| API certificate | `/var/lib/portunia-gate/api.crt`, `api.key` | New fingerprint to accept in Portunia. |
| Signing key | `/var/lib/portunia-gate/signature.key` | New sessions are signed by another key. |
| PostgreSQL database | database `portunia-gate` | Accounts, servers, rules, sessions, log. |
| Recordings | `/var/lib/portunia-gate/enregistrements/` | Sessions impossible to replay. |
| Settings | `/etc/portunia-gate/portunia-gate.env` (Docker: `.env`) | Ports, time zone, database access to redo. |

### Debian package

```
sudo -u postgres pg_dump portunia-gate | gzip > portunia-gate-db-$(date +%F).sql.gz
sudo tar czf portunia-gate-data-$(date +%F).tar.gz -C /var/lib portunia-gate -C /etc portunia-gate
```

### Docker (repository folder)

```
sudo docker compose exec -T postgres pg_dump -U portunia-gate portunia-gate | gzip > portunia-gate-db-$(date +%F).sql.gz
sudo docker cp "$(sudo docker compose ps -q portunia-gate)":/var/lib/portunia-gate - | gzip > portunia-gate-data-$(date +%F).tar.gz
```

> [!CAUTION]
> **The data archive contains the vault key** — Protect it like a password: encrypted storage, restricted access, off the server. Ideally, back up the database and the archive every night, and test a restore every quarter.

### Restore (Debian package)

Recommended procedure, to rehearse first on a test machine:

1.  On a new server, install **the same version** of Portunia Gate (`sudo bash installer.sh --package portunia_gate_X.Y.Z_amd64.deb`), then stop the service: `sudo systemctl stop portunia-gate`.
2.  Put the data and settings back. The archive stores `/var/lib/portunia-gate` and `/etc/portunia-gate` in a single `portunia-gate/` folder: extract it aside, then put each file back:
    
    ```
    mkdir /tmp/restore && sudo tar xzf portunia-gate-data-YYYY-MM-DD.tar.gz -C /tmp/restore
    sudo cp -p /tmp/restore/portunia-gate/portunia-gate.env /etc/portunia-gate/
    sudo rm /tmp/restore/portunia-gate/portunia-gate.env
    sudo cp -a /tmp/restore/portunia-gate/. /var/lib/portunia-gate/
    sudo chown -R _portunia_gate:_portunia_gate /var/lib/portunia-gate
    sudo rm -rf /tmp/restore
    ```
    
3.  Replace the database: `sudo -u postgres dropdb portunia_gate`, `sudo -u postgres createdb -O _portunia_gate portunia_gate`, then `gunzip -c portunia-gate-db-….sql.gz | sudo -u _portunia_gate psql portunia-gate`.
4.  Start: `sudo systemctl start portunia-gate`, then `sudo portunia-gate info` (same fingerprints as the old Gate) and `sudo portunia-gate audit --verify`.

<a id="maj"></a>

## 15\. Update

```
sudo bash installer.sh update       # Debian package
sudo ./installer.sh update          # Docker, from the repository folder
```

1.  Read the release notes (Portunia-Gate-releases) and check that no minimum Portunia version is required.
2.  **Warn users**: the update **cuts open sessions** (SSH, RDP, portal, assistance).
3.  Make a backup (§14).
4.  Run the update: with the package, download and fingerprint check, then installation; with Docker, `git pull` then rebuild. The database schema is updated at startup.
5.  Check: `sudo portunia-gate info` (version), `sudo systemctl status portunia-gate`, a sign-in from Portunia.

The web portal loads its new pages by itself after an update. If Portunia says that a feature requires a more recent Gate (portal: 0.7, assistance: 0.11, RDP vault: 0.29), this is the command to run.

<a id="surveiller"></a>

## 16\. Monitor the service

| Need | Debian package | Docker |
| --- | --- | --- |
| State | `sudo systemctl status portunia-gate` | `sudo docker compose ps` |
| Live service log | `sudo journalctl -u portunia-gate -f` | `sudo docker compose logs -f portunia-gate` |
| Before an unexpected restart | `sudo journalctl -u portunia-gate -n 100` | `sudo docker compose logs --tail 100 portunia-gate` |
| Information | `sudo portunia-gate info` — version, fingerprints, number of accounts |

**Internal watchdog**: every 20 s, the Gate checks that its database answers and that its SSH accepts a connection; a separate thread checks that its tasks make progress. Stuck for about a minute, it writes the cause (`surveillance : …`) and stops so that systemd or Docker restarts it. If this happens, keep the lines that precede the restart.

**Recommended external monitoring**: a TCP check on 2222 and 8443, the disk space of `/var/lib/portunia-gate`, and an alert on service restarts.

<a id="securite"></a>

## 17\. Security best practices

### Harden the server

On a server dedicated to the Gate, the assistant can keep only what is needed: firewall closed except for the useful ports, needless services stopped, system SSH and kernel tightened, automatic security updates. Optional: `--harden` at installation, or `sudo bash installer.sh harden` afterwards. `harden plan` shows the plan without changing anything, `harden status` shows the current state, and `harden undo` removes the hardening, except for removed packages. Without confirmation of a new SSH connection within 5 minutes, the firewall and the system SSH return to their previous state.

Portunia's **Administer › Hardening** tab shows the state read on the server every 15 minutes and flags a stopped firewall, a restarted service or a reading that is too old. It changes nothing: hardening is driven on the server, as root. It is a dashboard, not proof if the server is compromised.

### Limits and automatic closing

Since Portunia Gate 0.14, sign-in attempts are counted together for SSH, Portunia and the portal, over 10 minutes: 5 for an account from one address, 20 for an address, 30 for an account across all addresses.

A session closes in less than 30 seconds if the account is disabled, or if the rule is removed, expired or **outside its hours**. A recorded session does not open if recording is impossible, and closes if a write fails. Below `PORTUNIA_GATE_MIN_FREE_MB` (512 MB by default), new recorded sessions are refused. Watch the events `auth.bloquee`, `session.coupee`, `enregistrement.echec` and `disque.presque_plein`.

| Topic | Recommendation |
| --- | --- |
| Exposure | Ports 2222 and 8443 open only from the internal network or the VPN. From outside, the web portal behind Cloudflare Tunnel + Access, with no inbound port. |
| Servers | Allow SSH and RDP on the servers **only from the Gate** (firewall), otherwise it can be bypassed. |
| Accounts | TOTP code for everyone; "Password only" forbidden outside tests. One named account per person, never a shared account on the Gate. |
| Administrators | Two at least, no more than necessary. Auditors do not need to be administrators. |
| Vault | Prefer vault mode with "Generate a key" (Linux) or the account password (Windows); then change the account's password on the server. |
| Recorded keystrokes | Only where justified (contractors, critical servers): they capture typed passwords. Inform the people concerned (banner, charter). |
| Vault key | Off the server, in a company vault; never in a ticket or an e-mail. |
| Fingerprints | Always compare the Gate's fingerprint when adding it; never "forget" a server's fingerprint without knowing why it changed. |
| Time | NTP on the Gate: TOTP codes, time ranges and log timestamps depend on it. |
| System | Debian/Ubuntu security updates (`unattended-upgrades`), the server's administration SSH (port 22) reserved for system administrators. |
| Workstations | Master password and locking in Portunia on administration workstations. |

<a id="depannage"></a>

## 18\. Troubleshooting

### Installation

| Symptom | Cause and solution |
| --- | --- |
| "Docker does not start" in an LXC container | Enable *nesting* and *keyctl* (Proxmox › Options › Features), restart the container, run the assistant again. |
| Build stopped (`killed`, `signal 9`, `out of memory`) | Not enough memory (4 GB recommended): add memory or swap, run again; the installation resumes. |
| "Port 2222 is already in use" | Choose another port when asked. |
| "Download impossible" | No access to github.com: copy the `.deb` from a workstation, then `--package`. |
| "The downloaded package is altered" | Corrupted or intercepted download: run again; if it persists, do not install and report it. |
| The service does not start (package) | `sudo journalctl -u portunia-gate -n 50`. Unprivileged LXC container: enable *nesting*. |
| `git clone` refuses the password | GitHub asks for an access token instead. |

### Sign-in and connection

| Symptom | Cause and solution |
| --- | --- |
| Portunia cannot reach the Gate | Port 8443 closed or wrong address: `sudo bash installer.sh info`. |
| Fingerprint different from `portunia-gate info` | Do not sign in: wrong Gate or interception. Check the address. |
| "Wrong login, password or code" | The real reason is in the log: `sudo portunia-gate audit --last 10`. |
| Code refused at first sign-in | Delete the old Portunia Gate entry from the app, clear the Code field, sign in again for a new QR code. |
| Correct code but refused | Clock drift (NTP on the server and the phone) or code already used: wait for the next one. |
| "Too many attempts" | 5 failures in 10 minutes: wait a few minutes. |
| Lost phone | Accounts › Reset the code, or `sudo portunia-gate user totp <account>`. |
| No administrator left | `sudo bash installer.sh admin rescue` on the server. |
| "Gate session expired" | Sign in again from the Gates section. |
| Empty server list | No active rule for one's groups (see §8). Hours follow the Gate's time zone. |
| "The vault account was refused" | Secret changed on the server: replace the secret. |
| Windows server in the vault refused, `cible.coffre_refuse` "serveur sans NLA" | The server does not require NLA: enable "Allow connections only from computers running Remote Desktop with Network Level Authentication" in its system properties, or switch it to recorded mode. |
| Connection refused, `cible.empreinte_differente` | Server reinstalled? Check, then Forget the fingerprint. |
| RDP servers greyed out | Remote Desktop extension not enabled in Portunia. |

### Portal and assistance

| Symptom | Cause and solution |
| --- | --- |
| Tunnel "not set up", "not connected" or "error" | Read the message under the state in Administer › Web portal; paste the installation command again; the Gate must be able to reach GitHub and Cloudflare outbound. |
| Cloudflare 502 page | Wrong **Service URL** in the route (tunnel's Routes tab): `http://localhost:8445`. The **Test the public address** button reports it. |
| "Portal closed" | Switch off in Administer › Web portal. |
| Sign-in refused on the portal although Portunia works | Account outside the allowed group, first sign-in not yet done in Portunia, or public address mistyped. |
| Portunia Assistance "missing" | The Gate could not download the program (outbound access to github.com); it retries every day, and each time the relay is enabled. |
| The helper does not see "Remote assistance" | Portunia Gate and Assistance extensions to enable in their Portunia; account member of the helpers' group. |

To report a problem: version (`sudo portunia-gate info`), the last 100 lines of the service and the log events around the time of the problem.

<a id="cli"></a>

## Appendix A — Command line

The `portunia-gate` command is used with sudo, from any folder, with the package as with Docker. `sudo portunia-gate --help` and `sudo portunia-gate user --help` list the options.

```
# Installation and service
sudo portunia-gate init --admin julien --temporary --name "ACME Gate"   # database, keys, administrator; safe to run again
sudo portunia-gate init --no-admin                             # database and keys only
# portunia-gate serve: started by the portunia-gate service (do not start it by hand)

# Information
sudo portunia-gate info                                        # version, SSH and API fingerprints, number of accounts

# Accounts
sudo portunia-gate user add marc --temporary                  # temporary password, code enrolled in Portunia
sudo portunia-gate user add marc                              # password and code typed here
sudo portunia-gate user add sophie --role admin --temporary  # role: user (default), auditor or admin
sudo portunia-gate user add contractor --policy "key+totp"    # SSH key and code required
sudo portunia-gate user add robot --no-password --policy key  # key-only technical account
sudo portunia-gate user key marc "ssh-ed25519 AAAA… marc@pc"  # or the path of a .pub file
sudo portunia-gate user role marc auditor                     # user, auditor or admin
sudo portunia-gate user list
sudo portunia-gate user passwd marc
sudo portunia-gate user totp marc                             # (re)enrols the code
sudo portunia-gate user disable marc                          # --undo to enable again
sudo portunia-gate user policy contractor "key+totp"          # the account's combinations; "" = general setting
sudo portunia-gate user reset-password marc                   # new temporary password, shown once
sudo portunia-gate user reset-code marc                       # erases the code: new enrolment (Portunia or user totp)
sudo portunia-gate user delete marc

# Groups
sudo portunia-gate group add "Linux admins"
sudo portunia-gate group member "Linux admins" marc julien
sudo portunia-gate group remove "Linux admins" marc               # the sessions they lose close within 30 s
sudo portunia-gate group list                                  # groups and members
sudo portunia-gate group delete Interns                        # also deletes its rules

# Servers
sudo portunia-gate target add web-01 --host 10.0.1.11 --mode coffre --account admin --password --folder Production
sudo portunia-gate target add app-02 --host 10.0.1.12 --mode coffre --account deploy --key-file /root/deploy_ed25519
sudo portunia-gate target add stg-app --host 10.0.2.5 --mode enregistre --folder Staging
sudo portunia-gate target add fw-edge --host 10.0.0.1 --mode rebond --folder Network
sudo portunia-gate target add srv-ad-01 --host 10.0.3.10 --protocol rdp --mode enregistre --folder Windows
sudo portunia-gate target add srv-payroll --host 10.0.3.12 --protocol rdp --mode coffre --account "ACME\svc-payroll" --password   # RDP in the vault (Gate 0.29)
sudo portunia-gate target add vcenter --host 10.0.4.10 --protocol https --folder Infrastructure   # web interface, recorded mode (Gate 0.30)
sudo portunia-gate target add rds-02 --host 10.0.3.11 --port 3390 --protocol rdp --mode rebond --account "ACME\support"
sudo portunia-gate target list
sudo portunia-gate target edit web-01 --folder Prod --account root   # also --rename, --host, --port, --mode
sudo portunia-gate target edit web-01 --host 10.0.1.21 --password      # new address: secret to give again, fingerprint forgotten
sudo portunia-gate target secret web-01 --password             # replaces the vault secret (or --key-file)
sudo portunia-gate target secret app-02 --generate             # SSH: the Gate creates a key, installs it with the current password, forgets it
sudo portunia-gate target forget srv-ad-01                     # reinstalled server: forgets the fingerprint
sudo portunia-gate target delete fw-edge                       # secret erased; sessions and recordings kept

# Rules
sudo portunia-gate rule add prod --group "Linux admins" --targets web-01,stg-app --sftp --tunnels --days mon-fri --hours 7-20
sudo portunia-gate rule add contractor-db --group Contractors --targets db-01 --record-input --expires 2026-12-31
sudo portunia-gate rule add windows --group "Windows admins" --targets srv-ad-01,rds-02 --days mon-fri --hours 7-20
sudo portunia-gate rule list                                   # with their state: active, outside range, expired
sudo portunia-gate rule edit prod --sftp off --targets web-01,web-02   # switches: on or off
sudo portunia-gate rule edit contractor-db --expires 2027-03-31 --hours ""   # "" clears days, hours or expiry
sudo portunia-gate rule add accounting-files --group Accounting --targets srv-accounts --files   # Remote Desktop: file copy allowed and traced
sudo portunia-gate rule edit accounting-files --files off     # blocked again
sudo portunia-gate rule delete contractor-db                   # the sessions it allowed close within 30 s

# Settings
sudo portunia-gate setting auth_policy "password+totp,key+totp"
sudo portunia-gate setting banner "Restricted access. Sessions are recorded."
sudo portunia-gate setting name "ACME Gate"
sudo portunia-gate setting language en                        # en, fr, or "" for the system language
sudo portunia-gate setting                                   # every setting and its value
sudo portunia-gate setting session_hours 8                   # duration of a Portunia sign-in (1 to 168 h)
sudo portunia-gate setting remember_password off             # remember the Gate password in Portunia
sudo portunia-gate setting own_replay on                     # each account replays its own sessions
sudo portunia-gate setting sensitive_commands "rm -rf,shutdown"   # flagged in the player
sudo portunia-gate setting file_copies_mb 200                 # keeps files copied in Remote Desktop (MB per session; 0: none)

# Control
sudo portunia-gate audit --verify --last 50
sudo portunia-gate sessions --last 20
sudo portunia-gate audit --search marc --last 500 --csv log.csv   # search and export for a spreadsheet
sudo portunia-gate sessions --user marc --server web-01 --since 2026-11-01 --text "rm -rf"
sudo portunia-gate sessions show 3f2a9c1e                     # details, file, signature check
sudo portunia-gate sessions --files                           # Remote Desktop sessions with copied files
sudo portunia-gate hardening                                  # hardening state and alerts

# Web portal (see §10)
sudo portunia-gate portal set --url https://access.acme.com --group "Accès web" --duration 4 --idle 30 --clipboard on
sudo portunia-gate portal set --turnstile-site 0x4AAA… --turnstile-secret   # secret key asked (empty: erases it)
sudo portunia-gate portal set --code off                       # no more code on the portal (except accounts that require it)
sudo portunia-gate portal on                                   # opens the portal; off closes it (web sessions closed within 30 s)
sudo portunia-gate portal close-sessions                       # ends web sessions without closing the portal
sudo portunia-gate portal show                                 # state, settings, exposure
sudo portunia-gate portal test                                 # does the Gate answer at the public address?
sudo portunia-gate portal acme --domains access.acme.com --email it@acme.com --accept-terms   # Let's Encrypt; --renew, --off, no option: state
sudo portunia-gate portal show                                 # mode, HTTPS listener, certificate, tunnel
sudo portunia-gate portal expose cloudflare                    # modes: cloudflare, internal, direct, proxy, relay
sudo portunia-gate portal expose internal --networks 10.0.0.0/8,192.168.0.0/16
sudo portunia-gate portal expose proxy --trusted 10.0.5.10,10.0.5.11 --header X-Forwarded-For
sudo portunia-gate portal https 8446                           # HTTPS listener; without a value, shows it; --off closes it
sudo portunia-gate portal cert                                 # shows the portal certificate
sudo portunia-gate portal cert --import portal.pfx             # or --import cert.pem --key key.pem; --remove: self-signed
sudo portunia-gate portal tunnel "eyJh…"                       # Cloudflare token ("-": read from standard input); --remove
sudo portunia-gate portal tunnel                               # tunnel state
sudo portunia-gate portal relay-add dmz-01 10.0.5.20:8447      # dedicated relay: shows its pairing code
sudo portunia-gate portal relay-list
sudo portunia-gate portal relay-remove dmz-01

# Remote assistance (see §11)
sudo portunia-gate assist set --group Dépanneurs --duration 4  # helpers' group, maximum duration (1 to 12 h)
sudo portunia-gate assist on                                   # off closes it (sessions stopped within 30 s); show shows it

# Dedicated relay: on the relay machine (Debian amd64), not on the Gate
sudo bash installer.sh relay CODE access.acme.com             # installs, pairs (edge enroll) and starts the relay
sudo runuser -u _portunia_gate -- /usr/lib/portunia-gate/portunia-gate edge certificate cert.pem key.pem
sudo systemctl restart portunia-gate-relais                    # after a certificate change
# edge enroll CODE [--name NAME] [--public 0.0.0.0:443] [--tunnel 0.0.0.0:8447] [--cert C --key K]: done by the installer
# edge serve: started by the portunia-gate-relais service
```

Modes also accept `vault`, `recorded`, `jump`; days `mon-fri` or `lun,mer,ven`. Options in English or French: `--temporary` or `--provisoire`, `--name` or `--nom`; settings `banner`/`banniere`, `name`/`nom`, `language`/`langue`. Without a value, `setting` shows the current value.

Since Portunia Gate 0.26, everything Administer does in Portunia can also be done on the command line, and everything is written to the audit log (actor `console`): a Gate can be run entirely from a Linux or macOS workstation. Only **session replay** (player, revealing masked input) stays in Portunia. A command applies to the database at once; what it changes for open sessions (closing, rights, language) takes effect within 30 seconds. Full reference: [command line](command-line.md) ([français](ligne-de-commande.md)).

Over SSH, an account with a **temporary password** is refused until the password has been changed (in Portunia, or `user passwd`); its code must also be enrolled (Portunia, or `user totp`).

<a id="sans-portunia"></a>

## Appendix B — Connecting without Portunia

```
ssh -p 2222 julien@gate               # menu of allowed servers
ssh -p 2222 julien%web-01@gate        # straight to web-01
ssh -J julien@gate:2222 admin@fw-01   # end-to-end jump (servers in jump mode)
sftp -P 2222 julien%web-01@gate       # files (vault, if the rule allows SFTP)
ssh -p 2222 julien%web-01@gate "uptime"            # single command (vault)
ssh -p 2222 -L 5432:db-internal:5432 julien%web-01@gate   # tunnel (vault, if allowed)
```

Works with OpenSSH, PuTTY, MobaXterm, WinSCP. The TOTP code must have been enrolled beforehand (Portunia or `sudo portunia-gate user totp`). Single command, SFTP and tunnels: servers in the vault only.

<a id="flux"></a>

## Appendix C — Ports and network flows

| From | To | Port | Use |
| --- | --- | --- | --- |
| Workstations | Gate | 2222/tcp | SSH: sessions, SSH and RDP jump. |
| Workstations | Gate | 8443/tcp | Portunia's HTTPS API: sign-in, tokens, administration, audit, recorded and vault Remote Desktop, assistance (helper). |
| Gate (local) | Gate | 8445/tcp | Portal over HTTP for cloudflared; listens on `127.0.0.1`, never exposed. |
| Cloudflare | Gate | 8446/tcp | "Cloudflare proxy" option instead of the tunnel; Cloudflare addresses only. |
| Gate | Cloudflare | outbound | cloudflared tunnel started by the Gate (portal and assistance page); cloudflared downloaded from GitHub. |
| Gate | github.com | 443/tcp | Download of cloudflared (once per version) and of Portunia Assistance. |
| Gate | Servers | 22, 3389, 443… | Connections to the protected servers. |
| Gate | github.com | 443 | Updates (package) and download of Portunia Assistance. |
| Workstations | github.com | 443 | Portunia updates. |

<a id="fichiers"></a>

## Appendix D — Files and locations

| Path | Content |
| --- | --- |
| `/usr/lib/portunia-gate/portunia-gate` | The program. |
| `/usr/bin/portunia-gate` | Administration command (starts the program with the service's environment). |
| `/etc/portunia-gate/portunia-gate.env` | Ports, time zone, database access, certificate, language. |
| `/var/lib/portunia-gate/` | Keys (`vault.key`, `signature.key`, `ssh_host_ed25519_key`), certificate (`api.crt`, `api.key`), `enregistrements/` (and, next to a Remote Desktop session, `<session>-fichiers/`: copies of the copied files, `file_copies_mb` setting), `assist/` (Portunia Assistance). |
| Database `portunia-gate` | Accounts, groups, servers, rules, settings, sessions, log. |
| Service | `portunia-gate.service` (systemd), account `_portunia_gate`, automatic restart on failure. |

### Variables of `portunia-gate.env`

| Variable | Default | Meaning |
| --- | --- | --- |
| `PORTUNIA_GATE_SSH_LISTEN` | `0.0.0.0:2222` | SSH listener. |
| `PORTUNIA_GATE_API_LISTEN` | `0.0.0.0:8443` | API listener. |
| `PORTUNIA_GATE_PORTAL_LISTEN` | `127.0.0.1:8445` | Portal for Cloudflare Tunnel. |
| `PORTUNIA_GATE_PORTAL_TLS_LISTEN` | not set | Portal HTTPS listener; if set, it overrides the one chosen in Administer › Web portal. |
| `PORTUNIA_GATE_TLS_CERT`, `PORTUNIA_GATE_TLS_KEY` | self-signed | API certificate (and the portal's if it has no other). |
| `PORTUNIA_GATE_PORTAL_TLS_CERT`, `PORTUNIA_GATE_PORTAL_TLS_KEY` | not set | Portal certificate imposed by files (read again as soon as they change); overrides the one set in Administer › Web portal. |
| `PORTUNIA_GATE_LANG` | system language | Language of the service. |
| `TZ` | chosen time zone | Time ranges of the rules. |

After a change: `sudo systemctl restart portunia-gate` (cuts open sessions).

<a id="evenements"></a>

## Appendix E — Log events

| Family | Events |
| --- | --- |
| Service | `service.demarre` `installation` |
| Authentication | `auth.reussie` `auth.echec` `acces.refuse` |
| Accounts | `compte.cree` `compte.modifie` `compte.supprime` `compte.desactive` `compte.reactive` `compte.mot_de_passe` `compte.mot_de_passe_provisoire` `compte.totp_enrole` `compte.totp_reinitialise` `compte.cle_ajoutee` |
| Groups, rules, settings | `groupe.cree` `groupe.supprime` `groupe.membre_ajoute` `regle.creee` `regle.modifiee` `regle.supprimee` `reglage.modifie` |
| Servers | `cible.creee` `cible.modifiee` `cible.supprimee` `cible.empreinte_memorisee` `cible.empreinte_differente` `cible.empreinte_oubliee` `cible.cle_installee` `cible.coffre_refuse` `cible.identifiants_refuses` |
| Files (Remote Desktop) | `fichier.copie` (direction, name, size, SHA-256, copy kept) `fichier.bloque` (attempts held back for lack of permission) |
| Sessions | `session.ouverte` `session.fermee` `commande.executee` `sftp.ouvert` `rebond.ouvert` `rebond.ferme` `tunnel.ouvert` `tunnel.ferme` `web.ouvert` `web.page` |
| Audit | `enregistrement.consulte` `enregistrement.revele` |
| Portal | `portail.connexion` `portail.echec` `portail.active` `portail.desactive` `portail.modifie` `portail.sessions_fermees` |
| Assistance | `assistance.ouverte` `assistance.fermee` `assistance.reglage` `assistance.place_inconnue` `assistance.programme` |

> [!NOTE]
> **To watch** — Bursts of `auth.echec` (brute-force attack), `cible.empreinte_differente` (server replaced or spoofed), `enregistrement.revele` (revealed input), `compte.*` and `regle.*` outside planned changes. Event names stay in French whatever the language of the Gate, so that filters and exports stay stable.

<a id="routine"></a>

## Appendix F — Periodic checks

| When | Check |
| --- | --- |
| Every day | Last night's backup present and readable; service running; disk space. |
| Every week | Unusual sign-in failures and access refusals; revealed input; tunnel state if the portal is used. |
| Every month | `sudo portunia-gate audit --verify`; update of Portunia Gate and of the system; review of *expired* rules and of disabled accounts to delete. |
| Every quarter | Access review with the managers (who is in which group, who is administrator); restore test; check that the copy of `vault.key` is still readable. |
| At each departure | Account disabled the same day; personal accounts on the servers (recorded and jump modes) disabled. |

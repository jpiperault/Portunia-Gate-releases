# Portunia Gate

[English](README.md) · **Français**

**La passerelle SSH et Bureau à distance de [Portunia](https://github.com/jpiperault/Portunia-releases).**
Les utilisateurs passent par la Gate pour atteindre les serveurs : la Gate vérifie qui ils sont, décide où ils peuvent aller, fournit si besoin le compte du serveur sans jamais le montrer, enregistre les sessions et tient un journal d'audit infalsifiable.

> Portunia Gate s'appelait **Heimdia** jusqu'à la version 0.11 (les anciennes versions ci-dessous gardent ce nom). Commandes, paquet, service et variables s'appellent désormais `portunia-gate` / `PORTUNIA_GATE_*`.

## Ce que fait la Gate

| | |
|---|---|
| **Trois modes par serveur** | **Rebond** : relais de bout en bout ; la Gate trace la connexion sans en voir le contenu. **Enregistré** : la Gate relaie et enregistre la session ; l'utilisateur tape ses propres identifiants. **Coffre** : comme Enregistré, et la Gate fournit le compte du serveur, que l'utilisateur ne connaît jamais. |
| **Double authentification** | Mot de passe, clé SSH et code TOTP (Google Authenticator, Microsoft Authenticator…), combinables. Un code ne sert qu'une fois. |
| **Droits** | Des règles relient un groupe à des serveurs, avec jours, heures, date d'expiration, SFTP et tunnels permis ou non. Tout ce qui n'est pas permis est refusé. |
| **Enregistrement et relecture** | Chaque session en mode Coffre ou Enregistré est enregistrée et signée. Dans Portunia, relisez-la comme une vidéo : frappes en sous-titres, temps morts sautés, commandes sensibles signalées, recherche dans toutes les sessions. Les mots de passe tapés sont masqués ; chaque relecture est tracée. |
| **Journal d'audit** | Chaque événement contient l'empreinte du précédent : une ligne supprimée ou modifiée se voit. |
| **Bureau à distance** | Serveurs Windows (RDP) derrière la Gate, depuis Portunia ou le navigateur. |
| **Portail web** | Pour qui n'a pas Portunia sous la main : connexion, terminal et Bureau à distance dans le navigateur. Publié par un tunnel Cloudflare lancé par la Gate, un relais dédié en DMZ, un proxy ou WAF, ou directement en HTTPS, avec le certificat de votre PKI ou Let's Encrypt. Tout se règle dans Portunia. |
| **Assistance à distance** | Dépanner quelqu'un sur son PC Windows : il lance un petit programme, vous lit un code et choisit voir seulement ou voir et contrôler. Chiffré de bout en bout, relayé par la Gate. |
| **Portunia** | La Gate apparaît dans la barre latérale de Portunia ; ses serveurs s'ouvrent d'un clic ; administrateurs et auditeurs gèrent tout depuis Portunia Équipes. |
| **Autres clients** | Tout client SSH : OpenSSH, PuTTY, MobaXterm, WinSCP (SFTP). |

## Documentation

- **[Guide d'administration](docs/guide-administration.md)** : installation, comptes, serveurs et coffre, règles, portail web, assistance, audit, sauvegardes, mises à jour, dépannage, ligne de commande.
- **[Guide d'utilisation](https://github.com/jpiperault/Portunia-releases/blob/main/docs/guide-utilisation.md)** : Portunia, connexion par une Gate, portail web, assistance.
- Guide d'exploitation installé avec le paquet : `/usr/share/doc/portunia-gate/exploitation.fr.md`.

## Installer

**Serveur** : Debian 12 ou 13, Ubuntu 22.04 ou plus récent, processeur x86-64. 1 processeur, 2 Go de mémoire et 10 Go de disque suffisent. Une machine virtuelle ou un container LXC conviennent (rien à activer).

```bash
curl -fsSLO https://github.com/jpiperault/Portunia-Gate-releases/releases/latest/download/installer.sh
sudo bash installer.sh
```

L'assistant télécharge la dernière version, vérifie son empreinte SHA-256, installe le paquet et PostgreSQL, puis pose cinq questions (Entrée garde la valeur proposée) :

| Question | Valeur proposée |
|---|---|
| Nom de la Gate | Portunia Gate |
| Identifiant de l'administrateur | admin |
| Port SSH | 2222 |
| Port de l'API (utilisé par Portunia) | 8443 |
| Fuseau horaire | celui du serveur |

Il crée ensuite les clés, le certificat, la base et le compte administrateur, démarre la Gate et affiche ce qu'il faut saisir dans Portunia : **adresse**, **empreinte du certificat**, **identifiant** et **mot de passe provisoire**. La Gate tourne sous son propre compte sans privilèges, redémarre avec le serveur et se relance seule si elle cesse de répondre.

### Mettre la clé du coffre à l'abri, tout de suite

La clé du coffre chiffre les mots de passe des serveurs et les codes TOTP : sans elle, une restauration perd tous ces secrets. L'assistant en dépose une copie dans votre dossier personnel (`~/portunia-gate-vault.key`) et affiche les commandes pour la copier sur votre poste et l'effacer du serveur. Gardez-la dans un gestionnaire de mots de passe ou le coffre de l'entreprise.

### Relancer l'assistant

| | |
|---|---|
| `sudo bash installer.sh` | Menu : informations, mise à jour, administrateur de secours, portail web, durcissement |
| `sudo bash installer.sh info` | Adresse et empreinte à donner aux utilisateurs |
| `sudo bash installer.sh mise-a-jour` | Installe la dernière version et redémarre (**les sessions ouvertes sont coupées**) |
| `sudo bash installer.sh admin julien` | Crée un administrateur de secours avec un mot de passe provisoire |
| `sudo bash installer.sh durcir` | Durcit un serveur dédié à la Gate (`simulation` montre ce qui changerait, `annuler` revient en arrière) |
| `sudo bash installer.sh relais CODE` | Fait de cette machine un relais dédié du portail web (code donné par la Gate) |
| `sudo bash installer.sh --paquet portunia-gate_X.Y.Z_amd64.deb` | Installe un paquet téléchargé à l'avance (serveur sans Internet) |

Sans terminal (automatisation), l'assistant prend les valeurs proposées ou celles des variables `PORTUNIA_GATE_NOM`, `PORTUNIA_GATE_ADMIN`, `PORTUNIA_GATE_SSH_PORT`, `PORTUNIA_GATE_API_PORT` et `TZ`.

L'assistant, la Gate et le portail web parlent anglais, ou français quand le système ou le navigateur est en français.

## Ajouter la Gate dans Portunia

1. Portunia › **Extensions** › Portunia Gate › **Activer** : la section **Gates** apparaît dans la barre latérale.
2. Gates › **+** › saisissez l'adresse donnée par l'assistant.
3. Comparez l'empreinte affichée par Portunia à celle de l'assistant (ou `sudo portunia-gate info` sur le serveur). Si elle diffère, n'allez pas plus loin.
4. Identifiant et mot de passe provisoire ; **laissez le champ Code vide**.
5. Choisissez votre mot de passe, scannez le QR code avec votre application d'authentification et tapez le code qu'elle affiche.
6. Les serveurs autorisés apparaissent sous la Gate ; un clic ouvre la session.
7. Administration (Portunia Équipes) : menu **⋯** de la Gate › **Administrer** : serveurs et coffre, comptes, groupes, règles, portail web, assistance, réglages. Audit : menu **⋯** de la Gate › **Audit** : sessions et journal.

## Portail web

Dans Portunia : Administrer › **Portail web**, choisissez la façon de publier le portail, puis l'adresse publique, le groupe autorisé et l'interrupteur.

- **Tunnel Cloudflare** (aucun port ouvert) : dans Cloudflare, Networking › Tunnels › **Create Tunnel**, copiez la commande d'installation affichée et collez-la dans Portunia : la Gate lance le tunnel elle-même. Puis, dans le tunnel, Routes › Add route › **Published application**, Service URL `http://localhost:8445`.
- **Interne / VPN**, **HTTPS direct** ou **proxy / WAF** : ouvrez l'écoute HTTPS depuis le même écran (port 8446 par défaut ; redirigez-y le 443 sur le pare-feu), puis importez le certificat de votre PKI (PEM ou PFX) ou laissez la Gate en obtenir un auprès de **Let's Encrypt** et le renouveler.
- **Relais dédié** : une machine en DMZ sert le portail ; la Gate s'y connecte, jamais l'inverse.

Détails dans le [guide d'administration](docs/guide-administration.md), chapitre 10.

## Se connecter sans Portunia

```bash
ssh -p 2222 julien@gate              # menu des serveurs autorisés
ssh -p 2222 julien%web-01@gate       # directement sur web-01
ssh -J julien@gate:2222 admin@fw-01  # rebond de bout en bout
sftp -P 2222 julien%web-01@gate      # fichiers (mode Coffre, si la règle le permet)
```

## Signaler un problème

Ouvrez une [issue](https://github.com/jpiperault/Portunia-Gate-releases/issues) avec la version (`sudo portunia-gate info`), ce que vous faisiez et le message affiché (`sudo journalctl -u portunia-gate -n 50` pour le journal du service).

---

© Portunia. Tous droits réservés. Portunia Gate s'appuie sur des logiciels libres (russh, IronRDP, xterm.js…), sous licences MIT et Apache 2.0.

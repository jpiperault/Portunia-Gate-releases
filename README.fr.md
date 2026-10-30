# Heimdia

[English](README.md) · **Français**

**Le bastion SSH et Bureau à distance de [Portunia](https://github.com/jpiperault/Portunia-releases).**
Les utilisateurs passent par Heimdia pour atteindre les serveurs : Heimdia vérifie qui ils sont, décide où ils peuvent aller, fournit si besoin le compte du serveur sans le leur montrer, enregistre les sessions et trace tout dans un journal infalsifiable.

## Ce que fait Heimdia

| | |
|---|---|
| **Trois modes par serveur** | **Rebond** : relais de bout en bout ; le bastion journalise la connexion sans voir son contenu. **Enregistré** : le bastion relaie et enregistre la session ; l'utilisateur tape ses propres identifiants. **Coffre** : comme Enregistré, et le bastion fournit le compte du serveur, que l'utilisateur ne connaît jamais. |
| **Double authentification** | Mot de passe, clé SSH et code TOTP (Google Authenticator, Microsoft Authenticator…), combinables au choix. Un code ne sert qu'une fois. |
| **Droits** | Des règles relient un groupe à des serveurs, avec jours, heures, date d'expiration, SFTP et tunnels autorisés ou non. Tout ce qui n'est pas permis est refusé. |
| **Enregistrement et relecture** | Chaque session en mode coffre ou enregistré est enregistrée et signée. Dans Portunia, elle se relit comme une vidéo : la frappe en sous-titres, les pauses sautées, les commandes sensibles repérées, la recherche dans toutes les sessions. Les mots de passe tapés sont masqués ; chaque relecture est inscrite au journal. |
| **Journal d'audit** | Chaque événement contient l'empreinte du précédent : une ligne supprimée ou modifiée se voit. |
| **Bureau à distance** | Serveurs Windows (RDP) derrière le bastion, depuis Portunia ou le navigateur. |
| **Portail web** | Pour qui n'a pas Portunia sous la main : connexion, terminal et Bureau à distance dans le navigateur, derrière Cloudflare, sans port ouvert. Activé ou coupé d'un clic dans Portunia. |
| **Portunia** | Le bastion apparaît dans la barre latérale de Portunia ; ses serveurs s'ouvrent d'un clic ; administrateurs et auditeurs gèrent tout depuis Portunia Équipes. |
| **Autres clients** | N'importe quel client SSH : OpenSSH, PuTTY, MobaXterm, WinSCP (SFTP). |

## Installer

**Serveur** : Debian 12 ou 13, Ubuntu 22.04 ou suivantes, processeur x86-64. 1 processeur, 2 Go de mémoire et 10 Go de disque suffisent. Une machine virtuelle ou un container LXC conviennent (rien à activer).

```bash
curl -fsSLO https://github.com/jpiperault/Heimdia-releases/releases/latest/download/installer.sh
sudo bash installer.sh
```

L'assistant télécharge la dernière version, vérifie son empreinte SHA-256, installe le paquet et PostgreSQL, puis pose cinq questions (la touche Entrée garde la valeur proposée) :

| Question | Valeur proposée |
|---|---|
| Nom du bastion | Heimdia |
| Identifiant de l'administrateur | admin |
| Port SSH | 2222 |
| Port de l'API (utilisée par Portunia) | 8443 |
| Fuseau horaire | celui du serveur |

Il crée ensuite les clés, le certificat, la base et le compte administrateur, démarre le bastion et affiche ce qu'il faut saisir dans Portunia : **adresse**, **empreinte du certificat**, **identifiant** et **mot de passe provisoire**. Le bastion redémarre seul avec le serveur, et de lui-même s'il venait à ne plus répondre.

### Sauvegarder la clé du coffre, tout de suite

La clé du coffre chiffre les mots de passe des serveurs et les codes TOTP : sans elle, une restauration perd tous ces secrets. L'assistant en dépose une copie dans votre dossier personnel (`~/heimdia-coffre.key`) et affiche les commandes pour la copier sur votre poste puis l'effacer du serveur. Rangez-la dans un gestionnaire de mots de passe ou le coffre de l'entreprise.

### Relancer l'assistant

| | |
|---|---|
| `sudo bash installer.sh` | Menu : informations, mise à jour, administrateur de secours, portail web |
| `sudo bash installer.sh info` | Adresse et empreinte à donner aux utilisateurs |
| `sudo bash installer.sh mise-a-jour` | Installe la dernière version et redémarre (**les sessions ouvertes sont coupées**) |
| `sudo bash installer.sh admin julien` | Crée un administrateur de secours avec un mot de passe provisoire |
| `sudo bash installer.sh portail` | Installe Cloudflare Tunnel pour le portail web |
| `sudo bash installer.sh --paquet heimdia_X.Y.Z_amd64.deb` | Installe un paquet téléchargé à l'avance (serveur sans Internet) |

Sans terminal (automatisation), l'assistant prend les valeurs proposées ou celles des variables `HEIMDIA_NOM`, `HEIMDIA_ADMIN`, `HEIMDIA_SSH_PORT`, `HEIMDIA_API_PORT` et `TZ`.

L'assistant, le bastion et le portail web parlent français quand le système ou le navigateur est en français, anglais sinon.

Le guide d'exploitation (sauvegarde, journaux, dépannage) est installé avec le paquet : `/usr/share/doc/heimdia/exploitation.fr.md`.

## Ajouter le bastion dans Portunia

1. Portunia › **Extensions** › Heimdia › **Activer** : la section **Bastions** apparaît dans la barre latérale.
2. Bastions › **+** › tapez l'adresse donnée par l'assistant.
3. Comparez l'empreinte affichée par Portunia à celle de l'assistant (ou `sudo heimdia info` sur le serveur). Si elle diffère, n'allez pas plus loin.
4. Identifiant et mot de passe provisoire ; **laissez le champ Code vide**.
5. Choisissez votre mot de passe, scannez le code QR avec votre application d'authentification et tapez le code qu'elle affiche.
6. Les serveurs autorisés apparaissent sous le bastion ; un clic ouvre la session.
7. Administration (Portunia Équipes) : menu **⋯** du bastion › **Administrer** : serveurs et coffre, comptes, groupes, règles, portail web, réglages. Audit : menu **⋯** du bastion › **Audit** : sessions et journal.

## Portail web

1. Dans Cloudflare Zero Trust › Networks › Tunnels, créez un tunnel (Cloudflared) et copiez son jeton.
2. Sur le bastion : `sudo bash installer.sh portail`, puis collez le jeton.
3. Dans l'onglet *Public Hostname* du tunnel : votre adresse, type **HTTP**, URL `localhost:8445`.
4. Dans Portunia : Administrer › **Portail web** : adresse publique, groupe autorisé, puis l'interrupteur. Cloudflare Access devant l'adresse est conseillé.

## Se connecter sans Portunia

```bash
ssh -p 2222 julien@bastion              # menu des serveurs autorisés
ssh -p 2222 julien%web-01@bastion       # directement sur web-01
ssh -J julien@bastion:2222 admin@fw-01  # rebond de bout en bout
sftp -P 2222 julien%web-01@bastion      # fichiers (mode coffre, si la règle l'autorise)
```

## Signaler un problème

Ouvrez une [issue](https://github.com/jpiperault/Heimdia-releases/issues) avec la version (`sudo heimdia info`), ce que vous faisiez et le message affiché (`sudo journalctl -u heimdia -n 50` pour le journal du service).

---

© Heimdia. Tous droits réservés. Heimdia s'appuie sur des logiciels libres (russh, IronRDP, xterm.js…), sous licences MIT et Apache 2.0.

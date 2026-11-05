# Guide d'administration

*Portunia Gate 0.30.0 · Portunia 0.69.0 · Mise à jour : 5 novembre 2026*

> Le guide d'utilisation de Portunia est dans [Portunia-releases](https://github.com/jpiperault/Portunia-releases/blob/main/docs/guide-utilisation.md). English version: [administration guide](administration-guide.md).

Installer, régler et exploiter la Gate, et préparer Portunia sur les postes : comptes, serveurs, règles, portail web, assistance, audit, sauvegardes, mises à jour et dépannage. Chaque écran est décrit avec ses champs, ses valeurs par défaut et les commandes équivalentes sur le serveur.

<a id="avant"></a>

## 1\. Avant de commencer

Portunia Gate est le point de passage obligé entre les utilisateurs et les serveurs. Il vérifie **qui** se connecte (mot de passe, clé SSH, code TOTP), décide **où** chacun peut aller (règles), fournit si besoin le compte du serveur **sans le montrer** (coffre), **enregistre** les sessions et inscrit tout dans un **journal infalsifiable**.

L'administration se fait depuis Portunia (onglet **Administrer** de la Gate) ou en ligne de commande sur le serveur (`sudo portunia-gate …`). Les deux font la même chose et sont tracées au journal de la même façon.

### Les rôles

| Rôle | Peut | Ne peut pas |
| --- | --- | --- |
| **Utilisateur** (`user`) | Ouvrir les serveurs que ses règles autorisent. Relire ses propres sessions si le réglage le permet (« Mes sessions »). | Voir l'administration ou l'audit des autres. |
| **Auditeur** (`auditor`) | Relire toutes les sessions, chercher dans leur contenu, consulter et exporter le journal. | Modifier quoi que ce soit ; révéler les saisies masquées. |
| **Administrateur** (`admin`) | Tout : serveurs et coffre, comptes, groupes, règles, réglages, portail, assistance, audit, révélation des saisies masquées. | Se retirer ses propres droits d'administrateur ou désactiver son propre compte (la Gate le refuse). |

### Le vocabulaire

| Terme | Sens |
| --- | --- |
| Serveur (cible) | Une machine derrière la Gate : SSH (Linux, équipements réseau) ou Bureau à distance (Windows). |
| Mode | Façon dont la Gate relaie : **rebond**, **enregistré** ou **coffre** (voir [§7](#modes)). |
| Groupe | Ensemble de comptes. Les droits se donnent toujours à un groupe, jamais à un compte seul. |
| Règle | Relie un groupe à des serveurs, avec jours, heures, expiration, SFTP, tunnels, frappe enregistrée. |
| Coffre | Les mots de passe et clés des serveurs, chiffrés par la **clé du coffre** (`vault.key`). |
| Empreinte | Identité d'une clé ou d'un certificat. Portunia épingle celle de la Gate ; la Gate épingle celle de chaque serveur. |
| Jeton | Laissez-passer à usage unique (60 s) que Portunia demande à chaque ouverture : on ne retape pas son code. |

### Ce qu'il faut avoir

| Élément | Détail |
| --- | --- |
| Serveur | Debian 12 ou 13, Ubuntu 22.04 ou plus récent ; machine virtuelle ou container LXC. Paquet : 1 processeur, 2 Go de mémoire, 10 Go de disque. Docker : 2 processeurs, 4 Go, 20 Go. |
| Réseau | Ports **2222** (SSH) et **8443** (API) joignables depuis les postes ; la Gate doit joindre les serveurs (22, 3389…). Détail en [annexe C](#flux). |
| Heure | Serveur et téléphones à l'heure (NTP) : les codes TOTP en dépendent. |
| Postes | Portunia 0.48 ou suivant. L'onglet **Administrer** demande l'édition **Équipes** (pendant l'avant-première, toutes les fonctions sont ouvertes). |
| Téléphone | Une application d'authentification : Google Authenticator, Microsoft Authenticator, FreeOTP, 1Password… |
| Portail web et assistance (option) | Un nom de domaine géré par Cloudflare et un compte Cloudflare (gratuit) pour le tunnel. |

<a id="installer"></a>

## 2\. Installer la Gate

|  | Paquet Debian (conseillé) | Docker |
| --- | --- | --- |
| Durée | 1 à 2 minutes (paquet déjà compilé) | 10 à 30 minutes (compilation sur le serveur) |
| Fonctionnement | Service systemd durci, PostgreSQL de la distribution | Conteneurs Portunia Gate et PostgreSQL 17 (et cloudflared en option) |
| Container LXC | Rien à activer (sauf si le service ne démarre pas : *nesting*) | Options *nesting* et *keyctl* obligatoires |
| Processeurs | x86-64 | x86-64 et ARM |

### Paquet Debian

```
curl -fsSLO https://github.com/jpiperault/Portunia-Gate-releases/releases/latest/download/installer.sh
sudo bash installer.sh
```

L'assistant télécharge la dernière version, **vérifie son empreinte SHA-256**, installe le paquet et PostgreSQL, crée la base `portunia-gate` (accès local par socket, sans mot de passe, propriétaire `_portunia_gate`) et le service `portunia-gate`. Le service tourne sous le compte système `_portunia_gate` et n'écrit que dans `/var/lib/portunia-gate`.

Sans Internet, ou pour une version précise : copiez le `.deb` sur le serveur puis `sudo bash installer.sh --paquet portunia-gate_0.11.0_amd64.deb`.

### Docker

```
git clone https://github.com/jpiperault/Portunia-Gate.git
cd Portunia-Gate
sudo ./installer.sh --docker
```

Le dépôt est privé : `git clone` demande un identifiant GitHub et, comme mot de passe, un jeton d'accès en lecture. L'assistant installe Docker s'il manque et, avec moins de 4 Go de mémoire, propose un fichier d'échange temporaire pour la compilation. Toutes les commandes `installer.sh` et `docker compose` se lancent ensuite depuis ce dossier.

### Les cinq questions

| Question | Proposé | Conseil |
| --- | --- | --- |
| Nom de la Gate | Portunia Gate | Un nom parlant (« Gate ACME ») : il apparaît dans Portunia et dans l'application d'authentification de chacun. |
| Identifiant de l'administrateur | admin | Plutôt votre nom (`julien`) : le journal dira qui a fait quoi. Minuscules, chiffres, point, tiret, souligné ; 32 caractères au plus. |
| Port SSH | 2222 | Gardez 2222 sauf conflit. Le port 22 reste au SSH d'administration du serveur lui-même. |
| Port de l'API | 8443 | Si vous le changez, les utilisateurs ajouteront `adresse:port` dans Portunia. |
| Fuseau horaire | celui du serveur | Celui des utilisateurs : il sert aux plages horaires des règles. |

L'assistant crée ensuite les clés, le certificat, la base et le compte administrateur, démarre la Gate et affiche un encadré avec l'**adresse**, l'**empreinte** du certificat, l'**identifiant** et un **mot de passe provisoire** (trois groupes de quatre caractères). Notez-les pour la première connexion. Une installation interrompue se reprend en relançant l'assistant : les réponses déjà données sont reproposées.

> [!CAUTION]
> **Tout de suite : la clé du coffre** — L'assistant dépose une copie de `vault.key` dans votre dossier personnel (`~/portunia-gate-vault.key`). Copiez-la hors du serveur (gestionnaire de mots de passe, coffre de l'entreprise), puis effacez la copie : `scp moi@gate:~/portunia-gate-vault.key .` depuis votre poste, puis `rm ~/portunia-gate-vault.key` sur le serveur. Sans cette clé, une restauration perd tous les secrets du coffre et tous les codes TOTP.

### Installation sans questions

Hors terminal (outil d'automatisation), l'assistant prend les valeurs proposées ou celles des variables `PORTUNIA_GATE_NOM`, `PORTUNIA_GATE_ADMIN`, `PORTUNIA_GATE_SSH_PORT`, `PORTUNIA_GATE_API_PORT`, `TZ` :

```
sudo PORTUNIA_GATE_ADMIN=julien PORTUNIA_GATE_NOM="Gate ACME" bash installer.sh </dev/null
```

### Relancer l'assistant

| Commande | Effet |
| --- | --- |
| `sudo bash installer.sh` | Sur une Gate installé : menu des choix ci-dessous. |
| `sudo bash installer.sh info` | Adresse, ports et empreinte à donner aux utilisateurs. |
| `sudo bash installer.sh mise-a-jour` | Dernière version, redémarrage. **Coupe les sessions ouvertes.** |
| `sudo bash installer.sh admin secours` | Crée un administrateur avec mot de passe provisoire (dernier administrateur bloqué, téléphone perdu). |
| `sudo bash installer.sh portail` | Confie le tunnel Cloudflare à la Gate (commande ou jeton à coller), comme Administrer › Portail web. |
| `sudo bash installer.sh durcir` | Durcit le serveur (voir [§17](#securite)) ; `durcir simulation`, `durcir etat`, `durcir annuler`. |
| `sudo bash installer.sh relais CODE NOM` | Sur une **autre** machine : installe un relais dédié avec le code donné par la Gate (voir Annexe A). |

En Docker, remplacez `sudo bash installer.sh` par `sudo ./installer.sh` dans le dossier du dépôt.

### Certificat de l'entreprise (option)

Le certificat de l'API est auto-signé : Portunia le reconnaît par son empreinte, ce qui suffit. Celui que voient les navigateurs sur le portail est distinct et se règle dans Administrer › Portail web (voir le portail web). Pour utiliser un certificat de l'entreprise pour l'API (PEM, certificat suivi de sa chaîne) :

```
sudo install -o _portunia_gate -g _portunia_gate -m 600 cert.pem /var/lib/portunia-gate/entreprise.crt
sudo install -o _portunia_gate -g _portunia_gate -m 600 cle.pem  /var/lib/portunia-gate/entreprise.key
echo 'PORTUNIA_GATE_TLS_CERT=/var/lib/portunia-gate/entreprise.crt' | sudo tee -a /etc/portunia-gate/portunia-gate.env
echo 'PORTUNIA_GATE_TLS_KEY=/var/lib/portunia-gate/entreprise.key'  | sudo tee -a /etc/portunia-gate/portunia-gate.env
sudo systemctl restart portunia-gate
```

> [!WARNING]
> **L'empreinte change** — Prévenez les utilisateurs avant : Portunia leur présentera la nouvelle empreinte, à comparer avec `sudo portunia-gate info`. Même chose au renouvellement du certificat.

<a id="jour1"></a>

## 3\. Premier jour : la liste

1.  **Mettre la clé du coffre à l'abri** et effacer la copie du dossier personnel (voir ci-dessus).
2.  **Ajouter la Gate dans Portunia** : Extensions › Portunia Gate › Activer ; Gates › **+** › adresse. **Comparez l'empreinte** affichée avec celle de l'assistant ; si elle diffère, arrêtez-vous.
3.  **Première connexion** : identifiant et mot de passe provisoire, **champ Code vide**. Choisissez un mot de passe (10 caractères au moins), scannez le QR, tapez le code à 6 chiffres.
4.  **Créer un second administrateur** (un collègue, ou un compte de secours dont le téléphone est rangé à part) : si vous perdez votre téléphone, il vous rendra l'accès.
5.  **Réglages** : nom, bannière légale, connexion par défaut, durée de session (voir [§9](#reglages)).
6.  **Groupes** par métier : « Admins Linux », « Admins Windows », « Réseau », « Prestataires »…
7.  **Serveurs** : commencez par un serveur de test en mode coffre avec « Générer une clé » (voir [§7](#serveurs)).
8.  **Une règle** qui relie un groupe à ce serveur. Ouvrez-le depuis Portunia, puis relisez la session dans Audit : la chaîne est vérifiée de bout en bout.
9.  **Comptes des utilisateurs** : créez-les, transmettez les mots de passe provisoires par un canal séparé (voir [§5](#comptes)).
10.  **Planifier la sauvegarde** de la base et du dossier de données (voir [§14](#sauvegarde)).

> [!NOTE]
> **Rien ne s'affiche sous la Gate ?** — C'est normal tant qu'aucune règle ne donne accès à un serveur : il faut un serveur, un groupe qui contient le compte et une règle active qui les relie (jour, heure et date compris).

<a id="portunia"></a>

## 4\. Portunia sur les postes

| Sujet | À savoir |
| --- | --- |
| Installation | Installateur Windows (NSIS) : **pour l'utilisateur** (sans droits d'administrateur) ou **pour tous les utilisateurs du poste** (Program Files, droits d'administrateur, conseillé sur les postes partagés et les serveurs de rebond). Les mises à jour gardent le mode choisi ; en mode « tous », Windows demande l'autorisation à chaque mise à jour. Les données restent propres à chaque compte Windows. Téléchargement : `github.com/jpiperault/Portunia-releases`, dernière version. |
| Mises à jour | Paramètres › Mises à jour : vérification au démarrage (activée par défaut), une bannière propose la nouvelle version, rien ne s'installe sans accord. Le paquet est signé, Portunia vérifie la signature avant d'installer. |
| Extensions à activer | **Portunia Gate** (section Gates), **Bureau à distance** (sinon les serveurs RDP restent grisés), **Assistance à distance** pour les dépanneurs. Menu Extensions de la barre latérale. |
| Licence | Paramètres › Licence : coller le texte reçu par e-mail. Vérifiée hors ligne, sans compte en ligne. L'édition s'affiche en bas de la barre latérale. Administrer et Audit demandent **Équipes**. |
| Verrouillage | Conseillé sur les postes d'administration : Paramètres › Sécurité, mot de passe maître (les données locales sont alors chiffrées), Windows Hello en option, verrouillage après inactivité, Ctrl+Maj+L pour verrouiller. Mot de passe maître oublié sans Windows Hello = données locales perdues, par conception. |
| Jeton de la Gate | Rangé dans le Gestionnaire d'identifiants de Windows. Le mot de passe de la Gate n'y est rangé que si l'utilisateur le demande **et** si la Gate l'autorise (réglage « mémoriser le mot de passe ») ; le code reste demandé. |
| Session expirée | Au-delà de la durée réglée sur la Gate (8 h par défaut), Portunia propose de se reconnecter puis reprend l'action en cours. |

> [!WARNING]
> **Avant la vente des licences** — Aujourd'hui Portunia est en avant-première : toutes les fonctions Pro et Équipes sont ouvertes, sans licence. Une fois la vérification des licences mise en service, les postes d'administration et d'audit auront besoin d'une licence Équipes.

<a id="comptes"></a>

## 5\. Comptes

Portunia › menu **⋯** de la Gate › **Administrer** › **Comptes**. Le tableau montre l'identifiant, le rôle, les groupes, la façon de se connecter et l'état : *mot de passe provisoire*, *code enrôlé*, *à la 1ʳᵉ connexion*, *désactivé*.

### Créer un compte

1.  **Créer un compte** : identifiant, rôle, connexion (ou « Réglage général »), groupes.
2.  Portunia affiche **une seule fois** un mot de passe provisoire. Transmettez-le par un autre canal que l'adresse de la Gate (téléphone, SMS, en main propre).
3.  L'utilisateur ajoute la Gate dans Portunia, se connecte avec ce mot de passe et le champ Code vide, choisit son mot de passe et enrôle son code.

En ligne de commande : `sudo portunia-gate user add marc --provisoire` (même parcours), ou `sudo portunia-gate user add marc` pour saisir le mot de passe et enrôler le code sur place (QR affiché dans le terminal ; un enrôlement abandonné supprime le compte au lieu de le laisser à moitié créé).

### Façons de se connecter

| Choix dans Portunia | Valeur | Usage |
| --- | --- | --- |
| Réglage général | (vide) | Suit le réglage de la Gate — par défaut `password+totp,key+totp`. |
| Mot de passe + code | `password+totp` | Le cas courant. |
| Mot de passe ou clé, + code | `password+totp,key+totp` | Portunia avec mot de passe, client SSH avec clé ; le code toujours. |
| Clé SSH + code | `key+totp` | Prestataires, accès sensibles. Ce compte n'utilise pas Portunia pour se connecter à la Gate (voir limites). |
| Mot de passe seul | `password` | À éviter ; réservé aux essais. |
| Clé SSH seule | `key` | Comptes techniques (scripts) ; client SSH standard seulement. |

Un code ne sert qu'une fois : le même code à 6 chiffres est refusé s'il a déjà été utilisé, il faut attendre le suivant (30 s). Cinq échecs en dix minutes sur un compte imposent une attente.

### Les actions sur un compte

| Bouton | Effet | Commande |
| --- | --- | --- |
| Modifier | Rôle, connexion, groupes, désactivation. | `user role`, `user disable` |
| Mot de passe provisoire | Nouveau mot de passe à transmettre ; **ferme toutes les sessions du compte** : Portunia, portail web et sessions ouvertes. Pour un mot de passe oublié ou volé. | `user passwd` |
| Réinitialiser le code | Téléphone perdu ou changé : l'utilisateur enrôlera un nouveau code à sa prochaine connexion. Ferme aussi toutes ses sessions. | `user totp` |
| Clé SSH | Ajoute une clé publique (`ssh-ed25519 AAAA…`) ; Portunia affiche son empreinte. | `user key` |
| Supprimer | Efface le compte ; l'historique de ses sessions et le journal sont gardés. | `user delete` |

### Départ d'un collaborateur

1.  **Désactiver** le compte (Modifier › Compte désactivé) plutôt que le supprimer tout de suite : ses jetons Portunia sont effacés aussitôt et il ne peut plus se connecter ; son nom reste lisible dans les filtres d'audit.
2.  S'il avait un accès **enregistré** ou **rebond** avec ses propres identifiants sur les serveurs, faites désactiver ces comptes-là aussi : la Gate ne les connaît pas.
3.  Pour les serveurs en **coffre**, rien à changer : il n'a jamais connu les mots de passe. C'est l'avantage du coffre.
4.  Supprimez le compte plus tard, une fois les audits du départ terminés.

> [!WARNING]
> **Session déjà ouverte** — La désactivation empêche toute nouvelle connexion ; une session SSH déjà ouverte va jusqu'à sa fermeture. En urgence, redémarrer le service (`sudo systemctl restart portunia-gate`) coupe toutes les sessions, y compris celles des autres.

<a id="groupes"></a>

## 6\. Groupes

Administrer › **Groupes** : créer, supprimer. Les **membres se choisissent dans la fiche de chaque compte** (Comptes › Modifier › Groupes), ou en ligne de commande : `sudo portunia-gate group member "Admins Linux" marc julien` (`group remove` pour en retirer).

Supprimer un groupe **supprime aussi ses règles**. Deux groupes ont un rôle particulier :

| Groupe (nom par défaut) | Rôle | Où le choisir |
| --- | --- | --- |
| Accès web | Seuls ses membres peuvent se connecter au portail web. | Administrer › Portail web |
| Dépanneurs | Seuls ses membres peuvent lancer une assistance à distance. | Administrer › Assistance |

> [!NOTE]
> **Ces groupes ne sont pas créés d'office** — Si l'écran indique « le groupe … n'existe pas encore », créez-le dans Groupes avec exactement ce nom (ou choisissez-en un autre dans la liste), puis ajoutez-y les comptes.

<a id="serveurs"></a>

## 7\. Serveurs et coffre

Administrer › **Serveurs et coffre** › **Ajouter un serveur**. Chaque modification est tracée au journal.

### Les champs

| Champ | Détail |
| --- | --- |
| Protocole | SSH, Bureau à distance (RDP), ou Interface web (HTTPS ou HTTP, Gate 0.30). Le port passe à 22, 3389, 443 ou 80 tant que vous ne l'avez pas changé. |
| Nom | Sans espace (`web-01`) : c'est aussi le nom à taper avec un client SSH (`julien%web-01@gate`). |
| Dossier | Range les serveurs sous la Gate dans Portunia (Production, Recette, Réseau…). |
| Adresse, Port | Telles que la Gate les joint (adresse interne). Changer l'adresse d'un serveur qui a un secret dans le coffre oblige à **ressaisir le secret** : un jeton d'administration volé ne suffit pas à l'envoyer vers une autre machine. Le journal note l'ancienne et la nouvelle adresse. Les adresses de métadonnées des hébergeurs cloud (`169.254.169.254`…) sont refusées. |
| Mode | Rebond, enregistré ou coffre (ci-dessous). |
| Compte sur le serveur | Obligatoire en coffre. En Bureau à distance : `DOMAINE\compte`, proposé à l'ouverture. |
| Secret du coffre | En coffre seulement : *Générer une clé et l'installer*, *Mot de passe du compte*, *Clé privée existante* (sans phrase de passe) ; en Bureau à distance, le mot de passe du compte seulement. En modification : *Garder le secret actuel*. |

<a id="modes"></a>

### Choisir le mode

|  | Rebond | Enregistré | Coffre |
| --- | --- | --- | --- |
| Identifiants du serveur | Tapés par l'utilisateur dans Portunia | Tapés par l'utilisateur dans le terminal (3 essais) | Fournis par la Gate, jamais montrés |
| La Gate voit le contenu | Non (chiffré de bout en bout) | Oui | Oui |
| Enregistrement | Non : connexion, durée, octets | Oui, signé | Oui, signé |
| SFTP, commande, tunnel | — | — | Oui, selon la règle |
| Portail web | RDP seulement, relayé sans enregistrement (SSH : Portunia seulement) | SSH et RDP | SSH et RDP |
| Bureau à distance | Oui | Oui | Oui (Gate 0.29) : session Windows ouverte par la Gate |
| Quand le choisir | Équipements dont le contenu ne doit pas passer en clair par la Gate ; besoin de traçabilité minimale. | Chacun garde ses comptes personnels sur les serveurs, mais on veut l'enregistrement. | **Le cas conseillé** : comptes partagés, prestataires, départs sans changer de mot de passe. |

### Mettre un serveur au coffre

**Générer une clé et l'installer** (conseillé pour Linux) : donnez une seule fois le mot de passe actuel du compte. La Gate crée une clé, la pose dans `~/.ssh/authorized_keys` du compte, vérifie qu'elle fonctionne puis oublie le mot de passe (`cible.cle_installee`). Vous pouvez ensuite changer ou désactiver le mot de passe du compte sur le serveur.

Un secret **ne se relit jamais**, ni dans Portunia ni en ligne de commande. Si le mot de passe change sur le serveur, la connexion échoue avec « Le compte du coffre a été refusé » (`cible.coffre_refuse`) : remplacez le secret (Modifier › Secret du coffre).

### Empreinte d'un serveur

À la première connexion, la Gate mémorise l'empreinte du serveur (colonne Empreinte : *connue*). Si elle change ensuite, la connexion est refusée (`cible.empreinte_differente`) : c'est soit une réinstallation, soit une machine qui se fait passer pour le serveur. Vérifiez auprès de l'équipe concernée, puis **Oublier l'empreinte** ; la suivante sera mémorisée à la prochaine connexion. Même principe pour le certificat des serveurs Windows en mode enregistré.

### Serveurs Windows (Bureau à distance)

-   Modes **rebond** et **enregistré** : l'utilisateur tape ses identifiants Windows à chaque ouverture, ni Portunia ni la Gate ne les gardent.
-   En enregistré, l'authentification Windows (NLA) se fait entre le poste et le serveur : la Gate enregistre l'image, les clics, les copier-coller (masqués jusqu'à révélation) et la frappe si la règle le demande, sans voir le mot de passe.
-   **Coffre** (Gate 0.29) : la Gate garde le mot de passe du compte Windows (local ou `DOMAINE\compte`) et ouvre elle-même la session : l'authentification NLA se fait entre la Gate et le serveur, puis la Gate relaie et enregistre comme en mode enregistré. L'utilisateur n'a rien à taper et ne voit jamais le mot de passe ; Portunia et le portail web s'authentifient auprès de la Gate avec leur jeton à usage unique. Le serveur doit proposer NLA (réglage par défaut de Windows) ; sinon la connexion est refusée (`cible.coffre_refuse`, raison « serveur sans NLA »). Le coffre RDP garde un mot de passe, pas de clé.
-   **Rebond depuis le portail web** (Gate 0.29) : le navigateur ne peut pas faire de tunnel de bout en bout ; la Gate relaie donc la session sans l'enregistrer (`rebond.ouvert`, `rebond.ferme` : durée et octets). Depuis Portunia, le rebond reste de bout en bout. Si le contenu ne doit jamais passer en clair par la Gate, ne mettez pas ces serveurs dans le groupe du portail.
-   La Gate doit joindre le port 3389. Les serveurs RDP n'apparaissent pas dans le menu SSH de la Gate.
-   À l'ouverture, l'utilisateur choisit la **qualité d'affichage**, dans Portunia comme dans le portail web : *Équilibrée* (par défaut), *Rapide* (sans fond d'écran, thèmes ni animations, pour les liaisons lentes) ou *Maximale* (tous les effets visuels). Ce choix ne change rien à l'enregistrement ni à sa relecture.
-   **Copie de fichiers** (Gate 0.28) : permise seulement par une règle avec l'option **Fichiers** ; sinon la Gate la bloque, même avec un client qui la tenterait. Chaque fichier copié est noté au journal avec son empreinte SHA-256 ; `sudo portunia-gate setting file_copies_mb 200` garde en plus une copie des fichiers (jusqu'à 200 Mo par session) à côté de l'enregistrement. En mode rebond, la Gate ne voit pas le flux : bloquez alors la copie par une stratégie de groupe Windows.

**En ligne de commande**, un serveur Windows se déclare avec `--protocol rdp` et un `--mode` : sans lui, la commande prend le mode coffre, qui demande alors le compte et son mot de passe. L'accès se donne ensuite comme pour SSH, par un groupe et une règle :

```
sudo portunia-gate target add srv-compta --host 10.0.5.12 --protocol rdp --mode enregistre --account "ACME\compta" --folder Windows
sudo portunia-gate target add srv-paie --host 10.0.5.13 --protocol rdp --mode coffre --account "ACME\svc-paie" --password --folder Windows
sudo portunia-gate group add Comptabilité
sudo portunia-gate group member Comptabilité alice bob
sudo portunia-gate rule add compta-rdp --group Comptabilité --targets srv-compta --days lun-ven --hours 7-20
sudo portunia-gate target list        # protocole, mode, empreinte (« à mémoriser » avant la première connexion)
```

`--port` si le serveur n'écoute pas sur 3389 ; en rebond et enregistré, `--account` est facultatif (compte proposé à l'ouverture, jamais de mot de passe) ; en coffre, il est obligatoire avec `--password`. Les options `--sftp`, `--tunnels` d'une règle ne concernent que SSH. Ensuite : `target edit` (adresse, mode, compte…), `target forget` après une réinstallation de Windows, `target delete` (Gate 0.25 et suivantes, voir Annexe A).

<a id="web"></a>

### Interfaces web (Gate 0.30)

-   Pour les consoles d'administration : vCenter, iDRAC, iLO, pare-feu, NAS… Protocole **Interface web (HTTPS)** ou **(HTTP)**, adresse et port du serveur ; mode **enregistré** seulement. Les identifiants de l'application se tapent dans sa page, comme d'habitude.
-   La Gate relaie chaque requête : la règle est revérifiée au plus toutes les 30 secondes, chaque **page ouverte** est notée au journal (`web.page` : serveur, chemin), l'ouverture aussi (`web.ouvert`). Les images, scripts et appels de l'application ne sont pas notés. Ni l'image ni la frappe ne sont enregistrées.
-   Le certificat d'un serveur HTTPS est vérifié par son empreinte, gardée à la première connexion, comme pour le Bureau à distance.
-   **Depuis Portunia** : le serveur s'ouvre dans une fenêtre dédiée, à travers un relais local vers la Gate ; la session web dure 8 heures.
-   **Depuis le portail web** : dans un onglet, à l'adresse `/web/<nom>/`. Un navigateur a **une interface web à la fois** : en ouvrir une seconde remplace la première. L'application partage l'adresse du portail : ne publiez sur le portail que des applications de confiance.
-   Les cookies de la Gate ne sont jamais transmis au serveur ; l'adresse du visiteur non plus (le serveur ne voit que la Gate). Les WebSocket de l'application (consoles, rafraîchissements) passent.

```
sudo portunia-gate target add vcenter --host 10.0.4.10 --protocol https --folder Infrastructure
sudo portunia-gate rule add vsphere --group "Admins VMware" --targets vcenter --days lun-ven --hours 7-20
```

### Supprimer un serveur

Son secret est effacé ; l'historique des sessions et leurs enregistrements sont gardés.

<a id="regles"></a>

## 8\. Règles d'accès

Administrer › **Règles** › **Nouvelle règle**. Principe : **tout ce qu'aucune règle ne permet est refusé**. Plusieurs règles qui touchent le même serveur pour un compte **s'additionnent**. Une modification s'applique à la **prochaine ouverture** de session ; une session déjà ouverte continue.

| Champ | Détail |
| --- | --- |
| Nom, Groupe | Un groupe par règle. Créez d'abord le groupe. |
| Serveurs | Un ou plusieurs. |
| Jours | Cases L M M J V S D ; toutes cochées = tous les jours. |
| De / À (heure) | De 0 à 23, à 1 à 24 ; vide = toute la journée. Une plage peut passer minuit (22 → 6). Fuseau de la Gate (`TZ`). |
| Transferts de fichiers | SFTP, `scp`, `rsync` (serveurs en coffre). |
| Tunnels | Redirection de ports `ssh -L` à travers le serveur (coffre). Les redirections distantes `-R` sont toujours refusées. |
| Fichiers (Bureau à distance) | Copie de fichiers par le presse-papiers, dans les deux sens, sur les serveurs Windows en mode enregistré (Gate 0.28) ou coffre (Gate 0.29). **Sans cette option, la Gate la bloque** ; le texte copié passe toujours. Avec elle, chaque fichier est inscrit au journal (`fichier.copie` : sens, nom, taille, empreinte SHA-256) et apparaît dans l'enregistrement. En ligne de commande : `--files` ; dans Portunia (0.69) : case « Bureau à distance : copie de fichiers » de la règle, et la colonne Options affiche « fichiers RDP ». Les onglets Bureau à distance de Portunia 0.69 envoient et récupèrent des fichiers ; en mode rebond, où la Gate ne voit rien, c'est Portunia qui applique l'option. |
| Enregistrer aussi la frappe | Sous-titres et recherche dans ce qui est tapé. **Capte aussi les mots de passe tapés** (masqués à la relecture, révélables par un administrateur). |
| Expire le | Vide = jamais. Après la date, la règle passe « expirée ». |

La colonne État montre *active*, *hors plage* (en dehors des jours ou heures, en ce moment) ou *expirée*.

### Exemples

| Besoin | Règle |
| --- | --- |
| Équipe Linux en heures ouvrées | Groupe « Admins Linux », serveurs de production, lun–ven, 7 h–20 h, SFTP.   `sudo portunia-gate rule add prod --group "Admins Linux" --targets web-01,db-01 --sftp --days lun-ven --hours 7-20` |
| Astreinte de nuit | Groupe « Astreinte », mêmes serveurs, tous les jours, 20 h → 7 h (passe minuit). |
| Prestataire pour une mission | Groupe « Prestataires », un serveur, frappe enregistrée, expiration à la fin du contrat.   `sudo portunia-gate rule add presta-db --group Prestataires --targets db-01 --record-input --expires 2026-12-31` |
| Échanges de fichiers avec un serveur Windows | Groupe « Comptabilité », serveur Windows enregistré, copie de fichiers permise et tracée.   `sudo portunia-gate rule add compta-fichiers --group Comptabilité --targets srv-compta --files` |
| Accès base de données depuis un poste | Groupe « DBA », serveur rebond SSH en coffre, **Tunnels** cochés : `ssh -L 5432:db-interne:5432`. |

> [!NOTE]
> **Un utilisateur ne voit rien ?** — Vérifiez dans l'ordre : le compte est dans le groupe (fiche du compte) ; la règle vise ce serveur ; elle est *active* maintenant (jour, heure du fuseau de la Gate, date) ; le compte n'est pas désactivé. Un refus est journalisé `acces.refuse`.

<a id="reglages"></a>

## 9\. Réglages généraux

Administrer › **Réglages**.

| Réglage | Défaut | Détail |
| --- | --- | --- |
| Nom de la Gate | celui de l'installation | Affiché dans Portunia et dans l'application d'authentification (pour les codes enrôlés ensuite). |
| Bannière de connexion | — | Avertissement légal avant l'identification SSH, ex. « Accès réservé. Les sessions sont enregistrées. » Conseillé : informer que les sessions sont enregistrées. |
| Connexion par défaut des comptes | Mot de passe ou clé, + code | S'applique aux comptes réglés sur « Réglage général ». |
| Durée d'une session dans Portunia | 8 h | De 1 à 168 h. Au-delà, l'utilisateur se reconnecte (mot de passe et code). |
| Langue de la Gate | langue du système | Menu et messages SSH, bannière par défaut, journaux du service. Portunia et le portail suivent la langue de chacun. |
| Mémoriser le mot de passe | non | Autorise Portunia à garder le mot de passe de la Gate ; le code reste demandé. |
| Relire ses propres sessions | non | Ajoute « Mes sessions » au menu de la Gate pour chaque compte, sans pouvoir révéler les saisies masquées. |
| Commandes sensibles | `rm -rf`, `rm -fr`, `mkfs`, `dd if=`, `chmod -R 777`, `shutdown`, `reboot`, `iptables -F`, `userdel`, `DROP TABLE`, `DROP DATABASE` | Une par ligne ; marquées en rouge sur la barre de temps du lecteur. Ajoutez celles de votre métier (`systemctl stop`, `kubectl delete`…). |
| Copies des fichiers copiés (Gate 0.28) | 0 Mo | Taille gardée par session des fichiers copiés en Bureau à distance, à côté de l'enregistrement (0 à 10240 Mo ; 0 : seuls nom, taille et SHA-256 sont notés). Dans Portunia 0.69 : Réglages › Copie de fichiers en Bureau à distance ; en ligne de commande : `setting file_copies_mb`. |

En ligne de commande : `sudo portunia-gate setting auth_policy "password+totp,key+totp"`, `setting banniere "…"`, `setting nom "…"`, `setting langue fr` (sans valeur : affiche la valeur actuelle).

<a id="portail"></a>

## 10\. Portail web

Pour ouvrir ses serveurs depuis un navigateur, sans Portunia : connexion, liste des serveurs autorisés, terminal SSH ou Bureau à distance dans un onglet. Pas d'administration. Mêmes règles, même coffre, même enregistrement, même journal ; une session web apparaît dans l'audit avec « (web) » après l'adresse.

Interfaces web (Gate 0.30) : bouton du serveur, puis l'application dans un nouvel onglet (voir [§7](#web)). Bureau à distance dans le navigateur, dans les trois modes (Gate 0.29) : l'utilisateur tape ses identifiants Windows (rien en mode coffre, où la Gate ouvre la session) et choisit la **qualité d'affichage** (Équilibrée, Rapide ou Maximale, retenue par le navigateur) ; la taille du bureau Windows suit celle de la fenêtre. Le client se charge une fois, puis reste en cache jusqu'à la version suivante de la Gate.

**Fichiers** (Gate 0.28, règle avec l'option Fichiers, copier-coller du portail permis) : bouton **Envoyer des fichiers** ou fichiers déposés sur le bureau, puis collés dans Windows (Ctrl+V) ; dans l'autre sens, les fichiers copiés dans Windows (Ctrl+C) apparaissent dans un panneau, avec un bouton **Télécharger**. Chaque fichier est noté au journal avec son empreinte.

### Mise en place

1.  **Créer le tunnel** dans Cloudflare : **Networking › Tunnels › Create Tunnel**, un nom (ex. `portunia-gate`), puis copier la commande d'installation affichée, pour n'importe quel système. Elle contient le jeton ; ne pas l'exécuter.
2.  **La donner à la Gate** : Administrer › **Portail web** › mode **Cloudflare**, coller la commande, **Enregistrer le jeton**. La Gate télécharge et lance `cloudflared` elle-même ; le tunnel passe à **Healthy** dans Cloudflare. Sans Portunia : `sudo bash installer.sh portail`.
3.  **Publier l'adresse** : dans le tunnel, onglet **Routes › Add route › Published application**. **Subdomain** et **Domain** (le *Full hostname*, ex. `acces.votre-domaine.fr`, est l'adresse publique), **Path** vide, **Service URL** `http://localhost:8445`, puis **Add route**. Jamais de route `ssh://` ou `rdp://` vers vos serveurs : elle contournerait la Gate.
4.  **Vérifier** : adresse publique `https://` + Full hostname, **Enregistrer**, puis **Tester l'adresse publique**.
5.  **Conseillé** : une application **Cloudflare Access** sur cette adresse (compte de l'entreprise exigé avant même la page), et **Turnstile** (anti-robots).
6.  **Dans Portunia** : Administrer › **Portail web** : groupe, durées, copier-coller, Turnstile, Enregistrer, puis l'interrupteur. En ligne de commande : `sudo portunia-gate portal set --url https://acces.votre-domaine.fr` (et `--group`, `--duration`, `--idle`, `--clipboard`, `--code`), puis `sudo portunia-gate portal on`.
7.  **Créer le groupe** (« Accès web » par défaut) et y ajouter les comptes concernés.

### L'écran Portail web

| Élément | Détail |
| --- | --- |
| Interrupteur | Coupé : la page affiche « Portail fermé » et les sessions web en cours se ferment. |
| Compteurs | Sessions web en cours, connexions et échecs du jour, état du tunnel : *relié*, *non relié*, *non configuré*. En mode Cloudflare, l'état détaillé (téléchargement, démarrage, connexions, erreur de cloudflared) s'affiche au-dessus des étapes. |
| Fermer toutes les sessions web | Sans couper le portail ; les utilisateurs devront se reconnecter. |
| Adresse publique | Celle que publie Cloudflare ; vérifiée à chaque connexion (une requête venue d'ailleurs est refusée). |
| Groupe autorisé | Seuls ses membres se connectent. |
| Durée d'une session web | 1, 4 ou 8 h (1 h par défaut ; de 1 à 12 h). |
| Fermeture après inactivité | 15 min, 30 min ou 1 h (15 min par défaut ; de 5 à 240 min). Avertissement 90 s avant. |
| Copier-coller | Autorise le copier-coller de texte dans le terminal web. |
| Exiger le code d'authentification | Coché par défaut. Décoché : plus de code sur le portail pour les comptes qui suivent le réglage général. Un compte qui a ses propres combinaisons de connexion les suit : `password` le dispense du code même case cochée (exception), `password+totp` le lui impose même case décochée. En ligne de commande : `sudo portunia-gate portal set --code off` ; pour un seul compte : `sudo portunia-gate user policy marc password`. |
| Turnstile | Clé du site et clé secrète (gardée chiffrée par la Gate ; laissez vide pour la garder). |

### Ce qui est toujours imposé

-   Mot de passe, et code sauf exception (réglage ci-dessus ou combinaisons propres du compte). Sur Internet, un compte sans code n'est protégé que par son mot de passe : gardez Turnstile.
-   Cinq essais par compte et vingt par adresse en dix minutes.
-   Première connexion et code perdu : pas sur le portail, mais dans Portunia ou avec un administrateur.
-   Session du navigateur revérifiée à chaque appel : un compte désactivé ou retiré du groupe est coupé tout de suite.

**Écoute HTTPS** (modes Interne, HTTPS direct, Proxy) : Administrer › Portail web › **Écoute HTTPS**, un port ou adresse:port (ex. `8446`), **Ouvrir** ; appliquée sous quelques secondes, sans redémarrer la Gate. Ports sous 1024 refusés par le système : redirigez le 443 vers le 8446 sur la box ou le pare-feu ; le pare-feu durci autorise le 8446. En ligne de commande : `sudo portunia-gate portal https 8446` (`--off` la ferme).

**Certificat du portail** (modes Interne, HTTPS direct, Proxy), distinct de celui de l'API : le changer ne touche pas les consoles Portunia. Administrer › Portail web › **Certificat du portail** :

-   **Importer un certificat** d'une PKI interne ou publique : PEM avec sa chaîne et sa clé non chiffrée, ou PFX (.pfx, .p12) et son mot de passe ; appliqué sous quelques secondes. En ligne de commande : `sudo portunia-gate portal cert --import cert.pem --key cle.pem`, `--remove` pour revenir à l'auto-signé.
-   **Let's Encrypt** (Pro) : noms de domaine, e-mail, vérification *TLS-ALPN-01* (écoute HTTPS ouverte, 443 public redirigé vers elle) ou *DNS-01 par Cloudflare* (jeton d'API Zone › DNS › Edit ; rien à ouvrir, noms génériques permis). Renouvellement seul 30 jours avant l'échéance ; après un échec, nouvel essai toutes les 6 heures ou **Renouveler maintenant**. Autorité d'essai disponible pour les tests.
-   Une alerte s'affiche 21 jours avant l'échéance d'un certificat importé.

**Proxy Cloudflare au lieu du tunnel** : `sudo portunia-gate portal https 8446` (Docker : `PORTUNIA_GATE_PORTAL_PORT=0.0.0.0:8446` dans `.env`, puis redémarrer) ; l'adresse proxifiée pointe vers ce port en HTTPS (mode SSL « Full »). La Gate n'y accepte que les adresses de Cloudflare.

<a id="assistance"></a>

## 11\. Assistance à distance

Un dépanneur prend la main sur le PC d'un proche ou d'un client. La personne va sur `https://adresse-du-portail/assistance`, télécharge **Portunia Assistance** (un programme, sans installation ni compte), et lit un **code à 9 chiffres** au dépanneur, qui le tape dans Portunia (**Assistance à distance** dans la barre latérale). Elle voit le nom du dépanneur et choisit *voir seulement* ou *voir et contrôler*. La liaison est chiffrée de bout en bout à partir du code : la Gate ne voit ni l'écran ni la frappe et ne les enregistre pas.

### Mise en place

1.  Avoir l'adresse publique du portail (§10, étapes 1 à 3). Le portail lui-même peut rester coupé.
2.  Dans Portunia, activer l'extension **Assistance à distance** sur les postes des dépanneurs.
3.  Administrer › **Assistance** : groupe des dépanneurs, durée maximale, **Enregistrer**, puis l'interrupteur. Ou `sudo portunia-gate assist set --group Dépanneurs --duration 4` puis `sudo portunia-gate assist on`.
4.  Créer le groupe (« Dépanneurs » par défaut) et y ajouter les comptes.
5.  Vérifier que la case **Portunia Assistance** passe à *prêt* : la Gate télécharge le programme depuis les versions de Portunia (accès Internet sortant nécessaire), puis une fois par jour, et vérifie son empreinte SHA-256.

### L'écran Assistance

| Élément | Détail |
| --- | --- |
| Interrupteur | Coupé par défaut. Le couper met fin aux assistances en cours. |
| Compteurs | Personnes en attente, assistances du jour, total ; état du programme (*prêt* / *absent*). |
| Page de la personne aidée | L'adresse à donner, avec un bouton Copier. |
| Groupe des dépanneurs | Seuls ses membres, connectés à la Gate depuis Portunia, peuvent lancer une assistance. |
| Durée maximale | 1, 4 ou 8 h (4 h par défaut ; de 1 à 12 h). L'assistance s'arrête d'elle-même ensuite. |

### À dire aux personnes aidées

-   Windows SmartScreen avertit à la première ouverture (programme pas encore signé) : « Informations complémentaires » › « Exécuter quand même ».
-   Un cadre et une barre **Arrêter** restent sur l'écran ; Échap trois fois arrête aussi.
-   Un code faux met fin à la tentative ; la personne en reçoit un nouveau. Une place attend 30 minutes au plus.

Journal : `assistance.ouverte`, `assistance.fermee`, `assistance.reglage`, `assistance.place_inconnue`, `assistance.programme`. Une même adresse ne peut ouvrir plus de 20 connexions en 10 minutes.

<a id="audit"></a>

## 12\. Audit et relecture des sessions

Menu **⋯** de la Gate › **Audit** › **Sessions** (administrateurs et auditeurs). Filtres par compte, serveur, période (7 jours, 30 jours, 12 mois), cases « Avec frappe » et « Avec copie de fichiers » (Portunia 0.69 ; la session porte alors le nombre de fichiers, et le lecteur les place sur la barre de temps). La recherche porte sur la frappe et le texte affiché de toutes les sessions : la colonne « Trouvé » donne le moment et l'extrait, un clic ouvre la relecture à cet endroit.

### Le lecteur

| Fonction | Détail |
| --- | --- |
| Lecture | Espace lecture/pause, flèches ±5 s, boutons ±10 s, vitesse 1× à 16×, plein écran ; clic sur la barre de temps ou une ligne du panneau pour y sauter. |
| Pauses | Les silences de plus de 5 s sont hachurés et sautés (« Sauter les pauses »). |
| Sous-titres | **CC Frappe** : ce que la personne tape, touches spéciales comprises. |
| Repères | Chaque commande ; en orange sudo, Ctrl+C et saisies masquées ; en rouge les commandes sensibles ; en bas les erreurs affichées. |
| Signature | Vérifiée à chaque ouverture ; un fichier modifié depuis la session est signalé en rouge. |
| Exports | Enregistrement `.cast` (lisible par asciinema) et transcription horodatée. |

### Saisies masquées et révélation

Une frappe qui ne s'affiche pas à l'écran (mot de passe de sudo, su, ssh…) est remplacée par des points, dans le lecteur comme dans la recherche. Elle reste dans le fichier signé : un **administrateur** peut la **révéler** ; cette relecture est inscrite au journal (`enregistrement.revele`). Toute relecture est journalisée (`enregistrement.consulte`) et le lecteur affiche en filigrane le compte qui regarde.

> [!WARNING]
> **Encadrer la révélation** — Décidez par écrit qui peut révéler et dans quels cas (incident, enquête). Le journal le prouvera, mais la règle doit exister avant.

### Bureau à distance

Les sessions RDP enregistrées se relisent comme les sessions SSH (Portunia 0.47 et suivants) : image, clics, copier-coller masqués jusqu'à révélation, frappe si la règle l'enregistre.

### Espace disque

Les enregistrements sont rangés dans `/var/lib/portunia-gate/enregistrements/AAAA/MM/` ; une session RDP pèse nettement plus qu'une session SSH. Il n'y a pas encore de durée de conservation automatique : surveillez l'espace (`sudo du -sh /var/lib/portunia-gate/enregistrements/*`) et archivez les mois anciens hors du serveur en suivant votre politique de conservation. Un fichier déplacé ne se relit plus depuis Portunia.

<a id="journal"></a>

## 13\. Journal d'audit

Audit › **Journal** : tous les événements (connexions, échecs, sessions, administration, relectures), filtrables par compte, serveur ou événement, avec export CSV. Chaque événement contient l'empreinte du précédent : une ligne supprimée ou modifiée casse la chaîne.

**Vérifier la chaîne** : bouton dans Portunia, ou `sudo portunia-gate audit --verify` qui affiche le premier événement manquant ou modifié. À faire régulièrement et avant toute remise de preuves.

```
sudo portunia-gate audit --last 100          # derniers événements
sudo portunia-gate audit --verify --last 50  # vérifie toute la chaîne, affiche les 50 derniers
sudo portunia-gate sessions --last 20        # dernières sessions
```

Lecture utile : `auth.echec` donne la raison exacte d'un refus (compte inconnu, mot de passe, code…) que l'utilisateur, lui, ne voit pas ; `auth.reussie` donne les facteurs utilisés (le champ `via` vaut `portunia` pour une connexion à l'API). La liste complète est en [annexe E](#evenements).

<a id="sauvegarde"></a>

## 14\. Sauvegarder et restaurer

### Ce qu'il faut sauvegarder

| Élément | Où | Si on le perd |
| --- | --- | --- |
| **Clé du coffre** | `/var/lib/portunia-gate/vault.key` | **Secrets des serveurs et codes TOTP perdus.** À garder hors du serveur, à part de la base. |
| Clé d'hôte SSH | `/var/lib/portunia-gate/ssh_host_ed25519_key` | Nouvelle empreinte SSH à faire valider par tous. |
| Certificat de l'API | `/var/lib/portunia-gate/api.crt`, `api.key` | Nouvelle empreinte à valider dans Portunia. |
| Clé de signature | `/var/lib/portunia-gate/signature.key` | Les nouvelles sessions sont signées par une autre clé. |
| Base PostgreSQL | base `portunia-gate` | Comptes, serveurs, règles, sessions, journal. |
| Enregistrements | `/var/lib/portunia-gate/enregistrements/` | Sessions impossibles à relire. |
| Réglages | `/etc/portunia-gate/portunia-gate.env` (Docker : `.env`) | Ports, fuseau, accès à la base à refaire. |

### Paquet Debian

```
sudo -u postgres pg_dump portunia-gate | gzip > portunia-gate-base-$(date +%F).sql.gz
sudo tar czf portunia-gate-donnees-$(date +%F).tar.gz -C /var/lib portunia-gate -C /etc portunia-gate
```

### Docker (dossier du dépôt)

```
sudo docker compose exec -T postgres pg_dump -U portunia-gate portunia-gate | gzip > portunia-gate-base-$(date +%F).sql.gz
sudo docker cp "$(sudo docker compose ps -q portunia-gate)":/var/lib/portunia-gate - | gzip > portunia-gate-donnees-$(date +%F).tar.gz
```

> [!CAUTION]
> **L'archive des données contient la clé du coffre** — Protégez-la comme un mot de passe : stockage chiffré, accès restreint, hors du serveur. Idéalement, sauvegardez chaque nuit la base et l'archive, et testez une restauration chaque trimestre.

### Restaurer (paquet Debian)

Procédure conseillée, à répéter d'abord sur une machine d'essai :

1.  Sur un serveur neuf, installer **la même version** de Portunia Gate (`sudo bash installer.sh --paquet portunia_gate_X.Y.Z_amd64.deb`), puis arrêter le service : `sudo systemctl stop portunia-gate`.
2.  Remettre les données et les réglages. L'archive range `/var/lib/portunia-gate` et `/etc/portunia-gate` dans un même dossier `portunia-gate/` : l'extraire à part, puis replacer chaque fichier :
    
    ```
    mkdir /tmp/restau && sudo tar xzf portunia-gate-donnees-AAAA-MM-JJ.tar.gz -C /tmp/restau
    sudo cp -p /tmp/restau/portunia-gate/portunia-gate.env /etc/portunia-gate/
    sudo rm /tmp/restau/portunia-gate/portunia-gate.env
    sudo cp -a /tmp/restau/portunia-gate/. /var/lib/portunia-gate/
    sudo chown -R _portunia_gate:_portunia_gate /var/lib/portunia-gate
    sudo rm -rf /tmp/restau
    ```
    
3.  Remplacer la base : `sudo -u postgres dropdb portunia_gate`, `sudo -u postgres createdb -O _portunia_gate portunia_gate`, puis `gunzip -c portunia-gate-base-….sql.gz | sudo -u _portunia_gate psql portunia-gate`.
4.  Démarrer : `sudo systemctl start portunia-gate`, puis `sudo portunia-gate info` (empreintes identiques à l'ancien Gate) et `sudo portunia-gate audit --verify`.

<a id="maj"></a>

## 15\. Mettre à jour

```
sudo bash installer.sh mise-a-jour       # paquet Debian
sudo ./installer.sh mise-a-jour          # Docker, depuis le dossier du dépôt
```

1.  Lire les notes de version (Portunia-Gate-releases) et vérifier qu'aucune version minimale de Portunia n'est demandée.
2.  **Prévenir les utilisateurs** : la mise à jour **coupe les sessions ouvertes** (SSH, RDP, portail, assistance).
3.  Faire une sauvegarde (§14).
4.  Lancer la mise à jour : en paquet, téléchargement et vérification de l'empreinte puis installation ; en Docker, `git pull` puis recompilation. Le schéma de la base est mis à jour au démarrage.
5.  Contrôler : `sudo portunia-gate info` (version), `sudo systemctl status portunia-gate`, une connexion depuis Portunia.

Le portail web charge de lui-même ses nouvelles pages après une mise à jour. Si Portunia annonce qu'une fonction demande une version plus récente de la Gate (portail : 0.7, assistance : 0.11), c'est cette commande qu'il faut lancer.

<a id="surveiller"></a>

## 16\. Surveiller le service

| Besoin | Paquet Debian | Docker |
| --- | --- | --- |
| État | `sudo systemctl status portunia-gate` | `sudo docker compose ps` |
| Journal du service en direct | `sudo journalctl -u portunia-gate -f` | `sudo docker compose logs -f portunia-gate` |
| Avant un redémarrage inattendu | `sudo journalctl -u portunia-gate -n 100` | `sudo docker compose logs --tail 100 portunia-gate` |
| Informations | `sudo portunia-gate info` — version, empreintes, nombre de comptes |

**Surveillance interne** : toutes les 20 s, la Gate vérifie que sa base répond et que son SSH accueille une connexion ; un fil séparé vérifie que ses tâches avancent. Bloqué environ une minute, il écrit la cause (`surveillance : …`) et s'arrête pour être relancé par systemd ou Docker. Si cela se produit, gardez les lignes qui précèdent le redémarrage.

**Supervision externe conseillée** : un contrôle TCP sur 2222 et 8443, l'espace disque de `/var/lib/portunia-gate`, et une alerte sur les redémarrages du service.

<a id="securite"></a>

## 17\. Bonnes pratiques de sécurité

### Durcir le serveur

Sur un serveur dédié à la Gate, l'assistant peut ne garder que le nécessaire : pare-feu fermé sauf les ports utiles, services inutiles arrêtés, SSH du système et noyau resserrés, mises à jour de sécurité automatiques. Facultatif : `--durcir` à l'installation, ou `sudo bash installer.sh durcir` ensuite. `durcir simulation` montre le plan sans rien changer, et `durcir annuler` retire le durcissement, sauf les paquets supprimés. Sans confirmation d'une nouvelle connexion SSH sous 5 minutes, le pare-feu et le SSH du système reviennent à leur état d'avant.

L'onglet **Administrer › Durcissement** de Portunia affiche l'état relevé sur le serveur toutes les 15 minutes et signale un pare-feu arrêté, un service relancé ou un relevé trop ancien. Il ne modifie rien : le durcissement se pilote sur le serveur, en root. C'est un tableau de bord, pas une preuve si le serveur est compromis.

### Limites et fermetures automatiques

Depuis Portunia Gate 0.14, les essais de connexion sont comptés ensemble pour le SSH, Portunia et le portail, sur 10 minutes : 5 pour un compte depuis une adresse, 20 pour une adresse, 30 pour un compte toutes adresses confondues.

Une session se ferme en moins de 30 secondes si le compte est désactivé, ou si la règle est retirée, expirée ou **hors de ses horaires**. Une session enregistrée ne s'ouvre pas si l'enregistrement est impossible, et se ferme si une écriture échoue. Sous `PORTUNIA_GATE_MIN_FREE_MB` (512 Mo par défaut), les nouvelles sessions enregistrées sont refusées. Surveillez les événements `auth.bloquee`, `session.coupee`, `enregistrement.echec` et `disque.presque_plein`.

| Sujet | Recommandation |
| --- | --- |
| Exposition | Ports 2222 et 8443 ouverts seulement depuis le réseau interne ou le VPN. Pour l'extérieur, le portail web derrière Cloudflare Tunnel + Access, sans port entrant. |
| Serveurs | N'autoriser SSH et RDP sur les serveurs **que depuis la Gate** (pare-feu), sinon on le contourne. |
| Comptes | Code TOTP pour tous ; « Mot de passe seul » interdit hors essais. Un compte nominatif par personne, jamais de compte partagé sur la Gate. |
| Administrateurs | Deux au minimum, pas plus que nécessaire. Les auditeurs n'ont pas besoin d'être administrateurs. |
| Coffre | Préférer le mode coffre avec « Générer une clé » ; changer ensuite le mot de passe du compte sur le serveur. |
| Frappe enregistrée | Seulement là où c'est justifié (prestataires, serveurs critiques) : elle capte les mots de passe tapés. Informer les personnes (bannière, charte). |
| Clé du coffre | Hors du serveur, dans un coffre de l'entreprise ; jamais dans un ticket ou un e-mail. |
| Empreintes | Toujours comparer l'empreinte de la Gate à l'ajout ; ne jamais « oublier » l'empreinte d'un serveur sans savoir pourquoi elle a changé. |
| Heure | NTP sur la Gate : codes TOTP, plages horaires et horodatage du journal en dépendent. |
| Système | Mises à jour de sécurité de Debian/Ubuntu (`unattended-upgrades`), SSH d'administration du serveur (port 22) réservé aux administrateurs système. |
| Postes | Mot de passe maître et verrouillage dans Portunia sur les postes d'administration. |

<a id="depannage"></a>

## 18\. Dépannage

### Installation

| Symptôme | Cause et solution |
| --- | --- |
| « Docker ne démarre pas » dans un container LXC | Activer *nesting* et *keyctl* (Proxmox › Options › Features), redémarrer le container, relancer l'assistant. |
| Compilation arrêtée (`killed`, `signal 9`, `out of memory`) | Mémoire insuffisante (4 Go conseillés) : ajouter mémoire ou swap, relancer ; l'installation reprend. |
| « Le port 2222 est déjà utilisé » | Choisir un autre port à la question. |
| « Téléchargement impossible » | Pas d'accès à github.com : copier le `.deb` depuis un poste, puis `--paquet`. |
| « Le paquet téléchargé est altéré » | Téléchargement corrompu ou intercepté : relancer ; si cela persiste, ne pas installer et le signaler. |
| Le service ne démarre pas (paquet) | `sudo journalctl -u portunia-gate -n 50`. Container LXC non privilégié : activer *nesting*. |
| `git clone` refuse le mot de passe | GitHub demande un jeton d'accès à la place. |

### Connexion

| Symptôme | Cause et solution |
| --- | --- |
| Portunia ne joint pas la Gate | Port 8443 fermé ou mauvaise adresse : `sudo bash installer.sh info`. |
| Empreinte différente de `portunia-gate info` | Ne pas se connecter : mauvais Gate ou interception. Vérifier l'adresse. |
| « Identifiant, mot de passe ou code incorrect » | La vraie raison est au journal : `sudo portunia-gate audit --last 10`. |
| Code refusé à la première connexion | Supprimer l'ancienne entrée Portunia Gate de l'application, vider le champ Code, se reconnecter pour un nouveau QR. |
| Code juste mais refusé | Horloge décalée (NTP sur le serveur et le téléphone) ou code déjà utilisé : attendre le suivant. |
| « Trop d'essais » | 5 échecs en 10 minutes : attendre quelques minutes. |
| Téléphone perdu | Comptes › Réinitialiser le code, ou `sudo portunia-gate user totp <compte>`. |
| Plus aucun administrateur | `sudo bash installer.sh admin secours` sur le serveur. |
| « Session de la Gate expirée » | Se reconnecter depuis la section Gates. |
| Liste de serveurs vide | Pas de règle active pour ses groupes (voir §8). Les heures suivent le fuseau de la Gate. |
| « Le compte du coffre a été refusé » | Secret changé sur le serveur : remplacer le secret. |
| Serveur Windows au coffre refusé, `cible.coffre_refuse` « serveur sans NLA » | Le serveur n'exige pas NLA : activez « N'autoriser que les connexions des ordinateurs exécutant le Bureau à distance avec authentification NLA » dans ses propriétés système, ou passez-le en mode enregistré. |
| Connexion refusée, `cible.empreinte_differente` | Serveur réinstallé ? Vérifier, puis Oublier l'empreinte. |
| Serveurs RDP grisés | Extension Bureau à distance non activée dans Portunia. |

### Portail et assistance

| Symptôme | Cause et solution |
| --- | --- |
| Tunnel « non configuré », « non relié » ou « erreur » | Lire le message sous l'état dans Administrer › Portail web ; recoller la commande d'installation ; la Gate doit pouvoir joindre GitHub et Cloudflare en sortie. |
| Page Cloudflare 502 | Mauvaise **Service URL** dans la route (onglet Routes du tunnel) : `http://localhost:8445`. Le bouton **Tester l'adresse publique** le signale. |
| « Portail fermé » | Interrupteur coupé dans Administrer › Portail web. |
| Connexion refusée sur le portail alors que Portunia marche | Compte hors du groupe autorisé, première connexion pas encore faite dans Portunia, ou adresse publique mal saisie. |
| Portunia Assistance « absent » | La Gate n'a pas pu télécharger le programme (accès sortant à github.com) ; il réessaie chaque jour, et à chaque activation du relais. |
| Le dépanneur ne voit pas « Assistance à distance » | Extensions Portunia Gate et Assistance à activer dans son Portunia ; compte membre du groupe des dépanneurs. |

Pour signaler un problème : version (`sudo portunia-gate info`), les 100 dernières lignes du service et les événements du journal autour de l'heure du problème.

<a id="cli"></a>

## Annexe A — Ligne de commande

La commande `portunia-gate` s'utilise avec sudo, depuis n'importe quel dossier, en paquet comme en Docker. `sudo portunia-gate --help` et `sudo portunia-gate user --help` listent les options.

```
# Installation et service
sudo portunia-gate init --admin julien --temporary --name "Gate ACME"   # base, clés, administrateur ; relançable sans risque
sudo portunia-gate init --no-admin                             # base et clés seulement
# portunia-gate serve : lancé par le service portunia-gate (ne pas le lancer à la main)

# Informations
sudo portunia-gate info                                        # version, empreintes SSH et de l'API, nombre de comptes

# Comptes
sudo portunia-gate user add marc --provisoire                 # mot de passe provisoire, code enrôlé dans Portunia
sudo portunia-gate user add marc                              # mot de passe et code saisis ici
sudo portunia-gate user add sophie --role admin --temporary  # rôle : user (défaut), auditor ou admin
sudo portunia-gate user add presta --policy "key+totp"        # clé SSH et code obligatoires
sudo portunia-gate user add robot --no-password --policy key  # compte technique à clé seule
sudo portunia-gate user key marc "ssh-ed25519 AAAA… marc@pc"  # ou le chemin d'un fichier .pub
sudo portunia-gate user role marc auditor                     # user, auditor ou admin
sudo portunia-gate user list
sudo portunia-gate user passwd marc
sudo portunia-gate user totp marc                             # (ré)enrôle le code
sudo portunia-gate user disable marc                          # --undo pour réactiver
sudo portunia-gate user policy presta "key+totp"              # combinaisons du compte ; "" = réglage général
sudo portunia-gate user reset-password marc                   # nouveau mot de passe provisoire, affiché une fois
sudo portunia-gate user reset-code marc                       # efface le code : nouvel enrôlement (Portunia ou user totp)
sudo portunia-gate user delete marc

# Groupes
sudo portunia-gate group add "Admins Linux"
sudo portunia-gate group member "Admins Linux" marc julien
sudo portunia-gate group remove "Admins Linux" marc               # ses sessions perdues se ferment sous 30 s
sudo portunia-gate group list                                  # groupes et membres
sudo portunia-gate group delete Stagiaires                     # supprime aussi ses règles

# Serveurs
sudo portunia-gate target add web-01 --host 10.0.1.11 --mode coffre --account admin --password --folder Production
sudo portunia-gate target add app-02 --host 10.0.1.12 --mode coffre --account deploy --key-file /root/deploy_ed25519
sudo portunia-gate target add rec-app --host 10.0.2.5 --mode enregistre --folder Recette
sudo portunia-gate target add fw-edge --host 10.0.0.1 --mode rebond --folder Réseau
sudo portunia-gate target add srv-ad-01 --host 10.0.3.10 --protocol rdp --mode enregistre --folder Windows
sudo portunia-gate target add srv-paie --host 10.0.3.12 --protocol rdp --mode coffre --account "ACME\svc-paie" --password   # RDP au coffre (Gate 0.29)
sudo portunia-gate target add vcenter --host 10.0.4.10 --protocol https --folder Infrastructure   # interface web, mode enregistré (Gate 0.30)
sudo portunia-gate target add rds-02 --host 10.0.3.11 --port 3390 --protocol rdp --mode rebond --account "ACME\support"
sudo portunia-gate target list
sudo portunia-gate target edit web-01 --folder Prod --account root   # aussi --rename, --host, --port, --mode
sudo portunia-gate target edit web-01 --host 10.0.1.21 --password      # nouvelle adresse : secret à redonner, empreinte oubliée
sudo portunia-gate target secret web-01 --password             # remplace le secret du coffre (ou --key-file)
sudo portunia-gate target secret app-02 --generate             # SSH : la Gate crée une clé, l'installe avec le mot de passe actuel, l'oublie
sudo portunia-gate target forget srv-ad-01                     # serveur réinstallé : oublie l'empreinte
sudo portunia-gate target delete fw-edge                       # secret effacé ; sessions et enregistrements gardés

# Règles
sudo portunia-gate rule add prod --group "Admins Linux" --targets web-01,rec-app --sftp --tunnels --days lun-ven --hours 7-20
sudo portunia-gate rule add presta-db --group Prestataires --targets db-01 --record-input --expires 2026-12-31
sudo portunia-gate rule add windows --group "Admins Windows" --targets srv-ad-01,rds-02 --days lun-ven --hours 7-20
sudo portunia-gate rule list                                   # avec leur état : active, hors plage, expirée
sudo portunia-gate rule edit prod --sftp off --targets web-01,web-02   # interrupteurs : on ou off
sudo portunia-gate rule edit presta-db --expires 2027-03-31 --hours ""   # "" efface jours, heures ou expiration
sudo portunia-gate rule add compta-fichiers --group Comptabilité --targets srv-compta --files   # Bureau à distance : copie de fichiers permise et tracée
sudo portunia-gate rule edit compta-fichiers --files off      # de nouveau bloquée
sudo portunia-gate rule delete presta-db                       # les sessions qu'elle permettait se ferment sous 30 s

# Réglages
sudo portunia-gate setting auth_policy "password+totp,key+totp"
sudo portunia-gate setting banniere "Accès réservé. Les sessions sont enregistrées."
sudo portunia-gate setting nom "Gate ACME"
sudo portunia-gate setting langue fr                          # en, fr, ou "" pour la langue du système
sudo portunia-gate setting                                   # tous les réglages et leur valeur
sudo portunia-gate setting session_hours 8                   # durée d'une connexion Portunia (1 à 168 h)
sudo portunia-gate setting remember_password off             # mémoriser le mot de passe de la Gate dans Portunia
sudo portunia-gate setting own_replay on                     # chaque compte relit ses propres sessions
sudo portunia-gate setting sensitive_commands "rm -rf,shutdown"   # repérées dans le lecteur
sudo portunia-gate setting file_copies_mb 200                 # garde les fichiers copiés en Bureau à distance (Mo par session ; 0 : aucun)

# Contrôle
sudo portunia-gate audit --verify --last 50
sudo portunia-gate sessions --last 20
sudo portunia-gate audit --search marc --last 500 --csv journal.csv   # recherche et export pour un tableur
sudo portunia-gate sessions --user marc --server web-01 --since 2026-11-01 --text "rm -rf"
sudo portunia-gate sessions show 3f2a9c1e                     # détail, fichier, contrôle de la signature
sudo portunia-gate sessions --files                           # sessions Bureau à distance avec des fichiers copiés
sudo portunia-gate hardening                                  # état du durcissement et alertes

# Portail web (voir §10)
sudo portunia-gate portal set --url https://acces.acme.fr --group "Accès web" --duration 4 --idle 30 --clipboard on
sudo portunia-gate portal set --turnstile-site 0x4AAA… --turnstile-secret   # clé secrète demandée (vide : l'efface)
sudo portunia-gate portal set --code off                       # plus de code sur le portail (sauf comptes qui l'imposent)
sudo portunia-gate portal on                                   # ouvre le portail ; off le ferme (sessions web fermées sous 30 s)
sudo portunia-gate portal close-sessions                       # termine les sessions web sans fermer le portail
sudo portunia-gate portal show                                 # état, réglages, exposition
sudo portunia-gate portal test                                 # la Gate répond-elle à l'adresse publique ?
sudo portunia-gate portal acme --domains acces.acme.fr --email it@acme.fr --accept-terms   # Let's Encrypt ; --renew, --off, sans option : état
sudo portunia-gate portal show                                 # mode, écoute HTTPS, certificat, tunnel
sudo portunia-gate portal expose cloudflare                    # modes : cloudflare, internal, direct, proxy, relay
sudo portunia-gate portal expose internal --networks 10.0.0.0/8,192.168.0.0/16
sudo portunia-gate portal expose proxy --trusted 10.0.5.10,10.0.5.11 --header X-Forwarded-For
sudo portunia-gate portal https 8446                           # écoute HTTPS ; sans valeur, l'affiche ; --off la ferme
sudo portunia-gate portal cert                                 # affiche le certificat du portail
sudo portunia-gate portal cert --import portail.pfx            # ou --import cert.pem --key cle.pem ; --remove : auto-signé
sudo portunia-gate portal tunnel "eyJh…"                       # jeton Cloudflare (« - » : lu sur l'entrée standard) ; --remove
sudo portunia-gate portal tunnel                               # état du tunnel
sudo portunia-gate portal relay-add dmz-01 10.0.5.20:8447      # relais dédié : affiche son code d'appairage
sudo portunia-gate portal relay-list
sudo portunia-gate portal relay-remove dmz-01

# Assistance à distance (voir §11)
sudo portunia-gate assist set --group Dépanneurs --duration 4  # groupe des dépanneurs, durée maximale (1 à 12 h)
sudo portunia-gate assist on                                   # off la ferme (sessions arrêtées sous 30 s) ; show l'affiche

# Relais dédié : sur la machine du relais (Debian amd64), pas sur la Gate
sudo bash installer.sh relais CODE acces.acme.fr               # installe, appaire (edge enroll) et démarre le relais
sudo runuser -u _portunia_gate -- /usr/lib/portunia-gate/portunia-gate edge certificate cert.pem cle.pem
sudo systemctl restart portunia-gate-relais                    # après un changement de certificat
# edge enroll CODE [--name NOM] [--public 0.0.0.0:443] [--tunnel 0.0.0.0:8447] [--cert C --key K] : fait par l'installateur
# edge serve : lancé par le service portunia-gate-relais
```

Les modes acceptent aussi `vault`, `recorded`, `jump` ; les jours `mon-fri` ou `lun,mer,ven`. Options en anglais ou en français : `--temporary` ou `--provisoire`, `--name` ou `--nom` ; réglages `banner`/`banniere`, `name`/`nom`, `language`/`langue`. Sans valeur, `setting` affiche la valeur actuelle.

Depuis Portunia Gate 0.26, tout ce que fait Administrer dans Portunia se fait aussi en ligne de commande, et tout est inscrit au journal d'audit (acteur `console`) : une Gate s'exploite entièrement depuis un poste Linux ou macOS. Seule la **relecture des sessions** (lecteur, révélation des saisies masquées) reste dans Portunia. Une commande s'applique tout de suite en base ; ce qu'elle change pour les sessions ouvertes (fermeture, droits, langue) prend effet sous 30 secondes. Référence complète : [ligne de commande](ligne-de-commande.md) ([English](command-line.md)).

En SSH, un compte au **mot de passe provisoire** est refusé tant que le mot de passe n'a pas été changé (dans Portunia, ou `user passwd`) ; son code doit aussi être enrôlé (Portunia, ou `user totp`).

<a id="sans-portunia"></a>

## Annexe B — Se connecter sans Portunia

```
ssh -p 2222 julien@gate               # menu des serveurs autorisés
ssh -p 2222 julien%web-01@gate        # directement sur web-01
ssh -J julien@gate:2222 admin@fw-01   # rebond de bout en bout (serveurs en mode rebond)
sftp -P 2222 julien%web-01@gate       # fichiers (coffre, si la règle autorise SFTP)
ssh -p 2222 julien%web-01@gate "uptime"            # commande unique (coffre)
ssh -p 2222 -L 5432:db-interne:5432 julien%web-01@gate   # tunnel (coffre, si autorisé)
```

Fonctionne avec OpenSSH, PuTTY, MobaXterm, WinSCP. Le code TOTP doit avoir été enrôlé avant (Portunia ou `sudo portunia-gate user totp`). Commande unique, SFTP et tunnels : serveurs en coffre seulement.

<a id="flux"></a>

## Annexe C — Ports et flux réseau

| De | Vers | Port | Usage |
| --- | --- | --- | --- |
| Postes | Gate | 2222/tcp | SSH : sessions, rebond SSH et RDP. |
| Postes | Gate | 8443/tcp | API HTTPS de Portunia : connexion, jetons, administration, audit, Bureau à distance enregistré, assistance (dépanneur). |
| Gate (local) | Gate | 8445/tcp | Portail en HTTP pour cloudflared ; écoute `127.0.0.1`, jamais exposé. |
| Cloudflare | Gate | 8446/tcp | Option « proxy Cloudflare » au lieu du tunnel ; adresses Cloudflare seulement. |
| Gate | Cloudflare | sortant | Tunnel cloudflared lancé par la Gate (portail et page d'assistance) ; téléchargement de cloudflared depuis GitHub. |
| Gate | github.com | 443/tcp | Téléchargement de cloudflared (une fois par version) et de Portunia Assistance. |
| Gate | Serveurs | 22, 3389, 443… | Connexions vers les serveurs protégés. |
| Gate | github.com | 443 | Mises à jour (paquet) et téléchargement de Portunia Assistance. |
| Postes | github.com | 443 | Mises à jour de Portunia. |

<a id="fichiers"></a>

## Annexe D — Fichiers et emplacements

| Chemin | Contenu |
| --- | --- |
| `/usr/lib/portunia-gate/portunia-gate` | Le programme. |
| `/usr/bin/portunia-gate` | Commande d'administration (lance le programme avec l'environnement du service). |
| `/etc/portunia-gate/portunia-gate.env` | Ports, fuseau, accès à la base, certificat, langue. |
| `/var/lib/portunia-gate/` | Clés (`vault.key`, `signature.key`, `ssh_host_ed25519_key`), certificat (`api.crt`, `api.key`), `enregistrements/` (et, à côté d'une session Bureau à distance, `<session>-fichiers/` : copies des fichiers copiés, réglage `file_copies_mb`), `assist/` (Portunia Assistance). |
| Base `portunia-gate` | Comptes, groupes, serveurs, règles, réglages, sessions, journal. |
| Service | `portunia-gate.service` (systemd), compte `_portunia_gate`, relance automatique en cas d'échec. |

### Variables de `portunia-gate.env`

| Variable | Défaut | Sens |
| --- | --- | --- |
| `PORTUNIA_GATE_SSH_LISTEN` | `0.0.0.0:2222` | Écoute SSH. |
| `PORTUNIA_GATE_API_LISTEN` | `0.0.0.0:8443` | Écoute de l'API. |
| `PORTUNIA_GATE_PORTAL_LISTEN` | `127.0.0.1:8445` | Portail pour Cloudflare Tunnel. |
| `PORTUNIA_GATE_PORTAL_TLS_LISTEN` | absent | Écoute HTTPS du portail ; si réglée, l'emporte sur celle choisie dans Administrer › Portail web. |
| `PORTUNIA_GATE_TLS_CERT`, `PORTUNIA_GATE_TLS_KEY` | auto-signé | Certificat de l'API (et du portail s'il n'en a pas d'autre). |
| `PORTUNIA_GATE_PORTAL_TLS_CERT`, `PORTUNIA_GATE_PORTAL_TLS_KEY` | absent | Certificat du portail imposé par fichiers (relus dès qu'ils changent) ; l'emporte sur celui réglé dans Administrer › Portail web. |
| `PORTUNIA_GATE_LANG` | langue du système | Langue du service. |
| `TZ` | fuseau choisi | Plages horaires des règles. |

Après une modification : `sudo systemctl restart portunia-gate` (coupe les sessions ouvertes).

<a id="evenements"></a>

## Annexe E — Événements du journal

| Famille | Événements |
| --- | --- |
| Service | `service.demarre` `installation` |
| Authentification | `auth.reussie` `auth.echec` `acces.refuse` |
| Comptes | `compte.cree` `compte.modifie` `compte.supprime` `compte.desactive` `compte.reactive` `compte.mot_de_passe` `compte.mot_de_passe_provisoire` `compte.totp_enrole` `compte.totp_reinitialise` `compte.cle_ajoutee` |
| Groupes, règles, réglages | `groupe.cree` `groupe.supprime` `groupe.membre_ajoute` `regle.creee` `regle.modifiee` `regle.supprimee` `reglage.modifie` |
| Serveurs | `cible.creee` `cible.modifiee` `cible.supprimee` `cible.empreinte_memorisee` `cible.empreinte_differente` `cible.empreinte_oubliee` `cible.cle_installee` `cible.coffre_refuse` `cible.identifiants_refuses` |
| Fichiers (Bureau à distance) | `fichier.copie` (sens, nom, taille, SHA-256, copie gardée) `fichier.bloque` (tentatives retenues faute d'autorisation) |
| Sessions | `session.ouverte` `session.fermee` `commande.executee` `sftp.ouvert` `rebond.ouvert` `rebond.ferme` `tunnel.ouvert` `tunnel.ferme` `web.ouvert` `web.page` |
| Audit | `enregistrement.consulte` `enregistrement.revele` |
| Portail | `portail.connexion` `portail.echec` `portail.active` `portail.desactive` `portail.modifie` `portail.sessions_fermees` |
| Assistance | `assistance.ouverte` `assistance.fermee` `assistance.reglage` `assistance.place_inconnue` `assistance.programme` |

> [!NOTE]
> **À surveiller** — `auth.echec` en rafale (attaque par essais), `cible.empreinte_differente` (serveur remplacé ou usurpé), `enregistrement.revele` (révélation de saisies), `compte.*` et `regle.*` hors des changements prévus.

<a id="routine"></a>

## Annexe F — Contrôles périodiques

| Quand | Contrôle |
| --- | --- |
| Chaque jour | Sauvegarde de la nuit présente et lisible ; service actif ; espace disque. |
| Chaque semaine | Échecs de connexion et refus d'accès inhabituels ; révélations de saisies ; état du tunnel si le portail est utilisé. |
| Chaque mois | `sudo portunia-gate audit --verify` ; mise à jour de Portunia Gate et du système ; revue des règles *expirées* et des comptes désactivés à supprimer. |
| Chaque trimestre | Revue des accès avec les responsables (qui est dans quel groupe, qui est administrateur) ; essai de restauration ; vérification que la copie de `vault.key` est toujours lisible. |
| À chaque départ | Compte désactivé le jour même ; comptes personnels sur les serveurs (modes enregistré et rebond) désactivés. |

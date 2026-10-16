# UE3 — Module 14 : Installation, Utilisateurs et Permissions

---

# PARTIE I — COURS

## 1. Titre du cours
Administration Linux fondamentale : utilisateurs, groupes et permissions

## 2. Niveau
Élève ingénieur — cycle ingénieur, entrée en UE3 administration Linux (S2)

## 3. Durée
CM : 4h (module d'ouverture UE3 — +3h TD, +5h TP en Parties II et III)

## 4. Prérequis
- UE1-Module 6 : boot process, systemd, PID 1
- UE1-Module 2 : processus, notion de propriétaire de processus (UID/GID déjà entrevus)
- UE1-Module 4 : permissions Unix de base (rwx), déjà mentionnées en survol

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Décrire les grandes étapes d'une installation Linux serveur et les choix qui engagent la suite de l'administration (partitionnement, sélection de paquets minimale)
- Créer, modifier et supprimer des utilisateurs et des groupes en ligne de commande
- Expliquer la structure de `/etc/passwd`, `/etc/shadow` et `/etc/group`
- Calculer et interpréter des permissions Unix en notation octale et symbolique
- Expliquer et utiliser les permissions spéciales (SUID, SGID, sticky bit) et justifier leurs cas d'usage
- Configurer des ACL étendues pour des besoins de permissions plus fins que le modèle Unix classique
- Configurer `sudo` de façon sécurisée et expliquer sa différence fondamentale avec `su`

## 6. Motivation

Un serveur en production héberge souvent plusieurs applications, exploitées par plusieurs équipes, avec des besoins d'accès très différents : l'équipe de développement doit pouvoir déployer une application sans avoir accès aux données clients gérées par une autre équipe ; un script de sauvegarde automatisé doit pouvoir lire tous les fichiers sans pouvoir les modifier ; un administrateur doit pouvoir intervenir en urgence sans que son compte personnel n'ait des privilèges permanents excessifs. Toute cette gouvernance repose sur un socle unique : le système d'utilisateurs, de groupes et de permissions de Linux.

Une mauvaise gestion des permissions est l'une des causes les plus fréquentes d'incidents de sécurité en production — un `chmod 777` "temporaire" pour débloquer un problème, jamais revenu en arrière, un compte de service avec des droits `sudo` illimités par facilité. Ce module pose les bases rigoureuses qui évitent ces dérives, prérequis de tout le reste de l'UE3.

## 7. Concepts fondamentaux

### 7.1 Installation Linux serveur — choix structurants

Une installation Linux serveur typique (par opposition à un poste de travail) implique des choix qui engagent durablement l'administration future :

- **Partitionnement** : séparer `/`, `/home`, `/var`, `/tmp` sur des partitions ou volumes distincts limite l'impact d'un remplissage excessif d'une zone (ex : des logs qui explosent dans `/var/log` ne doivent pas pouvoir saturer la partition racine et bloquer le système entier — rappel direct du risque de saturation disque vu en UE1-M4)
- **Sélection de paquets minimale** : installer uniquement les paquets strictement nécessaires réduit la surface d'attaque (moins de logiciels installés = moins de vulnérabilités potentielles) et simplifie la maintenance — principe de base retrouvé tout au long de l'UE3 et de l'UE8 sécurité
- **Choix du système de fichiers** : ext4 (robuste, éprouvé, par défaut sur de nombreuses distributions) ou XFS (performant sur de gros volumes) selon le contexte — rappel UE1-M4 sur les systèmes de fichiers journalisés

**Ce cours ne détaille pas la procédure d'installation graphique/interactive elle-même** (spécifique à chaque distribution et rapidement obsolète) mais les choix structurants qui doivent être anticipés, quelle que soit la distribution utilisée.

### 7.2 Utilisateurs et groupes — modèle Linux

Chaque utilisateur Linux possède un **UID (User ID)** numérique unique, et appartient à un **groupe primaire** ainsi qu'éventuellement plusieurs **groupes secondaires**. Le compte `root` (UID 0) dispose de tous les privilèges sur le système, sans restriction de permissions — d'où l'importance de ne jamais l'utiliser pour un usage quotidien (principe du moindre privilège, déjà rencontré en UE1).

**Fichiers de référence :**
- `/etc/passwd` : liste des comptes utilisateurs (nom, UID, GID primaire, répertoire personnel, shell par défaut) — lisible par tous, ne contient **plus** les mots de passe depuis l'introduction du "shadow password"
- `/etc/shadow` : mots de passe chiffrés (hachés) et politiques d'expiration, lisible uniquement par `root` — séparé de `/etc/passwd` précisément pour limiter l'exposition des hachages de mot de passe à un fichier à permissions restreintes
- `/etc/group` : liste des groupes et de leurs membres

**Distinction cruciale groupe primaire / groupe secondaire :** chaque utilisateur a exactement **un** groupe primaire (utilisé par défaut pour tout fichier qu'il crée), mais peut appartenir à un nombre arbitraire de groupes secondaires (donnant accès à des ressources supplémentaires sans changer la propriété par défaut de ses créations).

### 7.3 Permissions Unix — rappel et approfondissement

Chaque fichier/répertoire porte trois jeux de permissions (lecture `r`, écriture `w`, exécution `x`) pour trois catégories : **propriétaire (user)**, **groupe (group)**, **autres (others)**.

**Notation symbolique et octale :**

| Symbolique | Octal | Signification |
|---|---|---|
| `rwx` | 7 | lecture + écriture + exécution |
| `rw-` | 6 | lecture + écriture |
| `r-x` | 5 | lecture + exécution |
| `r--` | 4 | lecture seule |
| `---` | 0 | aucun droit |

Une permission complète comme `rwxr-xr--` se lit : propriétaire = `rwx` (7), groupe = `r-x` (5), autres = `r--` (4), soit `754` en notation octale.

**Sens de `x` selon le type d'objet — point souvent mal compris :**
- Sur un **fichier** : autorise l'exécution (si c'est un binaire ou un script avec l'interpréteur approprié)
- Sur un **répertoire** : autorise la **traversée** (accéder à un fichier dont on connaît le nom à l'intérieur, ou faire `cd` dedans) — sans `x` sur un répertoire, même la simple traversée est impossible, indépendamment de la permission `r` (qui ne permet que de **lister** le contenu, une distinction fréquemment confondue)

### 7.4 `umask`

Le **umask** définit les permissions **retirées par défaut** à la création d'un nouveau fichier ou répertoire, exprimé comme un masque soustractif. Un umask de `022` (valeur par défaut courante) retire l'écriture pour le groupe et les autres : un fichier nouvellement créé avec une permission "maximale" théorique de `666` (fichiers, pas d'exécution par défaut) se retrouve avec `644` (666 - 022) ; un répertoire théoriquement `777` se retrouve avec `755`.

### 7.5 Permissions spéciales : SUID, SGID, sticky bit

- **SUID (Set User ID, symbole `s` à la place du `x` du propriétaire, octal `4000`)** : quand appliqué à un **exécutable**, le programme s'exécute avec les privilèges de son **propriétaire**, pas de l'utilisateur qui le lance. Exemple classique : `/usr/bin/passwd` appartient à `root` avec SUID activé, permettant à un utilisateur standard de modifier son propre mot de passe (opération qui nécessite d'écrire dans `/etc/shadow`, normalement réservé à `root`) sans avoir lui-même de privilèges root généraux.
- **SGID (Set Group ID, symbole `s` à la place du `x` du groupe, octal `2000`)** : sur un exécutable, comportement analogue au SUID mais pour le groupe. Sur un **répertoire**, comportement différent et très utile : tout nouveau fichier créé dans ce répertoire hérite automatiquement du **groupe du répertoire** plutôt que du groupe primaire de l'utilisateur créateur — mécanisme précieux pour un répertoire de travail partagé entre plusieurs utilisateurs d'un même groupe.
- **Sticky bit (symbole `t` à la place du `x` des autres, octal `1000`)** : sur un répertoire, restreint la suppression/renommage d'un fichier à son seul propriétaire (ou à `root`), même si d'autres utilisateurs ont un droit d'écriture sur le répertoire lui-même. Exemple emblématique : `/tmp`, où tout le monde peut écrire, mais où personne ne peut supprimer les fichiers d'un autre utilisateur.

**Sécurité — vigilance sur SUID :** un exécutable SUID appartenant à `root` est une cible de choix pour l'escalade de privilèges si le binaire présente une faille — un audit régulier des binaires SUID présents sur un système (`find / -perm -4000`) est une pratique de durcissement de base, approfondie en UE8.

### 7.6 ACL (Access Control Lists) — au-delà du modèle Unix classique

Le modèle Unix classique (propriétaire/groupe/autres) atteint ses limites dès qu'un besoin de permission plus fin apparaît — par exemple, donner un accès en lecture à un utilisateur précis **en plus** du propriétaire et du groupe habituel, sans le faire rejoindre un groupe dédié pour ce seul besoin ponctuel. Les **ACL étendues** répondent à ce besoin :

```bash
setfacl -m u:alice:rx /chemin/vers/fichier   # donne à alice un accès r-x spécifique
getfacl /chemin/vers/fichier                   # affiche les ACL actives
```

**Point de vigilance :** un fichier avec des ACL actives affiche un `+` après ses permissions classiques dans `ls -l` (ex : `rwxr-xr-x+`) — un signal visuel à ne jamais ignorer lors d'un audit de permissions, car les permissions "apparentes" (rwx classiques) ne racontent alors plus toute l'histoire réelle des accès autorisés.

### 7.7 `sudo` vs `su` — élévation de privilèges

- **`su`** (substitute user) : ouvre une nouvelle session sous l'identité d'un autre utilisateur (typiquement `root`), en exigeant le mot de passe de **cet utilisateur cible** — une fois la session ouverte, toutes les commandes s'exécutent avec ses privilèges, sans limitation, jusqu'à sortie explicite de cette session.
- **`sudo`** (superuser do) : exécute une **commande précise** avec des privilèges élevés, en exigeant le mot de passe de l'**utilisateur courant** (pas celui de root), selon une politique définie dans `/etc/sudoers` (à éditer exclusivement via `visudo`, qui valide la syntaxe avant sauvegarde et évite de casser irrémédiablement l'accès administrateur par une erreur de frappe).

**Pourquoi `sudo` est généralement préféré en production :** il permet une **granularité fine** (autoriser un utilisateur à exécuter uniquement certaines commandes précises avec privilèges, pas un accès root complet), une **traçabilité** (chaque exécution `sudo` est journalisée, avec l'identité de l'utilisateur réel ayant exécuté chaque commande — contrairement à une session `su` où toutes les actions se confondent sous l'identité de root dans les journaux), et évite la nécessité de partager ou connaître le mot de passe root lui-même.

## 8. Architecture

```
┌──────────────────────────────────────────────────────────┐
│  /etc/passwd                                                  │
│  alice:x:1001:1001:Alice Dupont:/home/alice:/bin/bash         │
│         │    │    │                                             │
│        UID  GID  GID primaire (référence /etc/group)             │
├──────────────────────────────────────────────────────────┤
│  /etc/shadow (root uniquement)                                  │
│  alice:$6$...hash...:19700:0:99999:7:::                          │
├──────────────────────────────────────────────────────────┤
│  /etc/group                                                     │
│  developpeurs:x:1050:alice,bob,carla   ← groupe secondaire        │
└──────────────────────────────────────────────────────────┘

Fichier créé par alice → propriétaire=alice, groupe=alice (primaire)
                          sauf si SGID sur le répertoire parent →
                          groupe hérité du répertoire
```

## 9. Fonctionnement interne

Le noyau Linux vérifie les permissions d'accès à chaque appel système impliquant un fichier (`open`, `read`, `write`...), en comparant l'UID/GID effectif du processus appelant (rappel UE1-M2 : chaque processus porte l'identité de son propriétaire dans son `task_struct`) aux permissions du fichier ciblé — c'est ce mécanisme qui explique pourquoi un exécutable SUID modifie effectivement le comportement du noyau : au moment de l'`exec()`, le noyau positionne l'UID **effectif** du nouveau processus à celui du propriétaire du fichier exécutable (si SUID actif) plutôt qu'à l'UID réel de l'utilisateur qui a lancé la commande — une distinction UID réel/UID effectif au cœur du mécanisme SUID, essentielle pour en comprendre la portée exacte et les limites.

## 10. Démonstration

```bash
# Créer un utilisateur avec groupe primaire dédié et répertoire personnel
sudo useradd -m -s /bin/bash stagiaire1

# Observer les fichiers de référence
getent passwd stagiaire1
sudo getent shadow stagiaire1

# Démonstration SUID
ls -l /usr/bin/passwd   # observer le 's' à la place du 'x' propriétaire

# Démonstration ACL
touch /tmp/demo_acl.txt
setfacl -m u:stagiaire1:r /tmp/demo_acl.txt
getfacl /tmp/demo_acl.txt
```

## 11. Commandes / outils

| Commande | Rôle | Exemple |
|---|---|---|
| `useradd` / `userdel` / `usermod` | Créer/supprimer/modifier un utilisateur | `sudo useradd -m -G developpeurs alice` |
| `groupadd` / `groupdel` | Créer/supprimer un groupe | `sudo groupadd developpeurs` |
| `passwd` | Définir/modifier un mot de passe | `sudo passwd alice` |
| `chmod` | Modifier les permissions | `chmod 750 script.sh` |
| `chown` / `chgrp` | Modifier le propriétaire/groupe | `sudo chown alice:developpeurs fichier` |
| `umask` | Consulter/définir le masque de création par défaut | `umask 027` |
| `setfacl` / `getfacl` | Définir/consulter les ACL étendues | `setfacl -m u:bob:rw fichier` |
| `visudo` | Éditer `/etc/sudoers` en sécurité | `sudo visudo` |
| `find / -perm -4000` | Auditer les binaires SUID du système | `sudo find / -perm -4000 -type f 2>/dev/null` |

**Erreur fréquente :** éditer `/etc/sudoers` directement avec un éditeur classique plutôt que `visudo` — une erreur de syntaxe non détectée peut rendre `sudo` totalement inutilisable pour tous les utilisateurs, y compris pour corriger l'erreur elle-même sans accès root direct par un autre moyen (console physique, mode single-user).

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** un audit de sécurité révèle qu'un serveur de production contient un binaire personnalisé, déployé plusieurs mois auparavant par un développeur pour "faciliter le déploiement", avec le bit SUID activé et appartenant à `root`. Le binaire en question exécute, sans validation stricte des entrées, une commande shell construite à partir d'un argument fourni par l'appelant.

**Analyse attendue :** ce scénario est une **vulnérabilité d'escalade de privilèges critique** : n'importe quel utilisateur du système, même avec des privilèges minimaux, peut exécuter ce binaire (s'il a la permission d'exécution) et, grâce au SUID actif et à la construction non sécurisée de la commande shell, injecter des commandes arbitraires exécutées avec les privilèges de `root` — un chemin direct vers une compromission totale du système à partir d'un compte utilisateur standard. Correction immédiate : retirer le bit SUID (`chmod u-s`) en urgence le temps d'investiguer, puis corriger le code (validation stricte des entrées, éviter la construction de commandes shell à partir d'entrées non fiables, ou repenser l'architecture pour ne pas nécessiter de privilèges root élevés en permanence pour cette opération). Mesure préventive : un audit périodique des binaires SUID (`find / -perm -4000`), comme mentionné en section 7.5, aurait dû détecter cet ajout non standard bien avant qu'un audit externe ne le révèle.

## 15. Sécurité

- Le principe du **moindre privilège** (déjà rencontré dans plusieurs modules précédents) trouve ici son application la plus directe : chaque compte, chaque groupe, chaque permission accordée doit correspondre à un besoin réel et documenté, jamais à une facilité de configuration
- Un compte de service (utilisé par une application, pas par un humain) ne devrait jamais avoir de shell interactif (`/usr/sbin/nologin` plutôt que `/bin/bash`) ni de mot de passe utilisable pour une connexion directe, réduisant sa surface d'exploitation en cas de compromission de l'application qu'il exécute
- La séparation `/etc/passwd` (lisible par tous) / `/etc/shadow` (root uniquement) est elle-même une mesure de sécurité historique — avant son introduction généralisée, les hachages de mots de passe, bien que chiffrés, étaient exposés à toute tentative de cassage hors ligne par quiconque avait un accès en lecture au système
- Une politique `sudo` doit toujours être aussi restrictive que possible : autoriser des commandes précises (`sudo systemctl restart monservice`) plutôt qu'un accès `ALL=(ALL) NOPASSWD: ALL` par confort, qui annule pratiquement tout l'intérêt de `sudo` par rapport à un accès root direct

## 16. Automatisation

Script Bash d'audit périodique des binaires SUID/SGID inattendus, comparés à une liste de référence connue et validée.

```bash
#!/usr/bin/env bash
# Compare les binaires SUID/SGID actuels à une liste de référence, alerte sur tout écart
set -euo pipefail

REFERENCE="/etc/security/suid_reference.txt"
LOGFILE="/var/log/suid_audit.log"

if [[ ! -f "$REFERENCE" ]]; then
    echo "Génération de la référence initiale (à valider manuellement une seule fois)"
    find / -xdev \( -perm -4000 -o -perm -2000 \) -type f 2>/dev/null | sort > "$REFERENCE"
    exit 0
fi

ACTUEL=$(find / -xdev \( -perm -4000 -o -perm -2000 \) -type f 2>/dev/null | sort)
NOUVEAUX=$(comm -13 "$REFERENCE" <(echo "$ACTUEL"))

if [[ -n "$NOUVEAUX" ]]; then
    echo "$(date '+%F %T') - ALERTE: nouveaux binaires SUID/SGID détectés" | tee -a "$LOGFILE"
    echo "$NOUVEAUX" | tee -a "$LOGFILE"
    exit 1
fi

echo "$(date '+%F %T') - OK: aucun nouveau binaire SUID/SGID" >> "$LOGFILE"
exit 0
```

## 17. Troubleshooting

Scénario court : un utilisateur signale qu'il ne parvient pas à accéder à un fichier partagé, alors que `ls -l` montre pourtant des permissions qui semblent l'y autoriser (il fait partie du bon groupe, avec une permission de lecture pour ce groupe). Démarche : vérifier l'ensemble de la chaîne de répertoires parents jusqu'à la racine (`namei -l /chemin/complet/fichier`) — un répertoire intermédiaire sans permission d'exécution (`x`) pour l'utilisateur ou son groupe bloque la traversée, indépendamment des permissions du fichier final lui-même (rappel section 7.3) → vérifier également la présence d'ACL actives (`getfacl`) qui pourraient restreindre l'accès au-delà de ce que les permissions classiques `rwx` laissent supposer, malgré une apparence favorable en `ls -l` simple.

## 18. Synthèse

- Une installation Linux serveur engage des choix durables : partitionnement séparé, sélection de paquets minimale
- `/etc/passwd` (public), `/etc/shadow` (root uniquement), `/etc/group` structurent le modèle utilisateurs/groupes
- Les permissions Unix (rwx × propriétaire/groupe/autres) se calculent en octal ; `x` sur un répertoire autorise la traversée, distinct de `r` qui autorise le listage
- SUID/SGID/sticky bit étendent le modèle pour des besoins précis (exécution avec privilèges du propriétaire, héritage de groupe, protection de suppression) — SUID est une surface d'attaque à auditer régulièrement
- Les ACL comblent les limites du modèle Unix classique pour des permissions fines, signalées par un `+` dans `ls -l`
- `sudo` (commande précise, granularité fine, traçabilité) est préféré à `su` (session complète) en production, toujours édité via `visudo`

## 19. Questions d'évaluation

**QCM**
1. La permission octale `640` correspond à :
 a) `rwxrw----` b) `rw-r-----` c) `r--r--r--` d) `rw-rw-rw-`

2. Le bit SUID sur un exécutable appartenant à root permet :
 a) de le rendre illisible b) à tout utilisateur de l'exécuter avec les privilèges de root c) de le rendre invisible dans `ls` d) d'empêcher sa suppression

**Questions ouvertes**
- Expliquez pourquoi la permission `x` sur un répertoire est indispensable pour y accéder, même avec la permission `r`.
- Expliquez la différence fondamentale entre `su` et `sudo` du point de vue de la traçabilité en production.

**Exercice technique**
- Donnez la commande permettant d'auditer l'ensemble des binaires SUID présents sur un système Linux.

**Problème de diagnostic**
- Un utilisateur membre du bon groupe, avec une permission de lecture apparemment correcte sur un fichier, ne parvient pourtant pas à y accéder. Proposez deux causes distinctes possibles, non liées à une erreur de permission sur le fichier lui-même.

## 20. Mini-projet

Développer un script (Bash ou Python) qui :
1. Audite l'ensemble des comptes utilisateurs du système et signale ceux disposant d'un shell interactif alors qu'ils semblent être des comptes de service (heuristique configurable, ex : UID système bas, nom correspondant à un service connu)
2. Audite les binaires SUID/SGID (reprise et extension du script de la section 16), avec une liste de référence validée manuellement une première fois
3. Audite la configuration `sudoers` (via `sudo -l` pour chaque utilisateur, ou lecture prudente des fichiers sous `/etc/sudoers.d/`) et signale toute règle trop permissive (`NOPASSWD: ALL` notamment)
4. Journalise chaque anomalie détectée avec horodatage
5. Génère un rapport de synthèse destiné à un audit de sécurité périodique

---

# PARTIE II — TD

### Niveau 1 — Compréhension

**Exercice 1.1**
Expliquez le rôle respectif de `/etc/passwd`, `/etc/shadow` et `/etc/group`.

**Exercice 1.2**
Convertissez la permission symbolique `rwxr-x---` en notation octale, et expliquez ce qu'elle autorise pour le propriétaire, le groupe et les autres.

**Exercice 1.3**
Expliquez la différence entre `su` et `sudo`, en particulier sur la question du mot de passe demandé.

### Niveau 2 — Application

**Exercice 2.1**
Donnez la commande permettant de créer un utilisateur `technicien1`, avec un répertoire personnel créé automatiquement et un shell `/bin/bash`.

**Exercice 2.2**
Un fichier a la permission octale `751`. Donnez sa notation symbolique complète.

**Exercice 2.3**
Donnez la commande permettant d'accorder à l'utilisateur `bob` un accès en lecture-écriture sur un fichier via une ACL étendue, sans modifier les permissions Unix classiques du fichier.

### Niveau 3 — Analyse

**Exercice 3.1**
Un répertoire partagé `/projets/equipe` a le groupe `dev_team` et le bit SGID activé. Un utilisateur membre de `dev_team` y crée un nouveau fichier. Quel sera le groupe propriétaire de ce fichier ? Cela aurait-il été différent sans le SGID ? Justifiez.

**Exercice 3.2**
Un audit révèle un binaire avec permission `rwsr-xr-x`, appartenant à `root`. Expliquez ce que cette notation indique précisément, et pourquoi ce binaire mérite une attention particulière lors d'un audit de sécurité.

### Niveau 4 — Problème complexe

**Exercice 4.1**
Un utilisateur `alice`, membre du groupe `compta`, se plaint de ne pas pouvoir lire le fichier `/data/compta/rapport.csv`, bien que `ls -l /data/compta/rapport.csv` affiche `-rw-r-----+ 1 root compta 2048 ... rapport.csv` (notez le `+` en fin de permissions). Un collègue lui assure que la permission de groupe (`r--` pour `compta`) devrait suffire. En vous appuyant sur les mécanismes vus en cours, formulez une hypothèse sur la cause réelle du problème, en identifiant précisément l'indice visuel qui la révèle, et proposez la commande permettant de la confirmer.

---

## Correction du TD

**1.1** — `/etc/passwd` contient la liste des comptes utilisateurs (nom, UID, GID primaire, répertoire personnel, shell), lisible par tous les utilisateurs du système. `/etc/shadow` contient les mots de passe chiffrés (hachés) et les politiques d'expiration associées, lisible uniquement par `root`, séparé de `/etc/passwd` pour limiter l'exposition des hachages. `/etc/group` contient la liste des groupes existants et de leurs membres (appartenance secondaire).

**1.2** — `rwxr-x---` : propriétaire = `rwx` (7), groupe = `r-x` (5), autres = `---` (0), soit `750` en octal. Cela autorise le propriétaire à lire, écrire et exécuter ; le groupe à lire et exécuter (mais pas écrire) ; les autres n'ont aucun droit.

**1.3** — `su` ouvre une session complète sous l'identité d'un autre utilisateur (typiquement root), en demandant le mot de passe de **l'utilisateur cible** ; une fois la session ouverte, toutes les commandes suivantes s'exécutent avec ses privilèges sans limitation jusqu'à la sortie de session. `sudo` exécute une commande précise avec des privilèges élevés, en demandant le mot de passe de **l'utilisateur courant** (pas celui de root), selon une politique définie dans `/etc/sudoers` — offrant une granularité plus fine et une meilleure traçabilité, chaque commande `sudo` étant journalisée individuellement avec l'identité réelle de l'exécutant.

**2.1** — `sudo useradd -m -s /bin/bash technicien1`.

**2.2** — `751` : propriétaire = 7 (`rwx`), groupe = 5 (`r-x`), autres = 1 (`--x`), soit `rwxr-x--x`.

**2.3** — `setfacl -m u:bob:rw <fichier>`.

**3.1** — Le fichier créé aura pour groupe propriétaire `dev_team`, c'est-à-dire le groupe du répertoire parent, et non le groupe primaire de l'utilisateur créateur. Ce comportement est directement dû au bit SGID activé sur le répertoire (rappel section 7.5 du cours) : sur un répertoire, SGID force l'héritage du groupe du répertoire pour tout nouveau fichier créé à l'intérieur. Sans SGID, le fichier aurait porté le groupe primaire de l'utilisateur créateur (potentiellement différent de `dev_team` si son groupe primaire personnel n'est pas ce groupe partagé), ce qui aurait pu empêcher les autres membres de l'équipe d'accéder correctement au fichier selon la configuration des permissions de groupe.

**3.2** — La notation `rwsr-xr-x` indique que le bit SUID est activé : le `s` à la position habituellement occupée par le `x` du propriétaire signifie que ce bit est actif **et** que la permission d'exécution du propriétaire est également active (un `S` majuscule indiquerait SUID actif mais sans permission d'exécution du propriétaire, une configuration incohérente rarement voulue). Ce binaire s'exécutera donc toujours avec les privilèges de son propriétaire (`root`), quel que soit l'utilisateur réel qui le lance — une caractéristique qui en fait une cible d'audit prioritaire, car toute faille dans ce binaire (validation d'entrée insuffisante, comportement non sécurisé) pourrait être exploitée par n'importe quel utilisateur du système pour obtenir une exécution de code avec des privilèges root, un chemin classique d'escalade de privilèges (rappel étude de cas industrielle du cours).

**4.1** — L'indice visuel qui révèle la cause réelle du problème est le symbole **`+`** apparaissant à la fin de la chaîne de permissions (`-rw-r-----+`), signalant la présence d'**ACL étendues** actives sur ce fichier (rappel section 7.6 du cours). Les permissions Unix classiques affichées par `ls -l` ne racontent alors plus toute l'histoire : une ACL peut potentiellement **restreindre** davantage l'accès pour certains utilisateurs ou groupes précis (via un masque ACL, ou une entrée ACL spécifique retirant un droit qu'on pourrait croire acquis par la seule appartenance au groupe `compta`), au-delà de ce que suggèrent les permissions classiques affichées. L'hypothèse la plus probable est donc qu'une ACL spécifique restreint explicitement l'accès d'alice (ou modifie le comportement effectif du masque de permission de groupe), malgré une appartenance légitime au groupe `compta`. Commande de confirmation : `getfacl /data/compta/rapport.csv`, qui révélerait l'intégralité des entrées ACL réellement appliquées à ce fichier, au-delà des permissions Unix classiques visibles en premier lieu.

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur système junior pour mettre en place la structure d'utilisateurs, de groupes et de permissions d'une petite équipe, en appliquant rigoureusement le principe du moindre privilège, puis pour auditer une configuration existante présentant des anomalies volontairement introduites.

## Architecture

Une machine Linux (VM ou conteneur avec privilèges suffisants pour la gestion d'utilisateurs — un conteneur standard peut nécessiter des ajustements selon l'environnement pédagogique, une VM est recommandée pour ce module).

## Objectifs
- Créer une structure d'utilisateurs et de groupes cohérente avec des besoins métier donnés
- Configurer un répertoire de travail partagé avec SGID
- Configurer une politique `sudo` restrictive et testée
- Auditer une configuration de permissions comportant des anomalies

## Préparation

```bash
which useradd groupadd chmod chown setfacl getfacl visudo 2>/dev/null
```

## Étapes

### Étape 1 — Créer la structure utilisateurs/groupes

Contexte : une équipe de 3 personnes (alice, bob, carla) travaille sur un projet commun ; alice est responsable technique (accès `sudo` limité à la gestion du service applicatif du projet), bob et carla sont développeurs (pas d'accès `sudo`).

```bash
sudo groupadd projet_x
sudo useradd -m -s /bin/bash -G projet_x alice
sudo useradd -m -s /bin/bash -G projet_x bob
sudo useradd -m -s /bin/bash -G projet_x carla
sudo passwd alice   # définir un mot de passe de test pour chaque compte
sudo passwd bob
sudo passwd carla

getent group projet_x
```

### Étape 2 — Répertoire de travail partagé avec SGID

```bash
sudo mkdir -p /projets/projet_x
sudo chown root:projet_x /projets/projet_x
sudo chmod 2775 /projets/projet_x   # SGID (2) + rwxrwxr-x

ls -ld /projets/projet_x

# Tester en tant que bob : créer un fichier et vérifier son groupe
sudo -u bob touch /projets/projet_x/test_bob.txt
ls -l /projets/projet_x/test_bob.txt
```
*Question :* le fichier créé par `bob` porte-t-il bien le groupe `projet_x`, malgré le groupe primaire personnel de `bob` (généralement un groupe portant son propre nom, créé automatiquement à la création de l'utilisateur) ? Reliez votre observation à l'exercice 3.1 du TD.

### Étape 3 — Politique `sudo` restrictive

```bash
sudo visudo -f /etc/sudoers.d/alice_projet_x
```
Contenu à ajouter (via l'éditeur ouvert par `visudo`) :
```
alice ALL=(root) /usr/bin/systemctl restart projet-x.service, /usr/bin/systemctl status projet-x.service
```
```bash
sudo -u alice sudo -l   # doit lister précisément ces deux commandes, rien de plus
sudo -u bob sudo -l 2>&1   # doit refuser, bob n'a aucune règle sudo définie
```
*Question :* que se serait-il passé si vous aviez écrit `alice ALL=(ALL) NOPASSWD: ALL` par simplicité ? En quoi cela aurait-il été contraire au principe du moindre privilège rappelé tout au long du cours ?

### Étape 4 — Audit d'une configuration avec anomalies

```bash
# Simuler une configuration à auditer (à but pédagogique)
sudo touch /tmp/audit_binaire_suspect
sudo chmod 4755 /tmp/audit_binaire_suspect   # SUID activé, à but de démonstration uniquement
sudo chown root:root /tmp/audit_binaire_suspect

# Lancer l'audit
find /tmp -perm -4000 -type f 2>/dev/null
```
*Résultat attendu :* le fichier factice créé apparaît dans la liste des binaires SUID détectés.

```bash
# Nettoyage après démonstration
sudo rm /tmp/audit_binaire_suspect
```

## Commandes de référence
`useradd`, `groupadd`, `chmod`, `chown`, `getent`, `visudo`, `sudo -l`, `find -perm`

## Résultats attendus
- Sortie `getent group projet_x` confirmant les 3 membres (étape 1)
- Preuve que le fichier créé par `bob` porte le groupe `projet_x` grâce au SGID (étape 2)
- Sorties `sudo -l` pour `alice` (règles précises) et `bob` (aucune règle) (étape 3)
- Détection du binaire SUID factice par la commande d'audit (étape 4)

## Questions

1. Dans l'étape 1, pourquoi est-il préférable de créer un groupe `projet_x` dédié plutôt que de donner à `bob` et `carla` un accès basé uniquement sur leur groupe primaire personnel (généralement propre à chaque utilisateur) ?
2. Si, à l'étape 3, vous aviez oublié d'utiliser `visudo` et édité directement `/etc/sudoers.d/alice_projet_x` avec un éditeur classique en y introduisant une erreur de syntaxe, quel risque concret cela aurait-il fait courir à l'administration future de la machine ?
3. L'audit de l'étape 4 (recherche de binaires SUID) est-il, à lui seul, suffisant pour garantir qu'aucun binaire SUID du système ne présente de faille exploitable ? Justifiez en distinguant ce que l'audit détecte de ce qu'il ne peut pas évaluer.

## Troubleshooting (panne à diagnostiquer)

**Situation fournie :** après la mise en place de la structure des étapes 1 et 2, `carla` signale qu'elle ne parvient pas à accéder au répertoire `/projets/projet_x/sous_dossier_prive`, un sous-répertoire créé entre-temps par `alice` à l'intérieur du répertoire partagé, alors que `carla` fait bien partie du groupe `projet_x` et que `ls -ld` sur ce sous-répertoire montre des permissions de groupe en apparence suffisantes.

**Démarche attendue :**
1. Symptôme : accès refusé à un sous-répertoire malgré une appartenance de groupe apparemment correcte
2. Hypothèse : le SGID appliqué au répertoire parent (`/projets/projet_x`) garantit l'héritage du **groupe propriétaire**, mais **pas** la propagation automatique des **permissions elles-mêmes** ni du SGID lui-même sur les sous-répertoires créés ultérieurement à l'intérieur — un sous-répertoire nouvellement créé par `alice` avec un `umask` restrictif par défaut pourrait très bien ne pas avoir hérité de permissions de groupe suffisantes (`rwx` pour le groupe), même si son groupe propriétaire est correctement `projet_x`
3. Vérification : `ls -ld /projets/projet_x/sous_dossier_prive` pour examiner précisément les permissions (pas seulement le groupe propriétaire) et vérifier si le SGID a lui-même été propagé (visible par un `s` à la position groupe) ou non
4. Isolation : confirmer que le sous-répertoire a été créé sans que le `umask` d'alice au moment de la création n'ait laissé de permission d'écriture/traversée suffisante pour le groupe
5. Correction : ajuster les permissions du sous-répertoire concerné (`chmod g+rwxs sous_dossier_prive` pour restaurer à la fois les droits de groupe et le SGID, afin que les futurs sous-répertoires créés à l'intérieur héritent correctement à leur tour) ; envisager, pour une solution plus durable, un `umask` par défaut plus adapté pour les utilisateurs travaillant régulièrement dans cette arborescence partagée
6. Vérification : `carla` teste à nouveau l'accès au sous-répertoire corrigé, confirmation du bon fonctionnement
7. Documentation : noter que le SGID sur un répertoire n'est **pas récursif par nature** — chaque nouveau sous-répertoire doit lui-même porter le SGID pour que l'héritage de groupe se propage correctement à ses propres futurs sous-répertoires, un piège classique dans la gestion d'arborescences partagées profondes

## Validation

Le TP est validé si l'étudiant fournit :
- Les 4 résultats/captures demandés
- Des réponses argumentées aux 3 questions
- Une identification correcte de la cause du scénario de troubleshooting (non-propagation automatique du SGID/permissions aux sous-répertoires), avec une correction cohérente

# UE1 — Module 4 : Systèmes de Fichiers et I/O

---

# PARTIE I — COURS

## 1. Titre du cours
Systèmes de fichiers et E/S : de l'appel système au disque

## 2. Niveau
Élève ingénieur — cycle ingénieur, tronc commun systèmes (S1)

## 3. Durée
CM : 4h (module complet UE1-M4 : +3h TD, +4h TP en Parties II et III)

## 4. Prérequis
- UE1-M1 : architecture générale d'un OS, kernel vs user space, system calls
- UE1-M2 : processus, états, appels système
- UE1-M3 : mémoire virtuelle (utile pour comprendre le cache de pages et `mmap`)

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Expliquer l'architecture d'un système de fichiers Unix (inode, superbloc, répertoires, liens)
- Décrire le rôle de la VFS (Virtual File System) comme couche d'abstraction
- Différencier E/S bloquantes, non bloquantes et asynchrones, et justifier un choix selon un cas d'usage
- Expliquer le chemin complet d'une opération `read()`/`write()` depuis l'appel système jusqu'au disque
- Expliquer le rôle du cache de pages (page cache) et ses implications en performance et en cohérence
- Diagnostiquer un problème d'E/S (latence disque, cache inefficace, fichier verrouillé) avec les outils Linux

## 6. Motivation

Un serveur de base de données écrit des transactions sur disque des milliers de fois par seconde. Un serveur web sert des fichiers statiques à des milliers de clients simultanés. Un script de sauvegarde copie des téraoctets de données chaque nuit. Dans tous ces cas, la performance et la fiabilité du système dépendent directement de la façon dont l'OS gère les fichiers et les entrées/sorties — un domaine souvent invisible tant que tout fonctionne, et brutalement critique dès qu'un incident survient.

Pourquoi un `write()` qui "réussit" en quelques microsecondes ne garantit-il pas que les données sont réellement sur le disque ? Pourquoi supprimer un gros fichier peut-il libérer de l'espace disque instantanément, ou au contraire mettre du temps ? Pourquoi certains programmes vont-ils beaucoup plus vite que d'autres pour lire le même fichier ? Ce module répond à ces questions en descendant depuis l'appel système applicatif jusqu'au comportement réel du noyau et du disque.

## 7. Concepts fondamentaux

### 7.1 Le modèle Unix : "tout est fichier"

Sous Unix/Linux, fichiers réguliers, répertoires, périphériques, pipes, sockets et même certaines interfaces noyau (`/proc`, `/sys`) sont accessibles via une interface uniforme : ouverture, lecture, écriture, fermeture (`open`, `read`, `write`, `close`). Cette uniformité simplifie considérablement la conception des programmes et des outils système (on peut rediriger la sortie d'une commande vers un fichier, un périphérique ou un pipe sans changer son code).

### 7.2 Architecture d'un système de fichiers Unix

- **Superbloc** : métadonnées globales du système de fichiers (taille, état, type, pointeurs vers les structures principales)
- **Inode (index node)** : structure contenant les métadonnées d'un fichier — taille, permissions, propriétaire, dates, et surtout les pointeurs vers les blocs de données physiques. **L'inode ne contient pas le nom du fichier.**
- **Répertoire** : un fichier spécial associant des noms à des numéros d'inode
- **Blocs de données** : contenu réel du fichier, découpé en blocs de taille fixe sur le disque

**Point clé pédagogique :** un nom de fichier n'est qu'une entrée dans un répertoire pointant vers un inode. Plusieurs noms (dans le même répertoire ou dans des répertoires différents) peuvent pointer vers le **même inode** : c'est le principe du **lien physique (hard link)**. Le fichier n'est réellement supprimé du disque que lorsque son compteur de liens tombe à zéro **et** qu'aucun processus ne le maintient ouvert.

Un **lien symbolique (symlink)**, à l'inverse, est un fichier à part entière (avec son propre inode) qui contient simplement le chemin vers un autre fichier — cassé si la cible est supprimée ou déplacée, contrairement au lien physique.

### 7.3 La VFS (Virtual File System)

Linux supporte de nombreux systèmes de fichiers (ext4, XFS, Btrfs, NTFS, réseau via NFS...). La **VFS** est une couche d'abstraction dans le noyau qui présente une interface uniforme (`open`, `read`, `write`...) aux applications, indépendamment du système de fichiers réel sous-jacent. Chaque système de fichiers implémente ses propres fonctions internes, appelées par la VFS via une table de fonctions (approche proche du polymorphisme orienté objet, implémentée en C via des pointeurs de fonctions).

```
Application → syscall (read/write) → VFS → pilote FS spécifique (ext4, XFS...) → pilote bloc → périphérique
```

**Pourquoi cette abstraction existe-t-elle ?** Sans la VFS, chaque application devrait connaître le système de fichiers spécifique de chaque périphérique qu'elle manipule — une explosion de complexité. La VFS permet à `cp` de fonctionner identiquement, que la source soit sur ext4, un montage réseau NFS, ou une clé USB en FAT32.

### 7.4 Montage (mount)

Un système de fichiers n'est accessible que s'il est **monté**, c'est-à-dire rattaché à un point précis de l'arborescence globale. C'est ce mécanisme qui permet à Linux de présenter plusieurs disques, partitions, ou systèmes de fichiers réseau comme une arborescence unique et cohérente, contrairement au modèle "lettres de lecteur" (`C:`, `D:`) de Windows.

### 7.5 Le cache de pages (page cache)

Pour éviter des accès disque coûteux à chaque lecture/écriture, Linux maintient en mémoire RAM un **cache de pages** : les blocs de fichiers récemment lus ou écrits restent en mémoire. Une lecture ultérieure du même bloc est alors servie directement depuis la RAM, sans accès disque.

**Conséquence pratique importante :** une commande `free -h` montrant peu de RAM "libre" n'indique pas nécessairement un problème — une grande partie de la mémoire "utilisée" peut être du cache de pages, immédiatement récupérable par le noyau si une application en a besoin (colonne `available` de `free`, vue en M3, tient déjà compte de cela).

Pour les **écritures**, le cache introduit une nuance essentielle : un `write()` peut retourner un succès alors que les données sont encore uniquement en cache (page "sale"/dirty), pas encore écrites physiquement sur le disque. Le noyau les écrit ensuite de façon asynchrone (writeback), sauf demande explicite de synchronisation (`fsync()`, montage `sync`). C'est un compromis performance/durabilité : rapide en fonctionnement normal, mais risque de perte de données récentes en cas de coupure brutale (kernel panic, perte d'alimentation) avant écriture effective.

### 7.6 E/S bloquantes, non bloquantes, asynchrones

- **E/S bloquante (par défaut)** : l'appel `read()`/`write()` suspend le processus jusqu'à ce que l'opération soit terminée (ou que les données soient disponibles). Simple à programmer, mais un thread bloqué ne peut rien faire d'autre pendant ce temps.
- **E/S non bloquante** : l'appel retourne immédiatement, même si l'opération n'est pas terminée (avec un code indiquant "réessayer plus tard"). Nécessite une boucle de vérification (polling) ou un mécanisme de notification (`select`, `poll`, `epoll` sous Linux) pour savoir quand une opération est prête.
- **E/S asynchrone** : le programme lance l'opération et est notifié (callback, signal, structure dédiée) une fois terminée, sans bloquer ni avoir à interroger activement.

**Pourquoi ce choix compte pour l'ingénieur ?** Un serveur qui gère des milliers de connexions simultanées avec un modèle *un thread bloquant par connexion* consommera énormément de mémoire (pile par thread) et de temps CPU en changements de contexte (vu en M2). C'est pourquoi les serveurs à haute performance (nginx, Node.js) utilisent des modèles non bloquants/asynchrones basés sur `epoll`.

### 7.7 Interruptions et le chemin d'une E/S disque

Une opération d'E/S physique (lecture disque) est lente comparée au CPU. Le CPU ne reste pas à attendre activement (busy-waiting) : le contrôleur du périphérique déclenche une **interruption matérielle** une fois l'opération terminée, ce qui réveille le processus en attente (transition Bloqué → Prêt, vue en M2) sans gaspiller de cycles CPU en attente active.

Chemin simplifié d'un `read()` sur un fichier non déjà en cache :
1. Application appelle `read()` → transition en mode noyau (system call)
2. VFS délègue au système de fichiers concerné
3. Le système de fichiers détermine les blocs physiques à lire (via l'inode)
4. Le pilote bloc envoie la requête au contrôleur du périphérique
5. Le processus appelant passe en état Bloqué (`D`, vu en M2) en attendant l'interruption
6. Le contrôleur termine l'opération et déclenche une interruption
7. Le noyau copie les données dans le cache de pages, puis dans l'espace utilisateur
8. Le processus repasse en état Prêt, puis Élu, et `read()` retourne

## 8. Architecture

```
┌───────────────────────────────────────────────────────┐
│                  Application (user space)                │
│              read()/write()/open()/close()                │
└───────────────────────┬───────────────────────────────┘
                         │ system call
┌───────────────────────▼───────────────────────────────┐
│                        VFS                                │
│         (interface uniforme, indépendante du FS)          │
└─────┬───────────────┬───────────────┬─────────────────┘
      │               │               │
┌─────▼─────┐   ┌─────▼─────┐   ┌─────▼─────┐
│   ext4     │   │   XFS      │   │   NFS      │
└─────┬─────┘   └─────┬─────┘   └─────┬─────┘
      │               │               │
┌─────▼───────────────▼───────────────▼─────────────────┐
│              Cache de pages (RAM)                         │
└─────────────────────────┬───────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────┐
│         Couche bloc + pilotes + contrôleur disque           │
│                    (interruption au terme de l'E/S)          │
└──────────────────────────────────────────────────────────┘
```

## 9. Fonctionnement interne (cas Linux)

Sous Linux, chaque fichier ouvert par un processus est représenté par une **table de descripteurs de fichiers** (file descriptors, `0`/`1`/`2` réservés à stdin/stdout/stderr), pointant vers une **table de fichiers ouverts** au niveau système (contenant la position courante du curseur, les modes d'ouverture), elle-même pointant vers l'inode réel en mémoire (`struct inode`).

Cette indirection à deux niveaux explique pourquoi deux processus qui ouvrent indépendamment le même fichier ont chacun leur propre position de curseur (`lseek`), alors que deux descripteurs issus d'un même `fork()` (donc de la même ouverture) partagent la même position — un piège classique en programmation système.

Le journaling (ext4, XFS) inscrit les métadonnées (et parfois les données) modifiées dans un journal avant de les appliquer réellement, permettant une récupération rapide et cohérente après un arrêt brutal (contrairement à un `fsck` complet nécessaire sur des systèmes de fichiers non journalisés).

## 10. Démonstration

```bash
# Observer inode et compteur de liens d'un fichier
touch /tmp/fichier_a
ln /tmp/fichier_a /tmp/fichier_b   # lien physique
ls -li /tmp/fichier_a /tmp/fichier_b   # même numéro d'inode, Links=2

# Observer les descripteurs de fichiers ouverts par un processus
sleep 100 &
PID=$!
ls -l /proc/$PID/fd

# Observer le cache de pages
free -h   # colonne "buff/cache"
```

## 11. Commandes / outils

| Commande | Rôle | Exemple |
|---|---|---|
| `ls -li` | Afficher le numéro d'inode | `ls -li /etc/passwd` |
| `df -h` | Espace disque par système de fichiers monté | `df -h` |
| `du -sh` | Taille réelle occupée par un répertoire | `du -sh /var/log` |
| `lsof` | Fichiers ouverts par processus/système | `lsof -p 1234` |
| `iostat -x 1` | Statistiques d'E/S disque en continu | `iostat -x 1 5` |
| `iotop` | Consommation d'E/S par processus (proche de `top`) | `iotop -o` |
| `mount` / `findmnt` | Points de montage actifs | `findmnt` |
| `fsync`/`sync` (commande `sync`) | Forcer l'écriture du cache vers le disque | `sync` |
| `stat` | Métadonnées complètes d'un fichier (inode, dates, liens) | `stat /etc/passwd` |

**Erreur fréquente :** confondre `du` (espace réellement occupé sur le disque) et la taille annoncée par `ls -l` (peut différer à cause des trous de fichier "sparse" ou de l'allocation par blocs) — pour un audit d'espace disque, toujours préférer `du`.

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** une équipe supprime un très gros fichier de logs (50 Go) pour libérer de l'espace disque sur un serveur en production. `df -h` continue pourtant d'afficher l'espace comme occupé plusieurs minutes après la suppression, provoquant une alerte de saturation disque toujours active.

**Analyse attendue :** la commande `rm` supprime l'entrée du répertoire (le nom pointant vers l'inode), mais tant qu'un **processus maintient le fichier ouvert** (ex : un service de journalisation qui écrivait encore dedans au moment de la suppression), l'espace disque physique n'est réellement libéré que lorsque ce processus ferme le descripteur — le compteur de liens *et* de références ouvertes de l'inode doit tomber à zéro. Piste de diagnostic : `lsof | grep deleted` révèle les fichiers supprimés mais toujours maintenus ouverts par un processus, avec la taille réellement occupée. Résolution : redémarrer (ou signaler proprement, ex: `SIGHUP` pour les services qui le supportent) le processus concerné pour qu'il libère le descripteur.

## 15. Sécurité

- Les **permissions Unix** (rwx, propriétaire/groupe/autres) et les **ACL** étendues contrôlent l'accès aux fichiers — un mauvais réglage (`chmod 777` par facilité) est une des causes les plus fréquentes de compromission
- Les **liens symboliques** peuvent être exploités dans des attaques de type *symlink attack* (un attaquant remplace un fichier temporaire par un lien vers un fichier sensible, avant qu'un programme privilégié n'écrive dedans) — d'où l'importance des flags `O_NOFOLLOW`/`O_EXCL` dans le code système sensible
- Le **cache de pages partagé** entre processus ayant mappé le même fichier (`mmap`) peut être un vecteur d'attaque si les permissions de mapping ne sont pas correctement restreintes (lien avec W^X vu en M3)
- Les fichiers supprimés mais encore ouverts (`lsof | grep deleted`) peuvent aussi représenter un risque de fuite d'information résiduelle si mal gérés lors d'un audit de sécurité

## 16. Automatisation

Script Bash de détection de fichiers supprimés mais toujours ouverts, consommant un espace disque significatif.

```bash
#!/usr/bin/env bash
# Détecte les fichiers supprimés mais toujours ouverts, occupant plus de X Mo
set -euo pipefail

SEUIL_MO=100
LOGFILE="/var/log/deleted_open_files.log"

RESULTAT=$(lsof 2>/dev/null | awk -v seuil="$SEUIL_MO" '
    /deleted/ {
        taille_octets = $7
        taille_mo = taille_octets / 1024 / 1024
        if (taille_mo > seuil) {
            printf "%s PID=%s CMD=%s TAILLE=%.1fMo FICHIER=%s\n", $0, $2, $1, taille_mo, $NF
        }
    }' | cut -d' ' -f2- 2>/dev/null || true)

if [[ -n "$RESULTAT" ]]; then
    echo "$(date '+%F %T') - ALERTE: fichiers supprimés toujours ouverts détectés" | tee -a "$LOGFILE"
    echo "$RESULTAT" | tee -a "$LOGFILE"
    exit 1
fi

echo "$(date '+%F %T') - OK: aucun fichier supprimé significatif toujours ouvert" >> "$LOGFILE"
exit 0
```

*Note pédagogique :* le parsing de `lsof` par colonnes peut varier selon la version/distribution — en contexte réel, préférer `lsof -F` (format machine-readable stable) pour un script robuste ; la version ci-dessus, volontairement simplifiée pour la lisibilité pédagogique, est à discuter en TD comme amélioration possible.

## 17. Troubleshooting

Scénario court : un serveur affiche `df -h` proche de 100% sur `/`, mais `du -sh /*` ne totalise qu'une fraction de l'espace annoncé occupé. Démarche : suspecter des fichiers supprimés toujours ouverts (`lsof | grep deleted`) → identifier le(s) processus responsables → décider entre redémarrage du service ou libération contrôlée du descripteur → vérifier via `df -h` que l'espace est bien libéré → documenter la cause pour éviter la récurrence (ex : rotation de logs mal configurée).

## 18. Synthèse

- Sous Unix, "tout est fichier" : interface uniforme via `open`/`read`/`write`/`close`
- L'inode contient les métadonnées et pointeurs vers les données ; le nom de fichier n'est qu'une entrée de répertoire
- La VFS abstrait les différents systèmes de fichiers derrière une interface commune
- Le cache de pages accélère les accès mais introduit un délai entre `write()` réussi et écriture physique réelle (pages "sales")
- E/S bloquantes, non bloquantes et asynchrones représentent des compromis différents pour la scalabilité (lien direct avec le coût des threads vu en M2)
- Un fichier supprimé mais encore ouvert par un processus continue d'occuper l'espace disque — cause fréquente d'incidents de saturation disque en production

## 19. Questions d'évaluation

**QCM**
1. L'inode d'un fichier contient :
 a) son nom b) ses métadonnées et pointeurs vers les données, mais pas son nom c) uniquement ses permissions d) le contenu complet du fichier pour les petits fichiers uniquement

2. Un `write()` qui retourne un succès garantit que :
 a) les données sont physiquement sur le disque b) les données sont dans le cache de pages, pas nécessairement encore sur le disque c) le fichier est fermé d) aucun autre processus ne peut lire le fichier

**Questions ouvertes**
- Expliquez pourquoi supprimer un fichier avec `rm` ne libère pas toujours immédiatement l'espace disque correspondant.
- Expliquez la différence entre lien physique et lien symbolique, avec un exemple de situation où l'un des deux serait cassé et pas l'autre.

**Exercice technique**
- Donnez la commande permettant d'identifier, parmi tous les processus du système, ceux qui maintiennent ouverts des fichiers supprimés, avec leur taille.

**Problème de diagnostic**
- `df -h` indique 98% d'utilisation sur `/var`, mais un audit manuel des répertoires ne trouve pas de fichier volumineux correspondant. Formulez l'hypothèse la plus probable et la commande pour la confirmer.

## 20. Mini-projet

Développer un script (Bash ou Python) qui :
1. Scanne périodiquement les fichiers supprimés mais toujours ouverts (`lsof`), au-delà d'un seuil de taille configurable
2. Identifie le processus responsable et son ancienneté (durée depuis son démarrage)
3. Journalise chaque détection avec horodatage, PID, commande, taille estimée
4. Propose (sans l'exécuter automatiquement, pour rester prudent) une commande de correction suggérée (ex : redémarrage du service identifié)
5. Génère un rapport hebdomadaire de synthèse

---

# PARTIE II — TD

### Niveau 1 — Compréhension

**Exercice 1.1**
Expliquez la différence entre un inode et un nom de fichier. Pourquoi peut-on dire que "le nom de fichier n'appartient pas au fichier lui-même" ?

**Exercice 1.2**
Qu'est-ce que la VFS, et quel problème résout-elle concrètement pour un développeur d'application ?

**Exercice 1.3**
Citez les trois modèles d'E/S vus en cours (bloquant, non bloquant, asynchrone) et donnez, pour chacun, un cas d'usage où il serait préférable aux deux autres.

### Niveau 2 — Application

**Exercice 2.1**
Vous créez un fichier `a.txt`, puis exécutez `ln a.txt b.txt`. Que vaut le compteur de liens de l'inode concerné après cette opération ? Que se passe-t-il si vous supprimez `a.txt` ensuite — `b.txt` reste-t-il accessible ?

**Exercice 2.2**
Donnez la commande permettant d'observer, pour un processus donné (PID 4521), la liste de tous ses descripteurs de fichiers actuellement ouverts.

**Exercice 2.3**
Donnez la commande permettant de forcer l'écriture immédiate de toutes les pages "sales" du cache vers le disque.

### Niveau 3 — Analyse

**Exercice 3.1**
Un serveur affiche `free -h` avec très peu de mémoire dans la colonne "libre" mais une valeur élevée dans "buff/cache", et la colonne "available" reste confortable. Un collègue s'inquiète d'un manque de RAM imminent. Est-ce justifié ? Argumentez en reliant votre réponse au mécanisme du cache de pages.

**Exercice 3.2**
`df -h` indique 100% d'utilisation sur une partition, alors qu'un audit exhaustif avec `du -sh` sur tous les répertoires visibles ne trouve qu'une fraction de l'espace annoncé occupé. Formulez une hypothèse cohérente avec les mécanismes vus en cours et la commande permettant de la vérifier.

### Niveau 4 — Problème complexe

**Exercice 4.1**
Un service de logs applicatif tourne en continu et écrit dans un fichier via un descripteur ouvert au démarrage. L'équipe d'exploitation configure une rotation de logs qui **renomme** l'ancien fichier puis en crée un nouveau du même nom, sans jamais redémarrer le service applicatif. Décrivez, en vous appuyant sur les mécanismes d'inode et de descripteurs de fichiers vus en cours, ce qui va se passer concrètement (où le service continuera-t-il réellement d'écrire ?), pourquoi ce comportement pose un problème pour l'espace disque, et une solution standard utilisée en production pour l'éviter (indice : chercher le mécanisme utilisé par les outils de rotation de logs matures comme `logrotate` avec l'option `copytruncate`, ou l'envoi d'un signal au service).

---

## Correction du TD

**1.1** — L'inode contient les métadonnées d'un fichier (taille, permissions, dates, pointeurs vers les blocs de données) mais ne contient jamais son nom. Le nom de fichier est une simple entrée dans un répertoire (lui-même un fichier spécial), qui associe un nom à un numéro d'inode. On peut donc dire que le nom "n'appartient pas" au fichier : c'est une référence externe, et plusieurs noms peuvent pointer vers le même inode (liens physiques).

**1.2** — La VFS est une couche d'abstraction du noyau qui présente une interface uniforme (`open`, `read`, `write`, `close`...) à toutes les applications, indépendamment du système de fichiers réel sous-jacent (ext4, XFS, NFS...). Elle résout le problème suivant pour le développeur : sans elle, chaque application devrait connaître et gérer explicitement les spécificités de chaque système de fichiers qu'elle pourrait rencontrer — la VFS permet d'écrire du code générique qui fonctionne identiquement quel que soit le FS réel.

**1.3** — Bloquant : simple à programmer, adapté à un script séquentiel ou une application avec peu de connexions simultanées (ex: un outil en ligne de commande). Non bloquant : adapté quand un même thread doit surveiller plusieurs sources d'E/S sans se figer sur aucune (ex: un petit serveur gérant quelques connexions avec `select`/`poll`). Asynchrone : adapté aux applications à très forte concurrence où bloquer ou même faire du polling actif serait trop coûteux (ex: serveur web haute performance gérant des dizaines de milliers de connexions avec `epoll`).

**2.1** — Le compteur de liens de l'inode passe à 2 après le `ln`. Après suppression de `a.txt`, le compteur redescend à 1 et l'inode (avec son contenu) reste accessible via `b.txt`, car le contenu n'est réellement libéré que lorsque le compteur de liens (et de références ouvertes) atteint zéro.

**2.2** — `ls -l /proc/4521/fd` (ou `lsof -p 4521` pour une vue plus détaillée avec les noms de fichiers résolus).

**2.3** — `sync` (force l'écriture de toutes les pages sales du système ; `fsync(fd)` au niveau programmatique cible un descripteur de fichier précis).

**3.1** — Ce n'est pas justifié comme signe d'alerte immédiat. Le cache de pages ("buff/cache") est de la mémoire utilisée de façon opportuniste pour accélérer les accès disque, mais immédiatement récupérable par le noyau dès qu'une application en a besoin — c'est exactement ce que reflète la colonne "available" (vue en M3), qui reste confortable ici. Un manque de RAM réel se manifesterait par une colonne "available" faible, pas par un cache élevé en tant que tel.

**3.2** — Hypothèse cohérente : un ou plusieurs fichiers ont été supprimés (`rm`) mais restent maintenus ouverts par un processus en cours d'exécution, donc leur espace disque n'a pas été réellement libéré même si aucune entrée de répertoire ne pointe plus vers eux (donc invisibles pour `du`, qui parcourt l'arborescence de noms). Commande de vérification : `lsof | grep deleted` (ou `lsof +L1` pour lister spécifiquement les fichiers avec un compteur de liens à zéro mais encore ouverts).

**4.1** — Le service applicatif a ouvert le fichier de logs au démarrage et détient un descripteur pointant vers l'**inode d'origine**. Quand `logrotate` (ou un outil équivalent) *renomme* ce fichier, l'inode ne change pas — seul le nom associé change. Le service continue donc d'écrire, via son descripteur toujours valide, dans **l'ancien inode renommé**, pas dans le nouveau fichier créé sous le nom d'origine. Résultat : le nouveau fichier de logs reste vide, tandis que l'ancien fichier (renommé, donc invisible sous son nom d'origine, mais toujours "vivant" via le descripteur ouvert) continue de grossir indéfiniment, occupant de l'espace disque sans qu'aucune commande basée sur les noms de fichiers (`ls`, `du` sur le nom attendu) ne le révèle facilement.
Solutions standards : soit envoyer un signal au service (souvent `SIGHUP`) pour qu'il rouvre explicitement son fichier de logs sous le nouveau nom après rotation, soit utiliser l'option `copytruncate` de `logrotate`, qui copie le contenu actuel puis **tronque** le fichier d'origine sur place (sans le renommer), de sorte que le descripteur déjà ouvert par le service continue de pointer vers le même inode, désormais vide, sans nécessiter de redémarrage.

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur système sur un serveur où l'espace disque semble se réduire de façon incohérente avec les fichiers visibles dans l'arborescence. Vous devez reproduire, comprendre et diagnostiquer ce phénomène, puis développer un outil de détection.

## Architecture

Une machine Linux unique (VM ou conteneur), avec les outils `lsof`, `du`, `df`, `stat` disponibles.

## Objectifs
- Observer concrètement inodes, liens physiques et symboliques
- Provoquer et diagnostiquer un cas de fichier supprimé mais toujours ouvert
- Observer le cache de pages et la différence entre `write()` et écriture physique
- Écrire un script de détection de fichiers supprimés toujours ouverts

## Préparation

```bash
which lsof du df stat ls 2>/dev/null
```

## Étapes

### Étape 1 — Inodes et liens

```bash
mkdir -p /tmp/lab_fs && cd /tmp/lab_fs
echo "contenu original" > original.txt
ln original.txt lien_physique.txt
ln -s original.txt lien_symbolique.txt

ls -li   # observer les numéros d'inode
stat original.txt | grep Links

rm original.txt
cat lien_physique.txt    # doit fonctionner
cat lien_symbolique.txt  # doit échouer (lien cassé)
```
*Question :* pourquoi le lien symbolique est-il cassé mais pas le lien physique ?

### Étape 2 — Fichier supprimé mais toujours ouvert

```bash
cd /tmp/lab_fs
python3 -c "
import time
f = open('gros_fichier.log', 'w')
for i in range(100000):
    f.write('ligne de log ' * 10 + '\n')
f.flush()
print('Fichier écrit, PID:', __import__('os').getpid())
time.sleep(60)
f.close()
" &
PID=$!
sleep 2
ls -lh gros_fichier.log
rm gros_fichier.log
ls -lh gros_fichier.log 2>&1   # doit indiquer "No such file"
lsof -p $PID | grep deleted
df -h /tmp
```
*Résultat attendu :* le fichier n'apparaît plus dans `ls`, mais `lsof` révèle qu'il est toujours ouvert par le PID Python, avec sa taille réelle.

### Étape 3 — Cache de pages et écriture différée

```bash
cd /tmp/lab_fs
free -h
dd if=/dev/zero of=fichier_test.dat bs=1M count=200 2>&1
free -h   # observer l'évolution de "buff/cache"
sync
free -h   # observer si "buff/cache" a changé après sync
```
*Question :* la commande `dd` a-t-elle terminé "instantanément" au sens de l'écriture physique réelle sur le disque, ou seulement au sens du cache ? Comment le vérifier plus précisément (indice : `iostat` pendant l'opération, dans un terminal séparé si possible) ?

### Étape 4 — Script de détection

Reprendre et tester le script de détection de la section 16 du cours (`deleted_open_files.log`), en l'exécutant pendant que le processus Python de l'étape 2 est toujours actif (avant l'expiration des 60 secondes de `time.sleep`).

```bash
bash deleted_open_files_detector.sh
```

## Commandes de référence
`ln`, `ls -li`, `stat`, `lsof`, `df -h`, `du -sh`, `dd`, `sync`, `free -h`

## Résultats attendus
- Capture montrant les numéros d'inode identiques pour `original.txt` et `lien_physique.txt` avant suppression
- Capture montrant `lsof` révélant le fichier supprimé toujours ouvert avec sa taille
- Capture montrant l'évolution de `buff/cache` avant/après l'écriture avec `dd`
- Log d'alerte généré par le script de détection à l'étape 4

## Questions

1. Si, à l'étape 2, vous aviez attendu la fin des 60 secondes (fermeture du fichier par le script Python) avant d'exécuter `df -h`, qu'auriez-vous observé de différent par rapport à l'exécuter immédiatement après le `rm` ?
2. Pourquoi `du -sh /tmp/lab_fs` ne montre-t-il, à l'étape 2 après le `rm`, aucune trace de `gros_fichier.log`, alors que l'espace disque réel reste occupé ?
3. En production, quelle serait votre première action face à ce type d'incident : redémarrer immédiatement le service concerné, ou chercher d'abord à comprendre pourquoi le fichier a été supprimé pendant qu'il était encore utilisé ? Justifiez.

## Troubleshooting (panne à diagnostiquer)

**Situation fournie :** après une rotation de logs automatisée (`logrotate` configuré sans `copytruncate` ni envoi de signal au service), l'espace disque de `/var/log` continue de croître de façon inexpliquée malgré la rotation apparemment fonctionnelle (les nouveaux fichiers de logs datés apparaissent bien chaque jour).

**Démarche attendue :**
1. Symptôme : croissance disque continue malgré une rotation en apparence correcte
2. Hypothèse (directement issue de l'exercice 4.1 du TD) : le service applicatif a conservé un descripteur ouvert vers l'ancien fichier renommé par `logrotate`, et continue d'écrire dedans indéfiniment
3. Vérification : `lsof | grep deleted` ou `lsof +L1` pour repérer un fichier au compteur de liens nul (renommé puis l'ancien nom supprimé par un cycle de rotation ultérieur) mais toujours ouvert, avec une taille anormalement élevée
4. Isolation : confirmer que le PID concerné correspond bien au service de journalisation suspecté
5. Correction immédiate : envoyer le signal approprié au service (souvent `SIGHUP`) pour qu'il rouvre son fichier de logs, ou redémarrer le service si aucun mécanisme de rechargement n'existe
6. Correction de fond : reconfigurer `logrotate` avec l'option `copytruncate`, ou avec un script `postrotate` envoyant le signal adéquat au service
7. Vérification : surveiller `lsof` et `df -h` sur plusieurs cycles de rotation suivants pour confirmer la résolution
8. Documentation : consigner la cause et le correctif appliqué, avec référence au comportement inode/descripteur expliqué en cours

## Validation

Le TP est validé si l'étudiant fournit :
- Les 4 captures/résultats demandés
- Des réponses argumentées aux 3 questions, notamment une explication correcte du lien entre `rm`, descripteurs ouverts et libération d'espace
- Une identification correcte de la cause dans le scénario de troubleshooting, avec la solution `copytruncate`/signal correctement expliquée

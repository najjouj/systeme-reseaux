# UE3 — Module 19 : Logs et Tâches Planifiées

---

# PARTIE I — COURS

## 1. Titre du cours
Journalisation durable et planification de tâches : logrotate, rsyslog et cron

## 2. Niveau
Élève ingénieur — cycle ingénieur, UE3 administration Linux (S2)

## 3. Durée
CM : 3h30 (module orienté pratique — +3h TD, +4h TP en Parties II et III)

## 4. Prérequis
- UE3-Module 16 : `journalctl` de base, timers systemd
- UE1-Module 4 : mécanisme de rotation de logs déjà rencontré (fichier supprimé mais toujours ouvert)
- UE1-Module 6 : distinction service `oneshot` / timer

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Configurer la persistance du journal `journald` au-delà de son comportement par défaut
- Écrire une configuration `logrotate` personnalisée pour une application maison
- Expliquer le rôle de `rsyslog` en complément de `journald` et le principe de centralisation de logs
- Écrire et déboguer une tâche `cron`, en anticipant les pièges classiques liés à l'environnement d'exécution
- Choisir entre `cron` et un timer systemd (rappel M16) selon le contexte
- Diagnostiquer une tâche planifiée qui ne s'exécute pas ou échoue silencieusement

## 6. Motivation

Un incident de production survient souvent à un moment où personne n'observe directement le système — c'est la journalisation, correctement configurée et conservée, qui permet de reconstituer après coup ce qui s'est passé. De la même façon, une grande partie des tâches de maintenance d'un serveur (sauvegardes, nettoyage, rapports) doivent s'exécuter de façon autonome, sans intervention humaine, à des horaires précis — c'est le rôle des tâches planifiées.

Ce module relie deux briques d'administration quotidienne trop souvent négligées jusqu'à ce qu'un incident révèle leur importance : des logs absents ou trop vite supprimés rendent un diagnostic impossible ; une tâche planifiée mal configurée peut échouer silencieusement pendant des mois sans que personne ne s'en aperçoive, jusqu'à ce que son absence (une sauvegarde jamais réalisée, par exemple) ait des conséquences graves.

## 7. Concepts fondamentaux

### 7.1 `journald` — persistance du journal

Par défaut sur de nombreuses distributions, le journal `journald` (rappel M16) est **volatile**, stocké uniquement en mémoire (`/run/log/journal`) et perdu au redémarrage. Pour une persistance à travers les redémarrages :

```bash
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal
sudo systemctl restart systemd-journald
```

La configuration de rétention se règle dans `/etc/systemd/journald.conf` :
```ini
[Journal]
Storage=persistent
SystemMaxUse=2G
MaxRetentionSec=90day
```

**`SystemMaxUse` — un rappel direct de l'UE1 :** sans cette limite, le journal pourrait croître indéfiniment jusqu'à saturer l'espace disque disponible (rappel UE1-M4 sur la saturation disque) — une limite explicite est indispensable en production, son absence n'étant pas différente en principe du risque déjà illustré avec le mécanisme de journalisation applicative classique.

### 7.2 `logrotate` — au-delà de la théorie déjà vue

`logrotate` (déjà mentionné en UE1-M4 à propos du piège "fichier renommé, descripteur toujours ouvert") gère la rotation périodique des fichiers de logs **applicatifs** classiques (pas le journal `journald` lui-même, qui gère sa propre rétention via `journald.conf`). Une configuration personnalisée pour une application maison :

```
# /etc/logrotate.d/monapp
/var/log/monapp/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 appuser appgroup
    sharedscripts
    postrotate
        systemctl kill -s HUP monapp.service
    endscript
}
```

- **`rotate 14`** : conserve 14 anciennes rotations avant suppression définitive
- **`compress`/`delaycompress`** : compresse les anciennes rotations pour économiser l'espace, en retardant la compression de la plus récente d'un cycle (pour laisser le temps à un outil de consultation externe de la lire sans avoir à gérer la décompression immédiatement)
- **`create`** : recrée le fichier de log immédiatement après rotation, avec les permissions spécifiées — indispensable si l'application ne recrée pas elle-même son fichier de log après une rotation
- **`postrotate`/`endscript`** : bloc de commandes exécutées après la rotation, typiquement pour signaler à l'application de rouvrir son fichier de log (rappel direct du piège "rotation cassée" du module UE1-M4 : c'est précisément ce bloc qui manquait dans le scénario fautif étudié à l'époque)

**Lien direct avec UE1-M4 :** le choix entre `copytruncate` (tronquer le fichier sur place, sans le renommer) et un `postrotate` envoyant un signal à l'application (comme ci-dessus) correspond exactement aux deux solutions déjà présentées comme correctifs du piège de rotation cassée étudié en UE1 — ce module en fait la mise en œuvre pratique et opérationnelle.

### 7.3 `rsyslog` — complément et centralisation

`rsyslog` (ou son équivalent `syslog-ng` sur certaines distributions) reste largement présent en complément de `journald`, notamment pour deux raisons : la compatibilité avec des applications plus anciennes écrivant directement au format syslog classique, et surtout la **centralisation de logs** — envoyer les journaux de plusieurs machines vers un serveur de collecte central, une capacité native de `rsyslog` (via le réseau, en UDP ou TCP) que `journald` seul ne propose pas nativement de façon aussi mature.

```
# /etc/rsyslog.d/50-remote.conf sur une machine cliente
*.* @@serveur-logs.exemple.lan:514   # @@ = TCP (fiable), @ seul = UDP (rappel UE2-M11 sur les compromis)
```

**Rappel direct de l'UE2-M11 :** le choix entre UDP (`@`) et TCP (`@@`) pour l'envoi de logs distants applique directement le compromis fiabilité/légèreté déjà étudié — UDP est plus léger mais peut perdre des messages de log sans notification, un compromis à évaluer selon la criticité des logs concernés (des logs de debug peuvent tolérer UDP, des logs d'audit de sécurité devraient privilégier TCP pour leur fiabilité).

### 7.4 `cron` — syntaxe et structure

```
# minute heure jour_du_mois mois jour_de_semaine commande
0 3 * * * /opt/scripts/sauvegarde.sh
*/15 * * * * /opt/scripts/verification.sh
0 2 * * 0 /opt/scripts/nettoyage_hebdomadaire.sh
```

Les cinq champs (minute 0-59, heure 0-23, jour du mois 1-31, mois 1-12, jour de la semaine 0-7 où 0 et 7 valent tous deux dimanche) acceptent des valeurs précises, des plages (`1-5`), des listes (`1,15`), des pas (`*/15`), et l'astérisque (`*`, "toute valeur").

**Emplacements des tâches cron :**
- `crontab -e` (par utilisateur) : tâches personnelles, exécutées avec les droits de l'utilisateur propriétaire
- `/etc/cron.d/` : fichiers de tâches système, avec un champ utilisateur supplémentaire précisant sous quelle identité exécuter chaque tâche
- `/etc/cron.daily/`, `/etc/cron.weekly/`, etc. : scripts exécutés selon une fréquence prédéfinie par le système, sans syntaxe cron explicite à écrire

### 7.5 Le piège classique de l'environnement `cron`

**L'erreur la plus fréquente et la plus fréquemment rencontrée en administration système :** un script fonctionne parfaitement lorsqu'il est lancé manuellement depuis un terminal, mais échoue silencieusement (ou de façon inattendue) lorsqu'il est exécuté par `cron`. La cause presque systématique : `cron` exécute chaque tâche avec un **environnement minimal**, très différent de celui d'une session interactive — notamment une variable `PATH` réduite, l'absence de variables d'environnement personnelles habituellement chargées par les fichiers de profil shell (`.bashrc`, `.profile`), et un répertoire de travail par défaut qui n'est pas nécessairement celui attendu par le script.

**Bonnes pratiques systématiques pour tout script destiné à `cron` :**
```bash
#!/usr/bin/env bash
# Toujours utiliser des chemins absolus, jamais supposer un PATH particulier
/usr/bin/rsync -av /data/ /backup/

# Ou définir explicitement le PATH nécessaire en tête de script
export PATH=/usr/local/bin:/usr/bin:/bin

# Toujours se positionner explicitement dans le bon répertoire si le script en dépend
cd /opt/monapp || exit 1
```

### 7.6 Redirection de sortie et notification d'échec

Par défaut, `cron` envoie la sortie standard et d'erreur d'une tâche par email local à l'utilisateur propriétaire (si un système de messagerie local est configuré, ce qui n'est pas toujours le cas) — une tâche dont personne ne consulte jamais cette sortie échoue **silencieusement** au sens opérationnel, même si techniquement une trace existe quelque part. Bonne pratique : rediriger explicitement vers un fichier de log dédié, ou vers `journald` via `logger`, et vérifier activement le code de retour :

```
0 3 * * * /opt/scripts/sauvegarde.sh >> /var/log/sauvegarde.log 2>&1 || echo "Échec sauvegarde $(date)" | logger -t sauvegarde-cron -p user.err
```

### 7.7 `cron` vs timer systemd — critères de choix (rappel et approfondissement de M16)

| Critère | cron | Timer systemd |
|---|---|---|
| Simplicité pour une tâche basique | Très simple, syntaxe concise | Plus verbeux (deux fichiers) |
| Journalisation intégrée | Non native (redirection manuelle nécessaire) | Automatique via journald |
| Gestion des dépendances | Aucune | Native (`After=`, `Requires=`) |
| Rattrapage si machine éteinte | Non natif (sauf `anacron` en complément) | `Persistent=true` natif |
| Limitation de ressources | Aucune native | Native (cgroups, comme tout service systemd) |

**Recommandation pratique :** pour une tâche simple et isolée sur un système déjà familier avec `cron`, `cron` reste parfaitement valide et largement utilisé en production ; pour une tâche critique nécessitant une traçabilité fine, des dépendances explicites, ou un rattrapage fiable après extinction, un timer systemd est structurellement préférable.

## 8. Architecture

```
┌────────────────────────────────────────────────────────────┐
│  Application ──stdout/stderr──▶ journald (volatile par défaut) │
│                                       │                          │
│                              Storage=persistent                  │
│                                       ▼                          │
│                              /var/log/journal/ (durable)          │
│                                                                   │
│  Application ──fichier .log──▶ logrotate (rotation périodique)     │
│                                       │                          │
│                              postrotate / copytruncate             │
│                                                                   │
│  rsyslog ──réseau (UDP/TCP)──▶ Serveur de collecte central          │
│                                                                   │
│  cron / timer systemd ──▶ exécution planifiée de scripts             │
└────────────────────────────────────────────────────────────┘
```

## 9. Fonctionnement interne

`journald` structure ses journaux dans un format binaire indexé (contrairement au texte brut historique de syslog), permettant des requêtes rapides et filtrées (déjà exploitées en pratique au module 16 via `journalctl -u`/`-p`) ; ce format binaire explique pourquoi `journalctl` est le seul outil réellement adapté pour consulter ce journal, un simple `cat`/`grep` sur les fichiers bruts sous `/var/log/journal/` n'étant ni pratique ni fiable. `cron` (le démon `crond`) relit ses fichiers de configuration à intervalle régulier ou sur notification de modification, et déclenche chaque tâche planifiée en créant un nouveau processus enfant à l'échéance prévue — un mécanisme relativement simple comparé à l'infrastructure de dépendances et de cgroups de systemd, ce qui explique historiquement sa popularité pour sa légèreté, mais aussi ses limites déjà évoquées en 7.7.

## 10. Démonstration

```bash
# Vérifier si le journal est actuellement volatile ou persistant
journalctl --disk-usage

# Tester une configuration logrotate sans attendre son déclenchement naturel
sudo logrotate --debug /etc/logrotate.d/monapp
sudo logrotate --force /etc/logrotate.d/monapp   # forcer l'exécution réelle

# Observer l'environnement minimal d'une tâche cron
* * * * * env > /tmp/env_cron.txt
# (comparer ensuite avec `env` exécuté dans un terminal interactif)
```

## 11. Commandes / outils

| Commande | Rôle | Exemple |
|---|---|---|
| `journalctl --disk-usage` | Vérifier l'espace occupé par le journal | `journalctl --disk-usage` |
| `logrotate --debug` | Simuler une rotation sans l'exécuter réellement | `sudo logrotate --debug /etc/logrotate.d/monapp` |
| `logrotate --force` | Forcer une rotation immédiate | `sudo logrotate --force /etc/logrotate.conf` |
| `crontab -l` / `-e` | Lister/éditer les tâches cron de l'utilisateur courant | `crontab -l` |
| `crontab -u <user> -l` | Lister les tâches cron d'un autre utilisateur (root requis) | `sudo crontab -u appuser -l` |
| `logger` | Envoyer un message au journal système depuis un script | `logger -t moncript "message"` |
| `run-parts --test` | Simuler l'exécution des scripts d'un répertoire `/etc/cron.daily` | `run-parts --test /etc/cron.daily` |

**Erreur fréquente :** modifier un fichier sous `/etc/cron.d/` sans respecter le format attendu (champ utilisateur supplémentaire par rapport à un `crontab` personnel) — une erreur de format silencieusement ignorée par `cron`, la tâche ne s'exécutant jamais sans qu'aucune erreur explicite ne soit remontée à l'administrateur.

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** une entreprise découvre, après une perte de données suite à une panne matérielle, que son script de sauvegarde quotidienne, planifié via `cron`, n'avait en réalité plus fonctionné correctement depuis plusieurs mois. Le script fonctionnait parfaitement lorsqu'un administrateur le testait manuellement.

**Analyse attendue :** ce scénario correspond très précisément au piège classique de la section 7.5 : le script, testé manuellement dans un terminal interactif (avec un `PATH` complet et les variables d'environnement habituelles), fonctionnait sans problème ; exécuté par `cron` avec son environnement minimal, une commande utilisée dans le script (probablement appelée sans chemin absolu, en supposant implicitement la présence d'un `PATH` complet) échouait silencieusement, sans qu'aucune notification n'ait été configurée pour alerter de cet échec (rappel section 7.6 : sortie cron non redirigée et non surveillée activement). Correction : réécrire le script avec des chemins absolus systématiques pour toute commande utilisée, ajouter une redirection de sortie vers un fichier de log dédié et une notification explicite en cas d'échec (email, alerte, ou à minima une entrée `journald` via `logger` consultée par un script de supervision), et envisager, pour cette tâche critique, une migration vers un timer systemd offrant une traçabilité native via `journalctl`. Mesure préventive généralisable : toute tâche planifiée critique doit disposer d'une vérification active de son bon déroulement (une alerte en cas d'échec, jamais seulement une trace passive consultée uniquement a posteriori après un incident).

## 15. Sécurité

- Un script exécuté par `cron` avec les privilèges de `root` (tâche système sous `/etc/cron.d/`) doit être protégé avec les mêmes rigueurs que tout binaire privilégié (rappel UE3-M14 sur les SUID) — un script modifiable par un utilisateur non privilégié mais exécuté périodiquement en tant que root constitue une voie d'escalade de privilèges si ses permissions ne sont pas correctement restreintes
- La centralisation de logs (rsyslog vers un serveur distant) protège contre la falsification a posteriori des journaux locaux par un attaquant ayant compromis une machine (effacer ses traces localement ne supprime pas la copie déjà envoyée au serveur central) — une pratique de sécurité de base pour toute investigation forensique crédible
- Un journal `journald` non persistant (volatile, perdu au redémarrage) peut, comme un espace disque saturé déjà mentionné en UE1, priver une investigation de sécurité de traces cruciales si un attaquant provoque délibérément un redémarrage après une intrusion
- Les fichiers de configuration `cron` et les scripts qu'ils invoquent doivent suivre le principe du moindre privilège déjà rencontré à plusieurs reprises : une tâche qui ne nécessite pas de privilèges root ne devrait jamais être placée dans un `crontab` root par simple facilité

## 16. Automatisation

Script Bash de vérification de bon fonctionnement des tâches cron critiques, via un mécanisme de "heartbeat" (chaque tâche surveillée doit mettre à jour un fichier témoin, vérifié périodiquement).

```bash
#!/usr/bin/env bash
# Vérifie que les tâches cron critiques ont bien mis à jour leur fichier témoin récemment
set -euo pipefail

declare -A TACHES_SURVEILLEES=(
    ["sauvegarde"]="/var/run/heartbeat_sauvegarde:90000"    # fichier:seuil_secondes
    ["nettoyage"]="/var/run/heartbeat_nettoyage:604800"
)
LOGFILE="/var/log/cron_heartbeat_watch.log"

for nom in "${!TACHES_SURVEILLEES[@]}"; do
    IFS=':' read -r fichier seuil <<< "${TACHES_SURVEILLEES[$nom]}"
    if [[ ! -f "$fichier" ]]; then
        echo "$(date '+%F %T') - ALERTE: heartbeat absent pour la tâche '$nom' ($fichier)" \
            | tee -a "$LOGFILE"
        continue
    fi
    AGE=$(( $(date +%s) - $(stat -c %Y "$fichier") ))
    if (( AGE > seuil )); then
        echo "$(date '+%F %T') - ALERTE: tâche '$nom' non exécutée depuis ${AGE}s (seuil: ${seuil}s)" \
            | tee -a "$LOGFILE"
    fi
done
```

*Note pédagogique :* ce mécanisme de heartbeat (le script de sauvegarde doit lui-même être modifié pour faire `touch /var/run/heartbeat_sauvegarde` à sa fin réussie) illustre concrètement la mesure préventive recommandée dans l'étude de cas — une vérification active plutôt qu'une confiance passive dans le simple fait qu'une tâche soit planifiée.

## 17. Troubleshooting

Scénario court : une tâche `cron` ajoutée via `crontab -e` ne s'exécute jamais, sans aucune erreur visible. Démarche : vérifier que le service `cron`/`crond` est bien actif (`systemctl status cron`, rappel M16) → vérifier la syntaxe exacte de la ligne cron (un champ manquant ou mal formé peut être silencieusement rejeté selon l'implémentation) → consulter les logs système du démon cron lui-même (`journalctl -u cron` ou `/var/log/syslog` selon la distribution) qui journalise généralement chaque déclenchement de tâche, permettant de vérifier si la tâche a été ne serait-ce que tentée → si la tâche est bien déclenchée mais échoue, appliquer la méthodologie de la section 7.5 (environnement minimal, chemins absolus).

## 18. Synthèse

- `journald` est volatile par défaut ; la persistance et une limite de taille (`SystemMaxUse`) doivent être explicitement configurées
- `logrotate` gère la rotation des logs applicatifs classiques, avec `postrotate`/`copytruncate` comme solutions directes au piège de rotation cassée déjà étudié en UE1
- `rsyslog` reste pertinent pour la centralisation de logs vers un serveur distant, avec le même compromis UDP/TCP déjà vu en UE2 pour la fiabilité
- Le piège le plus fréquent de `cron` est son environnement minimal (PATH réduit) — toujours utiliser des chemins absolus et ne jamais supposer l'environnement d'un terminal interactif
- La sortie d'une tâche cron doit être explicitement redirigée et surveillée activement, jamais laissée à un système d'email local non vérifié
- Un mécanisme de heartbeat/vérification active est la seule garantie fiable qu'une tâche planifiée critique fonctionne réellement, au-delà de sa simple présence dans une configuration

## 19. Questions d'évaluation

**QCM**
1. Par défaut, sur de nombreuses distributions, le journal `journald` est :
 a) toujours persistant b) volatile, perdu au redémarrage sauf configuration explicite c) désactivé par défaut d) stocké uniquement sur un serveur distant

2. La cause la plus fréquente d'échec d'un script fonctionnant manuellement mais pas via cron est :
 a) un problème de permissions systématique b) l'environnement minimal de cron (notamment un PATH réduit) c) l'absence de connexion réseau d) une erreur de syntaxe cron

**Questions ouvertes**
- Expliquez le lien entre le mécanisme `postrotate`/`copytruncate` de `logrotate` et le piège de rotation de logs déjà étudié en UE1-Module 4.
- Expliquez pourquoi une tâche cron critique devrait disposer d'un mécanisme de vérification active (heartbeat) plutôt que de se fier uniquement à sa présence dans la configuration.

**Exercice technique**
- Donnez la commande permettant de simuler l'exécution d'une configuration logrotate sans réellement effectuer la rotation.

**Problème de diagnostic**
- Un script s'exécute correctement en ligne de commande mais échoue silencieusement lorsqu'il est lancé par cron. Proposez la cause la plus probable et deux corrections concrètes à appliquer au script.

## 20. Mini-projet

Développer un script (Bash ou Python) qui :
1. Audite l'ensemble des tâches cron du système (crontabs utilisateur et `/etc/cron.d/`) et signale celles utilisant des commandes sans chemin absolu (heuristique de détection du piège de la section 7.5)
2. Vérifie la configuration de persistance de `journald` (`Storage=persistent` et présence d'une limite `SystemMaxUse`)
3. Implémente et surveille un mécanisme de heartbeat pour une liste configurable de tâches critiques (reprise et généralisation du script de la section 16)
4. Vérifie qu'au moins une configuration `logrotate` existe pour chaque application journalisant significativement dans `/var/log` (heuristique simple, ex : taille de fichier de log au-delà d'un seuil sans rotation associée détectée)
5. Journalise chaque détection avec horodatage et génère un rapport de synthèse périodique

---

# PARTIE II — TD

### Niveau 1 — Compréhension

**Exercice 1.1**
Expliquez pourquoi le journal `journald` est volatile par défaut sur de nombreuses distributions, et ce qui est nécessaire pour le rendre persistant.

**Exercice 1.2**
Décrivez la syntaxe des cinq champs d'une ligne cron, avec un exemple d'exécution tous les jours à 3h30 du matin.

**Exercice 1.3**
Expliquez la différence de rôle entre `logrotate` et `journald` en matière de gestion des logs.

### Niveau 2 — Application

**Exercice 2.1**
Donnez la commande permettant de simuler (sans l'exécuter réellement) une configuration `logrotate` donnée.

**Exercice 2.2**
Donnez la ligne `crontab` permettant d'exécuter un script `/opt/scripts/verif.sh` toutes les 15 minutes.

**Exercice 2.3**
Donnez la commande permettant de vérifier l'espace disque actuellement occupé par le journal `journald`.

### Niveau 3 — Analyse

**Exercice 3.1**
Un script utilise la commande `mysqldump` sans chemin absolu dans un contexte cron, et échoue avec une erreur "command not found" alors que la même commande fonctionne parfaitement en terminal interactif. Expliquez précisément la cause de cet écart.

**Exercice 3.2**
Une entreprise choisit d'utiliser un timer systemd plutôt que cron pour une tâche de sauvegarde critique. Justifiez ce choix en citant au moins deux avantages concrets du timer systemd pour ce cas d'usage précis, en vous appuyant sur le tableau comparatif du cours.

### Niveau 4 — Problème complexe

**Exercice 4.1**
Une entreprise découvre, après un incident de perte de données, qu'une tâche cron de sauvegarde n'a plus fonctionné correctement depuis plusieurs mois, sans qu'aucune alerte n'ait jamais été générée. L'investigation révèle que le script fonctionnait parfaitement en test manuel, que la ligne cron était syntaxiquement correcte et bien exécutée par le démon cron (confirmé dans les logs système), mais que le script lui-même utilisait des commandes sans chemin absolu. En vous appuyant sur l'ensemble des mécanismes vus en cours (environnement cron, redirection de sortie, notification d'échec), proposez une refonte complète de cette tâche de sauvegarde qui aurait empêché à la fois l'échec silencieux initial et l'absence de détection pendant plusieurs mois, en distinguant clairement la correction du script lui-même de la mise en place d'un mécanisme de détection indépendant du bon fonctionnement du script.

---

## Correction du TD

**1.1** — Par défaut, `journald` stocke ses journaux uniquement en mémoire volatile (`/run/log/journal`), un choix qui limite l'usage disque par défaut mais sacrifie la conservation des logs à travers un redémarrage. Pour le rendre persistant, il faut créer le répertoire `/var/log/journal` et configurer `Storage=persistent` dans `/etc/systemd/journald.conf`, accompagné idéalement d'une limite de taille (`SystemMaxUse`) pour éviter une croissance illimitée.

**1.2** — Les cinq champs sont, dans l'ordre : minute (0-59), heure (0-23), jour du mois (1-31), mois (1-12), jour de la semaine (0-7, 0 et 7 valant tous deux dimanche), suivis de la commande à exécuter. Exemple pour 3h30 tous les jours : `30 3 * * * /chemin/vers/commande`.

**1.3** — `logrotate` gère la rotation périodique des fichiers de logs **applicatifs classiques** (fichiers texte écrits directement par des applications dans `/var/log`), en les archivant, compressant et supprimant selon une politique de rétention configurée. `journald` gère son **propre** journal structuré et binaire, avec sa propre politique de rétention configurée séparément (`SystemMaxUse`, `MaxRetentionSec`) — les deux mécanismes coexistent et gèrent des types de journalisation distincts, sans que `logrotate` n'intervienne sur le journal `journald` lui-même.

**2.1** — `sudo logrotate --debug /etc/logrotate.d/<fichier_de_configuration>`.

**2.2** — `*/15 * * * * /opt/scripts/verif.sh`.

**2.3** — `journalctl --disk-usage`.

**3.1** — Cette différence de comportement s'explique par l'environnement minimal dans lequel `cron` exécute chaque tâche (rappel section 7.5 du cours) : en session interactive, la variable `PATH` de l'utilisateur (généralement complète, chargée via les fichiers de profil shell comme `.bashrc`) permet au shell de localiser automatiquement la commande `mysqldump` sans que son chemin complet ne soit nécessaire. `cron`, en revanche, exécute chaque tâche avec un `PATH` réduit et minimal, ne contenant généralement pas le répertoire où se trouve réellement l'exécutable `mysqldump` (souvent `/usr/bin` ou un chemin spécifique selon l'installation) — d'où l'erreur "command not found" spécifiquement dans ce contexte, la commande étant introuvable dans le `PATH` restreint de cron, alors qu'elle est parfaitement installée et fonctionnelle sur le système.

**3.2** — Deux avantages concrets pour une tâche de sauvegarde critique : (1) la **journalisation intégrée automatique** via `journald`, permettant de consulter facilement l'historique des exécutions et leurs résultats via `journalctl -u`, sans avoir à mettre en place manuellement une redirection de sortie comme c'est nécessaire avec cron (rappel section 7.6) ; (2) le **rattrapage automatique** via `Persistent=true`, garantissant que si la machine était éteinte au moment prévu de la sauvegarde, celle-ci s'exécutera tout de même au prochain démarrage plutôt que d'être simplement ignorée jusqu'à la prochaine occurrence planifiée — une garantie particulièrement précieuse pour une tâche aussi critique qu'une sauvegarde, où une exécution manquée sans rattrapage pourrait avoir des conséquences graves en cas d'incident survenant précisément durant cette fenêtre manquée.

**4.1** — Refonte proposée, distinguant clairement les deux axes de correction :

**Correction du script lui-même (empêcher l'échec silencieux initial) :** réécrire le script en utilisant systématiquement des chemins absolus pour toute commande invoquée (ou définir explicitement un `PATH` complet en tête de script), éliminant la cause racine de l'échec initial identifiée dans l'énoncé. Ajouter une gestion d'erreur explicite (`set -euo pipefail` en Bash, vérification du code de retour de chaque étape critique) pour que le script lui-même détecte et signale activement un échec plutôt que de se terminer silencieusement.

**Mise en place d'un mécanisme de détection indépendant (empêcher l'absence de détection pendant plusieurs mois) :** ce second axe est délibérément **indépendant** du bon fonctionnement interne du script lui-même — même un script parfaitement corrigé peut un jour échouer pour une toute autre raison (disque plein, service de base de données injoignable...), et la détection ne doit pas reposer uniquement sur la correction du bug initial. Mettre en place un mécanisme de heartbeat (rappel section 16 du cours) : le script, à sa toute fin réussie uniquement, met à jour un fichier témoin horodaté ; un second processus de surveillance totalement indépendant (autre tâche planifiée, ou outil de supervision existant) vérifie périodiquement que ce fichier témoin a bien été mis à jour dans le délai attendu, et déclenche une alerte active (email, notification, intégration à un système de supervision) si ce n'est pas le cas. Cette approche garantit qu'un futur échec, quelle qu'en soit la cause, sera détecté rapidement et activement signalé, plutôt que de rester invisible jusqu'à ce qu'un incident révèle after-coup l'absence de sauvegardes fonctionnelles — exactement la leçon centrale de l'étude de cas industrielle du cours.

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur système junior pour mettre en place une chaîne complète de journalisation durable et de tâche planifiée fiable pour une application de laboratoire, en reproduisant délibérément puis en corrigeant le piège classique de l'environnement cron.

## Architecture

Une VM ou un conteneur Linux avec systemd fonctionnel (pour `journald`), `cron` (ou `cronie`/`crond` selon la distribution) et `logrotate` installés.

## Objectifs
- Configurer la persistance du journal journald avec une limite de taille
- Écrire et tester une configuration logrotate personnalisée
- Reproduire puis corriger le piège classique de l'environnement cron
- Mettre en place un mécanisme de heartbeat pour une tâche planifiée

## Préparation

```bash
which journalctl logrotate crontab systemctl 2>/dev/null
journalctl --disk-usage
```

## Étapes

### Étape 1 — Persistance journald

```bash
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal

sudo tee -a /etc/systemd/journald.conf << 'EOF'
[Journal]
Storage=persistent
SystemMaxUse=500M
EOF

sudo systemctl restart systemd-journald
journalctl --disk-usage
```

### Étape 2 — Configuration logrotate personnalisée

```bash
sudo mkdir -p /var/log/lab_app
sudo tee /var/log/lab_app/app.log << 'EOF'
2024-01-01 Ligne de log initiale de test
EOF

sudo tee /etc/logrotate.d/lab_app << 'EOF'
/var/log/lab_app/*.log {
    daily
    rotate 5
    compress
    missingok
    notifempty
    create 0640 root root
}
EOF

sudo logrotate --debug /etc/logrotate.d/lab_app
sudo logrotate --force /etc/logrotate.d/lab_app
ls -l /var/log/lab_app/
```
*Question :* combien de fichiers observez-vous après la rotation forcée ? Le fichier `app.log` d'origine a-t-il bien été recréé conformément à la directive `create` ?

### Étape 3 — Reproduire et corriger le piège cron

```bash
# Version buguée : commande sans chemin absolu, dépendant implicitement du PATH interactif
cat << 'EOF' > /tmp/script_buggy.sh
#!/bin/bash
echo "Exécution démarrée: $(date)"
hostname > /tmp/resultat_hostname.txt
echo "Résultat: $(cat /tmp/resultat_hostname.txt)"
EOF
chmod +x /tmp/script_buggy.sh

# Test manuel : fonctionne normalement
/tmp/script_buggy.sh

# Ajouter en cron avec un PATH volontairement réduit pour simuler le piège
(crontab -l 2>/dev/null; echo "* * * * * PATH=/nonexistent /tmp/script_buggy.sh >> /tmp/cron_output.log 2>&1") | crontab -

sleep 65
cat /tmp/cron_output.log
```
*Résultat attendu :* une erreur "command not found" (ou équivalent) pour `hostname`, illustrant concrètement le piège malgré un script qui fonctionnait parfaitement en test manuel.

Correction :
```bash
crontab -r   # nettoyer la tâche de démonstration

cat << 'EOF' > /tmp/script_corrige.sh
#!/bin/bash
echo "Exécution démarrée: $(date)"
/usr/bin/hostname > /tmp/resultat_hostname.txt
echo "Résultat: $(cat /tmp/resultat_hostname.txt)"
EOF
chmod +x /tmp/script_corrige.sh

(crontab -l 2>/dev/null; echo "* * * * * PATH=/nonexistent /tmp/script_corrige.sh >> /tmp/cron_output.log 2>&1") | crontab -
sleep 65
tail -5 /tmp/cron_output.log
crontab -r   # nettoyage final
```

### Étape 4 — Mécanisme de heartbeat

```bash
cat << 'EOF' > /tmp/tache_avec_heartbeat.sh
#!/bin/bash
# Simule le travail de la tâche
echo "Travail effectué à $(date)"
# Heartbeat uniquement en cas de succès complet
/usr/bin/touch /tmp/heartbeat_lab_task
EOF
chmod +x /tmp/tache_avec_heartbeat.sh

/tmp/tache_avec_heartbeat.sh
ls -l /tmp/heartbeat_lab_task

# Reprendre et adapter le script de la section 16 du cours pour vérifier ce heartbeat
```

## Commandes de référence
`journalctl --disk-usage`, `logrotate --debug`/`--force`, `crontab -e`/`-l`/`-r`

## Résultats attendus
- Confirmation de la persistance journald activée (étape 1)
- Fichiers de rotation générés par logrotate (étape 2)
- Reproduction de l'échec cron puis preuve de la correction (étape 3)
- Fichier heartbeat créé et vérifié (étape 4)

## Questions

1. Dans l'étape 3, pourquoi avons-nous dû définir artificiellement `PATH=/nonexistent` dans la ligne cron pour reproduire le piège, alors que le cours indique que cron utilise déjà un PATH minimal par défaut ? Que cela révèle-t-il sur la variabilité réelle de cet environnement selon les systèmes ?
2. Si l'étape 2 avait omis la directive `create`, quel aurait été l'état du fichier `/var/log/lab_app/app.log` immédiatement après la rotation, et quel risque cela ferait-il courir à une application qui continuerait d'écrire dans ce chemin sans jamais rouvrir explicitement son descripteur (rappel UE1-M4) ?
3. Le mécanisme de heartbeat de l'étape 4 garantit-il que la tâche a réellement accompli **correctement** son travail, ou seulement qu'elle s'est exécutée jusqu'à son terme sans planter ? Cette nuance a-t-elle une importance pratique ?

## Troubleshooting (panne à diagnostiquer)

**Situation fournie :** après avoir mis en place le mécanisme de heartbeat de l'étape 4 sur une tâche réelle du système, un étudiant constate que le script de surveillance des heartbeats (section 16 du cours) déclenche une fausse alerte immédiatement après chaque redémarrage de la VM de laboratoire, avant même que la première exécution planifiée de la tâche surveillée n'ait eu lieu.

**Démarche attendue :**
1. Symptôme : fausse alerte systématique juste après redémarrage, avant la première exécution légitime attendue
2. Hypothèse : le fichier heartbeat, comme tout fichier sous `/tmp` ou `/var/run` selon son emplacement exact, peut être **supprimé au redémarrage** (comportement standard de certains répertoires temporaires, rappel indirect d'un piège similaire déjà rencontré en UE1 avec un état de script stocké dans `/tmp` non persistant) — le script de surveillance, exécuté juste après le redémarrage et avant la première occurrence planifiée de la tâche surveillée, ne trouve donc logiquement aucun fichier heartbeat récent, et déclenche une alerte qui n'est pas réellement le signe d'un problème mais un simple effet de bord du redémarrage
3. Vérification : consulter la documentation ou le comportement du répertoire utilisé pour stocker le fichier heartbeat (`/tmp` est généralement nettoyé à chaque redémarrage sur de nombreuses distributions, contrairement à `/var/log` ou un répertoire dédié comme `/var/lib/`)
4. Isolation : confirmer que l'alerte ne se produit que dans cette fenêtre spécifique juste après redémarrage, et disparaît d'elle-même une fois la première exécution légitime de la tâche survenue
5. Correction : déplacer l'emplacement du fichier heartbeat vers un répertoire réellement persistant (`/var/lib/heartbeats/` par exemple, à créer si nécessaire), et/ou ajouter une tolérance explicite dans le script de surveillance pour la période suivant immédiatement un redémarrage système (détectable via `uptime` ou l'heure de démarrage du système), évitant une fausse alerte pendant cette fenêtre légitimement transitoire
6. Vérification : redémarrer à nouveau la VM et confirmer l'absence de fausse alerte immédiate, avec une alerte correctement déclenchée uniquement si la tâche manque réellement son échéance après la fenêtre de tolérance
7. Documentation : noter cette leçon comme un rappel transversal supplémentaire (après le cas similaire déjà rencontré en UE1 pour un script utilisant `/tmp`) : tout mécanisme d'état devant survivre à un redémarrage doit être stocké dans un emplacement réellement persistant, jamais dans un répertoire temporaire par défaut sans avoir vérifié explicitement son comportement au redémarrage sur la distribution concernée

## Validation

Le TP est validé si l'étudiant fournit :
- Les 4 résultats/captures demandés
- Des réponses argumentées aux 3 questions
- Une identification correcte de la cause du scénario de troubleshooting (répertoire non persistant au redémarrage), avec une correction cohérente incluant à la fois le changement d'emplacement et une fenêtre de tolérance post-redémarrage

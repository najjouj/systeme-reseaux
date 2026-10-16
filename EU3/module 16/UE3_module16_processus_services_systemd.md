# UE3 — Module 16 : Processus, Services et systemd en Pratique

*Ce module applique en administration quotidienne la théorie de systemd déjà vue en UE1-Module 6 (boot process) — il ne reprend pas cette théorie mais la mobilise directement.*

---

# PARTIE I — COURS

## 1. Titre du cours
Administrer les services Linux au quotidien : unités systemd, journald et gestion des processus

## 2. Niveau
Élève ingénieur — cycle ingénieur, UE3 administration Linux (S2)

## 3. Durée
CM : 3h30 (module orienté pratique — +3h TD, +5h TP en Parties II et III)

## 4. Prérequis
- UE1-Module 6 : théorie complète de systemd (unités, targets, PID 1, `systemd-analyze`)
- UE1-Module 2 : processus, signaux, priorités
- UE3-Module 14 : utilisateurs et permissions (pour les comptes de service)

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Créer, activer et gérer une unité systemd personnalisée pour une application maison
- Configurer des dépendances, un redémarrage automatique et des limites de ressources sur un service
- Utiliser `journalctl` efficacement pour diagnostiquer un service en production
- Créer un timer systemd comme alternative moderne à une tâche planifiée simple
- Gérer les processus au quotidien (priorités, signaux, limitation de ressources) au-delà des commandes de base déjà vues
- Diagnostiquer un service qui ne démarre pas, plante en boucle, ou consomme des ressources excessives

## 6. Motivation

Déployer une application maison en production ne se limite pas à copier un exécutable sur un serveur et à le lancer manuellement dans un terminal — cette approche ne survit pas à un redémarrage, ne redémarre pas automatiquement en cas de plantage, et ne s'intègre à aucun mécanisme de supervision. La bonne pratique consiste à envelopper toute application de production dans une **unité systemd**, bénéficiant ainsi de tous les mécanismes déjà étudiés en théorie (UE1-M6) : démarrage automatique, gestion des dépendances, journalisation centralisée, redémarrage automatique en cas d'échec.

Ce module transforme la théorie systemd en compétence opérationnelle directe : écrire et déployer ses propres unités, diagnostiquer un service qui ne se comporte pas comme attendu, exploiter `journalctl` comme un véritable outil d'investigation plutôt qu'un simple visualiseur de logs — des gestes quotidiens de l'administrateur système.

## 7. Concepts fondamentaux

### 7.1 Anatomie d'un fichier d'unité de service

```ini
[Unit]
Description=Application métier de traitement de commandes
After=network-online.target postgresql.service
Wants=network-online.target
Requires=postgresql.service

[Service]
Type=simple
User=appuser
Group=appuser
WorkingDirectory=/opt/monapp
ExecStart=/opt/monapp/bin/serveur --config /etc/monapp/config.yml
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5
LimitNOFILE=65536
MemoryMax=512M

[Install]
WantedBy=multi-user.target
```

**Directives clés de la section `[Unit]` (rappel appliqué de UE1-M6) :**
- `After=` : ordre de démarrage relatif (ne garantit pas que l'unité citée soit pleinement opérationnelle, rappel du piège identifié en UE1-M6)
- `Requires=` : dépendance forte — si l'unité requise échoue, celle-ci échoue également
- `Wants=` : dépendance faible — un échec de l'unité voulue n'empêche pas le démarrage de celle-ci

**Directives clés de la section `[Service]` :**
- `Type=` : modèle de démarrage du processus — `simple` (le processus principal ne se détache pas, cas le plus courant pour une application moderne), `forking` (le processus se détache en arrière-plan lui-même, modèle plus ancien), `oneshot` (déjà rencontré en UE1-M6, pour une tâche qui se termine)
- `Restart=` : politique de redémarrage automatique — `on-failure` (seulement en cas d'échec, pas sur un arrêt volontaire), `always`, `no`
- `LimitNOFILE=` : nombre maximal de descripteurs de fichiers ouverts (rappel UE1-M4 sur les descripteurs) — une application réseau à fort trafic peut nécessiter d'augmenter cette limite au-delà de la valeur par défaut du système
- `MemoryMax=` : limite de mémoire imposée via les cgroups (mécanisme noyau sous-jacent à systemd, également utilisé par Docker — préfigurant l'UE9 conteneurisation)

### 7.2 Cycle de vie pratique d'une unité personnalisée

```bash
sudo cp monapp.service /etc/systemd/system/
sudo systemctl daemon-reload      # indispensable après toute modification d'un fichier d'unité
sudo systemctl enable monapp      # activation au démarrage (rappel UE1-M6 : distinct de start)
sudo systemctl start monapp       # démarrage immédiat
sudo systemctl status monapp      # vérification
```

**Erreur fréquente et rappel direct de l'UE1-M6 :** oublier `daemon-reload` après toute modification d'un fichier d'unité — systemd continue alors d'utiliser l'ancienne définition en mémoire, ignorant silencieusement les changements effectués sur le fichier jusqu'au prochain rechargement explicite.

### 7.3 `journalctl` — au-delà de la consultation simple

`journalctl` (rappel UE1-M6) ne se limite pas à afficher des logs bruts — c'est un véritable outil de filtrage et d'investigation :

```bash
journalctl -u monapp.service                    # logs d'une unité précise
journalctl -u monapp.service -f                   # suivi en temps réel (équivalent tail -f)
journalctl -u monapp.service --since "1 hour ago"   # filtrage temporel
journalctl -u monapp.service -p err                 # uniquement les messages de niveau erreur ou plus grave
journalctl -u monapp.service -b                      # logs du démarrage courant uniquement (rappel UE1-M6)
journalctl --disk-usage                                # espace disque occupé par le journal
```

**Niveaux de sévérité (syslog standard, du plus critique au moins critique) :** `emerg`, `alert`, `crit`, `err`, `warning`, `notice`, `info`, `debug` — filtrer par niveau (`-p`) permet d'isoler rapidement l'essentiel dans un journal volumineux, un réflexe de diagnostic bien plus efficace qu'un défilement manuel.

### 7.4 Redémarrage automatique et boucles de crash

La directive `Restart=on-failure` fait redémarrer automatiquement un service qui plante — un mécanisme précieux pour la résilience, mais qui peut masquer un problème sous-jacent non résolu : un service qui plante puis redémarre en boucle toutes les quelques secondes ("crash loop") consomme des ressources et génère un volume de logs important sans jamais réellement fonctionner. systemd propose des garde-fous :

- `RestartSec=` : délai entre chaque tentative de redémarrage (évite un redémarrage instantané en rafale)
- `StartLimitIntervalSec=` et `StartLimitBurst=` : nombre maximal de démarrages autorisés dans une fenêtre de temps donnée, au-delà duquel systemd cesse de retenter et place l'unité en échec définitif, nécessitant une intervention manuelle (`systemctl reset-failed` après correction) — un mécanisme qui transforme un problème silencieux et invisible (boucle infinie de tentatives) en un signal explicite et visible (échec définitif nécessitant une action humaine).

### 7.5 Timers systemd — alternative moderne à une tâche planifiée simple

systemd propose un mécanisme de planification intégré, les **timers**, alternative à `cron` (approfondi comme tel dans un module ultérieur de l'UE3) pour les tâches qui bénéficient de l'écosystème systemd (journalisation intégrée, dépendances, gestion d'échec) :

```ini
# monapp-backup.timer
[Unit]
Description=Déclenche la sauvegarde quotidienne de monapp

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

```ini
# monapp-backup.service (associé, sans section [Install], jamais démarré directement par un utilisateur)
[Unit]
Description=Sauvegarde de monapp

[Service]
Type=oneshot
ExecStart=/opt/monapp/scripts/backup.sh
```

**`Persistent=true` — un détail important :** si la machine était éteinte au moment où le timer aurait dû se déclencher, cette option garantit que la tâche s'exécutera au prochain démarrage, plutôt que d'être simplement ignorée jusqu'à la prochaine occurrence planifiée — un comportement particulièrement pertinent pour une tâche de sauvegarde quotidienne sur un poste qui n'est pas nécessairement allumé en permanence.

### 7.6 Gestion pratique des processus — au-delà de `ps`/`top`

**Priorités (rappel UE1-M2, application pratique) :**
```bash
nice -n 10 ./tache_longue.sh          # lancer avec une priorité basse dès le départ
renice -n 5 -p 1234                     # ajuster la priorité d'un processus déjà en cours
```

**Signaux (rappel UE1-M2, application pratique) :**
```bash
kill -TERM 1234    # demande d'arrêt propre (SIGTERM, interceptable, à toujours préférer en premier)
kill -HUP 1234       # souvent utilisé pour demander un rechargement de configuration sans redémarrage complet
kill -9 1234           # SIGKILL, dernier recours, non interceptable
```

**Limitation de ressources par utilisateur (`ulimit`, à distinguer des limites systemd de la section 7.1) :**
```bash
ulimit -n 4096   # nombre maximal de descripteurs de fichiers pour la session shell courante
```

## 8. Architecture

```
┌────────────────────────────────────────────────────────────┐
│  /etc/systemd/system/monapp.service                             │
│         │                                                         │
│         ▼                                                         │
│  systemd (PID 1) ── daemon-reload ── charge/actualise la définition │
│         │                                                         │
│         ▼                                                         │
│  Processus monapp (User=appuser, cgroup avec MemoryMax appliqué)     │
│         │                                                         │
│         ▼                                                         │
│  journald ── centralise stdout/stderr + métadonnées (unité, PID...)  │
│         │                                                         │
│         ▼                                                         │
│  journalctl -u monapp   ── interrogation filtrée                     │
└────────────────────────────────────────────────────────────┘
```

## 9. Fonctionnement interne

Chaque service géré par systemd s'exécute dans son propre **cgroup** (control group, mécanisme noyau), ce qui permet à systemd d'appliquer des limites de ressources (`MemoryMax`, `CPUQuota`...) de façon fiable et de **suivre l'ensemble des processus enfants** d'un service, même ceux créés indirectement (un service qui lance lui-même des sous-processus) — contrairement à un simple suivi par PID unique, facilement trompé si le processus principal se détache ou crée des enfants non rattachés explicitement. C'est ce mécanisme de cgroup qui garantit que `systemctl stop` termine réellement **tous** les processus appartenant à un service, pas seulement son processus principal — un comportement bien plus fiable qu'un simple `kill` manuel sur un PID unique, qui pourrait laisser des processus enfants orphelins actifs.

## 10. Démonstration

```bash
# Observer le cgroup d'un service actif
systemctl status sshd | grep -A2 CGroup

# Observer les limites de ressources appliquées à un service
systemctl show sshd -p MemoryMax -p LimitNOFILE

# Provoquer et observer une boucle de crash contrôlée (démonstration prudente)
```

## 11. Commandes / outils

| Commande | Rôle | Exemple |
|---|---|---|
| `systemctl daemon-reload` | Recharger les définitions d'unités après modification | après édition d'un `.service` |
| `systemctl edit <unit>` | Éditer une surcharge (override) sans modifier le fichier d'origine | `sudo systemctl edit monapp` |
| `systemctl reset-failed` | Réinitialiser l'état d'échec d'une unité bloquée | `sudo systemctl reset-failed monapp` |
| `journalctl -u <unit> -p err` | Filtrer les logs par sévérité | `journalctl -u monapp -p err` |
| `systemctl list-timers` | Lister les timers actifs et leur prochaine exécution | `systemctl list-timers` |
| `systemd-cgtop` | Vue en temps réel de la consommation de ressources par cgroup | `systemd-cgtop` |

**Erreur fréquente :** modifier directement un fichier d'unité fourni par un paquet système (sous `/usr/lib/systemd/system/`) plutôt que d'utiliser `systemctl edit` (qui crée un fichier de surcharge séparé, sous `/etc/systemd/system/<unit>.d/`) — une modification directe du fichier d'origine sera **écrasée** à la prochaine mise à jour du paquet concerné, perdant silencieusement toute personnalisation.

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** une application métier, déployée via une unité systemd avec `Restart=always` mais sans `StartLimitBurst` configuré, commence à planter systématiquement après une mise à jour de configuration erronée. Le service redémarre en boucle toutes les secondes, générant plusieurs milliers d'entrées de journal par minute, jusqu'à ce que l'espace disque alloué au journal (`journald`) atteigne sa limite configurée, provoquant la suppression automatique des entrées les plus anciennes — y compris potentiellement des informations de diagnostic antérieures à l'incident, utiles pour comprendre le contexte de la panne.

**Analyse attendue :** ce scénario illustre concrètement pourquoi les garde-fous de la section 7.4 (`StartLimitIntervalSec`, `StartLimitBurst`) ne sont pas de simples options facultatives mais une protection essentielle : sans eux, une boucle de crash peut se transformer en incident secondaire (saturation du journal, consommation CPU excessive par les tentatives de redémarrage répétées) qui aggrave la situation initiale et complique le diagnostic a posteriori. Correction immédiate : `systemctl stop` le service pour interrompre la boucle, corriger la configuration erronée, ajouter les limites de tentatives de redémarrage manquantes à l'unité, puis `systemctl reset-failed` avant de relancer. Mesure préventive : systématiquement inclure `StartLimitIntervalSec`/`StartLimitBurst` dans toute unité personnalisée utilisant `Restart=always` ou `on-failure`, jamais laisser un redémarrage automatique sans limite de fréquence.

## 15. Sécurité

- Un service exécuté sous un compte dédié (`User=appuser`, rappel UE3-M14 sur les comptes de service sans shell interactif) limite fortement l'impact d'une compromission de l'application — sans cette précaution, une faille applicative exploitée donnerait directement les privilèges du compte sous lequel le service tourne (root, si mal configuré)
- systemd propose des directives de durcissement supplémentaires pour une unité (`ProtectSystem=strict`, `NoNewPrivileges=true`, `PrivateTmp=true`...), restreignant ce qu'un service peut voir ou modifier sur le système, même s'il est compromis — approfondi en UE8 sécurité, mais à connaître dès ce module comme bonne pratique de conception d'unité
- Les limites de ressources (`MemoryMax`, `CPUQuota`) constituent aussi une mesure de résilience face à un service compromis ou bogué qui tenterait de consommer des ressources de façon excessive (déni de service local involontaire ou provoqué)
- Un `journalctl` accessible en lecture à des utilisateurs non privilégiés peut exposer des informations sensibles si des services y journalisent par erreur des données confidentielles (mots de passe, jetons) — une vigilance de conception applicative à rappeler, pas seulement une question de configuration système

## 16. Automatisation

Script Bash de détection de services en état d'échec ou en boucle de crash, avec résumé des logs récents pour accélérer le diagnostic.

```bash
#!/usr/bin/env bash
# Détecte les services en échec et résume leurs logs récents pour diagnostic rapide
set -euo pipefail

LOGFILE="/var/log/service_health_watch.log"

SERVICES_EN_ECHEC=$(systemctl list-units --type=service --state=failed --no-legend | awk '{print $1}')

if [[ -n "$SERVICES_EN_ECHEC" ]]; then
    echo "$(date '+%F %T') - ALERTE: services en échec détectés" | tee -a "$LOGFILE"
    while read -r service; do
        echo "--- $service ---" | tee -a "$LOGFILE"
        journalctl -u "$service" -n 5 --no-pager | tee -a "$LOGFILE"
    done <<< "$SERVICES_EN_ECHEC"
    exit 1
fi

echo "$(date '+%F %T') - OK: aucun service en échec" >> "$LOGFILE"
exit 0
```

## 17. Troubleshooting

Scénario court : après avoir modifié un fichier d'unité pour ajouter une nouvelle variable d'environnement, un administrateur exécute `systemctl restart` mais l'application continue de se comporter comme si la modification n'avait pas été prise en compte. Démarche : vérifier si `daemon-reload` a bien été exécuté après la modification du fichier (rappel section 7.2 — piège très fréquent) → si oui, vérifier avec `systemctl cat <unit>` que le contenu effectivement chargé par systemd correspond bien au fichier modifié (en cas de fichier de surcharge additionnel via `systemctl edit`, une variable définie dans le fichier principal peut être masquée ou complétée de façon inattendue par la surcharge) → vérifier enfin, avec `systemctl show <unit> -p Environment`, la valeur réellement effective de la variable telle que systemd la transmettra au processus.

## 18. Synthèse

- Une unité systemd personnalisée structure `[Unit]` (dépendances), `[Service]` (comportement d'exécution), `[Install]` (activation) — `daemon-reload` est indispensable après toute modification
- `journalctl` filtré par unité, sévérité et fenêtre temporelle est un outil de diagnostic bien plus efficace qu'un défilement manuel de logs bruts
- `Restart=` sans garde-fous (`StartLimitBurst`) peut transformer un problème visible en incident silencieux de boucle de crash, aggravant la situation
- Les timers systemd offrent une alternative à `cron` avec intégration native à l'écosystème systemd (dépendances, `Persistent=true` pour rattraper une exécution manquée)
- `systemctl edit` (surcharge) est préférable à une modification directe d'un fichier d'unité fourni par un paquet, pour survivre aux mises à jour
- Les cgroups sous-jacents à systemd garantissent un suivi et un arrêt fiable de l'ensemble des processus d'un service, y compris ses enfants indirects

## 19. Questions d'évaluation

**QCM**
1. Après modification d'un fichier d'unité systemd, la commande indispensable avant tout redémarrage du service est :
 a) `systemctl restart` uniquement b) `systemctl daemon-reload` c) `systemctl enable` d) aucune, la modification est automatiquement prise en compte

2. `StartLimitBurst` sert à :
 a) accélérer le démarrage d'un service b) limiter le nombre de tentatives de redémarrage automatique dans une fenêtre de temps, évitant une boucle de crash silencieuse c) augmenter la mémoire allouée d) désactiver totalement le redémarrage automatique

**Questions ouvertes**
- Expliquez pourquoi `systemctl edit` est préférable à une modification directe d'un fichier d'unité fourni par un paquet système.
- Expliquez comment les cgroups permettent à systemd d'arrêter fiablement l'ensemble des processus d'un service, y compris ses enfants indirects.

**Exercice technique**
- Donnez la commande permettant de filtrer les logs d'une unité `monapp.service` pour n'afficher que les messages de sévérité erreur ou plus grave.

**Problème de diagnostic**
- Un service redémarre en boucle toutes les quelques secondes, générant un volume de logs très important. Proposez une action immédiate et une correction de configuration pérenne.

## 20. Mini-projet

Développer une unité systemd complète et son script associé pour une application fictive de traitement de fichiers, incluant :
1. Un fichier `.service` avec `User=` dédié, `Restart=on-failure` avec garde-fous (`StartLimitIntervalSec`/`StartLimitBurst`), et au moins une directive de durcissement (`NoNewPrivileges=true` ou équivalent)
2. Un timer associé déclenchant une tâche de nettoyage quotidienne, avec `Persistent=true`
3. Un script de surveillance (repris et étendu de la section 16 du cours) intégrant une détection spécifique de boucle de crash (nombre de redémarrages sur une fenêtre glissante), au-delà de la simple détection d'état "failed"
4. Une documentation expliquant chaque choix de configuration et son lien avec les bonnes pratiques vues en cours

---

# PARTIE II — TD

### Niveau 1 — Compréhension

**Exercice 1.1**
Décrivez les trois sections d'un fichier d'unité systemd de type service (`[Unit]`, `[Service]`, `[Install]`) et le rôle général de chacune.

**Exercice 1.2**
Pourquoi `systemctl daemon-reload` est-il nécessaire après toute modification d'un fichier d'unité ?

**Exercice 1.3**
Expliquez la différence entre `Requires=` et `Wants=` dans la section `[Unit]` d'un fichier de service.

### Niveau 2 — Application

**Exercice 2.1**
Donnez la commande permettant de suivre en temps réel les logs d'une unité `monapp.service`.

**Exercice 2.2**
Donnez la commande permettant de lister tous les timers systemd actifs et leur prochaine date d'exécution.

**Exercice 2.3**
Un service est bloqué en état `failed` après avoir dépassé sa limite de tentatives de redémarrage (`StartLimitBurst`). Après avoir corrigé la cause du problème, quelle commande faut-il exécuter avant de pouvoir le redémarrer normalement ?

### Niveau 3 — Analyse

**Exercice 3.1**
Un administrateur configure `Restart=always` sur un service sans définir `StartLimitIntervalSec` ni `StartLimitBurst`. Le service commence à planter en boucle suite à une erreur de configuration. Décrivez les conséquences probables de cette absence de garde-fou, au-delà du simple fait que le service ne fonctionne pas.

**Exercice 3.2**
Un administrateur modifie directement le fichier `/usr/lib/systemd/system/nginx.service` fourni par le paquet du système pour y ajouter une option personnalisée, plutôt que d'utiliser `systemctl edit`. Quelques semaines plus tard, une mise à jour système est appliquée. Que risque-t-il de se passer concernant cette personnalisation, et quelle aurait été la bonne pratique ?

### Niveau 4 — Problème complexe

**Exercice 4.1**
Une équipe déploie une nouvelle version d'une application via une unité systemd avec `Restart=on-failure`, `RestartSec=2`, et sans limite de tentatives configurée. La nouvelle version contient un bug qui provoque un plantage systématique 3 secondes après chaque démarrage. Une semaine plus tard, l'équipe d'exploitation remarque que l'espace disque du serveur est anormalement bas, et que `journalctl --disk-usage` révèle un journal systemd occupant une taille disproportionnée. En vous appuyant sur les mécanismes vus en cours, expliquez la chaîne complète d'événements ayant mené à cette situation, depuis le déploiement du bug jusqu'à la saturation observée, et proposez à la fois une correction immédiate et une modification de configuration qui aurait limité l'ampleur de cet incident secondaire, sans empêcher pour autant la détection du bug initial.

---

## Correction du TD

**1.1** — `[Unit]` : décrit les métadonnées et les dépendances de l'unité (description, ordre de démarrage relatif via `After=`, dépendances fortes/faibles via `Requires=`/`Wants=`). `[Service]` : décrit le comportement d'exécution du service lui-même (commande à lancer, type de démarrage, utilisateur d'exécution, politique de redémarrage, limites de ressources). `[Install]` : décrit comment l'unité s'intègre au système d'activation au démarrage (généralement `WantedBy=` une target précise), utilisée uniquement lors de `systemctl enable`/`disable`.

**1.2** — `daemon-reload` indique à systemd de relire l'ensemble des fichiers d'unités depuis le disque et de reconstruire son graphe de dépendances interne. Sans cette commande, systemd continue d'utiliser la définition qu'il avait chargée en mémoire lors de son dernier chargement, ignorant silencieusement toute modification apportée au fichier sur le disque — un simple `systemctl restart` relance le service avec l'**ancienne** définition toujours en mémoire, pas la nouvelle version du fichier.

**1.3** — `Requires=` exprime une dépendance **forte** : si l'unité requise échoue à démarrer, l'unité qui la requiert échoue également. `Wants=` exprime une dépendance **faible** : systemd tente de démarrer l'unité voulue en parallèle, mais un échec de cette dernière n'empêche pas le démarrage de l'unité qui l'a déclarée comme voulue — une distinction importante pour éviter qu'une dépendance non strictement indispensable ne bloque tout un service en cas d'indisponibilité temporaire et non critique de cette dépendance.

**2.1** — `journalctl -u monapp.service -f`.

**2.2** — `systemctl list-timers`.

**2.3** — `systemctl reset-failed <nom_du_service>` (réinitialise l'état d'échec de l'unité, lui permettant d'être redémarrée normalement — sans cette commande, systemd continue de considérer l'unité comme définitivement en échec après avoir dépassé sa limite de tentatives).

**3.1** — Sans `StartLimitIntervalSec`/`StartLimitBurst`, le service tentera de redémarrer **indéfiniment**, à chaque échec, avec pour seul délai celui défini par `RestartSec=` (ou la valeur par défaut si non spécifiée). Conséquences probables : une consommation CPU significative due aux tentatives de démarrage répétées (chaque tentative implique une initialisation du processus, potentiellement coûteuse), un volume de journalisation très important (chaque tentative et son échec génèrent des entrées de log), pouvant à terme saturer l'espace disque alloué au journal systemd et provoquer la suppression d'entrées de log plus anciennes potentiellement utiles au diagnostic — le problème initial (le service qui plante) reste in fine invisible et non résolu pendant que cet incident secondaire s'aggrave silencieusement, contrairement à un service qui se bloquerait proprement en état `failed` après un nombre limité de tentatives, signalant clairement un problème nécessitant une intervention humaine.

**3.2** — La personnalisation ajoutée directement dans le fichier fourni par le paquet système (`/usr/lib/systemd/system/nginx.service`) risque d'être **écrasée** lors de la prochaine mise à jour du paquet `nginx`, celle-ci réinstallant généralement le fichier d'unité d'origine tel que fourni par le mainteneur du paquet, sans préserver les modifications locales apportées directement dans ce fichier. La bonne pratique aurait été d'utiliser `systemctl edit nginx`, qui crée un fichier de **surcharge** séparé (sous `/etc/systemd/system/nginx.service.d/override.conf`), appliqué par systemd par-dessus le fichier d'origine sans jamais le modifier directement — une surcharge qui survit intacte à toute mise à jour ultérieure du paquet système, puisqu'elle réside dans un emplacement totalement distinct de celui géré par le gestionnaire de paquets.

**4.1** — Chaîne d'événements : (1) la nouvelle version buguée est déployée, le service démarre puis plante systématiquement 3 secondes après chaque démarrage ; (2) `Restart=on-failure` avec `RestartSec=2` déclenche un nouveau redémarrage 2 secondes après chaque plantage, sans aucune limite de nombre de tentatives configurée ; (3) ce cycle se répète indéfiniment (plantage après 3s, attente de 2s, redémarrage, nouveau plantage après 3s...), soit environ une tentative toutes les 5 secondes, en continu, potentiellement pendant toute la semaine écoulée avant que l'équipe ne remarque le problème ; (4) chaque cycle de démarrage/plantage génère plusieurs entrées de journal (démarrage, message d'erreur du plantage, notification de redémarrage par systemd), qui s'accumulent au fil de milliers de cycles répétés sur une semaine entière, jusqu'à occuper une portion significative et disproportionnée de l'espace disque alloué au journal, comme révélé par `journalctl --disk-usage`. Correction immédiate : `systemctl stop` le service pour interrompre la boucle, corriger ou retirer la version buguée (retour à la version précédente stable, le temps de corriger le bug). Modification de configuration préventive : ajouter `StartLimitIntervalSec=60` et `StartLimitBurst=5` (par exemple, valeurs à ajuster selon le contexte) à la section `[Unit]` — après 5 tentatives de redémarrage échouées en 60 secondes, systemd cesserait ses tentatives et placerait l'unité en échec définitif, un signal visible et immédiat nécessitant une intervention humaine, au lieu d'une boucle silencieuse s'étalant sur plusieurs jours sans jamais être remarquée avant que l'incident secondaire (saturation disque du journal) ne devienne lui-même visible.

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur système junior pour déployer une application maison en tant que service systemd correctement configuré, incluant redémarrage automatique sécurisé, timer de maintenance, et pour diagnostiquer une boucle de crash provoquée volontairement.

## Architecture

Une VM ou un conteneur avec systemd fonctionnel comme PID 1 réel (rappel de la contrainte identifiée en UE1-M6 — un conteneur Docker standard ne convient généralement pas sans configuration spécifique).

## Objectifs
- Créer et déployer une unité de service personnalisée complète
- Utiliser `journalctl` pour diagnostiquer un comportement inattendu
- Créer un timer systemd avec `Persistent=true`
- Provoquer et corriger une boucle de crash, en observant l'effet des garde-fous

## Préparation

```bash
which systemctl journalctl 2>/dev/null
systemctl --version
```

## Étapes

### Étape 1 — Créer et déployer un service applicatif simple

```bash
sudo mkdir -p /opt/lab_app
sudo tee /opt/lab_app/serveur.sh << 'EOF'
#!/bin/bash
echo "Application démarrée, PID $$"
while true; do
    echo "Heartbeat: $(date)"
    sleep 5
done
EOF
sudo chmod +x /opt/lab_app/serveur.sh

sudo useradd -r -s /usr/sbin/nologin lab_app_user 2>/dev/null || true
sudo chown -R lab_app_user:lab_app_user /opt/lab_app

sudo tee /etc/systemd/system/lab-app.service << 'EOF'
[Unit]
Description=Application de laboratoire TP
After=network.target

[Service]
Type=simple
User=lab_app_user
ExecStart=/opt/lab_app/serveur.sh
Restart=on-failure
RestartSec=3
StartLimitIntervalSec=60
StartLimitBurst=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now lab-app.service
sudo systemctl status lab-app.service
```

### Étape 2 — Diagnostiquer avec journalctl

```bash
sudo journalctl -u lab-app.service -f &
sleep 15
kill %1   # arrêter le suivi en temps réel

sudo journalctl -u lab-app.service --since "2 minutes ago"
sudo journalctl -u lab-app.service -p info -n 5
```
*Question :* le compte `lab_app_user` créé avec `/usr/sbin/nologin` est-il cohérent avec les bonnes pratiques vues au module 14 pour un compte de service ? Justifiez.

### Étape 3 — Timer systemd

```bash
sudo tee /opt/lab_app/nettoyage.sh << 'EOF'
#!/bin/bash
echo "Nettoyage exécuté le $(date)" >> /var/log/lab_app_nettoyage.log
EOF
sudo chmod +x /opt/lab_app/nettoyage.sh

sudo tee /etc/systemd/system/lab-app-nettoyage.service << 'EOF'
[Unit]
Description=Nettoyage périodique de lab-app

[Service]
Type=oneshot
ExecStart=/opt/lab_app/nettoyage.sh
EOF

sudo tee /etc/systemd/system/lab-app-nettoyage.timer << 'EOF'
[Unit]
Description=Déclenche le nettoyage de lab-app toutes les 2 minutes (accéléré pour le TP)

[Timer]
OnUnitActiveSec=2min
Persistent=true

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now lab-app-nettoyage.timer
systemctl list-timers | grep lab-app
```
*Note :* l'intervalle est volontairement raccourci (2 minutes au lieu d'une planification quotidienne réaliste) pour permettre l'observation en séance de TP.

### Étape 4 — Provoquer et corriger une boucle de crash

```bash
# Remplacer le script par une version buguée qui plante immédiatement
sudo tee /opt/lab_app/serveur.sh << 'EOF'
#!/bin/bash
echo "Erreur fatale simulée"
exit 1
EOF

sudo systemctl restart lab-app.service
sleep 20
sudo systemctl status lab-app.service   # observer l'état "failed" après épuisement des tentatives

sudo journalctl -u lab-app.service --since "1 minute ago" | tail -20
```
*Résultat attendu :* après avoir atteint `StartLimitBurst` (5 tentatives en 60 secondes), le service passe en état `failed` définitif et cesse de retenter, conformément au garde-fou configuré à l'étape 1.

Correction :
```bash
# Restaurer le script fonctionnel de l'étape 1
sudo tee /opt/lab_app/serveur.sh << 'EOF'
#!/bin/bash
echo "Application démarrée, PID $$"
while true; do
    echo "Heartbeat: $(date)"
    sleep 5
done
EOF
sudo chmod +x /opt/lab_app/serveur.sh

sudo systemctl reset-failed lab-app.service
sudo systemctl start lab-app.service
sudo systemctl status lab-app.service
```

## Commandes de référence
`systemctl daemon-reload`, `systemctl enable --now`, `journalctl -u -f`, `journalctl -p`, `systemctl list-timers`, `systemctl reset-failed`

## Résultats attendus
- Service `lab-app` actif et fonctionnel (étape 1)
- Extraits de logs filtrés par temps et par sévérité (étape 2)
- Timer actif visible dans `systemctl list-timers` (étape 3)
- Preuve de l'état `failed` après épuisement des tentatives, puis correction réussie (étape 4)

## Questions

1. Dans l'étape 4, si `StartLimitBurst` n'avait pas été configuré à l'étape 1, quel aurait été le comportement observé à la place de l'état `failed` propre obtenu ? Reliez votre réponse à l'exercice 3.1 du TD.
2. Pourquoi le script de l'étape 1 utilise-t-il une boucle infinie avec `sleep 5` plutôt qu'un simple `echo` suivi d'une sortie immédiate ? Quel `Type=` de service serait approprié si l'application se comportait comme un script qui se termine après une tâche unique plutôt que de rester active en continu ?
3. Si vous deviez ajouter une directive de durcissement de sécurité simple à l'unité de l'étape 1 (rappel section 15 du cours), laquelle proposeriez-vous en premier, et pourquoi ?

## Troubleshooting (panne à diagnostiquer)

**Situation fournie :** après la correction de l'étape 4, un étudiant modifie à nouveau le fichier `/etc/systemd/system/lab-app.service` pour ajuster `RestartSec` de 3 à 10 secondes, exécute `systemctl restart lab-app.service`, mais observe (via `journalctl -u lab-app.service`) que le délai entre les tentatives de redémarrage reste toujours de 3 secondes lors d'un test ultérieur de plantage.

**Démarche attendue :**
1. Symptôme : une modification de configuration semble sans effet, comportement identique à l'ancienne configuration
2. Hypothèse : oubli de `systemctl daemon-reload` après la modification du fichier (rappel direct section 7.2 du cours et exercice 1.2/3.2 du TD) — `systemctl restart` relance le service mais avec la définition **toujours en mémoire** depuis le dernier chargement, ignorant la modification non rechargée
3. Vérification : `systemctl cat lab-app.service` pour comparer le contenu effectivement chargé par systemd avec le contenu réel du fichier sur le disque (`cat /etc/systemd/system/lab-app.service`) — une différence entre les deux confirme l'hypothèse
4. Isolation : confirmer qu'aucun `daemon-reload` n'a été exécuté entre la modification du fichier et le test observé
5. Correction : exécuter `sudo systemctl daemon-reload` puis `sudo systemctl restart lab-app.service`, et refaire le test de plantage pour confirmer le nouveau délai de 10 secondes
6. Vérification : `journalctl -u lab-app.service` confirme désormais le délai attendu entre les tentatives
7. Documentation : noter cette erreur comme rappel du piège le plus fréquent et le plus systématiquement rencontré en administration systemd au quotidien — `daemon-reload` après **toute** modification de fichier d'unité, sans exception, un réflexe à ancrer définitivement

## Validation

Le TP est validé si l'étudiant fournit :
- Les 4 résultats/captures demandés
- Des réponses argumentées aux 3 questions
- Une identification correcte de la cause du scénario de troubleshooting (oubli de `daemon-reload`), cohérente avec la leçon déjà présente dans le cours et le TD

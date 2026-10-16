# UE1 — Module 6 : Boot Process

---

# PARTIE I — COURS

## 1. Titre du cours
Le processus de démarrage : du firmware à l'espace utilisateur

## 2. Niveau
Élève ingénieur — cycle ingénieur, tronc commun systèmes (S1)

## 3. Durée
CM : 4h (module complet UE1-M6 : +3h TD, +4h TP en Parties II et III)

## 4. Prérequis
- UE1-M1 : architecture générale d'un OS, kernel vs user space
- UE1-M2 : processus, PID, arborescence parent/enfant (PID 1)
- UE1-M4 : systèmes de fichiers, montage (le boot doit monter la racine avant tout)

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Décrire les étapes chronologiques du démarrage, du firmware jusqu'au premier processus utilisateur
- Différencier BIOS et UEFI, et expliquer pourquoi UEFI a largement remplacé le BIOS historique
- Expliquer le rôle du bootloader (GRUB) et de l'initramfs
- Expliquer le rôle de systemd comme PID 1, ses unités et ses cibles (targets)
- Diagnostiquer un problème de démarrage en identifiant à quelle étape il se situe
- Utiliser les outils Linux (`systemctl`, `journalctl -b`, `systemd-analyze`) pour analyser et accélérer le démarrage

## 6. Motivation

Un serveur qui ne démarre plus après une mise à jour du noyau. Une machine qui reste bloquée sur un écran noir. Un service critique qui ne se lance pas automatiquement après un redémarrage planifié, provoquant une panne à 3h du matin. Dans tous ces cas, l'ingénieur système doit savoir *exactement* à quelle étape du démarrage la panne se situe pour intervenir efficacement — chercher une erreur applicative alors que le noyau lui-même ne charge pas est une perte de temps coûteuse.

Le démarrage d'un système Linux est une séquence précise et prévisible d'étapes, chacune ayant ses propres outils de diagnostic. Ce module donne la carte complète de cette séquence, indispensable pour tout ingénieur qui administre des serveurs en production.

## 7. Concepts fondamentaux

### 7.1 Vue d'ensemble de la séquence de démarrage

```
Mise sous tension
      │
      ▼
Firmware (BIOS ou UEFI)  — auto-test matériel (POST), sélection du périphérique de boot
      │
      ▼
Bootloader (GRUB)  — charge le noyau et l'initramfs en mémoire
      │
      ▼
Noyau Linux  — initialise le matériel, monte la racine réelle
      │
      ▼
Init / systemd (PID 1)  — démarre les services, atteint la cible de démarrage
      │
      ▼
Espace utilisateur prêt (login, services actifs)
```

### 7.2 Firmware : BIOS vs UEFI

Le **firmware** est le premier code exécuté à la mise sous tension, stocké sur une puce de la carte mère (indépendant de tout système d'exploitation installé). Il réalise le **POST** (Power-On Self-Test — vérification matérielle de base), puis localise et lance le bootloader.

- **BIOS (Basic Input/Output System)** : firmware historique, mode 16 bits, utilise le **MBR** (Master Boot Record, 512 premiers octets du disque) pour localiser le bootloader. Limité à des disques de 2 To et 4 partitions primaires.
- **UEFI (Unified Extensible Firmware Interface)** : firmware moderne, mode 32/64 bits, utilise une partition dédiée (**ESP — EFI System Partition**, formatée en FAT32) contenant des fichiers exécutables `.efi`. Supporte des disques de grande taille (via le partitionnement **GPT**), un démarrage plus rapide, et le **Secure Boot** (vérification cryptographique de la chaîne de démarrage).

**Pourquoi UEFI a-t-il remplacé le BIOS ?** Le BIOS était limité par la taille du MBR (512 octets, dont seulement 446 utilisables pour le code de boot lui-même) et son mode d'exécution 16 bits archaïque. UEFI apporte une architecture bien plus flexible (fichiers exécutables standards plutôt que code assembleur contraint), une meilleure sécurité (Secure Boot) et la compatibilité avec les grands disques modernes.

### 7.3 Le bootloader (GRUB)

Le **bootloader** (GRUB2 — GRand Unified Bootloader, le plus répandu sous Linux) a pour rôle de localiser, charger en mémoire, et transférer l'exécution vers le noyau Linux. Il permet aussi de :
- Présenter un menu de sélection (choix entre plusieurs noyaux installés, mode de secours)
- Passer des **paramètres de démarrage au noyau** (kernel parameters), par exemple pour désigner le système de fichiers racine, activer un mode debug, ou démarrer en mode single-user (maintenance)
- Charger l'**initramfs** (image mémoire temporaire contenant les pilotes et outils minimaux nécessaires *avant* que le système de fichiers racine réel ne soit accessible — utile par exemple si la racine est sur un disque nécessitant un pilote spécifique, ou chiffrée)

### 7.4 Initialisation du noyau

Une fois chargé, le noyau Linux :
1. Décompresse son image et s'initialise en mémoire
2. Détecte et initialise le matériel disponible (via ses pilotes intégrés ou modules chargés depuis l'initramfs)
3. Monte le système de fichiers racine temporaire (initramfs), exécute les scripts nécessaires (ex: activation de LVM, déchiffrement de disque)
4. Bascule ensuite (`pivot_root` ou équivalent) vers le système de fichiers racine **réel** défini par les paramètres de boot
5. Lance le tout premier processus en espace utilisateur : **PID 1** (systemd sur la quasi-totalité des distributions modernes)

**Point clé pédagogique :** l'initramfs existe précisément parce que le noyau a parfois besoin de pilotes ou de logique complexe (LVM, RAID logiciel, chiffrement disque) *avant même* de pouvoir accéder au système de fichiers racine définitif — c'est un problème d'œuf et de poule que l'initramfs résout en fournissant un environnement minimal intermédiaire.

### 7.5 systemd et PID 1

**systemd** est devenu le système d'init de référence sur la majorité des distributions Linux modernes (remplaçant l'ancien SysVinit et ses runlevels numérotés). En tant que **PID 1**, systemd a un rôle particulier : c'est l'ancêtre ultime de tous les autres processus, et c'est lui qui **adopte les processus orphelins** (rappel M2 : un processus zombie dont le parent meurt est adopté par PID 1, qui appelle alors `wait()` pour le nettoyer).

systemd organise le démarrage autour d'**unités** (units) :
- **service** : un démon ou processus à démarrer/arrêter/superviser
- **target** : un regroupement logique d'unités représentant un état du système (équivalent moderne et plus flexible des anciens runlevels), ex : `multi-user.target` (mode serveur, sans interface graphique) ou `graphical.target`
- **mount** : un point de montage géré par systemd
- **socket**, **timer**, **path** : autres types d'unités pour du démarrage à la demande ou planifié

**Différence fondamentale avec l'ancien SysVinit :** SysVinit démarrait les services **séquentiellement**, dans un ordre strict défini par des scripts numérotés. systemd démarre les unités **en parallèle** dès que leurs dépendances déclarées sont satisfaites, ce qui accélère considérablement le démarrage — au prix d'une gestion des dépendances plus explicite et plus complexe à maîtriser (directives `After=`, `Before=`, `Requires=`, `Wants=` dans les fichiers d'unité).

### 7.6 Cibles (targets) systemd courantes

| Target | Rôle | Équivalent SysVinit approximatif |
|---|---|---|
| `poweroff.target` | Arrêt du système | runlevel 0 |
| `rescue.target` | Mode de secours, un seul utilisateur | runlevel 1 |
| `multi-user.target` | Mode multi-utilisateur, sans interface graphique | runlevel 3 |
| `graphical.target` | Mode multi-utilisateur avec interface graphique | runlevel 5 |
| `reboot.target` | Redémarrage | runlevel 6 |

La cible par défaut est configurée via un lien symbolique (`/etc/systemd/system/default.target`), modifiable avec `systemctl set-default`.

## 8. Architecture

```
┌───────────────────────────────────────────────────────┐
│  Firmware (UEFI)                                          │
│  - POST                                                    │
│  - lit l'ESP, exécute le bootloader (.efi)                 │
└───────────────────────┬───────────────────────────────┘
                         ▼
┌───────────────────────────────────────────────────────┐
│  GRUB2                                                     │
│  - menu de sélection                                        │
│  - charge noyau + initramfs en mémoire                      │
│  - passe les paramètres de boot                             │
└───────────────────────┬───────────────────────────────┘
                         ▼
┌───────────────────────────────────────────────────────┐
│  Noyau Linux                                                │
│  - initialise le matériel                                    │
│  - monte l'initramfs, exécute sa logique (LVM, chiffrement)  │
│  - pivot_root vers la racine réelle                           │
│  - lance PID 1                                                │
└───────────────────────┬───────────────────────────────┘
                         ▼
┌───────────────────────────────────────────────────────┐
│  systemd (PID 1)                                            │
│  - résout les dépendances entre unités                       │
│  - démarre les services en parallèle                          │
│  - atteint la target par défaut (ex: multi-user.target)       │
└───────────────────────────────────────────────────────┘
```

## 9. Fonctionnement interne

Le passage de l'initramfs au système de fichiers racine réel s'effectue via un mécanisme appelé **pivot_root** (ou `switch_root` dans les initramfs modernes basés sur `dracut`/`initramfs-tools`), qui change atomiquement la racine du système de fichiers vue par le noyau, démonte l'ancienne racine temporaire (initramfs), et transfère l'exécution vers le véritable init (systemd) sur le système de fichiers définitif.

systemd construit un **graphe de dépendances** entre toutes les unités à activer pour atteindre la target par défaut, puis démarre en parallèle toutes les unités dont les dépendances sont déjà satisfaites, au lieu d'une exécution strictement séquentielle — c'est ce parallélisme contrôlé qui explique le gain de temps de démarrage par rapport à SysVinit, mais aussi la complexité accrue du diagnostic quand plusieurs services démarrent simultanément et qu'un ordre implicite était supposé par erreur par un administrateur.

## 10. Démonstration

```bash
# Voir le temps total de démarrage et sa décomposition
systemd-analyze

# Voir les unités classées par temps de démarrage
systemd-analyze blame

# Voir le graphe critique de dépendances (chaîne la plus longue)
systemd-analyze critical-chain

# Consulter les logs du dernier démarrage
journalctl -b

# Voir la target actuelle et la target par défaut
systemctl get-default
```

## 11. Commandes / outils

| Commande | Rôle | Exemple |
|---|---|---|
| `systemctl status <unit>` | État détaillé d'une unité | `systemctl status sshd` |
| `systemctl list-units --type=service` | Lister les services actifs | `systemctl list-units --type=service` |
| `systemctl enable/disable <unit>` | Activer/désactiver au démarrage | `systemctl enable nginx` |
| `systemctl get-default` / `set-default` | Consulter/changer la target par défaut | `systemctl set-default multi-user.target` |
| `journalctl -b` | Logs du démarrage courant | `journalctl -b -1` (démarrage précédent) |
| `systemd-analyze` | Temps de démarrage global | `systemd-analyze time` |
| `systemd-analyze blame` | Unités les plus lentes au démarrage | `systemd-analyze blame` |
| `systemd-analyze critical-chain` | Chaîne critique de dépendances | `systemd-analyze critical-chain` |

**Erreur fréquente :** confondre `systemctl stop` (arrêt immédiat, non permanent) et `systemctl disable` (empêche le démarrage automatique au prochain boot, sans arrêter le service en cours) — un service arrêté avec `stop` seul redémarrera au prochain reboot s'il est toujours `enabled`.

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** après une mise à jour du noyau sur un serveur de production, celui-ci ne redémarre pas correctement — la console affiche un message indiquant l'impossibilité de monter le système de fichiers racine, puis un shell de secours (initramfs).

**Analyse attendue :** ce symptôme situe précisément le problème **entre le chargement de l'initramfs et le montage de la racine réelle** (avant que systemd/PID 1 ne soit même atteint) — donc *avant* toute cause applicative. Causes fréquentes : l'initramfs généré pour le nouveau noyau ne contient pas les pilotes nécessaires (ex : contrôleur RAID/stockage spécifique), ou une erreur dans les paramètres de boot passés par GRUB (UUID de partition racine incorrect). Démarche : depuis le shell de secours initramfs, vérifier la présence du périphérique racine attendu (`ls /dev`), régénérer l'initramfs pour le noyau concerné (`update-initramfs` ou `dracut`), ou démarrer temporairement sur l'ancien noyau (toujours disponible dans le menu GRUB) en attendant la correction.

## 15. Sécurité

- **Secure Boot (UEFI)** vérifie cryptographiquement la signature du bootloader et du noyau avant exécution, empêchant le chargement d'un bootloader ou noyau modifié malveillamment (protection contre certains rootkits de bas niveau)
- Un accès physique à la console GRUB non protégé par mot de passe permet à un attaquant de modifier les paramètres de boot (ex : ajouter `init=/bin/bash` pour obtenir un accès root sans authentification) — d'où l'importance de protéger GRUB par mot de passe sur les systèmes sensibles
- Le mode de secours (`rescue.target`, ou single-user mode) doit lui-même être protégé (demande de mot de passe root) sur les systèmes en production, sous peine de contourner toute authentification via un simple accès physique/console
- Les unités systemd personnalisées doivent être auditées : un attaquant avec un accès suffisant pourrait créer une unité malveillante démarrée automatiquement à chaque boot (mécanisme de persistance classique)

## 16. Automatisation

Script Bash de vérification post-démarrage : confirme que les services critiques sont bien actifs après un reboot, et alerte sinon.

```bash
#!/usr/bin/env bash
# Vérifie l'état des services critiques après démarrage et journalise les anomalies
set -euo pipefail

SERVICES_CRITIQUES=("sshd" "nginx" "cron")
LOGFILE="/var/log/boot_check.log"
ECHEC=0

echo "$(date '+%F %T') - Vérification post-démarrage" >> "$LOGFILE"

for service in "${SERVICES_CRITIQUES[@]}"; do
    if systemctl is-active --quiet "$service"; then
        echo "  OK: $service actif" >> "$LOGFILE"
    else
        echo "  ALERTE: $service INACTIF" | tee -a "$LOGFILE"
        ECHEC=1
    fi
done

if [[ $ECHEC -eq 1 ]]; then
    echo "$(date '+%F %T') - Anomalies détectées au démarrage" >> "$LOGFILE"
    exit 1
fi

echo "$(date '+%F %T') - Tous les services critiques sont actifs" >> "$LOGFILE"
exit 0
```

*Usage typique :* déclenché via une unité systemd de type `oneshot` avec `WantedBy=multi-user.target`, pour s'exécuter automatiquement à chaque démarrage.

## 17. Troubleshooting

Scénario court : après un redémarrage planifié, un service métier critique n'a pas démarré automatiquement, sans qu'aucune panne matérielle ne soit visible. Démarche : `systemctl status <service>` pour voir l'état et la dernière erreur → vérifier si le service est bien `enabled` (`systemctl is-enabled <service>`) → consulter `journalctl -u <service> -b` pour les logs spécifiques au démarrage courant → vérifier les dépendances déclarées dans l'unité (`systemctl cat <service>`, directives `After=`/`Requires=`) qui pourraient ne pas être satisfaites à temps → corriger (activation manquante, dépendance mal déclarée, erreur de configuration) → tester avec un redémarrage contrôlé.

## 18. Synthèse

- Le démarrage suit une séquence stricte : Firmware → Bootloader → Noyau → initramfs → racine réelle → PID 1 (systemd) → target
- UEFI a remplacé le BIOS pour sa flexibilité, sa sécurité (Secure Boot) et son support des grands disques (GPT)
- L'initramfs résout le problème de "l'œuf et la poule" : fournir les pilotes nécessaires avant l'accès à la racine réelle
- systemd démarre les unités en parallèle selon un graphe de dépendances, contrairement au séquencement strict de l'ancien SysVinit
- Identifier *à quelle étape* un problème de démarrage se situe (firmware, bootloader, noyau/initramfs, ou systemd) est le premier réflexe de diagnostic, avant toute investigation applicative
- `systemctl`, `journalctl -b` et `systemd-analyze` sont les trois outils centraux du diagnostic de démarrage sous Linux moderne

## 19. Questions d'évaluation

**QCM**
1. L'initramfs existe principalement pour :
 a) accélérer le CPU b) fournir les pilotes nécessaires avant que la racine réelle ne soit accessible c) remplacer systemd d) gérer les threads

2. Contrairement à SysVinit, systemd démarre les services :
 a) toujours séquentiellement b) en parallèle selon un graphe de dépendances c) uniquement au premier boot d) sans aucune dépendance

**Questions ouvertes**
- Expliquez la différence entre `systemctl stop` et `systemctl disable`, et le risque associé à une confusion entre les deux en production.
- Un serveur affiche un shell de secours initramfs au démarrage. À quelle étape précise de la séquence ce blocage se situe-t-il, et qu'est-ce que cela exclut comme cause possible ?

**Exercice technique**
- Donnez la commande permettant d'identifier les 5 unités systemd ayant le plus contribué au temps de démarrage.

**Problème de diagnostic**
- Après un redémarrage planifié, un service métier n'a pas démarré automatiquement. Donnez la séquence de 3 commandes permettant de diagnostiquer la cause, en précisant ce que chacune doit révéler.

## 20. Mini-projet

Développer un script (Bash ou Python), packagé comme unité systemd de type `oneshot` exécutée à chaque démarrage, qui :
1. Vérifie l'état d'une liste configurable de services critiques (`systemctl is-active`)
2. Vérifie que la target atteinte correspond bien à la target attendue (`systemctl get-default` vs état réel)
3. Consulte `systemd-analyze blame` et signale toute unité dépassant un seuil de temps de démarrage configurable
4. Journalise un rapport structuré et envoie une alerte si une anomalie est détectée
5. Fournit le fichier d'unité systemd (`.service`) nécessaire à son propre déploiement automatique

---

# PARTIE II — TD

### Niveau 1 — Compréhension

**Exercice 1.1**
Listez, dans l'ordre chronologique, les grandes étapes du démarrage d'un système Linux moderne, du démarrage de la machine jusqu'à un système utilisable.

**Exercice 1.2**
Expliquez pourquoi l'initramfs est nécessaire, en donnant un exemple concret de situation où son absence poserait problème.

**Exercice 1.3**
Qu'est-ce qu'une target systemd, et en quoi diffère-t-elle conceptuellement d'un runlevel SysVinit, au-delà de la simple différence de nommage ?

### Niveau 2 — Application

**Exercice 2.1**
Donnez la commande permettant de savoir si le service `sshd` démarrera automatiquement au prochain redémarrage, sans avoir besoin de redémarrer la machine pour le vérifier.

**Exercice 2.2**
Donnez la commande permettant de consulter uniquement les logs générés par le service `nginx` lors du démarrage courant du système.

**Exercice 2.3**
Donnez la commande permettant d'identifier quelle unité systemd a le plus retardé le démarrage global du système.

### Niveau 3 — Analyse

**Exercice 3.1**
Un administrateur exécute `systemctl stop nginx` puis redémarre la machine par erreur avant d'avoir terminé sa maintenance. Le service `nginx` redémarre-t-il automatiquement après le reboot ? Justifiez précisément en vous appuyant sur la différence entre l'état d'exécution courant d'une unité et sa configuration d'activation au démarrage.

**Exercice 3.2**
`systemd-analyze critical-chain` révèle qu'un service `backup.service` retarde l'atteinte de `multi-user.target` de 45 secondes, alors que ce service ne devrait logiquement s'exécuter qu'une fois par nuit et n'a aucune raison de bloquer le démarrage. Formulez une hypothèse sur la cause probable (indice : penser aux directives de dépendance `After=`/`Before=`/`Wants=`/`Requires=` évoquées en cours) et la commande permettant de l'examiner.

### Niveau 4 — Problème complexe

**Exercice 4.1**
Une entreprise migre son parc de serveurs vers des disques chiffrés (LUKS) pour renforcer la sécurité des données au repos. Après la migration, plusieurs serveurs redémarrent normalement jusqu'à l'écran de saisie de la phrase de déchiffrement — mais ce processus manuel est incompatible avec des serveurs distants sans accès console physique facile. En vous appuyant sur les concepts de firmware, bootloader, initramfs et noyau vus en cours, expliquez à quelle étape précise ce blocage se situe-t-il nécessairement, pourquoi il ne peut pas se situer plus tard (ex : au niveau de systemd), et citez au moins une piste de solution technique standard (même si son détail d'implémentation dépasse ce module) permettant d'automatiser ou de déporter cette étape pour des serveurs distants.

---

## Correction du TD

**1.1** — Mise sous tension → Firmware (BIOS/UEFI, POST) → Bootloader (GRUB, charge noyau + initramfs) → Noyau (initialise le matériel, monte l'initramfs, exécute sa logique, puis pivot_root vers la racine réelle) → PID 1 / systemd (résout les dépendances, démarre les unités, atteint la target par défaut) → système utilisable.

**1.2** — L'initramfs fournit un environnement minimal (pilotes, outils) permettant au noyau d'accéder au système de fichiers racine réel, *avant* que celui-ci ne soit lui-même accessible. Exemple concret : si la partition racine est chiffrée (LUKS) ou répartie sur un ensemble LVM/RAID logiciel, le noyau a besoin d'exécuter une logique spécifique (déchiffrement, assemblage des volumes) *avant* de pouvoir monter la racine — logique qui réside précisément dans l'initramfs. Sans lui, le noyau serait bloqué, incapable d'accéder à la racine dont il a pourtant besoin pour continuer.

**1.3** — Une target regroupe logiquement un ensemble d'unités (services, montages, sockets...) représentant un état cohérent du système, avec des dépendances explicites entre elles, gérées par un graphe résolu et démarré en parallèle par systemd. Un runlevel SysVinit était un simple numéro associé à un répertoire de scripts exécutés le plus souvent séquentiellement, sans véritable modélisation de dépendances ni parallélisme — la différence n'est donc pas que de nommage : c'est un changement de modèle (graphe de dépendances explicite et démarrage parallèle vs séquence de scripts numérotés).

**2.1** — `systemctl is-enabled sshd` (retourne `enabled` ou `disabled`, indépendamment de l'état d'exécution courant du service).

**2.2** — `journalctl -u nginx -b` (logs de l'unité `nginx`, limités au démarrage courant du système).

**2.3** — `systemd-analyze blame` (liste les unités classées par temps de démarrage, la plus lente en tête).

**3.1** — Oui, `nginx` redémarre automatiquement après le reboot. `systemctl stop` n'agit que sur l'état d'exécution courant du service (l'arrête immédiatement), sans modifier sa configuration d'activation au démarrage (`enabled`/`disabled`). Si le service était `enabled` avant le `stop`, il le reste après, et systemd le redémarrera donc automatiquement au prochain boot, indépendamment du fait qu'il ait été arrêté manuellement juste avant le redémarrage. Pour empêcher ce redémarrage automatique, l'administrateur aurait dû utiliser `systemctl disable nginx` en complément (ou à la place) de `stop`.

**3.2** — Hypothèse probable : le fichier d'unité de `backup.service` déclare une dépendance forte (`Requires=` ou `Before=`/`After=` mal positionnée) qui le place, involontairement, dans la chaîne critique menant à `multi-user.target`, alors que sa nature (tâche planifiée nocturne) suggère qu'il devrait plutôt être déclenché par une unité `timer` indépendante du chemin de démarrage principal, sans bloquer l'atteinte de la target. Commande d'examen : `systemctl cat backup.service` (affiche le contenu du fichier d'unité et ses directives de dépendance).

**4.1** — Le blocage se situe nécessairement **avant que le noyau n'ait terminé son initialisation et son passage à la racine réelle, donc dans la phase initramfs** (ou éventuellement dès le bootloader si la phrase de déchiffrement était demandée par GRUB lui-même dans certaines configurations, mais typiquement c'est l'initramfs qui gère LUKS). Il ne peut **pas** se situer au niveau de systemd, car systemd (PID 1) n'est lancé qu'*après* que la racine réelle soit montée — or ici, la racine elle-même est chiffrée et ne peut être montée sans déchiffrement préalable ; systemd ne peut donc même pas encore exister à ce stade. Piste de solution standard : mettre en place un mécanisme de déchiffrement à distance de l'initramfs (par exemple un petit serveur SSH minimal embarqué dans l'initramfs, permettant à un administrateur de fournir la phrase de déchiffrement via une connexion réseau distante, sans accès console physique) — solution effectivement utilisée en production sur des flottes de serveurs distants avec disques chiffrés.

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur système pour auditer et fiabiliser le démarrage d'un serveur. L'équipe souhaite réduire le temps de démarrage et s'assurer que tous les services critiques redémarrent correctement après chaque reboot planifié.

## Architecture

**Important :** ce TP nécessite un environnement où **systemd fonctionne réellement comme PID 1** avec un vrai cycle de démarrage observable — une **VM légère** (ex : VM Vagrant/QEMU avec une distribution comme Debian ou Ubuntu Server) est fortement recommandée plutôt qu'un simple conteneur Docker, car un conteneur standard ne traverse pas les étapes firmware/bootloader/initramfs et systemd n'y tourne généralement pas comme PID 1 réel sans configuration spécifique (image systemd-enabled avec privilèges étendus).

*Si seule une infrastructure conteneurisée est disponible :* utiliser une image Docker "systemd-enabled" (ex: basée sur `jrei/systemd-ubuntu` ou équivalent, lancée avec `--privileged` et un point de montage cgroup approprié), en précisant explicitement aux étudiants que les étapes firmware/bootloader/initramfs ne seront pas observables dans ce mode — seul le comportement de systemd (targets, unités, `systemd-analyze`) sera manipulable.

## Objectifs
- Analyser le temps de démarrage réel d'une machine et identifier les unités les plus lentes
- Observer et manipuler les targets systemd
- Provoquer et diagnostiquer un service qui ne démarre pas automatiquement après reboot
- Déployer un script de vérification post-démarrage sous forme d'unité systemd

## Préparation

```bash
which systemctl journalctl systemd-analyze 2>/dev/null
systemctl --version
```

## Étapes

### Étape 1 — Analyse du temps de démarrage

```bash
systemd-analyze
systemd-analyze blame | head -10
systemd-analyze critical-chain
```
*Question :* quelle unité contribue le plus au temps de démarrage sur cette machine ? Est-elle sur la chaîne critique (`critical-chain`) ou simplement lente en parallèle sans bloquer les autres ?

### Étape 2 — Manipulation des targets

```bash
systemctl get-default
systemctl list-units --type=target
# NE PAS exécuter sur une machine de production réelle sans précaution :
# systemctl isolate multi-user.target   (bascule vers cette target sans reboot)
```
*Question :* quelle est la différence entre `systemctl isolate` (bascule immédiate) et `systemctl set-default` (change la target par défaut au prochain reboot, sans effet immédiat) ?

### Étape 3 — Provoquer et diagnostiquer un service qui ne redémarre pas

```bash
# Créer un faux service pour l'exercice
sudo tee /etc/systemd/system/lab-service.service << 'EOF'
[Unit]
Description=Service de laboratoire TP

[Service]
ExecStart=/bin/sleep infinity
Restart=no

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl start lab-service
sudo systemctl status lab-service
# Volontairement : ne pas faire "enable"
sudo systemctl is-enabled lab-service   # doit indiquer "disabled" ou une erreur
```
*Simuler un reboot logique (sans redémarrer réellement la machine) :*
```bash
sudo systemctl stop lab-service
sudo systemctl start lab-service   # un vrai reboot testerait le comportement "enabled" réel
```
*Question :* si vous redémarriez réellement cette machine maintenant, `lab-service` serait-il actif après le reboot ? Justifiez avec la commande utilisée pour le vérifier sans redémarrer.

Corrigez maintenant le problème :
```bash
sudo systemctl enable lab-service
sudo systemctl is-enabled lab-service   # doit indiquer "enabled"
```

### Étape 4 — Déployer le script de vérification post-démarrage

Reprendre le script `boot_check.sh` de la section 16 du cours, l'installer comme unité `oneshot` :

```bash
sudo cp boot_check.sh /usr/local/bin/boot_check.sh
sudo chmod +x /usr/local/bin/boot_check.sh

sudo tee /etc/systemd/system/boot-check.service << 'EOF'
[Unit]
Description=Vérification des services critiques au démarrage
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/boot_check.sh

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable boot-check.service
sudo systemctl start boot-check.service
sudo systemctl status boot-check.service
cat /var/log/boot_check.log
```

## Commandes de référence
`systemctl`, `journalctl -b`, `systemd-analyze`, `systemd-analyze blame`, `systemd-analyze critical-chain`

## Résultats attendus
- Capture de `systemd-analyze blame` (top 10)
- Réponse argumentée sur la chaîne critique (étape 1)
- Preuve (`systemctl is-enabled`) du service `lab-service` avant et après correction (étape 3)
- Capture de `systemctl status boot-check.service` et du contenu de `/var/log/boot_check.log` (étape 4)

## Questions

1. Pourquoi `systemctl stop` suivi immédiatement de `systemctl start` (étape 3) ne constitue-t-il pas un test valide de ce qui se passerait réellement après un reboot complet de la machine ?
2. Si `boot_check.sh` (étape 4) détecte une anomalie, quel type d'unité systemd (`socket`, `timer`, `path`...) pourrait compléter ce dispositif pour déclencher une alerte email en plus du simple log, sans modifier le script lui-même ?
3. Dans un environnement de production réel, quelle serait votre démarche pour tester en toute sécurité l'effet d'un changement de target par défaut, sans risquer de rendre un serveur distant inaccessible après un reboot malencontreux ?

## Troubleshooting (panne à diagnostiquer)

**Situation fournie :** après avoir ajouté l'unité `boot-check.service` (étape 4), un redémarrage réel de la VM de laboratoire montre que le service s'exécute *avant* que certains services applicatifs qu'il est censé vérifier (ex: un service web fictif `webapp.service` démarré à part) ne soient réellement actifs — provoquant de fausses alertes systématiques au démarrage.

**Démarche attendue :**
1. Symptôme : fausses alertes au démarrage alors que les services finissent par démarrer correctement quelques secondes plus tard
2. Hypothèse : la directive `After=multi-user.target` ne garantit pas que *tous* les services individuels de cette target sont déjà pleinement actifs — elle garantit seulement l'ordre relatif à la target elle-même, pas l'achèvement de chaque unité qui la compose
3. Vérification : `systemctl cat webapp.service` pour observer ses propres dépendances, et `journalctl -u boot-check -u webapp -b` pour comparer les horodatages réels de démarrage des deux unités
4. Isolation : confirmer que `boot-check.service` s'exécute avant la fin effective du démarrage de `webapp.service`
5. Correction : ajouter une dépendance explicite `After=webapp.service` (et éventuellement `Wants=webapp.service`) dans le fichier d'unité `boot-check.service`, plutôt que de se reposer uniquement sur `multi-user.target`
6. Vérification : relancer un reboot complet et confirmer l'absence de fausse alerte
7. Documentation : noter dans le changelog du script que la vérification post-démarrage doit toujours déclarer explicitement ses dépendances vis-à-vis des services qu'elle contrôle, et non se reposer implicitement sur une target générique

## Validation

Le TP est validé si l'étudiant fournit :
- Les 4 résultats/captures demandés
- Des réponses argumentées aux 3 questions
- Une identification correcte de la cause du scénario de troubleshooting (dépendance implicite insuffisante vs dépendance explicite nécessaire)

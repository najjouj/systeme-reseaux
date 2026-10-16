# UE3 — Module 15 : Filesystems, Mount, LVM et Stockage

---

# PARTIE I — COURS

## 1. Titre du cours
Gestion du stockage Linux : partitionnement, systèmes de fichiers et LVM

## 2. Niveau
Élève ingénieur — cycle ingénieur, UE3 administration Linux (S2)

## 3. Durée
CM : 4h (module dense — +3h TD, +5h TP en Parties II et III)

## 4. Prérequis
- UE1-Module 4 : systèmes de fichiers, inodes, VFS, montage (théorie)
- UE3-Module 14 : permissions (utiles pour la propriété des points de montage)

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Différencier partition, système de fichiers et point de montage
- Créer une partition, y appliquer un système de fichiers, et la monter durablement via `/etc/fstab`
- Expliquer l'architecture LVM (PV, VG, LV) et justifier son intérêt par rapport au partitionnement classique
- Étendre un volume logique LVM à chaud, sans interruption de service
- Configurer et gérer un espace de swap
- Diagnostiquer un problème de stockage (partition pleine, échec de montage, volume LVM corrompu) avec les outils Linux

## 6. Motivation

Un serveur de base de données voit son volume de données croître de façon imprévisible au fil des mois. Avec un partitionnement classique, agrandir cet espace signifie souvent redimensionner physiquement une partition — une opération risquée, parfois impossible sans interruption de service ou sans redémarrage. LVM (Logical Volume Manager) a été conçu précisément pour répondre à ce problème : découpler la notion de "disque physique" de celle "d'espace utilisable par le système de fichiers", permettant d'étendre, réduire, voire déplacer du stockage à chaud, sans jamais toucher au partitionnement physique sous-jacent une fois LVM en place.

Ce module donne les compétences pratiques indispensables pour dimensionner, faire évoluer et diagnostiquer le stockage d'un serveur Linux en production — une compétence quotidienne de l'administrateur système, directement liée à l'un des incidents les plus fréquents et les plus évitables : la saturation disque (déjà entrevue en UE1-M4, ici traitée du point de vue de la prévention et de la résolution structurelle).

## 7. Concepts fondamentaux

### 7.1 Partition, système de fichiers, point de montage — trois notions distinctes

- **Partition** : une subdivision d'un disque physique (ou d'un volume logique), définie par une table de partitionnement (MBR historique ou GPT moderne, rappel UE1-M6)
- **Système de fichiers (filesystem)** : la structure logique appliquée à une partition (ou un volume) pour y organiser des fichiers — ext4, XFS, Btrfs... (rappel UE1-M4 pour les concepts d'inode, superbloc)
- **Point de montage** : le répertoire de l'arborescence globale où le contenu d'une partition/volume devient accessible (rappel UE1-M4)

**Erreur fréquente chez les débutants :** confondre "créer une partition" et "y appliquer un système de fichiers" — ce sont deux opérations distinctes et successives (`fdisk`/`parted` pour la partition, `mkfs` pour le système de fichiers), une partition brute sans système de fichiers appliqué n'étant pas montable.

### 7.2 Outils de partitionnement

- **`fdisk`** : outil historique, interactif, adapté au MBR et au GPT (versions récentes)
- **`parted`** : outil plus moderne, supporte nativement GPT, utilisable en mode interactif ou scriptable
- **`lsblk`** : affiche l'arborescence des périphériques de bloc (disques, partitions, volumes LVM) de façon claire et hiérarchique — souvent le premier réflexe pour visualiser l'état du stockage d'une machine

### 7.3 Création et montage d'un système de fichiers

```bash
# Créer une partition (exemple simplifié, via parted en mode script)
sudo parted /dev/sdb mklabel gpt
sudo parted /dev/sdb mkpart primary ext4 0% 100%

# Appliquer un système de fichiers
sudo mkfs.ext4 /dev/sdb1

# Monter temporairement
sudo mkdir -p /mnt/donnees
sudo mount /dev/sdb1 /mnt/donnees
```

Un montage effectué directement avec `mount` est **temporaire** — il disparaît au redémarrage. Pour un montage **persistant**, l'entrée doit être déclarée dans `/etc/fstab` :

```
/dev/sdb1   /mnt/donnees   ext4   defaults   0   2
```

**Les 6 champs de `/etc/fstab` :** périphérique (idéalement identifié par UUID plutôt que par nom de device, voir 7.4), point de montage, type de système de fichiers, options de montage, indicateur de sauvegarde `dump` (obsolète en pratique, généralement `0`), ordre de vérification `fsck` au démarrage (`0` = pas de vérification, `1` = priorité pour la racine, `2` = autres partitions).

### 7.4 UUID plutôt que nom de périphérique

Le nom d'un périphérique (`/dev/sdb1`) n'est **pas garanti stable** d'un redémarrage à l'autre, notamment si l'ordre de détection des disques change (ajout/retrait de matériel, particulièrement fréquent en environnement virtualisé). Chaque système de fichiers porte un **UUID** unique, stable, généré à sa création — c'est pourquoi `/etc/fstab` doit référencer les systèmes de fichiers par leur UUID plutôt que par leur nom de périphérique, pour éviter qu'un changement d'ordre de détection ne monte accidentellement le mauvais disque au mauvais endroit après un redémarrage.

```bash
blkid   # affiche l'UUID de chaque système de fichiers détecté
```

```
UUID=1a2b3c4d-...   /mnt/donnees   ext4   defaults   0   2
```

### 7.5 LVM — architecture en trois couches

**LVM (Logical Volume Manager)** introduit une couche d'abstraction entre le stockage physique et les systèmes de fichiers, organisée en trois niveaux :

```
Disque physique (/dev/sdb)
        │
        ▼
  PV (Physical Volume)   ── un disque ou une partition, initialisé pour LVM
        │
        ▼
  VG (Volume Group)       ── regroupe un ou plusieurs PV en un pool d'espace unique
        │
        ▼
  LV (Logical Volume)     ── une "partition virtuelle" découpée dans le VG,
                              c'est sur un LV qu'on applique un système de fichiers
```

- **PV (Physical Volume)** : un disque entier ou une partition, "initialisé" pour être utilisable par LVM (`pvcreate`)
- **VG (Volume Group)** : un ou plusieurs PV regroupés en un **pool d'espace unique** (`vgcreate`) — l'espace total du VG est la somme de tous ses PV, indépendamment de leur répartition physique réelle sur un ou plusieurs disques
- **LV (Logical Volume)** : une subdivision du VG (`lvcreate`), comparable à une "partition virtuelle", sur laquelle on applique un système de fichiers classique (`mkfs`) exactement comme sur une partition physique traditionnelle

**Pourquoi cette abstraction est-elle si puissante ?** Parce qu'un LV n'est pas contraint par les limites physiques d'un disque unique — un VG peut regrouper plusieurs disques physiques, et un LV peut être **étendu** en puisant dans l'espace libre du VG, potentiellement réparti sur plusieurs disques physiques, sans jamais avoir à déplacer ou redimensionner une partition physique traditionnelle.

### 7.6 Extension d'un volume logique à chaud

L'un des principaux avantages opérationnels de LVM est la possibilité d'étendre un LV (et le système de fichiers qu'il contient) **sans interruption de service**, à condition que le VG dispose d'espace libre suffisant :

```bash
# Étendre le LV de 5 Go supplémentaires
sudo lvextend -L +5G /dev/vg_data/lv_donnees

# Étendre le système de fichiers pour utiliser le nouvel espace (ext4)
sudo resize2fs /dev/vg_data/lv_donnees

# Équivalent pour XFS (uniquement extension, jamais réduction, contrainte du FS)
sudo xfs_growfs /mnt/donnees
```

**Point clé pédagogique :** étendre le LV (`lvextend`) et étendre le système de fichiers pour occuper ce nouvel espace (`resize2fs`/`xfs_growfs`) sont deux opérations **distinctes et successives** — oublier la seconde étape est une erreur fréquente : le volume logique est bien agrandi au niveau LVM, mais le système de fichiers continue d'ignorer cet espace supplémentaire tant qu'il n'a pas été explicitement étendu à son tour.

**Asymétrie importante :** l'extension (grandir) est généralement simple et sûre pour la plupart des systèmes de fichiers modernes ; la **réduction** (rétrécir) est une opération plus risquée, pas supportée par tous les systèmes de fichiers (XFS, notamment, ne supporte **aucune réduction**, seulement l'extension), et nécessite généralement de démonter le système de fichiers au préalable — une asymétrie à connaître avant de dimensionner un LV, car revenir en arrière est structurellement plus difficile qu'avancer.

### 7.7 Snapshots LVM (aperçu)

LVM permet de créer des **snapshots** : une image figée d'un LV à un instant donné, utilisant un mécanisme de copie sur écriture (copy-on-write, rappel indirect du principe déjà vu pour la mémoire virtuelle en UE1-M3) — seuls les blocs modifiés après la création du snapshot consomment réellement de l'espace supplémentaire. Utile pour réaliser une sauvegarde cohérente d'un système en cours d'utilisation (base de données active, par exemple) sans interrompre le service, ou pour disposer d'un point de restauration rapide avant une opération risquée (mise à jour majeure, par exemple).

### 7.8 Espace de swap

Le **swap** (rappel UE1-M3, mémoire virtuelle et pression mémoire) peut être configuré soit comme une **partition dédiée**, soit comme un **fichier d'échange** (swapfile), ce dernier offrant davantage de flexibilité (redimensionnable sans repartitionnement) au prix d'une performance légèrement inférieure dans certains contextes.

```bash
# Créer et activer un fichier de swap de 2 Go
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

swapon --show   # vérifier l'espace de swap actif
```

## 8. Architecture

```
┌───────────────────────────────────────────────────────┐
│  Disque physique /dev/sdb        Disque physique /dev/sdc  │
│         │                              │                    │
│    PV /dev/sdb1                   PV /dev/sdc1               │
│         └──────────────┬──────────────┘                    │
│                          ▼                                   │
│              VG "vg_data" (pool unique)                        │
│         ┌────────────────┼────────────────┐                  │
│         ▼                 ▼                 ▼                  │
│   LV lv_donnees      LV lv_logs        LV lv_swap                │
│   (ext4, /data)      (xfs, /var/log)   (swap)                    │
└───────────────────────────────────────────────────────┘
```

## 9. Fonctionnement interne

Le noyau Linux gère les périphériques de bloc via un sous-système dédié (block layer), sur lequel LVM s'appuie via le **device mapper**, un mécanisme noyau permettant de créer des périphériques de bloc virtuels (les LV) construits par-dessus des périphériques physiques réels — chaque LV apparaît au reste du système comme un périphérique de bloc ordinaire (`/dev/vg_data/lv_donnees` ou `/dev/mapper/vg_data-lv_donnees`), sur lequel les opérations habituelles (`mkfs`, `mount`, `resize2fs`) s'appliquent sans traitement particulier, l'indirection LVM restant transparente pour ces couches supérieures — un exemple direct du principe d'abstraction en couches déjà rencontré à plusieurs reprises dans le programme (modèle OSI en UE2, VFS en UE1).

## 10. Démonstration

```bash
# Vue d'ensemble du stockage
lsblk
df -h
sudo pvs   # physical volumes
sudo vgs   # volume groups
sudo lvs   # logical volumes

# Détail d'un volume logique
sudo lvdisplay /dev/vg_data/lv_donnees
```

## 11. Commandes / outils

| Commande | Rôle | Exemple |
|---|---|---|
| `lsblk` | Vue hiérarchique des périphériques de bloc | `lsblk -f` (avec systèmes de fichiers) |
| `df -h` | Espace disque par système de fichiers monté | `df -h` |
| `blkid` | Afficher l'UUID de chaque système de fichiers | `sudo blkid` |
| `mkfs.ext4` / `mkfs.xfs` | Créer un système de fichiers | `sudo mkfs.ext4 /dev/sdb1` |
| `mount` / `umount` | Monter/démonter temporairement | `sudo mount /dev/sdb1 /mnt/data` |
| `pvcreate` / `vgcreate` / `lvcreate` | Créer PV / VG / LV | `sudo lvcreate -L 10G -n lv_data vg_data` |
| `lvextend` | Étendre un LV | `sudo lvextend -L +5G /dev/vg_data/lv_data` |
| `resize2fs` / `xfs_growfs` | Étendre le système de fichiers après extension du LV | `sudo resize2fs /dev/vg_data/lv_data` |
| `fsck` | Vérifier/réparer un système de fichiers (à froid, démonté) | `sudo fsck /dev/sdb1` |

**Erreur fréquente :** exécuter `fsck` sur un système de fichiers **monté** (particulièrement la partition racine) — une pratique dangereuse pouvant provoquer une corruption de données, `fsck` devant être exécuté sur un système de fichiers démonté (ou, pour la racine, depuis un environnement de secours/live, rappel UE1-M6 sur le mode de secours initramfs).

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** un serveur de base de données en production voit son espace disque `/var/lib/mysql` approcher dangereusement de la saturation. L'équipe avait anticipé ce risque en installant le système sur LVM dès le départ, avec un disque supplémentaire déjà physiquement attaché mais non encore utilisé.

**Analyse attendue :** grâce à l'architecture LVM déjà en place, la résolution ne nécessite ni interruption de service ni redémarrage : `pvcreate` sur le nouveau disque, `vgextend` pour l'ajouter au VG existant, `lvextend` pour agrandir le LV hébergeant `/var/lib/mysql`, puis `resize2fs`/`xfs_growfs` selon le système de fichiers utilisé pour que l'espace supplémentaire devienne effectivement utilisable — l'ensemble de l'opération s'effectue à chaud, le service de base de données restant actif et disponible pendant toute la procédure. Ce scénario illustre concrètement la valeur de la décision d'architecture prise en amont (installation sur LVM dès le départ, rappel section 7.1 du cours UE3-M14 sur les choix structurants d'installation) — une décision qui, sans coût significatif au moment de l'installation initiale, évite une interruption de service potentiellement coûteuse des mois ou années plus tard.

## 15. Sécurité

- Un espace disque saturé n'est pas qu'un problème de disponibilité : de nombreux mécanismes de sécurité (journalisation, `auditd`, logs d'authentification) dépendent d'un espace disque disponible pour fonctionner — un disque plein peut **silencieusement désactiver la traçabilité** d'un système au moment même où elle serait la plus nécessaire (par exemple pendant une tentative d'intrusion générant un volume de logs inhabituel)
- Le chiffrement de volume (LUKS, déjà mentionné en UE1-M6 à propos du démarrage) s'intègre naturellement avec LVM : un PV peut être créé par-dessus un volume LUKS déchiffré, combinant flexibilité de gestion du stockage et protection des données au repos
- Les snapshots LVM (7.7), bien qu'utiles pour la sauvegarde, doivent eux-mêmes être protégés par des permissions adéquates : un snapshot d'un volume contenant des données sensibles reproduit intégralement ces données sensibles, avec les mêmes exigences de protection que l'original

## 16. Automatisation

Script Bash de surveillance de l'espace disque avec alerte avant saturation critique, distinguant les systèmes de fichiers LVM extensibles de ceux qui ne le sont pas.

```bash
#!/usr/bin/env bash
# Surveille l'espace disque et alerte avant saturation, avec indication d'extensibilité LVM
set -euo pipefail

SEUIL_ALERTE=85
LOGFILE="/var/log/disk_space_watch.log"

df -h --output=target,pcent,source | tail -n +2 | while read -r point pourcentage source; do
    valeur=${pourcentage%\%}
    if (( valeur >= SEUIL_ALERTE )); then
        if [[ "$source" == /dev/mapper/* ]] || [[ "$source" == /dev/*vg*/* ]]; then
            extensible="OUI (volume LVM)"
        else
            extensible="NON (partition classique, nécessite un repartitionnement)"
        fi
        echo "$(date '+%F %T') - ALERTE: $point à ${valeur}% - Extensible via LVM: $extensible" \
            | tee -a "$LOGFILE"
    fi
done
```

## 17. Troubleshooting

Scénario court : après extension d'un volume logique et exécution de `resize2fs`, `df -h` continue d'afficher l'ancienne taille du système de fichiers, sans aucune erreur signalée lors des commandes précédentes. Démarche : vérifier avec `lvdisplay` que le LV a bien été étendu au niveau LVM lui-même (confirmer que l'étape `lvextend` a réellement réussi) → vérifier que la commande de redimensionnement du système de fichiers correspond bien au bon type (`resize2fs` pour ext4, `xfs_growfs` pour XFS — utiliser la mauvaise commande pour le mauvais type de système de fichiers échoue ou n'a aucun effet) → vérifier qu'aucune erreur silencieuse n'a été ignorée dans la sortie de la commande de redimensionnement (toujours vérifier le code de retour, pas seulement l'absence de message visible à l'écran) → si le problème persiste, vérifier qu'il ne s'agit pas d'un montage `bind` ou d'un chemin différent de celui réellement redimensionné.

## 18. Synthèse

- Partition, système de fichiers et point de montage sont trois notions distinctes et successives dans la mise en service d'un espace de stockage
- `/etc/fstab` doit référencer les systèmes de fichiers par UUID plutôt que par nom de périphérique, pour éviter tout risque lié à un changement d'ordre de détection au redémarrage
- LVM introduit trois couches (PV, VG, LV) découplant le stockage logique du matériel physique, permettant une extension à chaud sans interruption de service
- Étendre un LV (`lvextend`) et étendre le système de fichiers qu'il contient (`resize2fs`/`xfs_growfs`) sont deux opérations distinctes et successives, la seconde étant fréquemment oubliée
- La réduction d'un volume est structurellement plus risquée et pas toujours possible (XFS ne la supporte jamais) — une asymétrie à anticiper dès le dimensionnement initial
- Les snapshots LVM permettent des sauvegardes cohérentes à chaud via copie sur écriture
- Une saturation disque peut avoir des conséquences de sécurité indirectes en désactivant silencieusement la journalisation

## 19. Questions d'évaluation

**QCM**
1. Un PV, un VG et un LV, dans l'ordre logique de l'architecture LVM, forment :
 a) LV → VG → PV b) PV → VG → LV c) VG → LV → PV d) aucun ordre particulier

2. `/etc/fstab` doit idéalement référencer un système de fichiers par :
 a) son nom de périphérique (`/dev/sdb1`) b) son UUID, stable dans le temps c) sa taille en octets d) son numéro d'inode racine

**Questions ouvertes**
- Expliquez pourquoi étendre un LV avec `lvextend` ne suffit pas à rendre l'espace supplémentaire utilisable par le système de fichiers.
- Expliquez pourquoi XFS ne supporte aucune réduction de taille, contrairement à ext4, et quelle précaution cela implique pour le dimensionnement initial d'un volume XFS.

**Exercice technique**
- Donnez la commande permettant d'afficher l'UUID de tous les systèmes de fichiers détectés sur un système Linux.

**Problème de diagnostic**
- Un système de fichiers monté sur un LV, après extension confirmée du LV via `lvdisplay`, n'affiche toujours pas d'espace supplémentaire dans `df -h`. Proposez la cause la plus probable et la commande de correction.

## 20. Mini-projet

Développer un script (Bash ou Python) qui :
1. Surveille l'espace disque de l'ensemble des systèmes de fichiers montés (reprise et extension du script de la section 16)
2. Pour chaque système de fichiers proche de la saturation et identifié comme volume LVM, vérifie l'espace libre disponible dans le VG parent (`vgs`) pour évaluer si une extension à chaud est immédiatement possible
3. Si une extension est possible, journalise une recommandation précise (commandes `lvextend`/`resize2fs` suggérées, sans exécution automatique)
4. Si aucune extension n'est possible (VG sans espace libre suffisant), journalise une alerte de priorité plus élevée nécessitant une intervention matérielle (ajout de disque)
5. Génère un rapport hebdomadaire de synthèse sur l'état et les marges de manœuvre du stockage de l'ensemble du système

---

# PARTIE II — TD

### Niveau 1 — Compréhension

**Exercice 1.1**
Expliquez la différence entre une partition, un système de fichiers et un point de montage, avec l'ordre des opérations nécessaires pour rendre un nouveau disque utilisable.

**Exercice 1.2**
Pourquoi est-il recommandé d'utiliser l'UUID plutôt que le nom de périphérique (`/dev/sdb1`) dans `/etc/fstab` ?

**Exercice 1.3**
Décrivez les trois couches de l'architecture LVM (PV, VG, LV) et le rôle de chacune.

### Niveau 2 — Application

**Exercice 2.1**
Donnez la commande permettant d'afficher, pour chaque système de fichiers monté, son espace utilisé et disponible dans un format lisible par un humain.

**Exercice 2.2**
Donnez la séquence de deux commandes nécessaires pour étendre de 10 Go un volume logique `/dev/vg_data/lv_app` formaté en ext4.

**Exercice 2.3**
Donnez la commande permettant d'afficher l'ensemble des volume groups présents sur un système.

### Niveau 3 — Analyse

**Exercice 3.1**
Un administrateur exécute `lvextend -L +20G /dev/vg_data/lv_app` avec succès (confirmé par `lvdisplay`), mais oublie l'étape suivante. Une semaine plus tard, un développeur signale que l'espace disque de l'application semble toujours limité, malgré l'extension annoncée. Expliquez précisément ce qui s'est passé et la commande manquante.

**Exercice 3.2**
Une équipe envisage de réduire la taille d'un volume logique XFS pour libérer de l'espace au profit d'un autre volume du même VG. Est-ce réalisable directement ? Si non, proposez une méthode alternative permettant d'atteindre cet objectif malgré la contrainte du système de fichiers.

### Niveau 4 — Problème complexe

**Exercice 4.1**
Un serveur de production utilise LVM avec un VG unique regroupant deux disques physiques. Le volume logique hébergeant les données applicatives atteint 95% d'occupation, et l'équipe constate que le VG ne dispose plus d'aucun espace libre (`vgs` affiche 0 dans la colonne d'espace libre). Aucun nouveau disque physique n'est immédiatement disponible pour une extension. En vous appuyant sur les mécanismes vus en cours (y compris ceux évoqués en survol, comme les snapshots), proposez une démarche de gestion de crise à court terme (avant qu'un nouveau disque ne puisse être physiquement ajouté), en identifiant au moins une source d'espace potentiellement récupérable au sein de l'infrastructure LVM existante, et en justifiant pourquoi cette solution n'est que temporaire.

---

## Correction du TD

**1.1** — Une partition est une subdivision d'un disque physique définie par une table de partitionnement. Un système de fichiers est la structure logique appliquée à cette partition (ou à un volume) pour y organiser des fichiers. Un point de montage est le répertoire de l'arborescence globale où le contenu devient accessible. L'ordre nécessaire est : (1) créer la partition (`fdisk`/`parted`), (2) y appliquer un système de fichiers (`mkfs`), (3) la monter sur un point de montage (`mount`, puis déclaration persistante dans `/etc/fstab`).

**1.2** — Le nom de périphérique (`/dev/sdb1`) n'est pas garanti stable d'un redémarrage à l'autre, notamment si l'ordre de détection des disques change (ajout/retrait de matériel, fréquent en environnement virtualisé). L'UUID, généré à la création du système de fichiers, reste stable dans le temps indépendamment de l'ordre de détection — l'utiliser dans `/etc/fstab` évite qu'un changement d'ordre ne monte accidentellement le mauvais disque au mauvais emplacement après un redémarrage.

**1.3** — PV (Physical Volume) : un disque ou une partition initialisé pour être utilisable par LVM. VG (Volume Group) : un ou plusieurs PV regroupés en un pool d'espace unique, dont la taille totale est la somme de tous les PV qui le composent, indépendamment de leur répartition physique. LV (Logical Volume) : une subdivision du VG, comparable à une partition virtuelle, sur laquelle on applique un système de fichiers classique.

**2.1** — `df -h`.

**2.2** — `sudo lvextend -L +10G /dev/vg_data/lv_app` puis `sudo resize2fs /dev/vg_data/lv_app` (l'ordre est important : étendre d'abord le volume logique, puis le système de fichiers qu'il contient).

**2.3** — `sudo vgs`.

**3.1** — L'administrateur a bien étendu le **volume logique** LVM (confirmé par `lvdisplay`), mais a omis d'étendre le **système de fichiers** lui-même pour qu'il utilise réellement ce nouvel espace disponible. Le LV dispose bien de 20 Go supplémentaires au niveau du stockage logique, mais le système de fichiers (ext4, d'après le contexte) continue d'ignorer cet espace tant qu'il n'a pas été explicitement étendu à son tour. La commande manquante est `resize2fs /dev/vg_data/lv_app`, qui redimensionne le système de fichiers ext4 pour occuper l'intégralité de l'espace désormais disponible sur le volume logique.

**3.2** — Non, ce n'est pas réalisable directement : XFS ne supporte **aucune réduction** de taille, une limitation structurelle du système de fichiers rappelée en cours (section 7.6). Méthode alternative pour atteindre l'objectif malgré cette contrainte : créer un nouveau volume logique de la taille souhaitée (plus petite), y appliquer un nouveau système de fichiers XFS, copier l'intégralité des données de l'ancien volume vers le nouveau (via `rsync` ou équivalent, avec une interruption de service le temps de la bascule finale ou une synchronisation incrémentale suivie d'une courte coupure), démonter et supprimer l'ancien volume logique une fois la copie validée, libérant ainsi l'espace au profit du VG, puis remonter le nouveau volume à l'emplacement d'origine.

**4.1** — Démarche de gestion de crise proposée : examiner en priorité l'existence de **snapshots LVM** obsolètes ou non essentiels au sein du même VG (rappel section 7.7 du cours) — un snapshot ancien, notamment s'il a accumulé beaucoup de blocs modifiés depuis sa création (mécanisme de copie sur écriture), peut occuper un espace significatif au sein du VG ; sa suppression, si son contenu n'est plus nécessaire (ou after l'avoir sauvegardé ailleurs si besoin), libère immédiatement de l'espace réutilisable pour étendre le volume logique applicatif en tension. Autre piste complémentaire : vérifier si d'autres volumes logiques du même VG disposent d'un espace alloué mais significativement sous-utilisé (via `df -h` comparé à `lvdisplay` pour chaque LV du VG), auquel cas une **réduction contrôlée** d'un LV peu utilisé (si son système de fichiers le permet, ext4 par exemple, contrairement à XFS) pourrait libérer de l'espace au profit du volume en tension. Cette solution reste explicitement **temporaire** : elle ne fait que redistribuer un espace déjà fini au sein du VG existant, sans augmenter la capacité totale réelle de stockage physique disponible — l'ajout d'un nouveau disque physique (nouveau PV, `vgextend`) demeure la seule solution de fond face à une croissance continue des besoins, la gestion de crise ne faisant que gagner du temps jusqu'à ce que cet ajout matériel puisse être réalisé.

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur système junior pour mettre en place une architecture de stockage LVM sur un nouveau serveur, incluant une extension à chaud simulant une croissance de besoin en production, puis pour diagnostiquer un cas classique d'extension incomplète.

## Architecture

Une VM Linux disposant d'au moins un disque secondaire vierge (en plus du disque système), ou de la possibilité d'en attacher/simuler un (fichier disque loopback si aucun disque physique/virtuel supplémentaire n'est disponible — voir note ci-dessous). Un conteneur standard ne convient généralement pas pour ce module (gestion de périphériques de bloc), une VM est requise.

*Note pour environnement sans disque supplémentaire disponible :* un fichier disque peut être utilisé via un périphérique loopback :
```bash
sudo dd if=/dev/zero of=/tmp/disque_lab1.img bs=1M count=2048   # 2 Go
sudo losetup -fP /tmp/disque_lab1.img
losetup -a   # identifier le périphérique loop attribué, ex: /dev/loop0
```
Utiliser ce périphérique loop à la place de `/dev/sdb` dans les étapes suivantes.

## Objectifs
- Créer une architecture LVM complète (PV, VG, LV) et y appliquer un système de fichiers
- Monter durablement ce volume via `/etc/fstab` avec UUID
- Étendre le volume logique à chaud et redimensionner le système de fichiers
- Diagnostiquer un cas volontaire d'extension incomplète (LV étendu, FS non étendu)

## Préparation

```bash
which lsblk pvcreate vgcreate lvcreate mkfs.ext4 blkid 2>/dev/null
lsblk
```

## Étapes

### Étape 1 — Créer l'architecture LVM

```bash
DISQUE=/dev/loop0   # adapter selon votre environnement (ou /dev/sdb si disque réel disponible)

sudo pvcreate "$DISQUE"
sudo vgcreate vg_lab "$DISQUE"
sudo lvcreate -L 1G -n lv_donnees vg_lab

sudo pvs
sudo vgs
sudo lvs
```

### Étape 2 — Système de fichiers et montage persistant

```bash
sudo mkfs.ext4 /dev/vg_lab/lv_donnees
sudo mkdir -p /mnt/lab_donnees
sudo mount /dev/vg_lab/lv_donnees /mnt/lab_donnees

UUID=$(sudo blkid -s UUID -o value /dev/vg_lab/lv_donnees)
echo "UUID=$UUID /mnt/lab_donnees ext4 defaults 0 2" | sudo tee -a /etc/fstab

df -h /mnt/lab_donnees
sudo umount /mnt/lab_donnees
sudo mount -a   # remonte tout /etc/fstab, valide la syntaxe et l'UUID
df -h /mnt/lab_donnees
```
*Question :* pourquoi `sudo mount -a` après avoir modifié `/etc/fstab` est-il un bon réflexe de validation, plutôt que d'attendre un redémarrage complet pour découvrir une éventuelle erreur de syntaxe ?

### Étape 3 — Extension à chaud

```bash
# Simuler l'ajout d'espace (si un second périphérique loop est disponible) ou étendre depuis l'espace VG restant
sudo lvextend -L +500M /dev/vg_lab/lv_donnees
sudo lvdisplay /dev/vg_lab/lv_donnees | grep "LV Size"

df -h /mnt/lab_donnees   # observer : la taille n'a PAS encore changé ici

sudo resize2fs /dev/vg_lab/lv_donnees

df -h /mnt/lab_donnees   # observer : la taille a maintenant changé
```
*Question :* confirmez précisément, avec vos propres captures, le moment exact (avant/après `resize2fs`) où `df -h` reflète le nouvel espace — cela illustre-t-il bien la distinction en deux étapes vue en cours ?

### Étape 4 — Diagnostiquer une extension incomplète (scénario volontaire)

```bash
sudo lvextend -L +200M /dev/vg_lab/lv_donnees
# Volontairement, NE PAS exécuter resize2fs à cette étape

sudo lvdisplay /dev/vg_lab/lv_donnees | grep "LV Size"
df -h /mnt/lab_donnees
```
*Résultat attendu :* écart visible entre la taille du LV (`lvdisplay`) et l'espace réellement utilisable (`df -h`), reproduisant fidèlement le scénario de l'exercice 3.1 du TD.

Corrigez :
```bash
sudo resize2fs /dev/vg_lab/lv_donnees
df -h /mnt/lab_donnees
```

## Commandes de référence
`pvcreate`, `vgcreate`, `lvcreate`, `mkfs.ext4`, `blkid`, `mount -a`, `lvextend`, `resize2fs`, `lvdisplay`

## Résultats attendus
- Sorties `pvs`/`vgs`/`lvs` après création (étape 1)
- Confirmation du montage persistant via UUID et validation par `mount -a` (étape 2)
- Captures avant/après `resize2fs` démontrant la distinction en deux étapes (étape 3)
- Reproduction et correction du cas d'extension incomplète (étape 4)

## Questions

1. Dans l'étape 1, si vous aviez utilisé directement `/dev/loop0` pour créer un système de fichiers sans passer par LVM (`pvcreate`/`vgcreate`/`lvcreate`), qu'auriez-vous perdu en termes de flexibilité future, d'après ce qui a été vu en cours ?
2. Pourquoi l'entrée ajoutée dans `/etc/fstab` à l'étape 2 utilise-t-elle l'UUID plutôt que `/dev/vg_lab/lv_donnees` directement, alors que ce chemin LVM nommé semble déjà relativement stable comparé à `/dev/sdb1` ? (Indice : réfléchissez à la portabilité entre systèmes ou à la robustesse générale de la pratique, indépendamment du cas précis de LVM.)
3. Si l'étape 4 avait porté sur un volume XFS plutôt qu'ext4, la commande de correction aurait-elle été différente ? Laquelle, précisément ?

## Troubleshooting (panne à diagnostiquer)

**Situation fournie :** après avoir correctement suivi toutes les étapes du TP, un étudiant redémarre sa VM pour valider la persistance de sa configuration, mais le système ne démarre plus normalement, restant bloqué sur un shell de secours (emergency mode) avant même d'atteindre `multi-user.target`.

**Démarche attendue :**
1. Symptôme : échec de démarrage après ajout d'une entrée `/etc/fstab`, blocage en mode de secours (rappel direct UE1-M6 sur les shells de secours au démarrage)
2. Hypothèse : l'entrée `/etc/fstab` ajoutée référence un périphérique (le loop device `/dev/loop0`, ou son UUID associé) qui n'est **pas automatiquement recréé** au redémarrage — contrairement à un vrai disque physique ou une vraie partition, un périphérique loopback créé manuellement via `losetup` ne persiste pas nativement après un reboot sans configuration additionnelle spécifique ; systemd, au démarrage, tente de monter cette entrée `/etc/fstab` et échoue car le périphérique n'existe plus, bloquant la séquence de démarrage (rappel UE1-M6 : un montage en échec peut bloquer l'atteinte de la target par défaut)
3. Vérification : depuis le mode de secours, inspecter `/etc/fstab` et confirmer la présence de l'entrée problématique liée au périphérique loopback
4. Isolation : confirmer, via `losetup -a` depuis le mode de secours (si l'outil est accessible à ce stade), que le périphérique loop n'est effectivement plus attaché
5. Correction : ajouter l'option de montage `nofail` à l'entrée concernée dans `/etc/fstab` (permettant au démarrage de continuer même si ce montage spécifique échoue) — solution appropriée pour un environnement de laboratoire avec un périphérique non garanti persistant ; en environnement de production avec du vrai stockage physique, la vraie cause aurait été différente et cette option `nofail` ne devrait être utilisée qu'après une analyse consciente de son impact (un montage réellement critique ne devrait généralement pas être marqué `nofail`, au risque de masquer un problème réel plutôt que de le corriger)
6. Vérification : quitter le mode de secours, redémarrer, confirmer un démarrage normal jusqu'à `multi-user.target`
7. Documentation : noter cette limite propre à l'environnement de laboratoire (périphériques loopback non persistants) comme point de vigilance méthodologique — un exercice utilisant `losetup` doit anticiper ce comportement, soit en évitant de le déclarer dans `/etc/fstab` de façon permanente sans `nofail`, soit en configurant explicitement sa persistance via un mécanisme dédié (hors périmètre de ce TP), pour éviter de reproduire cette panne à chaque redémarrage de l'environnement de laboratoire

## Validation

Le TP est validé si l'étudiant fournit :
- Les 4 résultats/captures demandés
- Des réponses argumentées aux 3 questions
- Une identification correcte de la cause du scénario de troubleshooting (périphérique loopback non persistant, lien explicite avec le mécanisme de démarrage vu en UE1-M6), avec la correction `nofail` correctement justifiée et nuancée

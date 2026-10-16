# UE3 — Module 20 : Gestion des Paquets et du Noyau

---

# PARTIE I — COURS

## 1. Titre du cours
Gestion des paquets et du noyau : dépôts, dépendances, modules et mises à jour maîtrisées

## 2. Niveau
Élève ingénieur — cycle ingénieur, UE3 administration Linux (S2)

## 3. Durée
CM : 4h (module dense — +3h TD, +4h TP en Parties II et III)

## 4. Prérequis
- UE1-Module 1 : architecture kernel/user space
- UE3-Module 15 : gestion du stockage (pertinent pour l'espace occupé par les noyaux successifs)
- UE3-Module 16 : systemd (les paquets installent souvent des unités de service)

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Expliquer le rôle d'un gestionnaire de paquets et le mécanisme de résolution de dépendances
- Gérer des dépôts, installer, mettre à jour, figer (pin/hold) et supprimer des paquets avec APT ou DNF
- Vérifier l'intégrité et l'authenticité d'un paquet via la signature GPG des dépôts
- Expliquer la différence entre le noyau actif et les noyaux installés, et gérer leur cohabitation
- Charger, décharger et configurer des modules noyau
- Élaborer une stratégie de mise à jour maîtrisée, distinguant mises à jour de sécurité et montées de version majeures

## 6. Motivation

Un serveur en production ne reste jamais figé : des vulnérabilités sont découvertes en continu dans les logiciels installés, nécessitant des mises à jour régulières — mais une mise à jour mal maîtrisée peut elle-même casser un service en production, parfois plus sûrement qu'une vulnérabilité non exploitée. La gestion des paquets n'est donc pas une simple compétence technique de commande (`apt install`, `dnf update`) mais une véritable discipline de gestion du risque : comprendre ce qu'une mise à jour change réellement, savoir la tester, savoir revenir en arrière si nécessaire, et savoir distinguer une mise à jour de sécurité urgente d'une montée de version majeure à planifier avec précaution.

Ce module couvre également un aspect souvent négligé : le noyau lui-même est un paquet comme un autre, mais avec des implications particulières (un changement de noyau ne prend effet qu'au redémarrage, rappel direct de la séquence de démarrage étudiée en UE1-M6).

## 7. Concepts fondamentaux

### 7.1 Rôle d'un gestionnaire de paquets

Un **gestionnaire de paquets** automatise l'installation, la mise à jour et la suppression de logiciels, en gérant automatiquement leurs **dépendances** (bibliothèques et autres paquets nécessaires à leur fonctionnement). Sans ce mécanisme, installer un logiciel complexe nécessiterait de rechercher et installer manuellement chacune de ses dépendances, potentiellement en cascade — un travail fastidieux et sujet à erreur que le gestionnaire de paquets automatise entièrement.

**Deux familles principales sur les distributions Linux serveur :**
- **APT** (Debian, Ubuntu et dérivés) : gère des paquets `.deb`
- **DNF** (Fedora, RHEL/CentOS récents et dérivés, successeur de YUM) : gère des paquets `.rpm`

Les deux reposent sur le même principe conceptuel malgré des syntaxes différentes : des **dépôts** (repositories) distants hébergent des paquets indexés, le gestionnaire consulte cet index pour résoudre les dépendances et télécharger ce qui est nécessaire.

### 7.2 Dépôts et résolution de dépendances

```bash
# APT (Debian/Ubuntu)
sudo apt update              # rafraîchit l'index local des paquets disponibles
sudo apt install nginx       # installe nginx et résout ses dépendances automatiquement
sudo apt upgrade              # met à jour tous les paquets installés vers leur dernière version disponible

# DNF (Fedora/RHEL)
sudo dnf check-update
sudo dnf install nginx
sudo dnf upgrade
```

**`apt update` vs `apt upgrade` — une confusion fréquente chez les débutants :** `update` ne met à jour **aucun logiciel**, il rafraîchit uniquement l'index local des paquets disponibles (métadonnées) depuis les dépôts configurés ; c'est `upgrade` qui installe réellement les nouvelles versions disponibles selon cet index rafraîchi. Exécuter `upgrade` sans avoir fait `update` au préalable travaillerait sur un index potentiellement obsolète, ignorant silencieusement des mises à jour publiées depuis le dernier rafraîchissement.

**Conflits de dépendances :** il arrive qu'un paquet demandé nécessite une version d'une dépendance incompatible avec une version déjà requise par un autre paquet installé — le gestionnaire de paquets refuse alors l'opération plutôt que de risquer un système incohérent, signalant explicitement le conflit à résoudre manuellement (souvent en acceptant une mise à jour en cascade, ou en renonçant à l'installation du nouveau paquet).

### 7.3 Intégrité et authenticité — signature GPG des dépôts

Un paquet téléchargé depuis un dépôt distant représente une surface de confiance critique : un dépôt compromis (ou un attaquant interceptant la communication) pourrait potentiellement distribuer un paquet malveillant. Les gestionnaires de paquets modernes vérifient systématiquement une **signature cryptographique GPG** associée à chaque dépôt, garantissant que les paquets proviennent bien de la source légitime et n'ont pas été altérés en transit.

```bash
# Ajouter une clé GPG de dépôt (exemple générique, syntaxe selon distribution)
sudo apt-key add cle_depot.gpg   # méthode historique, dépréciée au profit de keyrings dédiés sur les versions récentes
# ou, méthode moderne recommandée :
sudo gpg --dearmor -o /etc/apt/keyrings/depot.gpg cle_depot.asc
```

**Ne jamais désactiver la vérification de signature** pour contourner une erreur d'installation, même temporairement — cette vérification est une des dernières lignes de défense contre la distribution de paquets malveillants, et sa désactivation expose directement le système à ce risque, un raccourci de facilité qui compromet une garantie de sécurité fondamentale.

### 7.4 Figer un paquet (pinning/hold)

Il est parfois nécessaire d'empêcher la mise à jour automatique d'un paquet spécifique, par exemple pour maintenir une version précise compatible avec une application critique déjà validée :

```bash
# APT
sudo apt-mark hold nginx
sudo apt-mark unhold nginx   # annuler le gel

# DNF
sudo dnf versionlock add nginx
sudo dnf versionlock delete nginx
```

**Compromis important à comprendre :** figer un paquet le protège d'une mise à jour non désirée, mais le prive également des correctifs de sécurité qui seraient publiés pour ce même paquet — une décision qui doit toujours être documentée avec sa justification et réévaluée périodiquement, jamais laissée en place indéfiniment sans révision (rappel du principe déjà rencontré à plusieurs reprises : toute exception de sécurité temporaire doit être suivie et non oubliée).

### 7.5 Le noyau comme paquet — spécificités

Le noyau Linux est distribué et géré comme n'importe quel autre paquet (`apt install linux-image-...` ou équivalent DNF), mais avec des implications particulières directement liées à la séquence de démarrage étudiée en UE1-M6 :

- Installer un nouveau noyau **n'active pas** immédiatement ce noyau — le système continue de fonctionner sur le noyau actuellement chargé en mémoire jusqu'au **prochain redémarrage** (rappel direct : le noyau est chargé une seule fois par GRUB au démarrage, il ne peut pas être "remplacé à chaud" pendant l'exécution)
- Plusieurs noyaux peuvent coexister installés simultanément sur un système, visibles dans le menu GRUB au démarrage — une sécurité précieuse permettant de redémarrer sur un noyau précédent en cas de problème avec le plus récent
- La commande `uname -r` affiche le noyau **actuellement actif** (chargé en mémoire), qui peut différer du dernier noyau **installé** si aucun redémarrage n'a eu lieu depuis sa mise à jour — une confusion fréquente et un piège de diagnostic classique

```bash
uname -r                              # noyau actif actuellement
dpkg --list | grep linux-image        # tous les noyaux installés (APT)
rpm -qa | grep kernel                  # équivalent DNF
```

### 7.6 Modules noyau

Le noyau Linux charge dynamiquement des **modules** (pilotes matériels, systèmes de fichiers additionnels, fonctionnalités réseau) selon les besoins, sans nécessiter de recompiler ou même de redémarrer le noyau lui-même dans la majorité des cas :

```bash
lsmod                              # lister les modules actuellement chargés
sudo modprobe nom_du_module          # charger un module (avec résolution de ses propres dépendances)
sudo modprobe -r nom_du_module        # décharger un module
modinfo nom_du_module                 # informations détaillées sur un module
```

**Persistance du chargement de modules :** un module chargé manuellement via `modprobe` ne survit pas à un redémarrage, sauf déclaration explicite dans `/etc/modules-load.d/` — une application directe, une fois de plus, du principe de persistance déjà rencontré pour la configuration réseau (UE3-M17) et les tâches planifiées (UE3-M19) : toute configuration appliquée en ligne de commande est volatile sauf déclaration explicite dans un mécanisme de persistance dédié.

### 7.7 Stratégie de mise à jour — sécurité vs stabilité

Toute organisation doit arbitrer entre deux exigences potentiellement en tension :
- **Réactivité de sécurité** : appliquer rapidement les correctifs de vulnérabilités critiques, pour limiter la fenêtre d'exposition
- **Stabilité de production** : éviter qu'une mise à jour non testée ne casse un service critique

**Approches courantes :**
- **Mises à jour de sécurité automatiques** (ex : `unattended-upgrades` sous Debian/Ubuntu), limitées aux correctifs de sécurité critiques uniquement (pas aux montées de version fonctionnelles), appliquées automatiquement sans intervention humaine
- **Environnement de test/staging** : appliquer d'abord toute mise à jour non critique sur un environnement de test représentatif avant la production, avec une fenêtre de validation
- **Déploiement progressif (canary)** : appliquer une mise à jour à un sous-ensemble limité de serveurs de production avant généralisation, permettant de détecter un problème avant qu'il n'affecte l'ensemble de l'infrastructure

**Aucune approche n'est universellement "meilleure"** — le choix dépend de la criticité du service, de la fréquence des vulnérabilités affectant sa pile technique, et de la maturité des processus de test disponibles dans l'organisation.

## 8. Architecture

```
┌────────────────────────────────────────────────────────────┐
│  Dépôt distant (signé GPG)                                       │
│         │                                                          │
│    apt update / dnf check-update  ──▶  index local rafraîchi        │
│         │                                                          │
│    apt install / dnf install  ──▶  résolution de dépendances         │
│         │                                                          │
│         ▼                                                          │
│  Paquets installés localement (dpkg/rpm)                            │
│         │                                                          │
│    Cas particulier : paquet noyau                                    │
│         ▼                                                          │
│  Nouveau noyau installé, actif seulement après redémarrage             │
│  (menu GRUB conserve les noyaux précédents comme filet de sécurité)     │
└────────────────────────────────────────────────────────────┘
```

## 9. Fonctionnement interne

Sous Debian/Ubuntu, `apt` s'appuie en réalité sur `dpkg` pour l'installation effective des fichiers d'un paquet (extraction, placement des fichiers, exécution de scripts de pré/post-installation), `apt` lui-même se chargeant de la couche supérieure de résolution de dépendances et de gestion des dépôts — une architecture en couches similaire dans son principe à d'autres abstractions déjà rencontrées dans le programme (VFS en UE1, LVM en UE3-M15). Le chargement d'un module noyau (`modprobe`) s'appuie sur l'appel système `init_module` (ou `finit_module`), qui charge le code du module directement en espace noyau — une opération privilégiée par nature, cohérente avec la distinction kernel/user space posée dès UE1-M1.

## 10. Démonstration

```bash
# Observer le noyau actif vs les noyaux installés
uname -r
dpkg --list | grep linux-image

# Observer les modules actuellement chargés
lsmod | head -10

# Simuler une mise à jour sans l'appliquer réellement (mode simulation)
sudo apt install --simulate nginx
```

## 11. Commandes / outils

| Commande | Rôle | Exemple |
|---|---|---|
| `apt update` / `apt upgrade` | Rafraîchir l'index / appliquer les mises à jour | `sudo apt update && sudo apt upgrade` |
| `apt install --simulate` | Simuler une installation sans l'exécuter | `apt install --simulate paquet` |
| `apt-mark hold` / `unhold` | Figer/dégeler un paquet | `sudo apt-mark hold nginx` |
| `apt list --upgradable` | Lister les paquets pour lesquels une mise à jour est disponible | `apt list --upgradable` |
| `uname -r` | Afficher le noyau actif | `uname -r` |
| `lsmod` / `modprobe` | Gérer les modules noyau | `sudo modprobe nom_module` |
| `apt-cache policy <paquet>` | Voir les versions disponibles et la version installée | `apt-cache policy nginx` |

**Erreur fréquente :** confondre le noyau **actif** (`uname -r`) avec le dernier noyau **installé** (visible via `dpkg`/`rpm`) — après une mise à jour du noyau sans redémarrage, ces deux informations diffèrent, un piège classique lors d'un audit de sécurité qui vérifierait uniquement `uname -r` en pensant confirmer l'application d'un correctif noyau déjà installé mais pas encore activé.

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** un audit de sécurité signale qu'un serveur reste vulnérable à une faille noyau corrigée plusieurs semaines auparavant. L'équipe d'exploitation affirme avoir appliqué la mise à jour dès sa publication, ce que confirme effectivement `dpkg --list | grep linux-image`, montrant bien la version corrigée installée.

**Analyse attendue :** ce scénario illustre directement le piège de la section 7.5 et de l'erreur fréquente associée : le paquet du noyau corrigé a bien été **installé**, mais le serveur n'a jamais été **redémarré** depuis cette installation — le noyau réellement **actif** en mémoire (vérifiable via `uname -r`) reste l'ancienne version vulnérable, le nouveau noyau installé n'ayant jamais pris effet faute de redémarrage. Ce cas souligne une réalité opérationnelle fréquemment sous-estimée : une politique de mise à jour de sécurité efficace pour le noyau doit explicitement inclure une planification de redémarrage régulière (fenêtre de maintenance), sans quoi l'installation seule du correctif reste insuffisante et trompeuse pour toute vérification de conformité basée uniquement sur l'état des paquets installés plutôt que sur l'état réellement actif du système. Correction : planifier un redémarrage dès que possible (idéalement lors d'une fenêtre de maintenance déjà prévue), et mettre en place une vérification de conformité qui compare systématiquement `uname -r` à la dernière version de noyau installée, pas seulement l'état des paquets.

## 15. Sécurité

- La vérification de signature GPG des dépôts est une mesure de sécurité fondamentale contre la distribution de paquets malveillants — jamais désactivée, même temporairement
- Un paquet figé (`hold`) reste privé des correctifs de sécurité qui le concernent — toute décision de gel doit être documentée, justifiée et réévaluée périodiquement
- Une politique de mise à jour de sécurité doit distinguer explicitement les correctifs critiques (à appliquer rapidement, éventuellement automatiquement) des montées de version fonctionnelles (à tester avant déploiement) — les confondre mène soit à un système jamais patché par excès de prudence, soit à un système instable par excès de réactivité non maîtrisée
- Un module noyau chargé peut représenter une surface d'attaque supplémentaire (code s'exécutant en espace noyau, avec des privilèges maximaux) — ne charger que les modules réellement nécessaires est une application directe du principe du moindre privilège déjà rencontré à plusieurs reprises dans le programme
- La conformité de sécurité d'un système doit être vérifiée sur son **état réellement actif** (noyau chargé en mémoire, services effectivement en cours d'exécution avec leur configuration réellement chargée — rappel du piège `daemon-reload` du module 16), jamais uniquement sur l'état déclaratif de sa configuration ou de ses paquets installés

## 16. Automatisation

Script Bash de vérification de cohérence entre le noyau installé et le noyau actif, signalant la nécessité d'un redémarrage.

```bash
#!/usr/bin/env bash
# Compare le noyau actif au dernier noyau installé, alerte si un redémarrage est nécessaire
set -euo pipefail

LOGFILE="/var/log/kernel_check.log"

NOYAU_ACTIF=$(uname -r)
DERNIER_INSTALLE=$(dpkg --list 2>/dev/null | grep '^ii.*linux-image-[0-9]' | \
    awk '{print $2}' | sed 's/linux-image-//' | sort -V | tail -1)

if [[ -z "$DERNIER_INSTALLE" ]]; then
    echo "$(date '+%F %T') - AVERTISSEMENT: impossible de déterminer le dernier noyau installé (adapter le script si distribution non basée sur dpkg)" \
        | tee -a "$LOGFILE"
    exit 2
fi

if [[ "$NOYAU_ACTIF" != "$DERNIER_INSTALLE" ]]; then
    echo "$(date '+%F %T') - ALERTE: noyau actif ($NOYAU_ACTIF) différent du dernier noyau installé ($DERNIER_INSTALLE) — redémarrage requis" \
        | tee -a "$LOGFILE"
    exit 1
fi

echo "$(date '+%F %T') - OK: noyau actif conforme au dernier noyau installé" >> "$LOGFILE"
exit 0
```

## 17. Troubleshooting

Scénario court : une tentative d'installation d'un nouveau paquet échoue avec un message de conflit de dépendances, impliquant un autre paquet déjà installé et critique pour un service en production. Démarche : examiner précisément le message de conflit (quelle version de quelle dépendance est en cause) → utiliser `apt-cache policy`/équivalent pour comparer la version actuellement installée, la version requise par le nouveau paquet, et les versions disponibles dans les dépôts configurés → évaluer si une mise à jour en cascade de la dépendance concernée est acceptable (test préalable en environnement non critique recommandé, rappel section 7.7) ou si le nouveau paquet doit être temporairement écarté, éventuellement au profit d'une version alternative compatible si elle existe → ne jamais forcer une installation en ignorant un conflit de dépendances signalé (option existante dans certains gestionnaires mais dangereuse), sous peine de laisser le système dans un état incohérent difficile à diagnostiquer ultérieurement.

## 18. Synthèse

- Un gestionnaire de paquets automatise l'installation et la résolution de dépendances ; `update` rafraîchit l'index, `upgrade` applique réellement les mises à jour — une distinction fréquemment confondue
- La vérification de signature GPG des dépôts est une mesure de sécurité fondamentale, jamais à désactiver
- Figer un paquet (`hold`) le protège d'une mise à jour non désirée mais le prive aussi des correctifs de sécurité — une décision à documenter et réévaluer
- Le noyau actif (`uname -r`) peut différer du dernier noyau installé si aucun redémarrage n'a eu lieu — un piège classique de conformité de sécurité
- Les modules noyau se chargent dynamiquement (`modprobe`), sans persistance native au redémarrage sauf déclaration explicite
- Une stratégie de mise à jour mature distingue correctifs de sécurité critiques (réactivité) et montées de version fonctionnelles (test préalable) — aucune approche unique n'est universellement adaptée

## 19. Questions d'évaluation

**QCM**
1. `apt update` sert à :
 a) installer toutes les mises à jour disponibles b) rafraîchir l'index local des paquets disponibles depuis les dépôts c) supprimer les paquets obsolètes d) redémarrer le système

2. Après l'installation d'un nouveau noyau, celui-ci devient actif :
 a) immédiatement b) seulement après le prochain redémarrage c) jamais automatiquement, une commande spécifique est requise d) uniquement si le service concerné est redémarré

**Questions ouvertes**
- Expliquez le piège classique entre noyau actif et noyau installé, et son impact sur un audit de conformité de sécurité.
- Expliquez le compromis induit par le fait de figer (hold) un paquet, et pourquoi cette décision doit être documentée.

**Exercice technique**
- Donnez la commande permettant d'afficher le noyau actuellement actif sur un système Linux.

**Problème de diagnostic**
- Un audit affirme qu'un correctif noyau critique est installé, mais le système reste vulnérable en pratique. Proposez la cause la plus probable et la vérification à effectuer.

## 20. Mini-projet

Développer un script (Bash ou Python) qui :
1. Vérifie la cohérence entre le noyau actif et le dernier noyau installé (reprise et extension du script de la section 16), avec calcul du nombre de jours depuis la dernière installation d'un noyau non encore actif
2. Liste les paquets actuellement figés (`hold`/`versionlock`) et signale ceux figés depuis plus d'une durée configurable sans réévaluation documentée (via un fichier de justification associé, à faire tenir à jour par l'équipe)
3. Vérifie la présence de mises à jour de sécurité disponibles non encore appliquées, en les distinguant des mises à jour fonctionnelles classiques si l'information est disponible dans les métadonnées du gestionnaire de paquets utilisé
4. Journalise chaque détection avec horodatage et niveau de gravité
5. Génère un rapport de synthèse périodique destiné à une revue de conformité de sécurité

---

# PARTIE II — TD

### Niveau 1 — Compréhension

**Exercice 1.1**
Expliquez le rôle d'un gestionnaire de paquets et pourquoi la résolution de dépendances est une fonctionnalité centrale.

**Exercice 1.2**
Expliquez la différence entre `apt update` et `apt upgrade`.

**Exercice 1.3**
Expliquez pourquoi installer un nouveau noyau ne suffit pas à le rendre actif immédiatement.

### Niveau 2 — Application

**Exercice 2.1**
Donnez la commande permettant de figer un paquet `postgresql` pour l'empêcher d'être mis à jour automatiquement.

**Exercice 2.2**
Donnez la commande permettant d'afficher tous les modules noyau actuellement chargés.

**Exercice 2.3**
Donnez la commande permettant de simuler une installation de paquet sans l'exécuter réellement.

### Niveau 3 — Analyse

**Exercice 3.1**
Un serveur affiche, via `dpkg --list`, un noyau corrigé récemment installé, mais `uname -r` affiche toujours l'ancienne version. L'équipe conclut à une erreur d'installation et tente de réinstaller le paquet, sans succès apparent (le paquet est déjà présent). Quelle est l'erreur de diagnostic commise, et quelle est la véritable action nécessaire ?

**Exercice 3.2**
Une organisation choisit de figer (hold) une dizaine de paquets critiques pour garantir la stabilité de son infrastructure, sans jamais revoir cette liste depuis deux ans. Un audit de sécurité récent révèle plusieurs vulnérabilités connues et corrigées affectant précisément ces paquets figés. Expliquez la relation de cause à effet, et proposez une pratique de gouvernance qui aurait permis d'éviter cette situation.

### Niveau 4 — Problème complexe

**Exercice 4.1**
Une équipe d'exploitation active `unattended-upgrades` (mises à jour de sécurité automatiques) sur l'ensemble de son parc de serveurs de production, sans mettre en place de fenêtre de test préalable ni de mécanisme de rollback. Quelques semaines plus tard, une mise à jour de sécurité, bien qu'automatiquement appliquée avec les meilleures intentions, introduit une régression de compatibilité inattendue avec une application métier critique, provoquant une panne en pleine nuit sans qu'aucun humain n'ait validé le changement au préalable. En vous appuyant sur les approches de la section 7.7 du cours, proposez une stratégie de mise à jour révisée qui conserve les bénéfices de la réactivité de sécurité (ne pas revenir à des mises à jour manuelles complètes, trop lentes) tout en réduisant significativement le risque d'un tel incident non maîtrisé, en détaillant concrètement comment elle se serait appliquée à ce scénario précis.

---

## Correction du TD

**1.1** — Un gestionnaire de paquets automatise l'installation, la mise à jour et la suppression de logiciels sur un système, en s'appuyant sur des dépôts distants indexés. La résolution de dépendances est centrale car la plupart des logiciels dépendent d'autres paquets (bibliothèques, outils) pour fonctionner ; sans automatisation, un administrateur devrait rechercher et installer manuellement chaque dépendance, potentiellement en cascade sur plusieurs niveaux, un travail fastidieux et source d'erreurs (versions incompatibles, dépendances oubliées) que le gestionnaire de paquets élimine.

**1.2** — `apt update` rafraîchit uniquement l'**index local** des paquets disponibles depuis les dépôts configurés (métadonnées), sans installer ni mettre à jour aucun logiciel. `apt upgrade` installe réellement les nouvelles versions disponibles des paquets déjà installés, en se basant sur cet index — nécessitant généralement d'avoir exécuté `update` au préalable pour disposer d'un index à jour, sans quoi `upgrade` travaillerait sur des informations potentiellement obsolètes.

**1.3** — Le noyau est chargé en mémoire une seule fois, au démarrage de la machine, par le bootloader (rappel UE1-M6) — il ne peut pas être remplacé "à chaud" pendant que le système fonctionne. Installer un nouveau paquet de noyau place les fichiers nécessaires sur le disque et les rend disponibles pour un futur démarrage, mais le système continue de fonctionner sur le noyau déjà chargé en mémoire jusqu'à ce qu'un redémarrage complet ait lieu, moment où le nouveau noyau sera effectivement chargé et devient actif.

**2.1** — `sudo apt-mark hold postgresql`.

**2.2** — `lsmod`.

**2.3** — `apt install --simulate <nom_du_paquet>` (ou `apt install -s <paquet>`, forme abrégée équivalente).

**3.1** — L'erreur de diagnostic est de confondre le noyau **installé** (correctement mis à jour, confirmé par `dpkg --list`) avec le noyau **actif** (`uname -r`, qui reste l'ancienne version tant qu'aucun redémarrage n'a eu lieu). Réinstaller le paquet ne changera rien, puisqu'il est déjà présent et à jour sur le disque — le problème n'est pas une erreur d'installation mais l'absence de redémarrage depuis cette installation. La véritable action nécessaire est simplement de **redémarrer le serveur**, ce qui chargera effectivement le nouveau noyau déjà installé et le rendra actif, confirmable ensuite par une nouvelle vérification de `uname -r` après le redémarrage.

**3.2** — Le fait d'avoir figé ces paquets a empêché l'application des correctifs de sécurité qui les concernaient au fil du temps (rappel section 7.4 du cours : figer un paquet le prive des mises à jour, y compris de sécurité), exactement le compromis attendu de cette décision — mais l'absence de toute réévaluation pendant deux ans a transformé une mesure de stabilité initialement justifiée en une accumulation silencieuse et non maîtrisée de vulnérabilités connues et publiquement documentées, un risque bien plus grave que l'instabilité que le gel visait initialement à prévenir. Une pratique de gouvernance qui aurait évité cette situation : associer systématiquement à toute décision de gel une date de réévaluation planifiée (par exemple, une revue trimestrielle ou semestrielle systématique de la liste des paquets figés), avec une justification documentée renouvelée à chaque revue, plutôt qu'un gel décidé une fois et jamais revisité — cohérent avec le principe déjà rencontré à plusieurs reprises dans le programme qu'une exception de sécurité temporaire doit être suivie activement, jamais simplement oubliée.

**4.1** — Stratégie révisée proposée, conservant la réactivité tout en réduisant le risque : conserver `unattended-upgrades` pour son bénéfice principal (réactivité face aux vulnérabilités critiques, sans dépendre systématiquement d'une intervention humaine potentiellement lente), mais l'organiser selon un **déploiement progressif par vagues** plutôt qu'une application simultanée sur l'ensemble du parc : appliquer d'abord les mises à jour de sécurité automatiques sur un sous-ensemble limité et non critique de serveurs (environnement de test représentatif, ou un petit groupe de serveurs de production à plus faible impact en cas de problème), observer leur comportement pendant une fenêtre de surveillance de quelques heures à un jour, puis étendre automatiquement l'application aux serveurs restants seulement si aucune anomalie n'a été détectée pendant cette fenêtre d'observation — une combinaison des approches "mise à jour automatique de sécurité" et "déploiement progressif (canary)" de la section 7.7. Appliqué au scénario précis de l'énoncé : si le sous-ensemble initial de serveurs avait inclus (ou idéalement, avait spécifiquement inclus) un environnement représentatif de l'application métier concernée, la régression de compatibilité aurait été détectée durant la fenêtre de surveillance limitée, avant toute application généralisée sur l'ensemble du parc de production — transformant un incident nocturne généralisé et non anticipé en une anomalie détectée et contenue sur un périmètre restreint, avec le temps nécessaire pour qu'une équipe humaine investigue et décide (report de la vague suivante, correctif applicatif, ou exception documentée pour ce paquet spécifique) avant tout impact sur l'ensemble du service.

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur système junior pour auditer l'état des paquets et du noyau d'un serveur de laboratoire, mettre en place une vérification automatisée de cohérence noyau actif/installé, et pratiquer la gestion prudente de conflits de dépendances.

## Architecture

Une VM Linux (APT ou DNF selon la distribution disponible dans l'environnement pédagogique — les commandes ci-dessous utilisent APT, à adapter si nécessaire).

## Objectifs
- Auditer l'état des paquets installés et des mises à jour disponibles
- Observer concrètement l'écart noyau actif/installé après une mise à jour
- Manipuler le gel de paquets et en observer l'effet
- Développer et tester un script de vérification de cohérence noyau

## Préparation

```bash
which apt dpkg uname lsmod 2>/dev/null
uname -r
```

## Étapes

### Étape 1 — Audit de l'état des paquets

```bash
sudo apt update
apt list --upgradable

apt-cache policy nginx 2>/dev/null || echo "nginx non installé, à adapter avec un paquet présent sur votre système"
```
*Question :* combien de paquets ont une mise à jour disponible sur votre système au moment du TP ? Distinguez, si l'information est visible, ce qui relève d'une mise à jour de sécurité de ce qui relève d'une mise à jour fonctionnelle classique.

### Étape 2 — Observer l'écart noyau actif/installé (si une mise à jour noyau est disponible)

```bash
uname -r
dpkg --list | grep '^ii.*linux-image-[0-9]'

# Si une mise à jour du noyau est disponible dans votre environnement de laboratoire :
sudo apt install --only-upgrade linux-image-generic 2>/dev/null || echo "Aucune mise à jour noyau disponible dans cet environnement, passer à la question ci-dessous en mode réflexion"

uname -r   # toujours l'ancienne version, tant qu'aucun redémarrage n'a eu lieu
dpkg --list | grep '^ii.*linux-image-[0-9]'   # peut désormais montrer une version plus récente installée
```
*Question :* si vous avez pu observer un écart réel entre `uname -r` et le dernier noyau listé par `dpkg`, décrivez précisément cet écart. Si aucune mise à jour n'était disponible dans votre environnement, expliquez néanmoins, en vous appuyant sur le cours, ce que vous observeriez dans ce cas et pourquoi.

### Étape 3 — Geler un paquet et observer l'effet

```bash
# Choisir un paquet installé sur votre système pour la démonstration (adapter le nom)
PAQUET_DEMO="curl"

apt-cache policy "$PAQUET_DEMO"
sudo apt-mark hold "$PAQUET_DEMO"
apt-mark showhold

sudo apt update
sudo apt upgrade --simulate | grep -i "$PAQUET_DEMO" || echo "$PAQUET_DEMO n'apparaît pas dans la simulation de mise à jour, conforme au gel appliqué"

sudo apt-mark unhold "$PAQUET_DEMO"
apt-mark showhold
```
*Question :* pourquoi la vérification via `apt upgrade --simulate` (plutôt qu'une exécution réelle) est-elle la méthode appropriée pour ce test en environnement de laboratoire partagé ou sensible ?

### Étape 4 — Script de vérification de cohérence noyau

Reprendre et tester le script de la section 16 du cours (`kernel_check.sh`) :

```bash
bash kernel_check.sh
cat /var/log/kernel_check.log
```

## Commandes de référence
`apt update`, `apt list --upgradable`, `apt-cache policy`, `apt-mark hold`/`unhold`, `uname -r`, `dpkg --list`

## Résultats attendus
- Liste des paquets avec mise à jour disponible (étape 1)
- Observation (réelle ou en réflexion argumentée) de l'écart noyau actif/installé (étape 2)
- Preuve du gel et du dégel effectif d'un paquet, avec vérification par simulation (étape 3)
- Sortie du script de vérification de cohérence noyau (étape 4)

## Questions

1. Dans l'étape 3, si vous aviez exécuté `apt upgrade` réellement (sans `--simulate`) pendant que le paquet était gelé, que se serait-il passé pour ce paquet spécifique par rapport aux autres paquets ayant une mise à jour disponible ?
2. Pourquoi la simple présence d'un nouveau noyau dans la sortie de `dpkg --list` ne constitue-t-elle pas, à elle seule, une preuve suffisante de conformité lors d'un audit de sécurité portant sur une vulnérabilité corrigée au niveau noyau ?
3. Le script de l'étape 4 est-il capable de détecter un noyau vulnérable si **aucune** mise à jour corrective n'a jamais été installée (paquet jamais mis à jour, pas seulement non redémarré) ? Justifiez et proposez, en réflexion, ce qu'un tel script devrait vérifier en complément pour couvrir également ce cas.

## Troubleshooting (panne à diagnostiquer)

**Situation fournie :** un étudiant exécute le script `kernel_check.sh` de l'étape 4 sur un système où aucune mise à jour de noyau n'a jamais été appliquée depuis l'installation initiale, et le script rapporte systématiquement "OK: noyau actif conforme au dernier noyau installé" — alors qu'un audit externe recommande fortement une mise à jour du noyau vers une version plus récente corrigeant plusieurs vulnérabilités connues.

**Démarche attendue :**
1. Symptôme : le script rapporte un état "OK" alors qu'une véritable action de sécurité (mise à jour du noyau) reste nécessaire
2. Hypothèse : le script vérifie uniquement la **cohérence** entre le noyau actif et le dernier noyau **installé** — il ne vérifie en aucun cas si ce noyau installé est lui-même **à jour** par rapport aux versions disponibles dans les dépôts, ni s'il existe des vulnérabilités connues affectant la version actuellement installée. Un système n'ayant jamais reçu de mise à jour de noyau obtient donc légitimement un résultat "OK" de ce script précis, car il n'y a effectivement aucun écart entre actif et installé — mais cette cohérence ne dit absolument rien sur l'obsolescence ou la vulnérabilité de cette version elle-même
3. Vérification : comparer manuellement la version de noyau actuellement installée avec la dernière version disponible dans les dépôts configurés (`apt list --upgradable | grep linux-image`, ou équivalent), pour confirmer qu'une mise à jour est effectivement disponible et non appliquée
4. Isolation : confirmer que le script de la section 16 du cours ne couvre, par sa conception même, qu'un seul type de risque (désynchronisation actif/installé après une mise à jour appliquée mais non activée), pas l'absence totale de mise à jour appliquée
5. Correction : étendre le script pour qu'il vérifie également, en complément de la cohérence actif/installé, la présence d'une mise à jour de noyau disponible et non installée (`apt list --upgradable`), avec une alerte de nature différente ("mise à jour disponible non appliquée" vs "redémarrage requis après mise à jour déjà installée") pour bien distinguer les deux catégories de risque au sein du rapport produit
6. Vérification : relancer le script étendu, confirmer qu'il détecte désormais correctement les deux types de situation (testable en simulant les deux cas si l'environnement le permet)
7. Documentation : noter cette limite comme un rappel important en conception d'outils d'audit — un script qui vérifie une **cohérence interne** (ici, actif vs installé) ne garantit en rien la **conformité globale** recherchée (ici, l'absence de vulnérabilité connue) ; les deux dimensions doivent être vérifiées séparément et explicitement, une leçon transversale applicable à de nombreux autres outils d'audit développés au fil du programme

## Validation

Le TP est validé si l'étudiant fournit :
- Les 4 résultats/captures/réflexions demandés
- Des réponses argumentées aux 3 questions
- Une identification correcte de la cause du scénario de troubleshooting (limite de portée du script, cohérence interne ≠ conformité globale), avec une proposition d'extension cohérente

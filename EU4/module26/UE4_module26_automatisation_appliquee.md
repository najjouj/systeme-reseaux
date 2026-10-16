# UE4 — Module 26 : Automatisation Appliquée

*Module de synthèse et de clôture de l'UE4 — mobilise les modules 23 à 25 (syntaxe, pipelines, robustesse) pour construire de vrais outils d'automatisation système, en s'appuyant sur l'ensemble des scripts déjà écrits de façon ponctuelle au fil des UE1, UE2 et UE3.*

---

# PARTIE I — COURS

## 1. Titre du cours
Construire des outils d'automatisation système robustes : monitoring, vérification, analyse de logs, détection d'anomalies

## 2. Niveau
Élève ingénieur — cycle ingénieur, clôture de l'UE4 automatisation (S2)

## 3. Durée
CM : 2h (module de synthèse orienté pratique — +2h TD, +6h TP en Parties II et III, volume TP élevé car module intégrateur)

## 4. Prérequis
- UE4-Modules 23, 24, 25 : syntaxe, pipelines, robustesse — mobilisés conjointement dans ce module
- L'ensemble des scripts déjà écrits dans les UE1, UE2 et UE3 (dizaines de scripts de supervision, tous à revisiter à la lumière de ce module)

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Concevoir l'architecture d'un outil de monitoring modulaire et réutilisable, plutôt qu'un script monolithique
- Écrire des vérifications de service robustes, distinguant les catégories de vérification déjà rencontrées (état actif, cohérence de configuration, seuils de performance)
- Automatiser l'analyse de logs pour la détection d'anomalies, en combinant les outils de traitement de texte du module 24
- Appliquer une détection d'anomalie par seuil et par tendance, en évitant les pièges déjà rencontrés (faux positifs après redémarrage, seuils arbitraires non calibrés)
- Intégrer un outil d'automatisation dans l'écosystème de planification déjà étudié (cron, timers systemd)
- Revisiter un script existant des UE précédentes pour en évaluer et améliorer la robustesse à la lumière de l'ensemble de l'UE4

## 6. Motivation

Ce module ne présente aucun concept Bash nouveau — les modules 23, 24 et 25 ont donné tout l'outillage syntaxique, de traitement de texte et de robustesse nécessaire. Ce qui manque encore est la **discipline de conception** : comment structurer un outil de supervision qui ne soit pas un script fragile écrit dans l'urgence, mais un outil réutilisable, testé, et intégré proprement dans l'écosystème d'automatisation déjà étudié en UE3.

Vous avez déjà écrit, au fil du programme, des dizaines de scripts de supervision : détection de zombies (UE1-M2), surveillance mémoire (UE1-M3), détection de fichiers supprimés ouverts (UE1-M4), surveillance ARP (UE2-M8), audit de permissions (UE3-M14), gestion de capacité LVM (UE3-M15), et bien d'autres. Ce module de clôture d'UE4 est l'occasion de revenir sur cette production abondante avec un regard neuf, outillé par la rigueur syntaxique (M23), la puissance des pipelines (M24) et la robustesse (M25) — transformant des scripts fonctionnels en véritables outils professionnels.

## 7. Concepts fondamentaux

### 7.1 Architecture modulaire d'un outil de monitoring

Un outil de supervision mature sépare généralement plusieurs responsabilités distinctes, plutôt que de tout mélanger dans un unique bloc de code :

```
┌─────────────────────────────────────────────┐
│  Bibliothèque commune (rappel M25)              │
│  set -euo pipefail, trap, validation, logging      │
├─────────────────────────────────────────────┤
│  Collecte    : fonctions récupérant l'état brut      │
│               (ex: lister les processus, lire un log) │
├─────────────────────────────────────────────┤
│  Analyse     : fonctions interprétant l'état collecté │
│               (ex: comparer à un seuil, détecter        │
│                une tendance)                              │
├─────────────────────────────────────────────┤
│  Rapport     : fonctions formatant et communiquant       │
│               le résultat (log, alerte, rapport)           │
└─────────────────────────────────────────────┘
```

**Pourquoi cette séparation compte-t-elle ?** Un script qui mélange collecte, analyse et rapport dans un seul bloc est difficile à tester (impossible de vérifier la logique d'analyse sans exécuter une vraie collecte système) et difficile à faire évoluer (changer le format de rapport implique de retoucher du code entremêlé avec la logique de détection). Séparer ces responsabilités en fonctions distinctes, avec `local` et des interfaces claires (rappel M23), permet de tester chaque partie isolément et de la remplacer sans affecter le reste.

### 7.2 Catégories de vérification de service — une taxonomie utile

Au fil des UE1-UE3, trois grandes catégories de vérification sont apparues, à distinguer explicitement dans la conception d'un outil :

- **Vérification d'état actif** : le service tourne-t-il ? (`systemctl is-active`, rappel UE3-M16)
- **Vérification de cohérence de configuration** : l'état réellement chargé correspond-il à ce qui est attendu ? (rappel du piège `daemon-reload` oublié, UE3-M16 ; rappel du piège noyau actif vs installé, UE3-M20)
- **Vérification de seuil de performance** : le service fonctionne-t-il dans des paramètres acceptables ? (rappel UE1-M3 mémoire, UE3-M21 performance)

**Une bonne pratique de conception, directement issue des leçons accumulées dans le programme :** ne jamais mélanger ces trois catégories dans un même indicateur binaire "OK"/"KO" — rappel direct du module UE3-M20, où un statut unique masquait la distinction entre cohérence interne et conformité globale. Un outil de qualité rapporte séparément l'état de chaque catégorie.

### 7.3 Analyse de logs automatisée — combiner M24 et M25

```bash
#!/usr/bin/env bash
set -euo pipefail

analyser_erreurs_recentes() {
    local fichier_log="$1"
    local fenetre_minutes="${2:-60}"

    if [[ ! -r "$fichier_log" ]]; then
        echo "Erreur: impossible de lire $fichier_log" >&2
        return 1
    fi

    local seuil_date
    seuil_date=$(date -d "-${fenetre_minutes} minutes" '+%Y-%m-%d %H:%M')

    awk -v seuil="$seuil_date" '$0 >= seuil' "$fichier_log" | \
        grep -iE "error|erreur|critical" | \
        awk '{print $NF}' | \
        sort | uniq -c | sort -rn
}
```

Ce type de fonction combine directement les acquis du module 24 (pipeline `grep`/`awk`/`sort`/`uniq`) avec la rigueur du module 25 (validation d'entrée, `set -euo pipefail`, retour explicite en cas d'échec) — une synthèse concrète plutôt qu'une simple juxtaposition des deux modules.

### 7.4 Détection d'anomalie — seuil fixe vs tendance, et leurs pièges respectifs

**Détection par seuil fixe :** simple, mais rappel direct du piège déjà rencontré en UE2-M10 (TCP State Health Monitor) — un seuil arbitraire non calibré par rapport à la ressource réellement limitante produit des faux positifs ou, pire, manque de vrais problèmes. Un seuil doit toujours être justifié par rapport à une limite réelle du système (plage de ports disponibles, capacité disque totale), pas fixé "au jugé".

**Détection par tendance (fenêtre glissante) :** rappel direct des scripts de détection de fuite mémoire (UE1-M3) et de croissance TIME_WAIT (UE2-M10) — détecter une **croissance continue** sur plusieurs mesures successives, plutôt qu'une simple valeur instantanée dépassant un seuil, permet de distinguer un pic ponctuel normal d'une dérive réellement préoccupante.

**Le piège du redémarrage — rappel direct de plusieurs modules (UE1-M6, UE3-M19) :** tout mécanisme de détection stateful (comparant une mesure à une mesure précédente, ou à un fichier d'état) doit explicitement gérer le cas d'un redémarrage récent du système ou du service surveillé — un état de référence perdu ou réinitialisé ne doit jamais être interprété à tort comme une anomalie.

### 7.5 Intégration dans l'écosystème de planification (rappel UE3-M16, M19)

Un outil d'automatisation mature ne se contente pas d'exister comme script isolé — il s'intègre proprement :

```ini
# monitoring-app.service
[Unit]
Description=Vérification périodique de l'application

[Service]
Type=oneshot
ExecStart=/opt/monitoring/verif_app.sh
```

```ini
# monitoring-app.timer
[Unit]
Description=Déclenche la vérification toutes les 5 minutes

[Timer]
OnCalendar=*:0/5
Persistent=true

[Install]
WantedBy=timers.target
```

**Rappel direct de la comparaison cron/timer systemd du module UE3-M16 :** pour un outil de supervision critique, un timer systemd offre une journalisation intégrée via `journalctl`, particulièrement utile pour l'historique des exécutions et de leurs résultats — un choix cohérent avec la nature de l'outil développé dans ce module.

### 7.6 Réviser un script existant — méthodologie de relecture critique

Face à un script déjà écrit dans une UE précédente, une relecture méthodique à la lumière de l'UE4 suit une check-list naturelle :

1. **Quotation (M23)** : chaque variable est-elle correctement entre guillemets doubles ?
2. **Boucles et substitutions (M23)** : une éventuelle boucle `for` sur substitution de commande non protégée devrait-elle être remplacée par `while read -r` ou un pipeline direct ?
3. **Pipelines (M24)** : le traitement peut-il être simplifié ou fiabilisé par une meilleure combinaison d'outils (`grep`/`awk`/`sort`/`uniq`) ?
4. **Robustesse (M25)** : `set -euo pipefail` est-il présent ? Un `trap` de nettoyage est-il nécessaire et présent ? Les erreurs sont-elles envoyées sur stderr ?
5. **`shellcheck` (M23)** : le script passe-t-il sans avertissement significatif ?

## 8. Architecture

```
┌────────────────────────────────────────────────────────────┐
│  Outil d'automatisation                                          │
│                                                                   │
│  Collecte ──▶ Analyse ──▶ Rapport                                    │
│     │            │            │                                     │
│  M24 (pipelines) │        M25 (logging stdout/stderr)                │
│                M21/UE1/UE2/UE3 (logique métier de détection            │
│                déjà étudiée : seuils, tendances, cohérence)             │
│                                                                       │
│  Le tout robustifié par M25 (set -euo pipefail, trap)                    │
│  et intégré via un timer systemd (rappel UE3-M16/M19)                     │
└────────────────────────────────────────────────────────────┘
```

## 9. Fonctionnement interne

Ce module ne présente aucun nouveau mécanisme système — sa valeur ajoutée est la **discipline d'assemblage** de ce qui a déjà été appris. Chaque script de ce module, exécuté périodiquement via un timer systemd, s'exécute dans son propre cgroup (rappel UE3-M16), ses erreurs sont capturées par `journald` (rappel UE3-M19) si stderr est correctement utilisé (rappel M25), et son analyse de logs s'appuie sur les mêmes primitives de traitement de texte (rappel M24) qui, en interne, s'appuient sur les pipes du noyau (rappel UE1-M5).

## 10. Démonstration

```bash
# Exemple de vérification multi-catégorie, distinguant les trois niveaux de la section 7.2
verifier_service_complet() {
    local service="$1"
    local seuil_memoire_mo="${2:-500}"

    # Catégorie 1 : état actif
    if ! systemctl is-active --quiet "$service"; then
        echo "ETAT: inactif"
        return 1
    fi
    echo "ETAT: actif"

    # Catégorie 2 : cohérence de configuration (rappel UE3-M16)
    local pid
    pid=$(systemctl show -p MainPID --value "$service")
    if [[ "$pid" -eq 0 ]]; then
        echo "COHERENCE: aucun PID principal identifiable"
    fi

    # Catégorie 3 : seuil de performance
    local memoire_ko
    memoire_ko=$(ps -o rss= -p "$pid" 2>/dev/null || echo 0)
    local memoire_mo=$((memoire_ko / 1024))
    if (( memoire_mo > seuil_memoire_mo )); then
        echo "PERFORMANCE: mémoire élevée (${memoire_mo}Mo > ${seuil_memoire_mo}Mo)"
    else
        echo "PERFORMANCE: mémoire normale (${memoire_mo}Mo)"
    fi
}
```

## 11. Commandes / outils

*Ce module ne présente aucun nouvel outil — la compétence consiste à assembler avec discipline l'ensemble des modules 23 à 25 et des acquis des UE1-UE3.*

| Composante | Modules mobilisés |
|---|---|
| Syntaxe et robustesse du script | M23, M25 |
| Traitement de logs | M24 |
| Logique de détection (seuils, tendances) | UE1-M3, UE2-M10, UE3-M20/M21 |
| Intégration planifiée | UE3-M16, M19 |

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** une équipe hérite d'une collection de vingt scripts de supervision, écrits par différents membres au fil de plusieurs années, sans convention commune : certains n'ont pas de `set -e`, d'autres mélangent logique de collecte et de rapport dans une seule fonction géante, plusieurs ne gèrent pas le cas d'un redémarrage récent et produisent des faux positifs systématiques chaque lundi matin après la maintenance hebdomadaire.

**Analyse attendue :** ce scénario, bien que fictif dans sa forme, synthétise fidèlement plusieurs pièges déjà rencontrés séparément dans tout le programme (UE1-M4/UE3-M19 sur les états non persistants, UE3-M20 sur les faux "OK" masquant un vrai risque, ce module sur le mélange de responsabilités). La démarche de refonte recommandée applique directement la méthodologie de relecture critique de la section 7.6 : auditer systématiquement chaque script avec `shellcheck`, ajouter `set -euo pipefail` partout où absent, séparer collecte/analyse/rapport dans chaque script mélangeant ces responsabilités, et surtout, pour les faux positifs récurrents du lundi matin, identifier explicitement s'il s'agit d'un état de référence non persistant perdu au redémarrage (rappel direct UE3-M19) et corriger l'emplacement de stockage de cet état en conséquence. Cette refonte, bien que représentant un investissement de temps significatif, transforme une collection de scripts fragiles hérités en une véritable suite d'outils fiables — un travail qui rejoint directement l'esprit du projet intégrateur de ce module.

## 15. Sécurité

- Un outil de supervision mal sécurisé (script exécuté avec des privilèges excessifs, ou lisible/modifiable par des utilisateurs non autorisés) peut lui-même devenir un vecteur d'attaque — rappel direct des principes déjà établis en UE3-M14 (SUID, moindre privilège) appliqués spécifiquement aux scripts d'automatisation
- Un outil de détection d'anomalie constitue une pièce du dispositif de sécurité global d'une infrastructure (détection d'une dérive de comportement pouvant signaler une compromission, pas seulement une panne) — sa fiabilité (absence de faux positifs qui useraient la vigilance de l'équipe, absence de faux négatifs qui laisseraient passer un incident réel) est directement une question de posture de sécurité
- La séparation collecte/analyse/rapport (section 7.1) facilite également l'audit de sécurité d'un outil : il est plus simple de vérifier qu'une fonction de collecte n'exécute aucune opération dangereuse avec des données non validées, si cette responsabilité est clairement isolée du reste du code

## 16. Automatisation

Squelette d'outil de supervision synthétisant l'ensemble du module — architecture modulaire, robustesse M25, pipeline M24.

```bash
#!/usr/bin/env bash
set -euo pipefail

# --- Bibliothèque commune (rappel M25) ---
log_info() { echo "[INFO] $*"; }
log_erreur() { echo "[ERREUR] $*" >&2; }

# --- Collecte ---
collecter_etat_service() {
    local service="$1"
    systemctl is-active "$service" 2>/dev/null || echo "inactive"
}

collecter_erreurs_log() {
    local fichier="$1"
    local minutes="$2"
    local seuil_date
    seuil_date=$(date -d "-${minutes} minutes" '+%Y-%m-%d %H:%M')
    awk -v seuil="$seuil_date" '$0 >= seuil' "$fichier" 2>/dev/null | grep -ciE "error|critical" || echo 0
}

# --- Analyse ---
analyser_resultat() {
    local etat_service="$1"
    local nb_erreurs="$2"
    local seuil_erreurs="$3"

    if [[ "$etat_service" != "active" ]]; then
        echo "CRITIQUE"
        return
    fi
    if (( nb_erreurs > seuil_erreurs )); then
        echo "ALERTE"
        return
    fi
    echo "OK"
}

# --- Rapport ---
main() {
    local service="${1:?Usage: $0 <service> <fichier_log> [seuil_erreurs]}"
    local fichier_log="${2:?Usage: $0 <service> <fichier_log> [seuil_erreurs]}"
    local seuil="${3:-10}"

    local etat nb_erreurs resultat
    etat=$(collecter_etat_service "$service")
    nb_erreurs=$(collecter_erreurs_log "$fichier_log" 60)
    resultat=$(analyser_resultat "$etat" "$nb_erreurs" "$seuil")

    case "$resultat" in
        OK) log_info "Service $service: OK (état=$etat, erreurs_1h=$nb_erreurs)" ;;
        ALERTE) log_erreur "Service $service: ALERTE (erreurs_1h=$nb_erreurs > seuil=$seuil)" ;;
        CRITIQUE) log_erreur "Service $service: CRITIQUE (état=$etat)" ;;
    esac
}

main "$@"
```

## 17. Troubleshooting

*Ce module étant lui-même un module de synthèse, ses scénarios complexes sont traités intégralement dans les Parties II et III plutôt que résumés ici.*

## 18. Synthèse

- Un outil de supervision mature sépare collecte, analyse et rapport — jamais mélangés dans un bloc unique
- Trois catégories de vérification (état actif, cohérence de configuration, seuil de performance) doivent être distinguées, jamais fusionnées en un indicateur binaire unique
- L'analyse de logs automatisée combine directement les pipelines du module 24 avec la robustesse du module 25
- La détection d'anomalie par seuil doit être calibrée par rapport à une limite réelle ; la détection par tendance doit gérer explicitement le cas d'un redémarrage récent
- Un outil d'automatisation s'intègre proprement dans l'écosystème de planification déjà étudié (timer systemd de préférence pour un outil critique)
- Réviser un script existant suit une check-list naturelle : quotation, boucles robustes, pipelines optimisés, robustesse `set -euo pipefail`/`trap`, validation `shellcheck`

## 19. Questions d'évaluation

**QCM**
1. Séparer collecte, analyse et rapport dans un outil de supervision permet notamment :
 a) d'accélérer l'exécution systématiquement b) de tester chaque partie isolément et de faire évoluer l'outil plus facilement c) de réduire la taille du fichier uniquement d) d'éviter tout usage de `set -e`

2. Un seuil de détection d'anomalie non calibré par rapport à une limite réelle du système risque de :
 a) toujours être plus précis b) produire des faux positifs ou manquer de vrais problèmes c) accélérer la détection sans inconvénient d) remplacer la nécessité d'une supervision

**Questions ouvertes**
- Expliquez pourquoi un indicateur binaire unique "OK"/"KO" est insuffisant pour un outil de supervision mature, avec un exemple concret.
- Décrivez la méthodologie de relecture critique d'un script existant à la lumière de l'UE4, en au moins 4 points.

**Exercice technique**
- Esquissez la structure (noms de fonctions et leur rôle, sans code complet) d'un outil de supervision suivant l'architecture collecte/analyse/rapport pour vérifier l'espace disque disponible.

**Problème de diagnostic**
- Un outil de détection de tendance produit des faux positifs systématiques après chaque redémarrage du serveur. Proposez la cause la plus probable et la correction.

## 20. Mini-projet — voir Projet intégrateur de fin d'UE4

Ce module étant la clôture de l'UE4, son mini-projet est directement absorbé par le projet intégrateur de fin d'unité (Projet 4 du programme général : *Automatiser l'administration Linux avec Bash*), qui reprend et étend l'ensemble des acquis des modules 23 à 26.

---

# PARTIE II — TD

*Ce TD s'appuie sur des scripts réalistes tirés en esprit des UE précédentes, à analyser et améliorer à la lumière de l'ensemble de l'UE4.*

### Niveau 1 — Compréhension

**Exercice 1.1**
Décrivez les trois responsabilités (collecte, analyse, rapport) d'un outil de supervision modulaire, avec un exemple de fonction pour chacune.

**Exercice 1.2**
Citez les trois catégories de vérification de service vues en cours, avec un exemple pour chacune.

**Exercice 1.3**
Expliquez pourquoi un mécanisme de détection par tendance doit gérer explicitement le cas d'un redémarrage récent du système.

### Niveau 2 — Application

**Exercice 2.1**
Écrivez la structure (fichiers `.service` et `.timer`) permettant d'exécuter un script de supervision `/opt/monitoring/check.sh` toutes les 10 minutes via systemd.

**Exercice 2.2**
Donnez, pour un script existant supposé ne pas utiliser `set -euo pipefail`, la ligne à ajouter et sa position correcte dans le script.

**Exercice 2.3**
Un script de supervision doit vérifier si un fichier de log contient plus de 5 occurrences du mot "critical" dans la dernière heure. Écrivez le pipeline correspondant (réutilisation directe du module 24).

### Niveau 3 — Analyse

**Exercice 3.1**
Un outil de supervision rapporte un indicateur binaire unique "OK" pour un service, alors qu'en réalité le service est actif mais consomme une mémoire anormalement élevée, proche d'un seuil critique. Expliquez pourquoi cette conception est insuffisante, et comment la refonte en trois catégories (section 7.2) résoudrait ce problème.

**Exercice 3.2**
Un script hérité d'une UE précédente utilise `for fichier in $(find /var/log -name "*.log")` pour itérer sur des fichiers de log, puis les analyse un par un avec `grep`. Appliquez la méthodologie de relecture critique du cours (section 7.6) à ce script : identifiez au moins deux améliorations distinctes à apporter, en citant le module concerné pour chacune.

### Niveau 4 — Problème complexe (scénario intégrateur)

**Exercice 4.1**
Une équipe hérite d'un script de supervision `verif_espace.sh` qui fonctionne ainsi : il compare l'espace disque actuel à une valeur stockée dans `/tmp/dernier_espace.txt` lors de l'exécution précédente, et alerte si l'espace libre a diminué de plus de 10% depuis cette dernière mesure. L'équipe constate que ce script génère une alerte systématique chaque lundi matin après la maintenance hebdomadaire du serveur (qui inclut un redémarrage), sans qu'aucun problème réel d'espace disque ne soit jamais constaté à ce moment précis. En vous appuyant sur l'ensemble des modules de l'UE4 (et les rappels transversaux de l'UE1/UE3 mobilisés dans ce module de synthèse), menez une analyse complète de ce script : identifiez la cause racine du faux positif récurrent, proposez une correction de l'emplacement de stockage de l'état, et proposez une seconde amélioration indépendante appliquant la méthodologie de relecture critique du cours à ce même script (au moins une amélioration de robustesse ou de structure, distincte de la correction de l'emplacement de stockage).

---

## Correction du TD

**1.1** — Collecte : une fonction qui récupère un état brut du système, par exemple `collecter_espace_disque()` qui exécute `df` et en extrait le pourcentage d'utilisation. Analyse : une fonction qui interprète cet état collecté, par exemple `analyser_seuil_disque(pourcentage, seuil)` qui compare la valeur collectée à un seuil et retourne une catégorie de résultat (OK/ALERTE/CRITIQUE). Rapport : une fonction qui communique le résultat de l'analyse, par exemple `rapporter_resultat(categorie, details)` qui formate et journalise (ou envoie une alerte) selon la catégorie obtenue.

**1.2** — État actif : le service tourne-t-il, vérifiable via `systemctl is-active` (exemple : un service web arrêté). Cohérence de configuration : l'état réellement chargé correspond-il à ce qui est attendu, par exemple un fichier d'unité modifié sans `daemon-reload` exécuté depuis (rappel UE3-M16), ou un noyau installé mais non actif (rappel UE3-M20). Seuil de performance : le service fonctionne-t-il dans des paramètres acceptables, par exemple une consommation mémoire dépassant un seuil configuré (rappel UE1-M3).

**1.3** — Un mécanisme de détection par tendance compare généralement une mesure actuelle à un état de référence stocké lors d'une exécution précédente. Un redémarrage récent du système (ou du service surveillé) peut réinitialiser cet état de référence (perte d'un fichier d'état non persistant, remise à zéro d'un compteur, changement légitime d'une valeur mesurée après le redémarrage) — sans gestion explicite de ce cas, le mécanisme de détection interpréterait à tort cette réinitialisation normale et attendue comme une anomalie réelle, générant un faux positif systématique à chaque redémarrage, un piège déjà rencontré à plusieurs reprises dans le programme (UE1, UE3-M19).

**2.1** — 
```ini
# /etc/systemd/system/check-app.service
[Unit]
Description=Supervision applicative
[Service]
Type=oneshot
ExecStart=/opt/monitoring/check.sh
```
```ini
# /etc/systemd/system/check-app.timer
[Unit]
Description=Déclenche la supervision toutes les 10 minutes
[Timer]
OnCalendar=*:0/10
Persistent=true
[Install]
WantedBy=timers.target
```

**2.2** — La ligne `set -euo pipefail` doit être ajoutée immédiatement après la ligne shebang (`#!/usr/bin/env bash`), avant toute autre instruction du script, pour que la robustesse s'applique dès la première commande réellement exécutée.

**2.3** — 
```bash
nb_occurrences=$(awk -v seuil="$(date -d '-60 minutes' '+%Y-%m-%d %H:%M')" '$0 >= seuil' fichier.log | grep -c "critical")
if (( nb_occurrences > 5 )); then
    echo "Alerte: $nb_occurrences occurrences de 'critical' dans la dernière heure"
fi
```

**3.1** — Un indicateur binaire unique "OK" pour ce service masque une information pourtant significative et potentiellement urgente : la consommation mémoire proche d'un seuil critique n'est reflétée nulle part dans ce résultat, alors qu'elle constitue un signal précurseur important (rappel direct de la logique de détection de fuite mémoire déjà étudiée en UE1-M3) qui mériterait une catégorie distincte de suivi ou d'alerte préventive, avant même que la situation ne dégénère en une véritable panne du service. La refonte en trois catégories (section 7.2) résoudrait ce problème en rapportant séparément : "ETAT: actif" (catégorie 1, correcte et rassurante en apparence) et "PERFORMANCE: mémoire élevée, proche du seuil critique" (catégorie 3, révélant le risque réel actuellement masqué par l'indicateur unique) — permettant à l'équipe de surveillance d'agir de façon préventive sur ce signal, plutôt que de découvrir le problème seulement lorsque le service finit par effectivement tomber en panne.

**3.2** — Deux améliorations distinctes possibles : (1) **Module 23** — `for fichier in $(find /var/log -name "*.log")` est soumis au découpage en mots sur la substitution de commande non protégée, un nom de fichier de log contenant un espace serait incorrectement fragmenté ; la correction robuste consisterait à utiliser `find /var/log -name "*.log" -print0 | while IFS= read -r -d '' fichier; do ... done`, comme déjà vu au module 23. (2) **Module 24/25** — le traitement "un par un avec grep" à l'intérieur d'une boucle Bash pourrait potentiellement être remplacé, selon le besoin exact, par un traitement direct de l'ensemble des fichiers concernés en une seule invocation de `grep -r` ou d'un pipeline combiné, souvent plus efficace et plus simple à maintenir qu'une boucle Bash explicite appelant `grep` répétitivement — un exemple direct du principe déjà souligné en cours qu'un pipeline bien conçu peut souvent remplacer avantageusement une boucle Bash manuelle pour ce type de traitement de texte.

**4.1** — Analyse complète du script hérité :

**Cause racine du faux positif récurrent :** le script stocke son état de référence dans `/tmp/dernier_espace.txt` — or `/tmp` est, sur de nombreuses distributions, **nettoyé au redémarrage** (rappel direct d'un piège déjà rencontré à plusieurs reprises dans le programme, notamment en UE1 et en UE3-M19 pour un mécanisme de heartbeat similaire). Après le redémarrage de maintenance du lundi matin, ce fichier d'état n'existe plus ; le script, lors de sa première exécution suivante, ne trouve donc aucune valeur de référence antérieure valide — selon son implémentation exacte, il interprète probablement cette absence comme une variation de 100% (ou toute autre valeur extrême) par rapport à une référence implicitement traitée comme zéro ou absente, déclenchant systématiquement une fausse alerte de "diminution de plus de 10%", sans rapport avec l'état réel du disque.

**Correction de l'emplacement de stockage :** déplacer le fichier d'état vers un emplacement réellement persistant à travers les redémarrages, par exemple `/var/lib/monitoring/dernier_espace.txt` (en créant ce répertoire si nécessaire, avec des permissions appropriées), plutôt que `/tmp`. Une amélioration complémentaire recommandée serait d'ajouter une détection explicite du cas "premier lancement / fichier d'état absent" (que ce soit après ce changement d'emplacement ou pour toute autre raison), traitant ce cas comme une simple initialisation de la référence plutôt que comme une anomalie à signaler, indépendamment de la persistance de l'emplacement choisi.

**Seconde amélioration indépendante (méthodologie de relecture critique) :** au-delà de la correction de l'emplacement de stockage, une revue du script à la lumière du **module 25** vérifierait la présence de `set -euo pipefail` (garantissant que toute erreur de lecture/écriture du fichier d'état, ou tout échec de la commande `df` sous-jacente, soit correctement détecté plutôt que silencieusement ignoré) et d'une gestion d'erreur explicite en cas d'échec de lecture du fichier d'état (distinguant clairement, dans le code lui-même, le cas légitime "fichier absent au premier lancement" du cas anormal "fichier présent mais illisible pour une autre raison, par exemple un problème de permissions") — deux améliorations de robustesse distinctes de la correction de fond sur l'emplacement de stockage, mais tout aussi importantes pour la fiabilité globale et durable de cet outil de supervision.

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur système junior pour construire un outil de supervision complet et modulaire depuis zéro, en appliquant l'intégralité des acquis de l'UE4, puis pour réviser un script hérité en appliquant la méthodologie de relecture critique du module.

## Architecture

Une machine Linux avec Bash, `shellcheck`, systemd disponibles.

## Objectifs
- Construire un outil de supervision suivant l'architecture collecte/analyse/rapport
- Intégrer cet outil dans un timer systemd
- Appliquer la méthodologie de relecture critique à un script hérité fourni
- Corriger le scénario intégrateur de l'exercice 4.1 du TD, reproduit concrètement

## Préparation

```bash
which bash shellcheck systemctl 2>/dev/null
```

## Étapes

### Étape 1 — Construire l'outil de supervision modulaire

```bash
mkdir -p /opt/lab_monitoring
cat << 'EOF' > /opt/lab_monitoring/superviser_disque.sh
#!/usr/bin/env bash
set -euo pipefail

# --- Collecte ---
collecter_utilisation_disque() {
    local point_montage="$1"
    df -h "$point_montage" | awk 'NR==2 {print $5}' | tr -d '%'
}

# --- Analyse ---
analyser_utilisation() {
    local pourcentage="$1"
    local seuil_alerte="$2"
    local seuil_critique="$3"

    if (( pourcentage >= seuil_critique )); then
        echo "CRITIQUE"
    elif (( pourcentage >= seuil_alerte )); then
        echo "ALERTE"
    else
        echo "OK"
    fi
}

# --- Rapport ---
rapporter() {
    local categorie="$1"
    local point_montage="$2"
    local pourcentage="$3"

    case "$categorie" in
        OK) echo "[INFO] $point_montage: ${pourcentage}% - OK" ;;
        ALERTE) echo "[ALERTE] $point_montage: ${pourcentage}% - seuil d'alerte dépassé" >&2 ;;
        CRITIQUE) echo "[CRITIQUE] $point_montage: ${pourcentage}% - seuil critique dépassé" >&2 ;;
    esac
}

main() {
    local point_montage="${1:-/}"
    local seuil_alerte="${2:-80}"
    local seuil_critique="${3:-90}"

    local pourcentage categorie
    pourcentage=$(collecter_utilisation_disque "$point_montage")
    categorie=$(analyser_utilisation "$pourcentage" "$seuil_alerte" "$seuil_critique")
    rapporter "$categorie" "$point_montage" "$pourcentage"
}

main "$@"
EOF
chmod +x /opt/lab_monitoring/superviser_disque.sh

shellcheck /opt/lab_monitoring/superviser_disque.sh

/opt/lab_monitoring/superviser_disque.sh / 1 2   # seuils volontairement bas pour déclencher CRITIQUE en démonstration
```
*Question :* `shellcheck` signale-t-il des avertissements sur ce script ? Si oui, corrigez-les. Le script respecte-t-il la séparation collecte/analyse/rapport vue en cours ?

### Étape 2 — Intégration systemd

```bash
sudo tee /etc/systemd/system/lab-superviser-disque.service << 'EOF'
[Unit]
Description=Supervision de l'espace disque (laboratoire)

[Service]
Type=oneshot
ExecStart=/opt/lab_monitoring/superviser_disque.sh / 80 90
EOF

sudo tee /etc/systemd/system/lab-superviser-disque.timer << 'EOF'
[Unit]
Description=Déclenche la supervision disque toutes les 2 minutes (accéléré pour le TP)

[Timer]
OnUnitActiveSec=2min
Persistent=true

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now lab-superviser-disque.timer
systemctl list-timers | grep lab-superviser

sleep 5
sudo systemctl start lab-superviser-disque.service
journalctl -u lab-superviser-disque.service -b --no-pager | tail -10
```

### Étape 3 — Réviser un script hérité (méthodologie de relecture critique)

```bash
cat << 'EOF' > /opt/lab_monitoring/script_herite.sh
#!/bin/bash
fichier=/tmp/etat_precedent.txt
espace_actuel=$(df / | tail -1 | awk '{print $5}' | tr -d '%')
if [ -f $fichier ]; then
    espace_precedent=$(cat $fichier)
    diff=$((espace_actuel - espace_precedent))
    if [ $diff -gt 10 ]; then
        echo alerte
    fi
fi
echo $espace_actuel > $fichier
EOF

shellcheck /opt/lab_monitoring/script_herite.sh
```
*À vous de jouer :* appliquez la méthodologie de relecture critique (section 7.6 du cours) à ce script. Produisez une version corrigée traitant à la fois les avertissements `shellcheck` ET le piège de persistance dans `/tmp` (rappel direct de l'exercice 4.1 du TD).

### Étape 4 — Reproduire et corriger le scénario du faux positif au redémarrage

```bash
# Simuler le comportement du script hérité après un "redémarrage" (nettoyage de /tmp)
rm -f /tmp/etat_precedent.txt
bash /opt/lab_monitoring/script_herite.sh   # première exécution, pas d'alerte possible (fichier absent)
echo "Résultat après nettoyage simulé du redémarrage: $(cat /tmp/etat_precedent.txt 2>/dev/null || echo 'aucun fichier')"

# Avec votre version corrigée (étape 3), reproduisez le même test :
# rm -f /var/lib/lab_monitoring/etat_precedent.txt (ou votre nouvel emplacement choisi)
# et confirmez qu'aucune fausse alerte n'est générée au premier lancement après cette "réinitialisation"
```

## Commandes de référence
`shellcheck`, `systemctl daemon-reload`/`enable --now`, `journalctl -u`, `systemctl list-timers`

## Résultats attendus
- Script de l'étape 1 validé par `shellcheck`, avec démonstration des trois catégories de résultat (OK/ALERTE/CRITIQUE)
- Timer actif et exécution confirmée via `journalctl` (étape 2)
- Version corrigée du script hérité, traitant `shellcheck` ET la persistance (étape 3)
- Preuve qu'aucune fausse alerte n'est générée après réinitialisation de l'état sur la version corrigée (étape 4)

## Questions

1. Dans l'étape 1, pourquoi la fonction `rapporter` envoie-t-elle les messages ALERTE/CRITIQUE sur stderr plutôt que stdout, alors que le message OK reste sur stdout ? Reliez votre réponse aux modules 24 et 25.
2. Dans l'étape 3, listez précisément tous les avertissements `shellcheck` détectés sur le script hérité, et pour chacun, indiquez à quel module de l'UE4 il se rapporte.
3. Le déplacement de l'emplacement de stockage de l'état (étape 3/4) suffit-il, à lui seul, à rendre ce script pleinement robuste selon l'ensemble des critères de l'UE4 ? Quelles autres améliorations, indépendantes de ce problème précis, resteraient recommandées ?

## Troubleshooting (panne à diagnostiquer — méta-scénario de clôture)

**Situation fournie :** après avoir développé et déployé l'outil de l'étape 1 en timer systemd, un étudiant constate, une semaine plus tard, que l'outil ne s'est exécuté que deux fois au lieu des centaines de fois attendues sur cette période (avec un intervalle de 2 minutes configuré pour le TP).

**Démarche attendue (synthèse complète de la méthodologie de troubleshooting de l'UE3-M22, appliquée ici à un outil de l'UE4) :**
1. Symptôme : nombre d'exécutions très inférieur à l'attendu sur la période
2. Hypothèses : (a) le timer a pu être désactivé ou son état "failed" atteint après un premier échec non anticipé (rappel UE3-M16, `StartLimitBurst`) ; (b) la machine de laboratoire a pu être éteinte ou suspendue une grande partie de cette période, réduisant mécaniquement le nombre d'occasions d'exécution ; (c) une erreur de configuration du timer lui-même (mauvaise syntaxe `OnUnitActiveSec`) aurait pu produire un intervalle bien plus long que prévu
3. Collecte : `systemctl status lab-superviser-disque.timer` (état actuel du timer), `systemctl list-timers` (prochaine exécution prévue et dernière exécution passée), `journalctl -u lab-superviser-disque.service -b` (historique complet des exécutions et éventuels échecs)
4. Test des hypothèses : si le timer est en état `failed`, l'hypothèse (a) est confirmée — vérifier alors si le service associé a échoué de façon répétée avant d'atteindre cet état ; si l'historique système confirme des périodes d'extinction de la VM de laboratoire, l'hypothèse (b) est plus probable et le comportement, bien qu'inattendu par l'étudiant, serait alors parfaitement normal et non un dysfonctionnement réel de l'outil lui-même
5. Isolation de la cause racine : selon l'hypothèse confirmée, la cause racine varie — un vrai bug de configuration (a, c) nécessite une correction technique ; une simple extinction de VM (b) ne révèle aucun problème réel de l'outil, seulement une limite inhérente à l'environnement de laboratoire
6. Correction : selon le cas, corriger la syntaxe du timer, ajuster les garde-fous de redémarrage automatique du service associé, ou simplement documenter que le comportement observé est cohérent avec l'historique réel de disponibilité de la machine
7. Vérification et documentation : ce méta-scénario, en clôture de l'UE4, démontre que la méthodologie de troubleshooting en 9 étapes (UE3-M22) reste pleinement pertinente et applicable à tout outil développé ultérieurement, y compris ceux construits avec toute la rigueur enseignée dans cette même unité — la rigueur de conception d'un outil ne dispense jamais de la rigueur de diagnostic face à un comportement inattendu de cet outil lui-même

## Validation

Le TP est validé si l'étudiant fournit :
- Les 4 résultats/versions demandés
- Des réponses argumentées aux 3 questions
- Une démarche complète et methodique pour le méta-scénario de troubleshooting, démontrant l'intégration réussie de la méthodologie UE3-M22 avec les acquis techniques de l'UE4

# UE3 — Module 22 : Troubleshooting Méthodique

*Module de synthèse et de clôture de l'UE3 — mobilise l'ensemble des modules 14 à 21, ainsi que les fondations de l'UE1.*

---

# PARTIE I — COURS

## 1. Titre du cours
Méthodologie professionnelle de troubleshooting système Linux

## 2. Niveau
Élève ingénieur — cycle ingénieur, clôture de l'UE3 administration Linux (S2)

## 3. Durée
CM : 3h (module de synthèse orienté pratique — +3h TD, +6h TP en Parties II et III, volume TP élevé car module intégrateur)

## 4. Prérequis
- L'intégralité de l'UE3 (modules 14 à 21) : utilisateurs/permissions, stockage/LVM, systemd, réseau/pare-feu, SSH, logs/cron, paquets/noyau, supervision/sauvegarde
- L'intégralité de l'UE1 : ce module n'introduit aucun nouveau concept technique, il structure et applique tout ce qui a déjà été appris

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Appliquer une méthodologie de troubleshooting structurée en 9 étapes, de façon disciplinée plutôt qu'intuitive
- Diagnostiquer une panne de service en mobilisant systemd, les logs et les dépendances applicatives
- Diagnostiquer une saturation disque en mobilisant stockage, LVM et les causes indirectes déjà étudiées (fichiers supprimés ouverts, logs non tournés)
- Diagnostiquer une panne réseau en mobilisant les couches OSI, ARP, routage et pare-feu
- Distinguer un symptôme d'une cause, et une cause immédiate d'une cause racine
- Documenter un incident de façon à en tirer une amélioration durable, pas seulement une résolution ponctuelle

## 6. Motivation

Chaque module de l'UE3 a enseigné un domaine technique précis — mais un incident réel en production ne se présente jamais avec une étiquette indiquant "ceci est un problème de systemd" ou "ceci est un problème de réseau". Un symptôme observé ("le service ne répond plus") peut avoir sa cause dans n'importe lequel des domaines étudiés, voire à l'intersection de plusieurs d'entre eux — un service qui ne démarre plus à cause d'un disque plein, lui-même causé par des logs non tournés, eux-mêmes non tournés à cause d'une tâche cron silencieusement défaillante.

Ce module ne présente aucune nouvelle commande — il structure la **démarche** qui permet de naviguer efficacement à travers tout ce qui a été appris, sans se perdre ni sauter prématurément à une conclusion non vérifiée (l'erreur méthodologique déjà illustrée à plusieurs reprises dans les modules précédents, notamment au module 21). C'est la compétence la plus transférable et la plus recherchée chez un ingénieur système : pas la connaissance de chaque commande individuellement, mais la capacité à les orchestrer sous pression face à un problème inconnu.

## 7. Concepts fondamentaux

### 7.1 La méthodologie en 9 étapes

```
1. Identifier le symptôme
2. Formuler des hypothèses
3. Collecter les informations
4. Tester les hypothèses
5. Isoler la cause
6. Corriger
7. Vérifier
8. Documenter
9. Mettre en place une prévention
```

**1. Identifier le symptôme** — Décrire précisément et factuellement ce qui est observé, sans interprétation prématurée. "Le site est lent" est vague ; "les requêtes HTTP vers `/api/commandes` prennent plus de 5 secondes depuis 14h32" est un symptôme exploitable. Cette étape échoue souvent par précipitation : sauter directement à une hypothèse sans avoir clairement caractérisé le symptôme observé.

**2. Formuler des hypothèses** — Lister, sans encore les tester, les causes plausibles compatibles avec le symptôme observé — en s'appuyant sur la connaissance des domaines techniques concernés (rappel direct de chaque module de l'UE3). Une bonne pratique consiste à formuler **plusieurs** hypothèses distinctes avant de privilégier la première venue à l'esprit (rappel de l'erreur du module 21 : conclure prématurément à partir d'un seul indicateur).

**3. Collecter les informations** — Rassembler les données factuelles nécessaires pour évaluer chaque hypothèse : logs (`journalctl`, rappel M16/M19), métriques de performance (`iostat`/`vmstat`, rappel M21), état des services (`systemctl status`, rappel M16), état réseau (rappel UE2 et M17), état du stockage (`df`/`lsblk`, rappel M15).

**4. Tester les hypothèses** — Confronter chaque hypothèse formulée aux informations collectées, en cherchant activement à la **réfuter** autant qu'à la confirmer (rappel de la démarche déjà appliquée en UE2-M10, exercice 4.1, où l'absence de fenêtre nulle permettait d'exclure une cause précise) — une bonne hypothèse de travail est une hypothèse qu'on a activement cherché à invalider sans y parvenir, pas simplement la première qui semblait plausible.

**5. Isoler la cause** — Une fois une hypothèse confirmée, remonter si possible jusqu'à la **cause racine**, pas seulement la cause immédiate visible. Distinction cruciale : la cause immédiate d'un service en panne peut être "disque plein" ; la cause racine peut être "les logs de ce service ne sont jamais tournés faute de configuration logrotate" (rappel direct M19) — corriger uniquement la cause immédiate (libérer de l'espace) sans traiter la cause racine garantit la récurrence de l'incident.

**6. Corriger** — Appliquer le correctif approprié à la cause identifiée (immédiate ET racine si possible), en respectant les précautions déjà enseignées à chaque module concerné (ex : `daemon-reload` après modification d'unité, M16 ; validation syntaxique avant redémarrage de pare-feu ou SSH, M17/M18).

**7. Vérifier** — Confirmer explicitement que le symptôme initial (étape 1) a bien disparu, avec une preuve concrète (pas une simple impression) — rappel du principe déjà établi à plusieurs modules : une absence apparente de problème n'est pas une preuve de résolution sans vérification active.

**8. Documenter** — Consigner l'incident : symptôme, hypothèses testées (y compris celles réfutées, précieuses pour de futures investigations similaires), cause racine identifiée, correction appliquée, durée de résolution — une documentation absente prive l'organisation de tout apprentissage collectif de l'incident (rappel du principe de documentation déjà souligné à plusieurs reprises dans le programme).

**9. Mettre en place une prévention** — Au-delà de la correction ponctuelle, identifier une mesure structurelle empêchant la récurrence : un mécanisme de heartbeat (M19/M21), une alerte de supervision proactive (M21), un garde-fou automatique (M17/M18) — cette étape transforme un incident isolé en une amélioration durable du système, cohérente avec l'esprit de chaque projet développé au fil de l'UE3.

### 7.2 Distinguer symptôme, cause immédiate et cause racine

```
Symptôme observé:        "Le service web ne répond plus"
        │
        ▼
Cause immédiate:         "Le service systemd est en état failed"
        │
        ▼
Cause plus profonde:      "Le disque / est plein, empêchant l'écriture
                            nécessaire au démarrage du service"
        │
        ▼
Cause racine:              "Les logs applicatifs de ce service ne sont
                             jamais tournés, faute de configuration
                             logrotate associée (rappel M19)"
```

**Chaque niveau de cette chaîne appelle une correction différente**, et seule une correction de la cause racine empêche réellement la récurrence — corriger uniquement le symptôme (redémarrer le service) ou la cause immédiate (libérer de l'espace disque manuellement) résout l'incident du jour, mais garantit sa réapparition prochaine si la cause racine n'est jamais traitée.

### 7.3 Les trois scénarios canoniques de ce module

Ce module structure sa pratique autour de trois grandes familles d'incidents, chacune mobilisant un ensemble différent (mais parfois recoupant) de modules de l'UE3 :

**a) Panne de service** — mobilise principalement systemd (M16), les logs (M19), potentiellement les permissions (M14) si le service ne peut accéder à une ressource, et le réseau (M17) si le service dépend d'une connectivité externe.

**b) Disque plein** — mobilise principalement le stockage/LVM (M15), les logs et leur rotation (M19), potentiellement les sauvegardes (M21) si elles-mêmes consomment l'espace de façon incontrôlée, et rejoint directement le piège UE1-M4 (fichier supprimé mais toujours ouvert).

**c) Réseau en panne ("réseau down")** — mobilise l'ensemble de l'UE2 (couches OSI, ARP, routage, DNS), le pare-feu (M17), et potentiellement SSH (M18) si c'est l'accès distant lui-même qui est affecté, rendant le diagnostic plus difficile (accès console physique/hors-bande parfois nécessaire).

**Pourquoi ces trois scénarios spécifiquement ?** Ce sont, empiriquement, parmi les catégories d'incidents les plus fréquentes en administration système de production — et chacune illustre bien la nécessité de mobiliser plusieurs domaines techniques simultanément plutôt qu'un seul isolément, exactement la compétence transversale que ce module cherche à consolider.

### 7.4 Panne de service — arbre de diagnostic type

```
Service en échec (systemctl status → failed)
    │
    ├─▶ journalctl -u <service> -b  → message d'erreur explicite ?
    │       │
    │       ├─▶ Erreur de configuration → corriger, daemon-reload, redémarrer (rappel M16)
    │       ├─▶ Erreur de permission → vérifier User=, propriétaire des fichiers (rappel M14)
    │       ├─▶ Dépendance indisponible → vérifier After=/Requires=, état de la dépendance (rappel M16)
    │       └─▶ Erreur disque/réseau → basculer vers l'arbre de diagnostic correspondant
    │
    └─▶ Aucune erreur explicite → vérifier StartLimitBurst épuisé (rappel M16),
         systemctl reset-failed après correction de la cause sous-jacente
```

### 7.5 Disque plein — arbre de diagnostic type

```
df -h révèle une partition proche de 100%
    │
    ├─▶ du -sh /* (ou sous-répertoires) → identifie le répertoire volumineux
    │       │
    │       ├─▶ Correspond à des fichiers légitimes en croissance → LVM extensible ? (rappel M15)
    │       └─▶ Ne correspond à aucun fichier visible en quantité suffisante
    │               │
    │               └─▶ lsof | grep deleted → fichier supprimé mais toujours ouvert (rappel UE1-M4)
    │
    └─▶ Logs applicatifs anormalement volumineux
            │
            └─▶ Configuration logrotate absente ou mal configurée (rappel M19)
```

### 7.6 Réseau en panne — arbre de diagnostic type (méthodologie couche par couche)

```
Connectivité perdue vers/depuis une machine
    │
    ├─▶ Couche 1/2 : ip link (interface UP ?), câble/interface physique (rappel UE2-M8)
    ├─▶ Couche 2/3 : ip neigh (ARP résolu ?), conflit d'adresse possible (rappel UE2-M8/M9)
    ├─▶ Couche 3 : ip route (route par défaut cohérente ?) (rappel UE2-M9)
    ├─▶ Couche 3 (ICMP) : ping — mais rappel du piège UE2-M7 : un ping filtré ≠ panne réelle
    ├─▶ Couche 4 : test de port applicatif (rappel UE2-M9/M10), refus explicite vs timeout
    └─▶ Pare-feu local : iptables -L -v -n (règle bloquante ? rappel M17)
```

**Note méthodologique transversale :** ces trois arbres ne sont pas des check-lists à suivre mécaniquement dans l'ordre à chaque incident, mais des **cartes mentales** orientant la collecte d'informations (étape 3) et la formulation d'hypothèses (étape 2) — un ingénieur expérimenté priorise intuitivement les branches les plus probables selon le contexte, une intuition qui se construit précisément par la pratique répétée de cette méthodologie explicite.

## 8. Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  MÉTHODOLOGIE EN 9 ÉTAPES                    │
│                                                               │
│  1.Symptôme → 2.Hypothèses → 3.Collecte → 4.Test → 5.Isoler   │
│         ↑                                          │         │
│         │                                          ▼         │
│  9.Prévention ← 8.Documenter ← 7.Vérifier ← 6.Corriger         │
└─────────────────────────────────────────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        ▼                  ▼                  ▼
  Panne de service    Disque plein       Réseau down
  (M16, M19, M14)     (M15, M19, UE1-M4)  (UE2, M17, M18)
```

## 9. Fonctionnement interne

Ce module ne présente pas de nouveau mécanisme noyau — il s'appuie sur la compréhension déjà acquise que chaque symptôme observable en surface (un service qui ne répond pas, un disque saturé, une connexion qui échoue) est l'expression visible d'un état interne précis du système (un cgroup en échec, un inode épuisé ou un espace disque réellement occupé, une table de routage ou un cache ARP incohérent) — la méthodologie de ce module est précisément la discipline qui relie systématiquement l'observable à l'interne, plutôt que de rester au niveau du symptôme.

## 10. Démonstration

```bash
# Kit de premiers réflexes multi-domaines (à adapter selon le symptôme initial)
systemctl --failed                      # services en échec
df -h                                     # état du stockage
ip addr show; ip route show                # état réseau de base
journalctl -p err -b --since "1 hour ago"    # erreurs récentes toutes unités confondues
```

## 11. Commandes / outils

*Ce module ne présente aucun nouvel outil — la compétence consiste à mobiliser à bon escient l'ensemble déjà appris.* Voir les tableaux de commandes des modules 14 à 21 pour référence complète.

| Domaine | Commandes de référence (rappel) |
|---|---|
| Service | `systemctl status/cat`, `journalctl -u` (M16) |
| Stockage | `df -h`, `du -sh`, `lsof \| grep deleted`, `lvs`/`vgs` (M15, UE1-M4) |
| Réseau | `ip link/addr/neigh/route`, `ping`, `traceroute`, `iptables -L` (UE2, M17) |
| Performance | `vmstat`, `iostat -x`, `sar` (M21) |
| Paquets/noyau | `uname -r`, `dpkg --list`, `apt list --upgradable` (M20) |

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** un service de traitement de commandes en ligne cesse de répondre un vendredi soir. L'équipe d'astreinte, sous pression temporelle, redémarre immédiatement le service sans investigation — celui-ci redémarre normalement et le service semble rétabli. Le lundi matin, le même incident se reproduit.

**Analyse attendue :** ce scénario illustre l'échec de ne pas suivre la méthodologie complète, en particulier les étapes 5 (isoler la cause racine, pas seulement le symptôme) et 9 (prévention). Le redémarrage du vendredi a traité le **symptôme** (service non répondant) sans jamais identifier la **cause** — appliquer rétrospectivement la méthodologie complète après la récurrence du lundi (avec les logs désormais disponibles pour les deux occurrences, rappel M19 sur la persistance du journal) permettrait de découvrir, par exemple, une fuite mémoire progressive de l'application (rappel UE1-M3, cette fois appliquée en contexte de service systemd) provoquant un épuisement de ressources après une durée d'exécution prolongée, expliquant la récurrence à intervalle similaire. Ce cas illustre une tension réelle et fréquente en production : la pression du moment pousse vers une résolution rapide du symptôme, alors que la méthodologie complète (en particulier documenter les observations même lors d'une résolution d'urgence, pour permettre une analyse a posteriori) est ce qui permet, in fine, d'éviter la récurrence — un redémarrage sans aucune collecte d'information (étape 3) prive les investigations futures de données précieuses qui n'existeront plus après un redémarrage qui efface l'état du processus en mémoire.

## 15. Sécurité

- Une méthodologie de troubleshooting rigoureuse s'applique également, avec les mêmes neuf étapes, à un incident de sécurité (intrusion suspectée, comportement anormal) — la discipline de ne pas conclure prématurément (étape 4, chercher à réfuter avant de confirmer) est encore plus critique en contexte de sécurité, où une conclusion hâtive peut faire manquer un indicateur de compromission réel
- La documentation systématique (étape 8) constitue, en contexte de sécurité, une trace d'investigation potentiellement nécessaire pour une analyse forensique ultérieure ou une obligation réglementaire de notification d'incident
- L'étape 9 (prévention) rejoint directement le principe de défense en profondeur déjà rencontré à plusieurs reprises : chaque incident correctement analysé jusqu'à sa cause racine devient une opportunité d'ajouter une couche de protection ou de détection supplémentaire, renforçant progressivement la résilience globale du système

## 16. Automatisation

Script Bash de collecte automatique d'un "snapshot" de diagnostic initial, rassemblant en une seule exécution les informations de base des trois scénarios canoniques (étape 3 de la méthodologie, accélérée).

```bash
#!/usr/bin/env bash
# Collecte un snapshot de diagnostic initial multi-domaines pour accélérer l'étape 3
set -uo pipefail   # pas de -e ici : on veut collecter un maximum d'informations même si une commande échoue

RAPPORT="/tmp/diagnostic_snapshot_$(date +%Y%m%d_%H%M%S).txt"

{
    echo "=== SNAPSHOT DE DIAGNOSTIC - $(date) ==="

    echo -e "\n--- Services en échec ---"
    systemctl --failed --no-legend

    echo -e "\n--- État du stockage ---"
    df -h

    echo -e "\n--- Charge et mémoire ---"
    vmstat 1 2 | tail -1
    free -h

    echo -e "\n--- Réseau ---"
    ip -brief addr show
    ip route show

    echo -e "\n--- Erreurs système récentes ---"
    journalctl -p err -b --since "1 hour ago" --no-pager | tail -30

    echo -e "\n--- Noyau actif ---"
    uname -r

} > "$RAPPORT" 2>&1

echo "Snapshot de diagnostic généré: $RAPPORT"
```

## 17. Troubleshooting

*Ce module étant lui-même consacré à la méthodologie de troubleshooting, les scénarios complexes sont traités intégralement dans les Parties II et III (TD et TP) plutôt que résumés ici en une seule section courte.*

## 18. Synthèse

- La méthodologie en 9 étapes (symptôme → hypothèses → collecte → test → isolation → correction → vérification → documentation → prévention) structure tout diagnostic, quel que soit le domaine technique concerné
- Symptôme, cause immédiate et cause racine sont trois niveaux distincts — seule une correction de la cause racine empêche la récurrence
- Les trois scénarios canoniques (panne de service, disque plein, réseau down) mobilisent chacun plusieurs modules de l'UE3 simultanément, rarement un seul isolément
- Chercher activement à réfuter une hypothèse est une pratique plus rigoureuse que de simplement chercher à la confirmer
- La pression opérationnelle (résoudre vite) entre parfois en tension avec la rigueur méthodologique (comprendre la cause racine) — les deux doivent être conciliées, notamment en documentant même lors d'une résolution d'urgence
- Cette méthodologie s'applique identiquement aux incidents de sécurité, avec une importance accrue de la documentation et de la non-précipitation

## 19. Questions d'évaluation

**QCM**
1. La différence entre cause immédiate et cause racine est que :
 a) elles désignent toujours la même chose b) la cause racine est la raison profonde qui, non traitée, garantit la récurrence de l'incident c) la cause immédiate est toujours plus importante à corriger d) seule la cause immédiate doit être documentée

2. Chercher activement à réfuter une hypothèse plutôt que simplement la confirmer :
 a) est une perte de temps inutile b) est une pratique de diagnostic plus rigoureuse, réduisant le risque de conclusion prématurée c) n'a aucun intérêt pratique d) ne s'applique qu'aux incidents de sécurité

**Questions ouvertes**
- Expliquez pourquoi un redémarrage de service sans investigation préalable, bien que résolvant temporairement un incident, peut nuire à la résolution durable du problème.
- Décrivez, pour le scénario "disque plein", au moins deux causes racines distinctes possibles derrière le même symptôme immédiat de saturation, en citant les modules de l'UE3 concernés.

**Exercice technique**
- Listez, dans l'ordre, les 9 étapes de la méthodologie de troubleshooting vue en cours.

**Problème de diagnostic**
- Un service redevient fonctionnel après un simple redémarrage, mais l'incident se reproduit une semaine plus tard à intervalle similaire. Quelle hypothèse cela suggère-t-il, et quelle étape de la méthodologie a probablement été négligée lors de la première occurrence ?

## 20. Mini-projet

Développer un outil de diagnostic guidé (Bash ou Python) qui :
1. Pose une série de questions à l'utilisateur (ou détecte automatiquement) pour catégoriser le symptôme initial parmi les trois scénarios canoniques du cours (ou un scénario mixte)
2. Exécute automatiquement la collecte d'informations pertinente selon le scénario identifié (reprise et enrichissement du script de la section 16, avec des collectes spécifiques et plus poussées selon la branche de l'arbre de diagnostic empruntée)
3. Présente les résultats de façon structurée selon les arbres de diagnostic des sections 7.4 à 7.6, suggérant les hypothèses les plus probables sans jamais imposer une conclusion automatique
4. Intègre une fonctionnalité de documentation assistée : génère un squelette de rapport d'incident pré-rempli avec les informations collectées, à compléter par l'ingénieur avec les étapes 4 à 9 de la méthodologie
5. Constitue, par cette conception, un outil pédagogique réutilisable pour structurer la pratique du troubleshooting au-delà de ce module

---

# PARTIE II — TD

*Ce TD utilise exclusivement des scénarios complexes et multi-causes, dans l'esprit du module de synthèse — chaque exercice mobilise plusieurs domaines de l'UE3 simultanément.*

### Niveau 1 — Compréhension

**Exercice 1.1**
Listez les 9 étapes de la méthodologie de troubleshooting, avec une phrase décrivant chacune.

**Exercice 1.2**
Expliquez la différence entre symptôme, cause immédiate et cause racine, avec un exemple original (non tiré du cours) illustrant les trois niveaux.

**Exercice 1.3**
Pourquoi l'étape "formuler des hypothèses" recommande-t-elle d'en lister plusieurs avant de les tester, plutôt que de tester immédiatement la première venue à l'esprit ?

### Niveau 2 — Application

**Exercice 2.1**
Pour le scénario "panne de service", citez trois catégories de cause possibles vues dans l'arbre de diagnostic du cours, avec le module de l'UE3 associé à chacune.

**Exercice 2.2**
Pour le scénario "réseau down", donnez l'ordre recommandé (du plus bas niveau au plus haut) des vérifications à effectuer, conformément à l'arbre de diagnostic du cours.

**Exercice 2.3**
Donnez un exemple de mesure de "prévention" (étape 9) qui aurait pu être mise en place suite à l'étude de cas industrielle du cours (fuite mémoire provoquant une panne récurrente).

### Niveau 3 — Analyse

**Exercice 3.1**
Un service web est en panne. `systemctl status` révèle l'état `failed`. `journalctl -u <service>` ne montre aucun message d'erreur explicite de l'application elle-même, mais révèle que le service a atteint sa limite `StartLimitBurst` (rappel M16). Décrivez la démarche complète (au moins 4 étapes de la méthodologie) permettant de progresser au-delà de ce constat initial, sans se contenter d'un simple `systemctl reset-failed` suivi d'un redémarrage aveugle.

**Exercice 3.2**
Une machine ne répond plus du tout en SSH, alors qu'elle répondait normalement une heure auparavant, et qu'aucune modification de configuration n'a été rapportée par l'équipe. Formulez au moins trois hypothèses distinctes, mobilisant des domaines différents de l'UE2/UE3, cohérentes avec ce symptôme.

### Niveau 4 — Problème complexe (scénario intégrateur)

**Exercice 4.1**
Un serveur applicatif présente le symptôme suivant : le service principal est en état `failed` (`systemctl status`), et `journalctl -u <service> -b` révèle un message d'erreur indiquant une impossibilité d'écrire dans son répertoire de logs. Une investigation sur l'espace disque (`df -h`) confirme que la partition concernée est à 100%. Une recherche plus poussée (`du -sh` sur les sous-répertoires) ne révèle cependant aucun fichier volumineux correspondant à l'espace manquant dans les répertoires visibles. En vous appuyant sur l'ensemble des modules de l'UE1 et de l'UE3 mobilisés dans ce module de synthèse, menez un raisonnement complet en suivant explicitement les 9 étapes de la méthodologie, jusqu'à formuler une hypothèse de cause racine cohérente avec l'ensemble des observations, et une mesure de prévention adaptée.

---

## Correction du TD

**1.1** — 1. Identifier le symptôme : décrire précisément et factuellement ce qui est observé. 2. Formuler des hypothèses : lister les causes plausibles sans encore les tester. 3. Collecter les informations : rassembler les données factuelles nécessaires. 4. Tester les hypothèses : confronter chaque hypothèse aux informations collectées, en cherchant à la réfuter. 5. Isoler la cause : remonter jusqu'à la cause racine, pas seulement la cause immédiate. 6. Corriger : appliquer le correctif approprié. 7. Vérifier : confirmer explicitement la disparition du symptôme initial. 8. Documenter : consigner l'incident pour un apprentissage collectif futur. 9. Mettre en place une prévention : identifier une mesure structurelle empêchant la récurrence.

**1.2** — Exemple original : un développeur ne parvient plus à pousser du code vers un dépôt distant (symptôme : "git push échoue avec une erreur d'authentification"). La cause immédiate est une clé SSH expirée ou révoquée côté serveur du dépôt. La cause racine pourrait être l'absence de tout processus de notification avant l'expiration programmée des clés d'accès dans la politique de sécurité de l'organisation, laissant les utilisateurs découvrir le problème seulement au moment de l'échec, sans avoir été prévenus à l'avance pour renouveler leur accès en amont.

**1.3** — Formuler plusieurs hypothèses avant de les tester réduit le risque de **biais de confirmation** : si l'on ne considère qu'une seule hypothèse dès le départ, on risque de chercher inconsciemment des informations qui la confirment plutôt que de rester ouvert à des causes alternatives potentiellement plus exactes — l'erreur méthodologique déjà illustrée au module 21 (conclusion prématurée à partir d'un load average élevé, sans avoir envisagé et testé d'autres hypothèses comme le thrashing mémoire ou l'attente disque avant de s'engager dans une piste d'investigation coûteuse en temps).

**2.1** — Erreur de configuration de l'unité (module 16) ; erreur de permission empêchant le service d'accéder à une ressource nécessaire (module 14) ; dépendance réseau indisponible, par exemple une base de données distante injoignable (module 17, voire l'ensemble de l'UE2 pour la cause réseau elle-même).

**2.2** — Couche 1/2 (état de l'interface physique/logique) → couche 2/3 (résolution ARP) → couche 3 (table de routage, puis test ICMP en gardant à l'esprit son piège de filtrage) → couche 4 (test de port applicatif, distinguant refus explicite et timeout) → pare-feu local (règles iptables/nftables).

**2.3** — Exemple de mesure de prévention : mettre en place une supervision de la consommation mémoire du service dans le temps (rappel module 21, outils de supervision de performance), avec une alerte si une tendance de croissance continue est détectée sur une fenêtre glissante (rappel direct du mini-projet du module UE1-M3 sur la mémoire) — permettant de détecter une fuite mémoire progressive avant qu'elle ne provoque un nouvel épuisement de ressources et une panne récurrente, plutôt que de découvrir le problème uniquement après chaque nouvelle occurrence de l'incident.

**3.1** — Démarche complète : (1) **Identifier le symptôme** précisément : le service est en `failed`, ayant épuisé ses tentatives de redémarrage automatique, sans message d'erreur applicatif explicite dans les logs consultés jusqu'ici. (2) **Formuler des hypothèses** : l'application plante peut-être pour une raison non journalisée par elle-même (crash brutal, signal reçu) ; ou le problème n'est pas dans l'application elle-même mais dans son environnement d'exécution (dépendance manquante au démarrage, permission insuffisante, ressource externe injoignable). (3) **Collecter des informations complémentaires** : examiner l'intégralité du journal autour de chaque tentative de démarrage (pas seulement chercher un message d'erreur explicite, mais aussi l'absence totale de sortie après le lancement, qui serait elle-même un indice), vérifier l'état des dépendances déclarées de l'unité (`systemctl cat`, `After=`/`Requires=`), vérifier les ressources système au moment des tentatives (mémoire, disque, rappel modules 15/21). (4) **Tester les hypothèses** ainsi affinées avec les nouvelles informations collectées, en cherchant à en réfuter certaines (par exemple, si les dépendances sont toutes bien actives et les ressources système normales, cela réfute l'hypothèse d'un problème d'environnement et recentre l'investigation sur l'application elle-même) — la démarche ne se limite donc jamais à un simple `reset-failed` suivi d'un redémarrage sans avoir d'abord progressé dans ces étapes de collecte et de test, qui seules permettent d'éviter une récurrence rapide du même échec.

**3.2** — Trois hypothèses distinctes possibles, mobilisant des domaines différents : (1) **Réseau** : un changement de routage ou un problème de couche basse (câble, interface, rappel UE2-M8/M9) a rendu la machine injoignable, indépendamment de SSH lui-même — à tester en vérifiant d'abord une simple joignabilité de base (ping, en gardant à l'esprit son propre piège de filtrage) avant de conclure à un problème SSH spécifique. (2) **Pare-feu** : une règle de pare-feu a été modifiée ou appliquée (peut-être par une automatisation planifiée non immédiatement associée par l'équipe à un "changement de configuration" rapporté humainement), bloquant désormais le port SSH (rappel direct M17, le scénario exact du TP de ce module). (3) **Service SSH lui-même** : le service `sshd` a pu planter ou être arrêté suite à un événement système local (rappel M16, éventuellement en lien avec une saturation de ressources, rappel M15/M21), sans rapport avec le réseau ou le pare-feu — à distinguer des deux hypothèses précédentes par un accès hors-bande si disponible (console physique/virtuelle), qui permettrait de vérifier l'état du service localement indépendamment de la question réseau.

**4.1** — Application complète de la méthodologie :

**1. Symptôme :** le service principal est en état `failed`, avec un message d'erreur explicite indiquant une impossibilité d'écriture dans son répertoire de logs ; la partition concernée est confirmée à 100% via `df -h`.

**2. Hypothèses :** (a) des fichiers de logs légitimes de ce service ont simplement grossi au-delà de l'espace disponible ; (b) un autre processus, sans rapport direct avec le service en panne, consomme l'espace de cette même partition ; (c) un fichier a été supprimé mais reste maintenu ouvert par un processus, continuant d'occuper l'espace disque sans être visible dans l'arborescence (rappel direct UE1-M4) — cette troisième hypothèse est particulièrement cohérente avec l'indice déjà fourni dans l'énoncé : `du -sh` sur les répertoires visibles ne révèle aucun fichier volumineux correspondant à l'espace manquant, un symptôme caractéristique précisément de cette situation.

**3. Collecte d'informations :** exécuter `lsof | grep deleted` (rappel direct UE1-M4) pour identifier tout fichier supprimé mais toujours ouvert par un processus, avec sa taille réelle occupée.

**4. Test des hypothèses :** la sortie de `lsof` confirme (dans ce scénario) la présence d'un fichier supprimé mais toujours ouvert par un processus, avec une taille correspondant approximativement à l'espace manquant — confirmant l'hypothèse (c) et réfutant implicitement les hypothèses (a) et (b) qui auraient dû être visibles via `du -sh`.

**5. Isolation de la cause racine :** le processus responsable est identifié (via le PID révélé par `lsof`) comme étant potentiellement le service en panne lui-même, ou un service tiers de journalisation, ayant eu son fichier de logs renommé ou supprimé par un mécanisme externe (par exemple un script de nettoyage mal conçu, ou une rotation `logrotate` mal configurée sans le mécanisme `postrotate`/`copytruncate` approprié — rappel direct M19 et du piège originel étudié en UE1-M4) sans jamais recevoir de signal lui indiquant de rouvrir son descripteur de fichier.

**6. Correction :** envoyer le signal approprié au processus concerné (souvent `SIGHUP`) pour qu'il rouvre son fichier de logs, ou redémarrer proprement le service responsable, libérant ainsi l'espace disque occupé par l'ancien descripteur ; vérifier et corriger la configuration `logrotate` manquante ou incorrecte pour ce service.

**7. Vérification :** confirmer via `df -h` que l'espace disque est bien libéré, et que le service principal initialement en panne démarre désormais normalement (`systemctl start`, `systemctl status`).

**8. Documentation :** consigner l'incident avec la chaîne causale complète (service en panne → disque plein → fichier supprimé toujours ouvert → configuration logrotate manquante), pour référence future.

**9. Prévention :** mettre en place (ou corriger) une configuration `logrotate` appropriée pour ce service, et déployer un script de détection périodique de fichiers supprimés mais toujours ouverts au-delà d'un seuil de taille significatif (rappel direct du mini-projet du module UE1-M4), permettant de détecter une situation similaire avant qu'elle ne cause une nouvelle panne de service par saturation disque.

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur système en astreinte pour diagnostiquer trois incidents simulés, représentatifs des trois scénarios canoniques du cours, en appliquant rigoureusement et explicitement la méthodologie en 9 étapes pour chacun.

## Architecture

Une VM Linux avec systemd, disposant des outils des modules 14 à 21 (logrotate, iptables, LVM si possible, sysstat).

## Objectifs
- Diagnostiquer une panne de service multi-causale en suivant la méthodologie complète
- Diagnostiquer une saturation disque avec cause cachée (fichier supprimé toujours ouvert)
- Diagnostiquer une panne réseau avec cause au niveau pare-feu
- Produire une documentation d'incident structurée pour chaque scénario

## Préparation

```bash
which systemctl journalctl df lsof iptables ip 2>/dev/null
```

## Étapes

### Scénario A — Panne de service (préparation par l'enseignant ou auto-provoquée)

```bash
# Provoquer le scénario : service avec une dépendance sur un répertoire inaccessible
sudo mkdir -p /opt/lab_incident_a
sudo tee /opt/lab_incident_a/app.sh << 'EOF'
#!/bin/bash
echo "Tentative d'écriture dans le répertoire de logs..."
echo "test" > /var/log/lab_incident_a/app.log
echo "Application démarrée"
sleep 3600
EOF
sudo chmod +x /opt/lab_incident_a/app.sh
# Volontairement : NE PAS créer /var/log/lab_incident_a, cause de la panne à diagnostiquer

sudo tee /etc/systemd/system/lab-incident-a.service << 'EOF'
[Unit]
Description=Service de laboratoire - Incident A

[Service]
Type=simple
ExecStart=/opt/lab_incident_a/app.sh
Restart=on-failure
RestartSec=3
StartLimitIntervalSec=30
StartLimitBurst=3

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl start lab-incident-a.service
sleep 15
```

**À vous de jouer :** appliquez la méthodologie en 9 étapes pour diagnostiquer et corriger cet incident. Documentez chaque étape dans votre rapport.

```bash
# Commandes de départ suggérées (à compléter vous-même selon votre investigation)
sudo systemctl status lab-incident-a.service
sudo journalctl -u lab-incident-a.service -b --no-pager
```

### Scénario B — Disque plein avec cause cachée

```bash
sudo mkdir -p /tmp/lab_incident_b
python3 -c "
import time
f = open('/tmp/lab_incident_b/gros_fichier.log', 'w')
for i in range(200000):
    f.write('donnee de test ' * 20 + '\n')
f.flush()
print('Fichier écrit, PID:', __import__('os').getpid())
time.sleep(600)
" &
sleep 2
sudo rm /tmp/lab_incident_b/gros_fichier.log   # supprimé mais toujours ouvert par le processus Python
df -h /tmp
```

**À vous de jouer :** appliquez la méthodologie en 9 étapes. Le symptôme initial fourni : "l'espace disque de `/tmp` semble anormalement occupé, mais aucun fichier volumineux n'est visible."

### Scénario C — Réseau down par pare-feu

```bash
# Provoquer une règle de pare-feu bloquante sur un port de test
sudo tee /tmp/service_test.py << 'EOF'
import http.server
import socketserver
PORT = 8888
Handler = http.server.SimpleHTTPRequestHandler
with socketserver.TCPServer(("", PORT), Handler) as httpd:
    httpd.serve_forever()
EOF
python3 /tmp/service_test.py &
sleep 1
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8888/   # doit fonctionner avant la règle

sudo iptables -A INPUT -p tcp --dport 8888 -j DROP

curl -s --connect-timeout 3 -o /dev/null -w "%{http_code}\n" http://localhost:8888/ || echo "Timeout ou échec"
```

**À vous de jouer :** appliquez la méthodologie en 9 étapes. Le symptôme initial fourni : "le service de test sur le port 8888 ne répond plus depuis quelques minutes, sans changement applicatif rapporté."

```bash
# Nettoyage final après investigation complète
sudo iptables -D INPUT -p tcp --dport 8888 -j DROP
kill %1 2>/dev/null
```

## Commandes de référence
L'ensemble des commandes des modules 14 à 21 — aucune commande nouvelle n'est introduite dans ce module.

## Résultats attendus

Pour **chacun** des trois scénarios, un rapport structuré suivant explicitement les 9 étapes de la méthodologie :
1. Symptôme observé (tel que fourni, reformulé précisément si nécessaire)
2. Hypothèses formulées (au moins deux par scénario)
3. Informations collectées (commandes exécutées et résultats obtenus)
4. Test des hypothèses (laquelle est confirmée, lesquelles sont réfutées et pourquoi)
5. Cause racine isolée
6. Correction appliquée
7. Preuve de vérification (symptôme initial confirmé disparu)
8. Synthèse documentaire de l'incident
9. Mesure de prévention proposée

## Questions

1. Pour le scénario A, la cause racine identifiée relève-t-elle davantage du module 14 (permissions) ou d'un défaut de conception du service lui-même (absence de vérification de la disponibilité de son répertoire de logs au démarrage) ? Les deux corrections sont-elles nécessairement exclusives l'une de l'autre ?
2. Pour le scénario B, si vous n'aviez eu accès qu'à `du -sh` sans jamais penser à `lsof`, combien de temps supplémentaire cette investigation aurait-elle pu prendre ? Qu'est-ce que cela révèle sur l'importance de connaître l'ensemble large des outils disponibles, pas seulement les plus évidents ?
3. Pour le scénario C, si vous n'aviez eu accès à la machine que via SSH sur un port différent de 8888, auriez-vous pu diagnostiquer ce problème aussi facilement ? Quelle hypothèse alternative auriez-vous dû envisager si votre propre accès SSH avait également été affecté par une règle de pare-feu trop large ?

## Troubleshooting (panne à diagnostiquer — méta-scénario)

**Situation fournie :** après avoir résolu les trois scénarios précédents, un étudiant constate que son rapport de documentation (étape 8) pour le scénario A est incomplet — il a noté la correction appliquée, mais a oublié de documenter les hypothèses qu'il avait initialement formulées puis réfutées durant son investigation.

**Démarche attendue (méta-application de la méthodologie à ce cas de documentation lui-même) :**
1. Symptôme : documentation incomplète, informations perdues sur le cheminement d'investigation réel
2. Hypothèse : la pression de "passer à la correction" une fois la cause trouvée a probablement fait négliger la valeur de noter également les pistes explorées puis abandonnées, une habitude à corriger
3. Vérification : relire la définition de l'étape 8 du cours — la documentation doit inclure "les hypothèses testées (y compris celles réfutées, précieuses pour de futures investigations similaires)"
4. Isolation : reconnaître que cette omission n'est pas une erreur technique mais une **erreur de discipline méthodologique**, le même type d'erreur que celle illustrée dans l'étude de cas industrielle du cours (redémarrage sans documentation complète menant à une récurrence non anticipée)
5. Correction : reconstituer, si possible à partir de l'historique de commandes exécutées (`history` en Bash, si conservé), les hypothèses réellement testées durant l'investigation, et compléter le rapport de documentation en conséquence
6. Vérification : le rapport complété permet-il désormais à un tiers qui n'aurait pas participé à l'investigation de comprendre non seulement la solution, mais aussi pourquoi les autres pistes explorées ne convenaient pas ?
7. Documentation (de ce méta-incident lui-même) : noter, comme leçon personnelle, la difficulté réelle et fréquente de maintenir une discipline de documentation complète sous la pression naturelle de "passer à la suite" une fois une solution trouvée — une habitude à cultiver consciemment, cohérente avec l'esprit de rigueur de l'ensemble de ce module de clôture

## Validation

Le TP est validé si l'étudiant fournit :
- Un rapport complet suivant les 9 étapes pour chacun des trois scénarios
- Des réponses argumentées aux 3 questions
- Une réflexion sur le méta-scénario de troubleshooting, démontrant la compréhension que la méthodologie s'applique aussi à la qualité de sa propre pratique de documentation

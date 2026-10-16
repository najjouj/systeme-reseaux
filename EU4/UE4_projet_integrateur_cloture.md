# UE4 — Projet Intégrateur de Clôture
## Automatiser l'administration Linux avec Bash

*Correspond au **Projet 4** du programme général (partie Bash) : "Automatiser l'administration Linux avec Bash". Mobilise l'intégralité des modules 23 à 26 de l'UE4, appliqués à l'ensemble des scripts déjà écrits dans les UE1, UE2 et UE3.*

---

## 1. Titre du projet
De l'ad-hoc au professionnel : construire une suite d'automatisation Bash robuste pour l'infrastructure Atelier Nomia

## 2. Niveau
Élève ingénieur — cycle ingénieur, clôture de l'UE4 automatisation (S2)

## 3. Durée
1,5 à 2 semaines de travail en autonomie (binôme recommandé), avec 2 points de passage encadrés et une soutenance finale de 15 minutes par binôme.

## 4. Prérequis
L'intégralité de l'UE4 (modules 23 à 26), ainsi que l'ensemble des scripts déjà écrits au fil des UE1, UE2 et UE3.

## 5. Contexte du projet

**Continuité narrative directe avec le projet de clôture de l'UE3 :** les serveurs `app-01` et `backup-01` de l'entreprise fictive **Atelier Nomia** fonctionnent en production depuis leur déploiement à la clôture de l'UE3. Au fil de leur mise en place, l'équipe a accumulé une dizaine de scripts d'automatisation et de supervision, écrits au moment de chaque besoin, sans convention commune ni rigueur homogène — exactement le scénario de l'étude de cas industrielle du module 26 (collection de scripts hérités, hétérogènes, avec des faux positifs après redémarrage).

*Si le projet de clôture UE3 n'a pas été réalisé, l'équipe peut partir d'un ensemble minimal de 3 scripts fictifs fournis en annexe (ou reconstitués à partir des exemples de scripts déjà donnés dans les modules UE1/UE2/UE3 de ce programme), représentatifs des mêmes pièges à corriger.*

Vous intervenez comme équipe d'ingénieurs pour professionnaliser cette collection de scripts : les auditer, les robustifier, en construire de nouveaux selon une architecture cohérente, et livrer une suite d'automatisation intégrée à l'écosystème systemd, avec une documentation permettant à n'importe quel futur membre de l'équipe de la comprendre et de la faire évoluer.

## 6. Objectifs pédagogiques

Ce projet doit démontrer la maîtrise intégrée de :
- La rigueur syntaxique Bash (quotation, boucles robustes, gestion d'arguments) — module 23
- La construction de pipelines efficaces de traitement de texte — module 24
- La robustesse systématique (`set -euo pipefail`, `trap`, codes de sortie documentés) — module 25
- L'architecture modulaire collecte/analyse/rapport et l'intégration à l'écosystème de planification — module 26
- La méthodologie de relecture critique appliquée à un ensemble de scripts hérités réels

## 7. Cahier des charges

### 7.1 Phase d'audit (obligatoire, base de tout le reste)

L'équipe doit constituer un ensemble d'au moins **5 scripts** représentatifs des différents domaines déjà couverts dans le programme (au choix, adaptés ou réécrits à partir des exemples déjà rencontrés dans les modules précédents si les scripts originaux ne sont pas disponibles) :
- Un script de détection d'anomalie processus/mémoire (esprit UE1-M2/M3)
- Un script d'audit de permissions ou de configuration système (esprit UE3-M14/M20)
- Un script de supervision réseau ou de service (esprit UE2/UE3-M16/M17)
- Un script de sauvegarde (esprit UE3-M21)
- Un script d'analyse de logs (esprit UE3-M19 ou UE4-M24)

Pour chacun, produire un **rapport d'audit individuel** suivant la méthodologie de relecture critique du module 26 (section 7.6) : quotation, boucles/substitutions, pipelines, robustesse `set -euo pipefail`/`trap`, résultat `shellcheck`.

### 7.2 Bibliothèque commune

Construire (ou reprendre et finaliser le mini-projet du module 25) une **bibliothèque Bash commune** (`bash_hardening.sh` ou nom équivalent), fournissant au minimum :
- Initialisation standard (`set -euo pipefail` + `trap` de nettoyage générique avec enregistrement de ressources)
- Fonctions de validation d'arguments courantes
- Fonctions de journalisation standardisées (stdout pour l'information, stderr pour les erreurs/alertes)

### 7.3 Refonte des scripts audités

Chaque script audité en 7.1 doit être **refondu** pour :
- Utiliser la bibliothèque commune de la section 7.2
- Passer `shellcheck` sans avertissement significatif non justifié
- Suivre l'architecture collecte/analyse/rapport (module 26) lorsque pertinent
- Corriger tout problème de persistance d'état identifié (rappel systématique du piège `/tmp`)

### 7.4 Nouveaux outils — architecture trois catégories

L'équipe doit développer **au moins un nouvel outil** de supervision suivant explicitement les trois catégories de vérification du module 26 (état actif / cohérence de configuration / seuil de performance), rapportées **séparément**, jamais fusionnées en un indicateur binaire unique — appliqué à un service réel de l'infrastructure `app-01`/`backup-01`.

### 7.5 Intégration et planification

- Chaque outil de supervision doit être intégré via un timer systemd (module 26, avec journalisation vérifiable via `journalctl`)
- Au moins un outil doit conserver un état entre exécutions (détection de tendance), stocké dans un emplacement **explicitement justifié comme persistant**

### 7.6 Incident volontaire de clôture

L'équipe doit reproduire volontairement, sur l'un des scripts refondus, le scénario du **faux positif au redémarrage** (module 26, section 14 et exercice 4.1 du TD) : introduire délibérément (dans une version de démonstration, pas dans le livrable final) un stockage d'état non persistant, démontrer le faux positif généré après un redémarrage simulé, puis présenter la version corrigée avec la justification de l'emplacement de stockage retenu.

## 8. Architecture cible (schéma de référence)

```
┌─────────────────────────────────────────────────────────┐
│  bash_hardening.sh (bibliothèque commune, module 25)          │
│  set -euo pipefail, trap générique, validation, logging          │
└───────────────────────┬───────────────────────────────┘
                          │ source
        ┌──────────────────┼──────────────────┬──────────────┐
        ▼                    ▼                    ▼                ▼
  script_processus.sh   script_permissions.sh  script_reseau.sh  script_backup.sh
  (refondu)                (refondu)              (refondu)        (refondu)
        │                    │                    │                │
        └──────────────────┴──────────────────┴──────────────┘
                          │
                 Nouvel outil (architecture 3 catégories)
                          │
              Timer systemd + journalctl (module 26)
```

## 9. Environnement de laboratoire

Réutilisation directe des VM `app-01`/`backup-01` du projet de clôture UE3 si disponibles, sinon toute VM Linux avec Bash, `shellcheck` et systemd.

## 10. Points de passage encadrés

### Point de passage 1 (mi-parcours)
- Ensemble des 5 scripts audités avec rapport individuel
- Bibliothèque commune fonctionnelle et testée isolément
- Validation par l'enseignant avant de poursuivre la refonte complète

### Point de passage 2 (fin de projet, avant soutenance)
- Tous les scripts refondus, validés `shellcheck`
- Nouvel outil trois-catégories fonctionnel et intégré en timer
- Démonstration du scénario faux positif/correction prête

## 11. Livrables attendus

- **Rapports d'audit individuels** pour les 5 scripts de départ
- **Bibliothèque commune** (code source + documentation)
- **Scripts refondus** (les 5 scripts d'origine, version finale)
- **Nouvel outil trois-catégories**, avec ses fichiers `.service`/`.timer`
- **Rapport de démonstration** du scénario faux positif/correction (module 26)
- **Rapport de synthèse global** résumant les problèmes les plus fréquemment rencontrés à travers l'ensemble des scripts audités, et les leçons transversales qui en découlent

## 12. Grille d'évaluation

| Critère | Pondération | Détail |
|---|---|---|
| Qualité des audits individuels | 20% | Méthodologie de relecture critique correctement appliquée aux 5 scripts |
| Bibliothèque commune | 15% | Fonctionnelle, réutilisée effectivement par tous les scripts refondus |
| Qualité de la refonte | 25% | `shellcheck` propre, robustesse systématique, architecture cohérente |
| Nouvel outil trois-catégories | 15% | Catégories correctement séparées et rapportées, intégration systemd fonctionnelle |
| Scénario faux positif/correction | 15% | Reproduction convaincante, correction justifiée |
| Documentation et synthèse globale | 10% | Clarté, leçons transversales bien identifiées |

**Note additionnelle, cohérente avec l'esprit du programme :** un script qui passe `shellcheck` sans erreur mais dont l'architecture reste un bloc monolithique sans séparation collecte/analyse/rapport n'est que partiellement conforme à l'esprit du module 26 — l'évaluation porte sur la qualité de conception autant que sur l'absence d'avertissement d'outil automatique.

## 13. Contraintes importantes

- Les rapports d'audit doivent identifier les problèmes **avant** la refonte, pas être reconstruits après coup pour justifier a posteriori le travail déjà fait
- La bibliothèque commune doit être réellement partagée par tous les scripts refondus, pas dupliquée ou réimplémentée différemment dans chacun
- Le scénario de faux positif (section 7.6) doit être démontré avec de vraies captures avant/après, pas seulement décrit

## 14. Clôture du programme UE1-UE4

Ce projet clôt un parcours continu depuis l'UE1 : les scripts audités et refondus ici sont, pour une large part, ceux-là mêmes écrits au fil des UE1 (processus, mémoire, filesystems, boot, IPC), UE2 (réseau) et UE3 (administration Linux) — ce projet transforme cette accumulation progressive de compétences en un livrable professionnel cohérent, démontrant la capacité à revenir sur son propre travail avec un regard critique et à l'élever au niveau d'exigence attendu d'un ingénieur système en production.

**Prolongement vers l'UE5 :** la rigueur de scripting acquise ici (quotation, robustesse, architecture modulaire) sera directement transférable à l'UE5 (PowerShell et automatisation Windows), où les mêmes principes de conception (séparation des responsabilités, gestion d'erreurs systématique, validation d'entrée) s'appliqueront dans un écosystème différent, avec ses propres spécificités déjà annoncées dans le programme général (pipeline d'objets PowerShell vs texte Unix).

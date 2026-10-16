# UE3 — Module 22 : Troubleshooting Méthodique
## Compléments : ÉVALUATION / PROJET

---

# PARTIE I — ÉVALUATION

**Durée :** 2h (module de synthèse — durée allongée, épreuve pouvant s'appuyer sur un scénario complet à traiter de bout en bout)
**Modalités :** documents non autorisés sauf `man` local si épreuve sur machine
**Barème total :** /20

## Section A — QCM (4 points, 0,5 pt/question)

1. La cause racine d'un incident se distingue de la cause immédiate par le fait que :
 a) elles sont toujours identiques b) la cause racine, non traitée, garantit la récurrence de l'incident c) seule la cause immédiate mérite d'être documentée d) la cause racine est toujours plus rapide à corriger

2. Formuler plusieurs hypothèses avant de les tester permet notamment d'éviter :
 a) une perte de temps systématique b) le biais de confirmation et une conclusion prématurée c) toute forme de documentation d) l'usage de plusieurs outils

3. Pour le scénario "réseau down", l'ordre de vérification recommandé va :
 a) du pare-feu vers la couche physique b) de la couche 1/2 vers la couche 4 puis le pare-feu c) uniquement au niveau applicatif d) sans ordre particulier

4. Un fichier supprimé mais toujours ouvert par un processus, ne consommant apparemment aucun espace visible via `du -sh` :
 a) est une situation impossible b) une hypothèse cohérente à vérifier via `lsof | grep deleted` c) toujours résolue par un redémarrage du disque d) sans rapport avec le stockage

5. L'étape "documenter" de la méthodologie doit inclure :
 a) uniquement la solution finale retenue b) également les hypothèses testées et réfutées, utiles pour de futures investigations c) uniquement le nom de l'ingénieur ayant résolu l'incident d) rien si l'incident a été résolu rapidement

6. Un redémarrage de service sans investigation préalable :
 a) constitue toujours une résolution complète et durable b) résout le symptôme sans nécessairement traiter la cause racine, au risque d'une récurrence c) empêche définitivement toute récurrence d) est la seule méthode recommandée en urgence

7. L'étape "prévention" (étape 9) vise à :
 a) uniquement documenter l'incident b) mettre en place une mesure structurelle empêchant la récurrence, au-delà de la correction ponctuelle c) remplacer l'étape de correction d) être appliquée uniquement pour les incidents de sécurité

8. Un `ping` sans réponse dans le cadre du diagnostic "réseau down" :
 a) prouve toujours une panne réseau complète b) doit être interprété avec prudence, un filtrage ICMP n'étant pas une preuve de panne réelle c) n'a aucune utilité diagnostique d) remplace tout test de couche 4

## Section B — Questions ouvertes (6 points)

**Q1 (2 pts).** Décrivez les 9 étapes de la méthodologie de troubleshooting, en une phrase courte par étape.

**Q2 (2 pts).** Expliquez, à l'aide d'un exemple, la différence entre symptôme, cause immédiate et cause racine, et pourquoi seule la correction de la cause racine empêche la récurrence.

**Q3 (2 pts).** Expliquez pourquoi la documentation d'un incident doit inclure les hypothèses réfutées, pas seulement la solution retenue.

## Section C — Exercice technique (4 points)

Pour le scénario "panne de service" (service en état `failed`), donnez, dans l'ordre logique de la méthodologie, les quatre premières commandes que vous exécuteriez, en précisant à quelle étape de la méthodologie chacune correspond.

## Section D — Problème de diagnostic intégrateur (6 points)

Un service applicatif est en état `failed`. `journalctl -u <service> -b` révèle un message d'erreur d'écriture dans son répertoire de logs. `df -h` confirme une partition à 100%. `du -sh` sur les répertoires visibles ne révèle aucun fichier volumineux correspondant à l'espace manquant.

**a) (2 pts)** Formulez au moins deux hypothèses distinctes cohérentes avec l'ensemble de ces observations.

**b) (2 pts)** Donnez la commande permettant de tester la plus probable de ces hypothèses, et ce que son résultat confirmerait ou infirmerait.

**c) (2 pts)** Une fois la cause racine isolée et corrigée, proposez une mesure de prévention adaptée, distincte de la simple correction immédiate.

## Grille de correction résumée

| Section | Points | Critères |
|---|---|---|
| A — QCM | /4 | Réponse exacte uniquement |
| B — Questions ouvertes | /6 | Précision technique, méthodologie correctement articulée sur les 9 étapes |
| C — Exercice technique | /4 | 4 commandes pertinentes, correctement associées à leur étape méthodologique |
| D — Diagnostic | /6 | Hypothèses cohérentes avec l'ensemble des indices, démarche méthodique complète, prévention distincte de la correction |

---

# PARTIE II — MINI-PROJET (échelle module — projet de synthèse UE3)

## Cahier des charges

**Titre :** Assistant de diagnostic guidé (Guided Troubleshooting Assistant)

**Contexte :** Ce projet clôt la série des huit outils développés au fil de l'UE3 (modules 14 à 21). L'équipe système souhaite un outil de diagnostic guidé, structurant la pratique de la méthodologie en 9 étapes pour toute personne intervenant en astreinte, y compris un ingénieur junior peu expérimenté face à un incident inconnu.

### Objectifs du livrable

Développer un outil (Bash ou Python, au choix justifié) qui :
1. Catégorise le symptôme initial parmi les trois scénarios canoniques (panne de service, disque plein, réseau down) via une interaction simple avec l'utilisateur (questions, ou détection automatique si possible), ou signale un scénario mixte/non catégorisable
2. Exécute automatiquement la collecte d'informations pertinente selon le scénario identifié, en suivant les arbres de diagnostic du cours (sections 7.4 à 7.6) — chaque branche de l'arbre doit correspondre à une collecte réelle et automatisée, pas une simple liste statique
3. Présente les résultats collectés de façon structurée, en suggérant les hypothèses les plus cohérentes avec les observations, **sans jamais imposer une conclusion automatique définitive** — l'outil assiste le raisonnement humain, il ne le remplace pas
4. Génère un squelette de rapport de documentation d'incident pré-rempli avec les informations collectées (étapes 1 à 3 de la méthodologie), laissant les champs des étapes 4 à 9 à compléter par l'ingénieur
5. Peut, en option, intégrer ou faire référence aux outils développés aux modules précédents (Permission & Account Auditor, LVM Capacity Manager, Service Health Guardian, Safe Firewall Deployer, SSH Access Auditor & Hardener, Log & Schedule Guardian, Package & Kernel Compliance Auditor, Performance & Backup Health Guardian) comme sources de collecte complémentaires si disponibles sur le système

### Contraintes techniques
- L'outil ne doit jamais prétendre fournir un diagnostic certain — chaque suggestion doit être présentée comme une hypothèse à vérifier, jamais comme une conclusion automatique
- Chaque branche de collecte automatisée doit être réellement fonctionnelle et testée, pas un placeholder
- Gestion d'erreurs robuste (commandes indisponibles selon l'environnement, permissions insuffisantes)
- Documentation d'installation et d'utilisation (README)
- Tests démontrés sur les trois scénarios canoniques du TP de ce module (panne de service, disque plein avec cause cachée, réseau down par pare-feu), avec le rapport de documentation généré pour chacun

### Livrables attendus
- Code source commenté
- README (installation, utilisation, structure des arbres de diagnostic implémentés, limites connues)
- Journal de tests montrant les 3 scénarios canoniques traités par l'outil, avec les rapports générés
- Rapport court (1 page) sur les choix techniques et une réflexion explicite sur la limite fondamentale de l'outil : il structure et accélère la collecte (étapes 1 à 3), mais le raisonnement de test d'hypothèses, d'isolation de cause racine et de décision de correction (étapes 4 à 9) reste une responsabilité humaine qu'aucun outil ne devrait prétendre automatiser entièrement

### Grille d'évaluation

| Critère | Pondération |
|---|---|
| Catégorisation correcte du symptôme initial | 15% |
| Collecte automatisée fonctionnelle pour les trois arbres de diagnostic | 35% |
| Présentation des hypothèses sans conclusion automatique imposée | 20% |
| Génération de squelette de documentation d'incident | 15% |
| Documentation et tests démontrés (3 scénarios) | 15% |

### Modalité
Travail individuel ou en binôme, durée indicative : 1,5 à 2 semaines (projet de synthèse, volume plus important).

**Clôture de l'UE3 :** ce projet, combiné aux huit outils déjà développés aux modules 14 à 21, constitue une suite complète de supervision, d'audit et de diagnostic pour une infrastructure Linux — une base directement réutilisable pour le projet intégrateur de fin d'UE3 (Projet 1 du programme général : déployer et sécuriser une infrastructure Linux), où l'ensemble de ces outils pourra être déployé et mis à l'épreuve sur une infrastructure réelle.

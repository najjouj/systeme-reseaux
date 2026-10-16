# UE3 — Module 20 : Gestion des Paquets et du Noyau
## Compléments : ÉVALUATION / PROJET

---

# PARTIE I — ÉVALUATION

**Durée :** 1h30
**Modalités :** documents non autorisés sauf `man` local si épreuve sur machine
**Barème total :** /20

## Section A — QCM (4 points, 0,5 pt/question)

1. `apt update` sert à :
 a) installer toutes les mises à jour disponibles b) rafraîchir l'index local des paquets depuis les dépôts c) redémarrer le système d) supprimer les paquets obsolètes

2. Après installation d'un nouveau paquet de noyau, celui-ci devient actif :
 a) immédiatement b) seulement après le prochain redémarrage c) uniquement via une commande dédiée sans redémarrage d) jamais automatiquement

3. `uname -r` affiche :
 a) le dernier noyau installé quel que soit son état b) le noyau actuellement actif en mémoire c) la liste de tous les noyaux disponibles d) la version d'APT

4. Figer un paquet (`hold`) :
 a) le met à jour automatiquement en priorité b) le protège d'une mise à jour non désirée mais le prive aussi des correctifs de sécurité le concernant c) le supprime définitivement d) n'a aucun effet réel

5. La vérification de signature GPG d'un dépôt de paquets :
 a) est une option facultative sans réelle importance b) est une mesure de sécurité fondamentale contre la distribution de paquets malveillants c) ralentit uniquement les installations d) concerne uniquement les paquets noyau

6. Un module noyau chargé manuellement via `modprobe` :
 a) survit automatiquement à tout redémarrage b) est volatile, sauf déclaration explicite de persistance c) ne peut jamais être déchargé d) remplace le noyau actif

7. Un audit de sécurité basé uniquement sur l'état des paquets installés, sans vérifier l'état réellement actif du système :
 a) est toujours suffisant b) peut être trompeur, notamment pour un noyau installé mais non encore actif c) est plus fiable qu'un audit de l'état actif d) ne concerne pas le noyau

8. Un déploiement progressif (canary) de mises à jour automatiques vise à :
 a) accélérer l'application à l'ensemble du parc simultanément b) détecter une régression sur un périmètre restreint avant généralisation c) remplacer entièrement les tests manuels d) désactiver les mises à jour de sécurité

## Section B — Questions ouvertes (6 points)

**Q1 (2 pts).** Expliquez pourquoi un audit de sécurité vérifiant uniquement `dpkg --list` (ou équivalent) peut conclure à tort qu'un système est protégé contre une vulnérabilité noyau corrigée.

**Q2 (2 pts).** Expliquez le compromis induit par le gel d'un paquet critique, et la pratique de gouvernance recommandée pour l'encadrer dans la durée.

**Q3 (2 pts).** Proposez une stratégie de mise à jour combinant réactivité de sécurité et réduction du risque de régression non maîtrisée, en vous appuyant sur les approches vues en cours.

## Section C — Exercice technique (4 points)

Donnez la séquence de commandes permettant de : rafraîchir l'index des paquets, lister les paquets pour lesquels une mise à jour est disponible, figer un paquet nommé `postgresql`, puis vérifier via une simulation que ce paquet n'est effectivement pas concerné par une mise à jour ultérieure.

## Section D — Problème de diagnostic (6 points)

Un script de vérification de cohérence noyau (actif vs dernier installé) rapporte systématiquement "OK", alors qu'un audit externe recommande une mise à jour du noyau vers une version corrigeant plusieurs vulnérabilités.

**a) (2 pts)** Expliquez pourquoi ce script peut rapporter "OK" tout en laissant subsister un risque de sécurité réel.

**b) (2 pts)** Proposez la vérification complémentaire manquante, avec la commande associée.

**c) (2 pts)** Expliquez la distinction générale entre "cohérence interne" et "conformité globale" illustrée par ce cas, applicable à d'autres outils d'audit.

## Grille de correction résumée

| Section | Points | Critères |
|---|---|---|
| A — QCM | /4 | Réponse exacte uniquement |
| B — Questions ouvertes | /6 | Précision technique, raisonnement sur le piège actif/installé et la gouvernance du gel |
| C — Exercice technique | /4 | Séquence de commandes complète et correcte |
| D — Diagnostic | /6 | Distinction cohérence interne/conformité globale correctement articulée, vérification complémentaire pertinente |

---

# PARTIE II — MINI-PROJET (échelle module)

## Cahier des charges

**Titre :** Auditeur de conformité paquets et noyau (Package & Kernel Compliance Auditor)

**Contexte :** Suite au scénario de troubleshooting du TP (un script de cohérence rapportant "OK" à tort faute de vérifier la conformité globale), l'équipe sécurité souhaite un outil d'audit qui couvre explicitement les deux dimensions distinctes identifiées en cours : la cohérence interne (noyau actif vs installé) et la conformité globale (existence de mises à jour de sécurité disponibles et non appliquées, paquets figés non réévalués).

### Objectifs du livrable

Développer un outil (Bash ou Python, au choix justifié) qui :
1. Vérifie la cohérence entre le noyau actif et le dernier noyau installé (reprise du script du cours), avec alerte "redémarrage requis" si un écart est détecté
2. Vérifie, indépendamment du point 1, l'existence de mises à jour de noyau disponibles dans les dépôts mais non installées, avec une alerte de nature distincte ("mise à jour noyau disponible non appliquée")
3. Liste les paquets actuellement figés (`hold`/`versionlock`) et calcule, pour chacun, une durée depuis un fichier de justification associé (à faire tenir à jour par l'équipe) ; signale ceux dépassant un seuil configurable sans réévaluation documentée
4. Liste l'ensemble des paquets pour lesquels une mise à jour est disponible, avec une tentative de distinction entre mise à jour de sécurité et mise à jour fonctionnelle si l'information est exploitable depuis le gestionnaire de paquets utilisé
5. Journalise chaque détection avec horodatage et catégorie claire (cohérence interne / conformité globale / gouvernance du gel)
6. Génère un rapport de synthèse structuré selon ces trois catégories distinctes, destiné à une revue de sécurité périodique

### Contraintes techniques
- Les trois catégories de vérification (cohérence interne, conformité globale, gouvernance du gel) doivent être clairement séparées dans le code et dans le rapport, jamais mélangées sous un unique statut "OK"/"KO" global trompeur
- Gestion d'erreurs robuste (gestionnaire de paquets non reconnu, absence de fichiers de justification pour le gel)
- Configuration externalisée (seuils, emplacement des fichiers de justification de gel)
- Documentation d'installation et d'utilisation (README)
- Tests démontrés sur au moins 3 scénarios : noyau installé mais non actif (redémarrage requis), noyau à jour et actif mais mise à jour disponible non installée (reproduisant précisément le piège du TP), paquet figé sans justification récente détecté

### Livrables attendus
- Code source commenté
- README (installation, configuration, format du rapport en trois catégories, limites connues)
- Journal de tests montrant les 3 scénarios validés, avec captures
- Rapport court (1 page) sur les choix techniques, expliquant explicitement pourquoi les trois catégories ne doivent jamais être fusionnées en un indicateur unique, et les limites connues (ex : ne peut pas garantir qu'une mise à jour disponible corrige effectivement une vulnérabilité critique sans consultation d'une base de vulnérabilités externe)

### Grille d'évaluation

| Critère | Pondération |
|---|---|
| Vérification de cohérence interne (actif/installé) | 20% |
| Vérification de conformité globale (mises à jour disponibles) | 30% |
| Gouvernance du gel de paquets (justification et péremption) | 20% |
| Séparation claire des trois catégories dans le rapport | 15% |
| Documentation et tests démontrés (3 scénarios) | 15% |

### Modalité
Travail individuel ou en binôme, durée indicative : 1 semaine. Cet outil complète la série d'audit système initiée aux modules 14 à 19 — l'ensemble converge vers une suite d'audit unifiée mobilisable pour le projet intégrateur de fin d'UE3.

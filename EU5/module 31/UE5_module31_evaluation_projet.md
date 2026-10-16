# UE5 — Module 31 : Permissions NTFS et ACL
## Compléments : ÉVALUATION / PROJET

---

# PARTIE I — ÉVALUATION

**Durée :** 1h30
**Modalités :** documents non autorisés sauf aide PowerShell locale si épreuve sur machine
**Barème total :** /20

## Section A — QCM (4 points, 0,5 pt/question)

1. Une ACL NTFS est composée de :
 a) trois catégories fixes uniquement b) une liste d'ACE (sujet, type, permissions) c) un seul propriétaire d) aucune structure définie

2. Lorsqu'une ACL contient une autorisation et un refus applicables au même sujet et à la même permission :
 a) l'autorisation l'emporte toujours b) le refus l'emporte toujours c) le dernier ajouté l'emporte d) l'accès est indéterminé

3. L'héritage NTFS, comparé au SGID Unix :
 a) sont strictement identiques b) propage les permissions elles-mêmes, contrairement au SGID qui ne propage que le groupe c) n'existe pas sous NTFS d) est moins puissant que SGID dans tous les cas

4. La combinaison de permissions de partage et NTFS suit la règle :
 a) la plus permissive l'emporte b) la plus restrictive l'emporte c) seul le partage compte d) seul le NTFS compte

5. Un déplacement de dossier entre deux volumes NTFS différents :
 a) conserve toujours les permissions d'origine b) peut réinitialiser les permissions aux valeurs par défaut du dossier de destination c) est impossible d) supprime automatiquement toutes les permissions

6. La bonne pratique recommandée pour éviter la confusion entre permissions de partage et NTFS consiste à :
 a) restreindre les deux de façon égale b) garder le partage large et gérer la granularité au niveau NTFS c) ignorer les permissions de partage d) désactiver l'héritage systématiquement

7. Un refus explicite NTFS non documenté :
 a) n'a aucun impact sur l'auditabilité b) peut rendre une structure de permissions difficile à comprendre et à auditer c) est automatiquement supprimé après 90 jours d) n'affecte que les administrateurs

8. `icacls` sert à :
 a) gérer les permissions NTFS en ligne de commande b) configurer Kerberos c) créer des GPO d) gérer les zones DNS

## Section B — Questions ouvertes (6 points)

**Q1 (2 pts).** Expliquez la règle de priorité du refus explicite NTFS, avec un exemple de cas d'usage légitime.

**Q2 (2 pts).** Expliquez pourquoi un audit de permissions limité au seul niveau NTFS d'un dossier peut manquer la véritable cause d'un problème d'accès, en citant deux causes alternatives possibles.

**Q3 (2 pts).** Comparez l'héritage NTFS et le mécanisme SGID Unix, en précisant ce que chacun propage réellement.

## Section C — Exercice technique (4 points)

Donnez la commande `icacls` permettant d'accorder Contrôle total (hérité par sous-fichiers et sous-dossiers) au groupe "NOMIA\GG_Direction" sur "C:\Data\Direction", et celle permettant de refuser explicitement l'accès en écriture à l'utilisateur "NOMIA\stagiaire" sur ce même dossier.

## Section D — Problème de diagnostic (6 points)

Un utilisateur membre d'un groupe autorisé en Modification sur un dossier ne parvient pas à y écrire via un partage réseau, sans refus explicite visible sur le dossier lui-même.

**a) (2 pts)** Proposez deux hypothèses distinctes non liées à un refus explicite directement visible sur ce dossier.

**b) (2 pts)** Pour chacune, donnez la commande de vérification appropriée.

**c) (2 pts)** Expliquez pourquoi une démarche de diagnostic limitée à la première vérification négative serait insuffisante.

## Grille de correction résumée

| Section | Points | Critères |
|---|---|---|
| A — QCM | /4 | Réponse exacte uniquement |
| B — Questions ouvertes | /6 | Précision technique, comparaison NTFS/Unix correctement articulée |
| C — Exercice technique | /4 | 2 commandes exactes |
| D — Diagnostic | /6 | Deux hypothèses distinctes et pertinentes, commandes exactes, raisonnement méthodologique correct |

---

# PARTIE II — MINI-PROJET (échelle module)

## Cahier des charges

**Titre :** Auditeur de permissions NTFS et partages (NTFS & Share Permission Auditor)

**Contexte :** Suite aux outils des modules 27 à 30, l'équipe système souhaite un cinquième outil PowerShell dédié à l'audit des permissions NTFS et de partage, avec une attention particulière au piège de combinaison identifié en cours et vécu en TP.

### Objectifs du livrable

Développer un script PowerShell qui :
1. Audite une arborescence pour détecter les refus explicites (reprise et extension du script du cours), en distinguant ceux visés sur un utilisateur individuel de ceux visés sur un groupe entier
2. Détecte les ruptures d'héritage non documentées
3. Pour chaque partage réseau configuré sur le serveur, compare systématiquement les permissions de partage et les permissions NTFS du dossier source, signalant explicitement toute combinaison où le partage est plus restrictif que le NTFS (source de confusion identifiée en cours) ou l'inverse
4. Calcule, pour un utilisateur ou groupe donné en paramètre, les permissions effectives sur un chemin précis, en tenant compte de l'héritage et de la règle de priorité du refus
5. Produit un rapport de synthèse en catégories distinctes (refus explicites, ruptures d'héritage, incohérences partage/NTFS, permissions effectives calculées)
6. Documente l'exécution avec horodatage

### Contraintes techniques
- La détection des incohérences partage/NTFS doit être fiable et testée (reproduction du scénario du TP)
- Gestion d'erreurs robuste (partage inexistant, chemin NTFS inaccessible)
- Documentation d'installation et d'utilisation (README)
- Tests démontrés sur au moins 3 scénarios : refus explicite détecté et catégorisé (individuel vs groupe), incohérence partage/NTFS détectée (reproduction du TP), calcul de permissions effectives correct sur un cas avec héritage et refus combinés

### Livrables attendus
- Code source commenté
- README (installation, utilisation, format du rapport, limites connues)
- Journal de tests montrant les 3 scénarios validés, avec captures
- Rapport court (1 page) sur les choix techniques et le parallèle explicite avec le Permission & Account Auditor développé en UE3-M14 pour Linux

### Grille d'évaluation

| Critère | Pondération |
|---|---|
| Détection des refus explicites (individuel vs groupe) | 20% |
| Détection des ruptures d'héritage | 15% |
| Détection des incohérences partage/NTFS | 30% |
| Calcul correct des permissions effectives | 20% |
| Documentation et tests démontrés (3 scénarios) | 15% |

### Modalité
Travail individuel ou en binôme, durée indicative : 1 semaine. Cet outil complète la suite d'audit Windows initiée aux modules 27 à 30, enrichie aux modules suivants (PowerShell avancé, durcissement).

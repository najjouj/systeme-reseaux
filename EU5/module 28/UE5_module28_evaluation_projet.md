# UE5 — Module 28 : Active Directory — Fondamentaux
## Compléments : ÉVALUATION / PROJET

---

# PARTIE I — ÉVALUATION

**Durée :** 1h30
**Modalités :** documents non autorisés sauf aide PowerShell locale si épreuve sur machine
**Barème total :** /20

## Section A — QCM (4 points, 0,5 pt/question)

1. Le regroupement le plus large dans la hiérarchie Active Directory est :
 a) le domaine b) l'arborescence c) la forêt d) l'unité d'organisation

2. Une unité d'organisation (OU) sert principalement à :
 a) attribuer des permissions à des fichiers b) organiser administrativement les objets et permettre la délégation de droits c) remplacer les groupes de sécurité d) stocker des mots de passe

3. Un utilisateur peut appartenir :
 a) à une seule OU à la fois, mais plusieurs groupes simultanément b) à plusieurs OU et un seul groupe c) ni à une OU ni à un groupe d) uniquement à des groupes de distribution

4. La réplication Active Directory est de type :
 a) maître-esclave strict b) multi-maître, avec des rôles FSMO pour certaines opérations spécifiques c) inexistante d) uniquement manuelle

5. La recommandation de déployer au moins deux contrôleurs de domaine vise à :
 a) accélérer l'authentification uniquement b) éviter un point de défaillance unique pour l'authentification et l'annuaire c) doubler automatiquement la sécurité des mots de passe d) réduire le nombre d'utilisateurs nécessaires

6. Un délai entre une modification effectuée sur un contrôleur de domaine et sa visibilité sur un autre contrôleur :
 a) indique toujours une panne b) peut être un délai de réplication normal, à vérifier avant de conclure à un problème c) est impossible dans Active Directory d) nécessite systématiquement un redémarrage

7. La délégation de droits d'administration sur une OU permet notamment de :
 a) accorder automatiquement les droits de Domain Admin b) restreindre un droit précis (ex: réinitialiser des mots de passe) à un périmètre organisationnel limité c) supprimer le besoin de contrôleurs de domaine d) remplacer les groupes de sécurité

8. Comparé à la gestion locale des comptes sous Linux (`/etc/passwd` par machine), Active Directory :
 a) fonctionne de façon identique, sans différence b) centralise la gestion d'identités, propageant automatiquement les modifications à toutes les machines membres c) est moins sécurisé par nature d) ne peut gérer qu'un seul utilisateur

## Section B — Questions ouvertes (6 points)

**Q1 (2 pts).** Expliquez le problème que résout Active Directory par rapport à une gestion locale des comptes, avec un exemple concret de risque de sécurité qu'il permet d'éviter.

**Q2 (2 pts).** Expliquez la différence entre une OU et un groupe Active Directory, en illustrant pourquoi les deux notions sont nécessaires et complémentaires.

**Q3 (2 pts).** Expliquez le mécanisme de réplication multi-maître d'Active Directory, et la nuance apportée par les rôles FSMO.

## Section C — Exercice technique (4 points)

Donnez les commandes PowerShell permettant de : créer une unité d'organisation nommée "Ventes", créer un utilisateur "Claire Petit" (SamAccountName "cpetit") dans cette OU, créer un groupe "GG_Ventes", et y ajouter cet utilisateur.

## Section D — Problème de diagnostic (6 points)

Un administrateur modifie l'appartenance à un groupe d'un utilisateur sur un contrôleur de domaine. Quelques minutes plus tard, un collègue signale que cette modification n'est toujours pas visible depuis un poste connecté à un autre contrôleur de domaine.

**a) (2 pts)** Formulez l'hypothèse la plus probable, en la distinguant explicitement d'une conclusion de panne prématurée.

**b) (2 pts)** Donnez la commande permettant de vérifier cette hypothèse.

**c) (2 pts)** Si la vérification révèle un retard de réplication anormalement long (plusieurs heures), proposez deux pistes d'investigation distinctes.

## Grille de correction résumée

| Section | Points | Critères |
|---|---|---|
| A — QCM | /4 | Réponse exacte uniquement |
| B — Questions ouvertes | /6 | Précision technique, distinction OU/groupe et nuance FSMO correctement articulées |
| C — Exercice technique | /4 | 4 commandes exactes et dans le bon ordre logique |
| D — Diagnostic | /6 | Hypothèse de réplication normale envisagée avant conclusion de panne, commande exacte, pistes d'investigation pertinentes |

---

# PARTIE II — MINI-PROJET (échelle module)

## Cahier des charges

**Titre :** Auditeur de structure Active Directory (AD Structure & Replication Auditor)

**Contexte :** Suite à l'outil de supervision de services développé au module 27, l'équipe système souhaite un second outil PowerShell dédié à la santé de l'annuaire Active Directory lui-même — comptes obsolètes, cohérence de la structure organisationnelle, et surtout état de la réplication, pour éviter la confusion entre délai normal et vraie panne déjà identifiée en cours et en TP.

### Objectifs du livrable

Développer un script PowerShell qui :
1. Audite les comptes utilisateurs désactivés depuis plus d'une durée configurable (reprise et extension du script du cours)
2. Vérifie l'état de réplication entre l'ensemble des contrôleurs de domaine, en distinguant explicitement un délai normal (sous un seuil configurable) d'un retard anormal nécessitant investigation
3. Audite la structure des unités d'organisation et signale celles vides (candidates à un nettoyage), ainsi que celles présentant une profondeur de hiérarchie inhabituelle (heuristique simple de complexité excessive)
4. Vérifie la cohérence entre les groupes de sécurité et leur usage réel (groupes sans membre, groupes non utilisés dans aucune permission si cette information est accessible)
5. Produit un rapport de synthèse structuré en catégories distinctes (comptes, réplication, structure OU, groupes), sur le modèle des rapports multi-catégories déjà développés en Bash (UE3-M20)
6. Documente l'exécution avec horodatage

### Contraintes techniques
- Le seuil de retard de réplication "normal" vs "anormal" doit être configurable et justifié dans la documentation, pas arbitraire (rappel direct de la leçon des seuils mal calibrés, déjà rencontrée en UE2-M10)
- Gestion d'erreurs robuste (module ActiveDirectory non chargé, contrôleur de domaine injoignable)
- Documentation d'installation et d'utilisation (README)
- Tests démontrés sur au moins 2 scénarios : audit sur une structure AD de laboratoire avec au moins un compte obsolète et une OU vide détectés correctement, vérification de réplication (réelle si environnement multi-DC disponible, ou simulée/documentée sinon)

### Livrables attendus
- Code source commenté
- README (installation, utilisation, format du rapport, justification des seuils choisis, limites connues)
- Journal de tests montrant les 2 scénarios validés, avec captures
- Rapport court (1 page) sur les choix techniques et le parallèle avec les outils d'audit Bash équivalents déjà développés (notamment la leçon sur les seuils calibrés de l'UE2-M10)

### Grille d'évaluation

| Critère | Pondération |
|---|---|
| Audit des comptes obsolètes | 20% |
| Vérification de réplication avec seuil justifié | 30% |
| Audit de structure OU et groupes | 25% |
| Documentation et tests démontrés (2 scénarios) | 15% |
| Parallèle avec les outils Bash équivalents | 10% |

### Modalité
Travail individuel ou en binôme, durée indicative : 1 semaine. Cet outil complète la suite d'audit Windows initiée au module 27, enrichie aux modules suivants (DNS/DHCP, GPO, NTFS).

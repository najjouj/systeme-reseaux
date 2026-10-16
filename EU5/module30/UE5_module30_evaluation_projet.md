# UE5 — Module 30 : Group Policy, Kerberos et LDAP
## Compléments : ÉVALUATION / PROJET

---

# PARTIE I — ÉVALUATION

**Durée :** 1h30
**Modalités :** documents non autorisés sauf aide PowerShell locale si épreuve sur machine
**Barème total :** /20

## Section A — QCM (4 points, 0,5 pt/question)

1. L'ordre d'application des GPO est résumé par l'acronyme :
 a) DOSL b) LSDOU c) ULDOS d) SDLOU

2. Un GPO plus spécifique (lié à une OU) l'emporte généralement sur un GPO plus général (lié au domaine) car :
 a) il est toujours plus récent b) il est appliqué plus tard dans l'ordre LSDOU c) il a une taille de fichier plus petite d) aucune règle ne détermine cela

3. Kerberos garantit notamment que :
 a) le mot de passe est chiffré mais transmis sur le réseau b) le mot de passe n'est jamais transmis sur le réseau, même chiffré c) aucune authentification n'est nécessaire d) les tickets sont valides indéfiniment

4. Un Ticket-Granting Ticket (TGT) sert à :
 a) accéder directement à une ressource sans autre étape b) obtenir des tickets de service sans avoir à ressaisir le mot de passe c) remplacer LDAP d) chiffrer le disque

5. LDAP, comparé à DNS :
 a) sont strictement identiques dans leur usage b) interroge un annuaire riche et structuré, tandis que DNS résout des noms vers des adresses IP c) remplace entièrement DNS d) ne concerne que les mots de passe

6. `gpresult /r` sert à :
 a) forcer l'application immédiate des GPO b) diagnostiquer précisément quels GPO sont effectivement appliqués sur un poste c) créer un nouveau GPO d) chiffrer les tickets Kerberos

7. Un changement de mot de passe n'invalide pas immédiatement :
 a) rien, tout est immédiatement invalidé b) les tickets Kerberos déjà émis, valides jusqu'à leur expiration naturelle c) le compte utilisateur lui-même d) la connexion réseau en cours

8. Une dérive d'horloge importante entre un poste client et un contrôleur de domaine peut provoquer :
 a) aucun effet notable b) des échecs d'authentification Kerberos, du fait de la tolérance temporelle limitée du protocole c) une désactivation automatique de LDAP d) un changement automatique de fuseau horaire

## Section B — Questions ouvertes (6 points)

**Q1 (2 pts).** Décrivez les trois étapes du fonctionnement de Kerberos et expliquez pourquoi le mot de passe n'est jamais transmis sur le réseau.

**Q2 (2 pts).** Expliquez l'ordre LSDOU et donnez un exemple de situation où un blocage d'héritage serait utile.

**Q3 (2 pts).** Expliquez pourquoi une dérive d'horloge peut provoquer des échecs d'authentification Kerberos difficiles à diagnostiquer, en reliant votre réponse au mécanisme de tolérance temporelle du protocole.

## Section C — Exercice technique (4 points)

Donnez la commande permettant de forcer l'application des GPO sur un poste, celle permettant d'observer les tickets Kerberos actuels, et le nom distinctif (Distinguished Name) LDAP pour un utilisateur "Claire Petit" dans l'OU "Ventes" du domaine "nomia.local".

## Section D — Problème de diagnostic (6 points)

Un GPO récemment créé et lié à une OU n'apparaît pas dans `gpresult /r` sur un poste appartenant censément à cette OU.

**a) (2 pts)** Formulez trois hypothèses distinctes pouvant expliquer cette absence.

**b) (2 pts)** Pour chacune, donnez une commande ou vérification permettant de la tester.

**c) (2 pts)** Expliquez pourquoi conclure hâtivement à une "erreur de configuration du GPO" sans avoir testé ces hypothèses serait une erreur méthodologique.

## Grille de correction résumée

| Section | Points | Critères |
|---|---|---|
| A — QCM | /4 | Réponse exacte uniquement |
| B — Questions ouvertes | /6 | Précision technique, mécanisme Kerberos et LSDOU correctement expliqués |
| C — Exercice technique | /4 | 3 éléments exacts |
| D — Diagnostic | /6 | Trois hypothèses distinctes et pertinentes, vérifications appropriées, raisonnement méthodologique correct |

---

# PARTIE II — MINI-PROJET (échelle module)

## Cahier des charges

**Titre :** Auditeur de stratégies de groupe et d'authentification (GPO & Kerberos Health Auditor)

**Contexte :** Suite aux outils des modules 27 à 29, l'équipe système souhaite un quatrième outil PowerShell dédié à la cohérence des GPO et à la détection précoce de problèmes d'authentification Kerberos liés à la dérive d'horloge — directement inspiré du scénario de l'exercice 4.1 du TD.

### Objectifs du livrable

Développer un script PowerShell qui :
1. Audite l'ensemble des OU et leurs GPO liés, avec l'état d'héritage et d'application forcée (reprise et extension du script du cours)
2. Détecte les OU avec blocage d'héritage actif, les signalant comme point d'attention nécessitant une documentation explicite de la justification (rappel du principe de traçabilité des exceptions déjà rencontré dans le programme)
3. Vérifie la dérive d'horloge entre le serveur exécutant le script et chaque contrôleur de domaine du domaine, alertant si un écart dépasse un seuil configurable proche de la tolérance Kerberos par défaut
4. Vérifie la cohérence de réplication SYSVOL pour les GPO récemment modifiés (dans la mesure des outils disponibles, une vérification simplifiée est acceptable avec limite documentée)
5. Produit un rapport de synthèse en catégories distinctes (structure GPO, blocages d'héritage, dérive d'horloge, cohérence SYSVOL)
6. Documente l'exécution avec horodatage

### Contraintes techniques
- La détection de dérive d'horloge doit être fiable et testée (comparaison explicite entre plusieurs sources de temps)
- Gestion d'erreurs robuste (module GroupPolicy non installé, contrôleur de domaine injoignable)
- Documentation d'installation et d'utilisation (README), avec la justification du seuil de dérive d'horloge choisi
- Tests démontrés sur au moins 2 scénarios : audit d'une structure GPO avec au moins un blocage d'héritage détecté, vérification de dérive d'horloge sur l'environnement de laboratoire disponible

### Livrables attendus
- Code source commenté
- README (installation, utilisation, format du rapport, justification du seuil de dérive d'horloge, limites connues)
- Journal de tests montrant les 2 scénarios validés, avec captures
- Rapport court (1 page) sur les choix techniques et le lien avec le scénario de l'exercice 4.1 du TD (dérive NTP causant des échecs Kerberos difficiles à diagnostiquer)

### Grille d'évaluation

| Critère | Pondération |
|---|---|
| Audit de structure GPO et héritage | 25% |
| Détection de blocages d'héritage non documentés | 20% |
| Détection de dérive d'horloge avec seuil justifié | 30% |
| Documentation et tests démontrés (2 scénarios) | 25% |

### Modalité
Travail individuel ou en binôme, durée indicative : 1 semaine. Cet outil complète la suite d'audit Windows initiée aux modules 27 à 29, enrichie au module suivant (NTFS/ACL).

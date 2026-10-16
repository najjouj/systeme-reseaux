# UE3 — Module 18 : SSH et Durcissement d'Accès Distant
## Compléments : ÉVALUATION / PROJET

---

# PARTIE I — ÉVALUATION

**Durée :** 1h30
**Modalités :** documents non autorisés sauf `man` local si épreuve sur machine
**Barème total :** /20

## Section A — QCM (4 points, 0,5 pt/question)

1. Dans l'authentification par clé SSH, ce qui ne transite jamais sur le réseau est :
 a) la clé publique b) la clé privée c) le nom d'utilisateur d) le port de connexion

2. `PasswordAuthentication no` dans `sshd_config` :
 a) désactive SSH entièrement b) force l'authentification par clé, éliminant le risque de force brute sur mot de passe c) autorise uniquement root d) déconnecte les sessions actives

3. Les permissions attendues pour une clé privée SSH sont :
 a) 644 b) 600 c) 777 d) 755

4. Le changement du port SSH par défaut :
 a) est une mesure de sécurité structurelle suffisante à elle seule b) réduit le bruit opérationnel mais n'empêche pas un scan ciblé c) désactive automatiquement l'authentification par mot de passe d) est obligatoire pour utiliser des clés

5. L'agent forwarding SSH (`-A`) présente un risque si :
 a) la machine intermédiaire n'est pas totalement fiable b) la clé privée est trop longue c) le port SSH a été changé d) aucune phrase de passe n'est définie

6. Avant de redémarrer le service SSH après modification de `sshd_config`, il faut :
 a) fermer sa session active b) valider la syntaxe avec `sshd -t` c) régénérer les clés d) désactiver le pare-feu

7. La méthodologie de prudence recommandée lors d'un durcissement SSH à distance consiste à :
 a) fermer immédiatement la session après modification b) tester une nouvelle connexion dans un second terminal avant de fermer la session active c) redémarrer la machine directement d) désactiver temporairement le pare-feu

8. Désactiver totalement la journalisation des échecs d'authentification SSH pour réduire le bruit :
 a) est une bonne pratique recommandée b) supprime la capacité de détecter un changement de profil d'attaque, sans traiter la cause réelle c) accélère les connexions SSH d) est nécessaire pour activer les clés

## Section B — Questions ouvertes (6 points)

**Q1 (2 pts).** Expliquez pourquoi l'authentification par clé est structurellement plus sûre qu'un mot de passe, en précisant ce qui transite réellement sur le réseau dans chaque cas.

**Q2 (2 pts).** Décrivez la méthodologie de prudence à respecter lors de l'application d'un durcissement SSH à distance, et le risque encouru si elle n'est pas suivie.

**Q3 (2 pts).** Un ingénieur propose de désactiver la journalisation des tentatives d'authentification SSH échouées pour réduire le volume de logs. Expliquez pourquoi c'est une erreur et proposez une alternative.

## Section C — Exercice technique (4 points)

Donnez les directives `sshd_config` (nom et valeur) permettant de : interdire la connexion directe en root, désactiver l'authentification par mot de passe, et restreindre l'accès aux seuls utilisateurs `alice` et `bob`.

## Section D — Problème de diagnostic (6 points)

Un utilisateur reçoit systématiquement "Permission denied (publickey)" en tentant de se connecter à un serveur, alors qu'il affirme avoir déployé sa clé publique avec `ssh-copy-id` la veille et que sa connexion fonctionnait initialement.

**a) (2 pts)** Proposez deux hypothèses distinctes pouvant expliquer cet échec, l'une liée à la clé elle-même, l'autre liée aux permissions.

**b) (2 pts)** Pour chacune des deux hypothèses, donnez la commande de vérification appropriée.

**c) (2 pts)** Donnez la commande permettant d'obtenir un diagnostic détaillé côté client sur la raison précise de l'échec.

## Grille de correction résumée

| Section | Points | Critères |
|---|---|---|
| A — QCM | /4 | Réponse exacte uniquement |
| B — Questions ouvertes | /6 | Précision technique, méthodologie de prudence correctement décrite |
| C — Exercice technique | /4 | 3 directives exactes avec valeurs correctes |
| D — Diagnostic | /6 | Deux hypothèses distinctes et pertinentes, commandes de vérification correctes |

---

# PARTIE II — MINI-PROJET (échelle module)

## Cahier des charges

**Titre :** Auditeur et sécurisateur d'accès SSH (SSH Access Auditor & Hardener)

**Contexte :** Suite aux outils des modules précédents (audit de comptes/permissions, gestion de capacité de stockage, supervision de services, déploiement sécurisé de pare-feu), l'équipe système souhaite un outil dédié à SSH — combinant audit de configuration et application prudente d'un durcissement, avec la même discipline de rollback déjà appliquée au pare-feu (module 17).

### Objectifs du livrable

Développer un outil (Bash ou Python, au choix justifié) qui :
1. Audite la configuration `sshd_config` actuelle et signale tout écart par rapport à un ensemble de bonnes pratiques configurables (`PermitRootLogin`, `PasswordAuthentication`, présence d'`AllowUsers`), en distinguant les écarts critiques des recommandations
2. Audite l'ensemble des `authorized_keys` du système : permissions correctes, cohérence avec `AllowUsers` (reprise et durcissement du script du cours), détection de clés potentiellement obsolètes (non utilisées depuis une durée configurable, si l'information est disponible via les journaux)
3. Applique, sur demande explicite, un jeu de directives de durcissement à `sshd_config`, avec **validation systématique de la syntaxe** (`sshd -t`) avant tout redémarrage du service
4. Intègre un mécanisme de rollback automatique (sur le modèle de l'outil du module 17) : sauvegarde de la configuration précédente, application, rollback automatique après un délai si aucune confirmation n'intervient
5. Refuse d'appliquer `PasswordAuthentication no` si aucune clé publique valide n'est détectée pour au moins un utilisateur autorisé (garde-fou structurel contre le scénario de coupure d'accès du TD/TP)
6. Journalise chaque audit et chaque application de configuration avec horodatage

### Contraintes techniques
- Le garde-fou de l'objectif 5 doit être fiable et testé explicitement (reproduction du scénario de perte d'accès du TD 3.1)
- Le mécanisme de rollback doit être cohérent avec celui déjà développé pour le pare-feu (réutilisation de la même approche si pertinent)
- Gestion d'erreurs robuste (fichier de configuration invalide, absence de répertoires `.ssh` sur certains comptes)
- Configuration externalisée (bonnes pratiques attendues, délai de rollback, liste des utilisateurs à auditer)
- Documentation d'installation et d'utilisation (README)
- Tests démontrés sur au moins 3 scénarios : audit d'une configuration non durcie (détection de tous les écarts), application réussie d'un durcissement avec confirmation, tentative d'application bloquée par le garde-fou (aucune clé valide détectée)

### Livrables attendus
- Code source commenté
- README (installation, utilisation, bonnes pratiques auditées, limites connues)
- Journal de tests montrant les 3 scénarios validés, avec captures
- Rapport court (1 page) sur les choix techniques, notamment le garde-fou anti-coupure d'accès, et les limites connues (ex : ne peut pas garantir qu'une clé détectée comme "valide" correspond réellement à une personne encore légitimement autorisée)

### Grille d'évaluation

| Critère | Pondération |
|---|---|
| Audit complet de `sshd_config` et des `authorized_keys` | 25% |
| Application sécurisée avec validation syntaxique systématique | 15% |
| Mécanisme de rollback automatique | 20% |
| Garde-fou anti-coupure d'accès (clé absente) | 25% |
| Documentation et tests démontrés (3 scénarios) | 15% |

### Modalité
Travail individuel ou en binôme, durée indicative : 1 semaine. Cet outil complète la série d'audit/déploiement sécurisé initiée aux modules 14 à 17 — il pourra être intégré comme brique de durcissement dans le projet intégrateur de fin d'UE3.

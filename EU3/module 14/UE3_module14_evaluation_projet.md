# UE3 — Module 14 : Installation, Utilisateurs et Permissions
## Compléments : ÉVALUATION / PROJET

---

# PARTIE I — ÉVALUATION

**Durée :** 1h30
**Modalités :** documents non autorisés sauf `man` local si épreuve sur machine
**Barème total :** /20

## Section A — QCM (4 points, 0,5 pt/question)

1. La permission octale `640` correspond à :
 a) `rwxr-----` b) `rw-r-----` c) `rw-rw----` d) `r--r-----`

2. La permission `x` sur un répertoire autorise :
 a) le listage de son contenu b) sa traversée (accès à un fichier connu, `cd`) c) sa suppression d) sa compression

3. Le fichier `/etc/shadow` est :
 a) lisible par tous les utilisateurs b) lisible uniquement par root c) supprimé par défaut sur les systèmes modernes d) identique à `/etc/passwd`

4. Le bit SUID sur un exécutable appartenant à root permet :
 a) de le rendre illisible aux autres utilisateurs b) à tout utilisateur autorisé à l'exécuter de le faire avec les privilèges de root c) d'empêcher son exécution d) de le rendre exécutable uniquement par root

5. Le SGID sur un répertoire provoque :
 a) l'héritage automatique du groupe du répertoire par les nouveaux fichiers créés à l'intérieur b) la suppression automatique des fichiers après 24h c) l'interdiction d'écriture pour tous sauf le propriétaire d) le chiffrement automatique du contenu

6. Un fichier affichant `rw-r-----+` dans `ls -l` indique :
 a) une erreur d'affichage b) la présence d'ACL étendues actives, potentiellement modifiant l'accès réel c) que le fichier est exécutable d) que le fichier appartient à plusieurs propriétaires

7. Comparé à `su`, `sudo` offre notamment :
 a) un accès root permanent sans mot de passe b) une granularité fine (commandes précises) et une traçabilité individuelle c) l'impossibilité de tracer les actions d) un accès identique dans tous les cas

8. `/etc/sudoers` doit être édité :
 a) avec n'importe quel éditeur de texte b) exclusivement via `visudo`, qui valide la syntaxe avant sauvegarde c) uniquement en mode graphique d) jamais, ce fichier est généré automatiquement

## Section B — Questions ouvertes (6 points)

**Q1 (2 pts).** Expliquez pourquoi la permission `r` seule sur un répertoire, sans `x`, ne permet pas d'accéder à un fichier à l'intérieur même en connaissant son nom exact.

**Q2 (2 pts).** Expliquez le mécanisme du SUID en termes d'UID réel et d'UID effectif, et pourquoi un binaire SUID appartenant à root est une cible d'audit prioritaire.

**Q3 (2 pts).** Un utilisateur membre du bon groupe et disposant en apparence des permissions suffisantes ne parvient pas à accéder à un fichier. Citez deux causes possibles non liées à une erreur de permission Unix classique sur le fichier lui-même.

## Section C — Exercice technique (4 points)

Donnez la séquence de commandes permettant de : créer un groupe `audit`, créer un répertoire `/data/audit` appartenant à ce groupe avec le bit SGID activé et des permissions `770`, puis vérifier que ces permissions sont correctement appliquées.

## Section D — Problème de diagnostic (6 points)

Un audit de sécurité révèle un binaire personnalisé sur un serveur de production, avec permission `rwsr-xr-x`, appartenant à `root`, exécutant une commande shell construite à partir d'un argument utilisateur non validé.

**a) (2 pts)** Expliquez précisément le risque que représente cette configuration, en citant le mécanisme exact permettant l'escalade de privilèges.

**b) (2 pts)** Proposez une action immédiate (avant investigation complète) pour limiter le risque.

**c) (2 pts)** Proposez une mesure préventive qui aurait permis de détecter ce binaire avant qu'un audit externe ne le révèle.

## Grille de correction résumée

| Section | Points | Critères |
|---|---|---|
| A — QCM | /4 | Réponse exacte uniquement |
| B — Questions ouvertes | /6 | Précision technique, mécanisme correctement expliqué (traversée, UID réel/effectif) |
| C — Exercice technique | /4 | Séquence de commandes complète et correcte |
| D — Diagnostic | /6 | Mécanisme d'escalade correctement expliqué, action immédiate pertinente, mesure préventive cohérente (audit SUID périodique) |

---

# PARTIE II — MINI-PROJET (échelle module)

## Cahier des charges

**Titre :** Auditeur de permissions et de comptes (Permission & Account Auditor)

**Contexte :** Suite à l'étude de cas du cours (binaire SUID vulnérable découvert tardivement par un audit externe), l'équipe sécurité souhaite un outil d'audit périodique automatisé des comptes, groupes et permissions sensibles d'un serveur Linux, pour détecter ce type de dérive avant qu'un incident ne survienne.

### Objectifs du livrable

Développer un outil (Bash ou Python, au choix justifié) qui :
1. Audite l'ensemble des comptes utilisateurs et signale ceux disposant d'un shell interactif (`/bin/bash`, `/bin/sh`...) alors qu'ils correspondent à un profil de compte de service (heuristique configurable : UID système, absence de mot de passe défini, nom correspondant à un service connu)
2. Audite les binaires SUID/SGID du système, comparés à une liste de référence validée manuellement une première fois (reprise et durcissement du script du cours), avec alerte sur tout nouvel ajout
3. Audite la configuration `sudo` de chaque utilisateur (via `sudo -l` ou lecture prudente de `/etc/sudoers.d/`) et signale toute règle jugée trop permissive selon des critères configurables (ex : `NOPASSWD: ALL`, `ALL=(ALL) ALL`)
4. Détecte les fichiers/répertoires sensibles (liste configurable, ex : `/etc/shadow`, `/etc/sudoers`) dont les permissions s'écartent d'une valeur de référence attendue
5. Journalise chaque anomalie détectée avec horodatage et niveau de gravité (critique, à surveiller)
6. Génère un rapport de synthèse périodique, structuré et exploitable pour une revue de sécurité

### Contraintes techniques
- La première exécution doit établir une base de référence validée manuellement (pas de fausse alerte massive au premier lancement) — reprise et généralisation de l'approche du script SUID du cours
- Gestion d'erreurs robuste (permissions insuffisantes pour certaines vérifications, absence de `/etc/sudoers.d/`)
- Configuration externalisée (heuristiques de détection de comptes de service, règles sudo jugées trop permissives, liste des fichiers sensibles à surveiller)
- Documentation d'installation et d'utilisation (README)
- Tests démontrés sur au moins 3 scénarios : ajout d'un nouveau binaire SUID (comme au TP), création d'une règle sudo trop permissive, modification des permissions d'un fichier sensible surveillé

### Livrables attendus
- Code source commenté
- README (installation, configuration, format du rapport, limites connues)
- Journal de tests montrant les 3 scénarios validés, avec captures avant/après détection
- Rapport court (1 page) sur les choix techniques et les limites connues (ex : ne peut pas évaluer si un binaire SUID légitime contient lui-même une faille de sécurité, seulement signaler sa présence pour investigation humaine)

### Grille d'évaluation

| Critère | Pondération |
|---|---|
| Audit des comptes de service mal configurés | 20% |
| Audit SUID/SGID avec référence validée | 25% |
| Audit des règles sudo trop permissives | 25% |
| Audit des permissions de fichiers sensibles | 15% |
| Documentation et tests démontrés (3 scénarios) | 15% |

### Modalité
Travail individuel ou en binôme, durée indicative : 1 semaine. Ce premier outil de l'UE3 pourra être enrichi au fil des modules suivants (stockage, réseau, logs) pour constituer, en fin d'UE3, une suite d'audit système complète — en écho à la série d'outils développée en UE1.

# UE3 — Module 17 : Réseau Côté Administration
## Compléments : ÉVALUATION / PROJET

---

# PARTIE I — ÉVALUATION

**Durée :** 1h30
**Modalités :** documents non autorisés sauf `man` local si épreuve sur machine
**Barème total :** /20

## Section A — QCM (4 points, 0,5 pt/question)

1. Une configuration réseau appliquée uniquement via `ip addr add` :
 a) survit à tout redémarrage b) est volatile, perdue au redémarrage sauf configuration persistante additionnelle c) est automatiquement sauvegardée d) nécessite un accès root permanent

2. La chaîne netfilter appropriée pour du trafic qui ne fait que traverser la machine (rôle de routeur) est :
 a) INPUT b) OUTPUT c) FORWARD d) LOCAL

3. Une politique par défaut `DROP` sur `INPUT`, avec autorisation explicite du nécessaire, correspond à une approche :
 a) liste noire b) liste blanche c) sans politique définie d) réservée aux tests

4. Le filtrage par état (`ESTABLISHED,RELATED`) permet notamment :
 a) de bloquer tout le trafic entrant sans exception b) d'autoriser automatiquement le trafic de retour d'une connexion légitimement établie c) de désactiver le NAT d) de remplacer entièrement la politique par défaut

5. `net.ipv4.ip_forward` doit être activé pour :
 a) accélérer le DNS b) permettre à une machine Linux de transmettre des paquets entre ses interfaces (rôle de routeur) c) activer SSH d) désactiver le pare-feu

6. Appliquer une politique `DROP` sur `INPUT` avant d'avoir autorisé sa propre connexion SSH distante risque de :
 a) accélérer la connexion b) couper immédiatement l'accès administrateur à distance c) n'avoir aucun effet d) désactiver automatiquement le pare-feu

7. Un problème sur la chaîne `OUTPUT` (connexions sortantes bloquées) est généralement :
 a) plus immédiatement visible qu'un problème INPUT b) potentiellement plus difficile à détecter, car n'affectant que certaines fonctionnalités précises c) impossible à diagnostiquer d) toujours identique à un problème INPUT

8. Un mécanisme de rollback automatique (via `at` par exemple) avant un changement de pare-feu à distance sert à :
 a) accélérer l'application des règles b) restaurer automatiquement une configuration de secours si aucune confirmation n'intervient, en cas de coupure d'accès c) remplacer la sauvegarde manuelle d) désactiver le pare-feu après usage

## Section B — Questions ouvertes (6 points)

**Q1 (2 pts).** Expliquez pourquoi l'ordre des opérations est crucial lors de l'application d'une politique de pare-feu restrictive en administration à distance, avec l'ordre correct à respecter.

**Q2 (2 pts).** Expliquez la différence entre les chaînes INPUT, OUTPUT et FORWARD, avec un exemple de trafic concerné par chacune.

**Q3 (2 pts).** Expliquez pourquoi un problème de configuration sur la chaîne OUTPUT peut rester inaperçu plus longtemps qu'un problème équivalent sur INPUT.

## Section C — Exercice technique (4 points)

Donnez la séquence complète de commandes `iptables` permettant d'appliquer, dans le bon ordre et sans risque de coupure d'accès SSH, une politique restrictive sur INPUT autorisant uniquement : le loopback, le trafic établi/lié, et le port SSH (22).

## Section D — Problème de diagnostic (6 points)

Un serveur configuré comme passerelle NAT ne transmet aucun trafic entre deux réseaux, malgré des règles de pare-feu et de routage en apparence correctes.

**a) (2 pts)** Donnez la première vérification à effectuer, souvent oubliée dans ce type de configuration.

**b) (2 pts)** Donnez la commande permettant de la réaliser et celle permettant de corriger le problème si cette hypothèse est confirmée.

**c) (2 pts)** Si cette première vérification ne révèle aucun problème, proposez une seconde piste de diagnostic, liée aux chaînes netfilter vues en cours.

## Grille de correction résumée

| Section | Points | Critères |
|---|---|---|
| A — QCM | /4 | Réponse exacte uniquement |
| B — Questions ouvertes | /6 | Précision technique, raisonnement sur l'ordre des opérations et la visibilité différentielle INPUT/OUTPUT |
| C — Exercice technique | /4 | Ordre correct des commandes (règles avant politique DROP) |
| D — Diagnostic | /6 | Vérification `ip_forward` en premier, commandes exactes, seconde piste FORWARD pertinente |

---

# PARTIE II — MINI-PROJET (échelle module)

## Cahier des charges

**Titre :** Gestionnaire de pare-feu sécurisé (Safe Firewall Deployer)

**Contexte :** Suite à l'incident classique identifié en cours et vécu en TP (coupure d'accès SSH par une mauvaise politique de pare-feu), l'équipe système souhaite un outil qui sécurise structurellement le déploiement de nouvelles règles de pare-feu à distance, en intégrant nativement le mécanisme de rollback automatique plutôt que de le laisser à la discipline manuelle de chaque administrateur.

### Objectifs du livrable

Développer un outil (Bash ou Python, au choix justifié) qui :
1. Sauvegarde automatiquement et horodate l'état actuel des règles de pare-feu avant toute application d'une nouvelle configuration
2. Applique un nouveau jeu de règles fourni en argument (fichier de règles `iptables-save`/`nftables` ou équivalent)
3. Programme systématiquement, sans que l'utilisateur ait à y penser, un rollback automatique après un délai configurable (par défaut prudent, ex : 5 minutes)
4. Fournit une commande de confirmation explicite que l'administrateur doit exécuter pour annuler le rollback programmé, une fois la nouvelle configuration validée comme fonctionnelle
5. Vérifie, avant même d'appliquer toute nouvelle politique `DROP` par défaut sur INPUT, qu'une règle autorisant la connexion SSH actuelle (ou le port SSH configuré) est bien présente dans le nouveau jeu de règles — refuse l'application et alerte si ce n'est pas le cas (garde-fou structurel contre l'erreur classique du cours)
6. Journalise chaque application de règles, chaque rollback (déclenché automatiquement ou annulé manuellement) avec horodatage

### Contraintes techniques
- Le garde-fou SSH (objectif 5) doit être fiable : détecter la présence d'une règle d'autorisation SSH même si elle n'est pas formulée exactement de la même façon que dans les exemples du cours (tester plusieurs formulations plausibles)
- Le mécanisme de rollback ne doit jamais dépendre uniquement de la disponibilité continue de la connexion qui a déclenché le changement (cohérent avec la question 2 du TP sur le risque si `atd` échoue)
- Gestion d'erreurs robuste (fichier de règles invalide, `at`/`atd` non disponible — proposer une alternative de repli, ex : un minuteur applicatif local)
- Documentation d'installation et d'utilisation (README)
- Tests démontrés sur au moins 3 scénarios : application réussie avec confirmation manuelle avant rollback, application risquée bloquée par le garde-fou SSH (règle SSH absente du nouveau jeu de règles), rollback automatique déclenché après expiration du délai sans confirmation

### Livrables attendus
- Code source commenté
- README (installation, utilisation, format des fichiers de règles acceptés, limites connues)
- Journal de tests montrant les 3 scénarios validés, avec captures
- Rapport court (1 page) sur les choix techniques, notamment la méthode de détection fiable d'une règle SSH, et les limites connues (ex : ne protège pas contre un changement de port SSH non anticipé dans sa configuration)

### Grille d'évaluation

| Critère | Pondération |
|---|---|
| Sauvegarde et application correctes des règles | 15% |
| Mécanisme de rollback automatique fiable | 25% |
| Garde-fou de détection de règle SSH manquante | 30% |
| Robustesse (repli si `at` indisponible) | 15% |
| Documentation et tests démontrés (3 scénarios) | 15% |

### Modalité
Travail individuel ou en binôme, durée indicative : 1 semaine. Cet outil complète la série d'audit/déploiement système initiée aux modules 14 à 16 — il pourra être intégré comme brique de sécurisation dans le projet intégrateur de fin d'UE3 (Projet 1 : déployer et sécuriser une infrastructure Linux).

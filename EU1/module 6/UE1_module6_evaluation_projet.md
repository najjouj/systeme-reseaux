# UE1 — Module 6 : Boot Process
## Compléments : ÉVALUATION / PROJET

---

# PARTIE I — ÉVALUATION

**Durée :** 1h30
**Modalités :** documents non autorisés sauf `man` local si épreuve sur machine
**Barème total :** /20

## Section A — QCM (4 points, 0,5 pt/question)

1. Le POST (Power-On Self-Test) est réalisé par :
 a) systemd b) le firmware (BIOS/UEFI) c) GRUB d) le noyau Linux

2. UEFI, comparé au BIOS historique, apporte notamment :
 a) le support du MBR uniquement b) le support du GPT et de disques de grande taille, ainsi que le Secure Boot c) la suppression du bootloader d) l'exécution exclusive en mode 16 bits

3. L'initramfs est chargé et exécuté :
 a) après que systemd ait démarré b) avant que le système de fichiers racine réel ne soit monté c) uniquement en cas d'erreur du noyau d) par GRUB directement, sans intervention du noyau

4. PID 1 (systemd) a pour particularité de :
 a) ne jamais pouvoir être arrêté b) adopter les processus orphelins et servir d'ancêtre ultime à tous les autres processus c) s'exécuter avant le noyau d) remplacer le bootloader

5. Contrairement à SysVinit, systemd démarre les unités :
 a) toujours dans un ordre strictement séquentiel b) en parallèle selon un graphe de dépendances explicite c) uniquement sur demande manuelle d) sans jamais résoudre de dépendances

6. `systemctl stop <service>` suivi d'un redémarrage réel de la machine :
 a) empêche systématiquement le service de redémarrer b) n'empêche pas le redémarrage automatique si le service est toujours `enabled` c) désactive définitivement le service d) supprime le fichier d'unité

7. Un déchiffrement de disque (LUKS) requis avant le montage de la racine se situe nécessairement :
 a) au niveau de systemd b) dans la phase initramfs (ou éventuellement bootloader), avant que systemd ne puisse être lancé c) après l'atteinte de multi-user.target d) dans le firmware exclusivement

8. La directive `After=` dans un fichier d'unité systemd garantit que :
 a) l'unité citée est totalement terminée et pleinement opérationnelle avant le démarrage de l'unité courante b) uniquement un ordre relatif de déclenchement, sans garantie que l'unité citée soit pleinement fonctionnelle c) aucune relation d'ordre n'existe entre les deux unités d) l'unité citée sera désactivée

## Section B — Questions ouvertes (6 points)

**Q1 (2 pts).** Décrivez les grandes étapes de la séquence de démarrage d'un système Linux moderne, du firmware jusqu'à l'espace utilisateur, en situant précisément où se place l'initramfs et pourquoi.

**Q2 (2 pts).** Expliquez pourquoi une directive `After=multi-user.target` dans une unité systemd de vérification post-démarrage peut être insuffisante pour garantir que les services qu'elle contrôle sont réellement actifs, et proposez la correction appropriée.

**Q3 (2 pts).** Un serveur affiche un shell de secours initramfs au démarrage après une mise à jour du noyau. Expliquez pourquoi cela exclut d'emblée une cause purement applicative (au niveau des services systemd), en justifiant par la position de cette étape dans la séquence de démarrage.

## Section C — Exercice technique (4 points)

Écrivez le contenu d'un fichier d'unité systemd de type `oneshot`, nommé `verif-demarrage.service`, qui exécute un script `/usr/local/bin/verif.sh` après l'atteinte de `multi-user.target`, et qui doit être activé automatiquement à chaque démarrage.

## Section D — Problème de diagnostic (6 points)

Après un redémarrage planifié d'un serveur, un service métier critique (`payment-gateway.service`) n'a pas démarré automatiquement, alors qu'il fonctionnait normalement avant le redémarrage.

**a) (2 pts)** Donnez la première commande à exécuter pour déterminer si le problème vient d'une désactivation de l'unité ou d'une erreur au démarrage, et précisez ce que chaque résultat possible indiquerait.

**b) (2 pts)** En supposant que l'unité est bien `enabled` mais qu'elle a échoué au démarrage, donnez la commande permettant d'examiner les logs spécifiques à cette unité pour ce démarrage précis.

**c) (2 pts)** Proposez deux hypothèses distinctes de cause racine (l'une liée aux dépendances déclarées de l'unité, l'autre à une cause purement applicative), et pour chacune, la commande permettant de l'investiguer.

## Grille de correction résumée

| Section | Points | Critères |
|---|---|---|
| A — QCM | /4 | Réponse exacte uniquement |
| B — Questions ouvertes | /6 | Précision technique, bonne situation des étapes dans la séquence de démarrage |
| C — Exercice technique | /4 | Fichier d'unité syntaxiquement correct et complet (sections [Unit]/[Service]/[Install]) |
| D — Diagnostic | /6 | Démarche méthodique, commandes pertinentes, hypothèses distinctes et argumentées |

---

# PARTIE II — MINI-PROJET (échelle module)

## Cahier des charges

**Titre :** Superviseur de démarrage (Boot Health Supervisor)

**Contexte :** Après les trois outils précédents (Process Watchdog, Memory Drift Detector, Ghost Disk Space Auditor), l'équipe DevOps souhaite fermer la boucle avec un outil dédié à la fiabilité du démarrage : détecter automatiquement, à chaque reboot, tout écart entre l'état attendu du système et son état réel, avant que les utilisateurs ne signalent une panne.

### Objectifs du livrable

Développer un outil (Bash ou Python, au choix justifié), packagé comme unité systemd de type `oneshot`, qui :
1. Vérifie qu'une liste configurable de services critiques sont `enabled` **et** effectivement `active` après démarrage (les deux vérifications sont distinctes, cf. cours §16 et TD 3.1)
2. Vérifie que la target atteinte correspond à la target par défaut attendue
3. Consulte `systemd-analyze blame` et signale toute unité dépassant un seuil de temps de démarrage configurable
4. Consulte `systemd-analyze critical-chain` et signale si une unité inattendue (hors liste blanche configurable) apparaît dans la chaîne critique
5. Déclare explicitement ses propres dépendances (`After=`) envers **chacun** des services qu'il vérifie, pour éviter le piège de fausse alerte identifié au TP (dépendance implicite insuffisante sur une target générique)
6. Journalise un rapport structuré et distingue clairement anomalies bloquantes (service critique inactif) et anomalies de performance (démarrage lent)

### Contraintes techniques
- Fichier d'unité systemd fourni et fonctionnel (`.service`), avec dépendances correctement déclarées
- Configuration externalisée (liste des services critiques, seuils de temps)
- Gestion d'erreurs robuste (service inexistant, `systemd-analyze` indisponible)
- Documentation d'installation et d'utilisation (README)
- Tests démontrés sur au moins 2 scénarios : un service critique volontairement `disabled` avant reboot, et une unité factice ralentissant artificiellement le démarrage

### Livrables attendus
- Code source commenté
- Fichier d'unité systemd (`.service`)
- README (installation, configuration, format du rapport)
- Journal de tests montrant les 2 scénarios validés, avec captures `systemctl status` avant/après correction
- Rapport court (1 page) sur les choix techniques, notamment la gestion des dépendances explicites (`After=`) et les limites connues (ex : ne détecte pas une régression de performance progressive sur plusieurs reboots sans historique conservé)

### Grille d'évaluation

| Critère | Pondération |
|---|---|
| Vérification correcte enabled + active (distinction respectée) | 20% |
| Détection des unités lentes / chaîne critique anormale | 20% |
| Dépendances systemd correctement déclarées (pas de fausse alerte) | 20% |
| Robustesse et configuration externalisée | 15% |
| Documentation et rapport | 15% |
| Tests démontrés (2 scénarios) | 10% |

### Modalité
Travail individuel ou en binôme, durée indicative : 1 semaine.

**Perspective de fin d'UE1 :** les quatre outils développés au fil des modules (Process Watchdog, Memory Drift Detector, Ghost Disk Space Auditor, Boot Health Supervisor) couvrent ensemble CPU/processus, mémoire, disque et démarrage — ils constituent une base de supervision système cohérente. Une synthèse ou une intégration de ces outils pourra être proposée comme projet de clôture d'UE1, une fois tous les modules validés.

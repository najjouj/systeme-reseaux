# UE3 — Module 19 : Logs et Tâches Planifiées
## Compléments : ÉVALUATION / PROJET

---

# PARTIE I — ÉVALUATION

**Durée :** 1h30
**Modalités :** documents non autorisés sauf `man` local si épreuve sur machine
**Barème total :** /20

## Section A — QCM (4 points, 0,5 pt/question)

1. Par défaut, sur de nombreuses distributions, le journal `journald` est :
 a) toujours persistant b) volatile, perdu au redémarrage sauf configuration explicite c) désactivé par défaut d) répliqué automatiquement sur le réseau

2. `SystemMaxUse` dans `journald.conf` sert à :
 a) accélérer les requêtes journalctl b) limiter l'espace disque occupé par le journal, évitant une saturation c) chiffrer le journal d) définir le niveau de sévérité minimal

3. La directive `postrotate`/`endscript` de `logrotate` sert typiquement à :
 a) supprimer définitivement les anciens logs b) signaler à l'application de rouvrir son fichier de log après rotation c) compresser les logs plus anciens d) désactiver la rotation suivante

4. La cause la plus fréquente d'échec d'un script fonctionnant manuellement mais pas via cron est :
 a) un problème de permissions systématique b) l'environnement minimal de cron, notamment un PATH réduit c) l'absence de réseau d) une erreur de syntaxe cron systématique

5. Le choix entre UDP et TCP pour l'envoi de logs vers un serveur distant via rsyslog reflète :
 a) une contrainte propre à rsyslog sans rapport avec la théorie réseau b) le même compromis fiabilité/légèreté déjà vu pour UDP/TCP en UE2 c) uniquement une préférence esthétique d) une obligation légale

6. Un timer systemd, comparé à cron, offre notamment :
 a) une syntaxe plus simple pour toute tâche b) une journalisation intégrée et un rattrapage natif via Persistent=true c) l'impossibilité de planifier des tâches quotidiennes d) une incompatibilité avec les scripts Bash

7. Un mécanisme de heartbeat pour une tâche planifiée critique sert à :
 a) accélérer son exécution b) détecter activement si la tâche a bien fonctionné, indépendamment de sa simple présence dans la configuration c) remplacer entièrement logrotate d) chiffrer ses résultats

8. Stocker un fichier d'état devant survivre à un redémarrage dans `/tmp` :
 a) est toujours sûr et recommandé b) risque une perte de cet état, `/tmp` étant généralement nettoyé au redémarrage sur de nombreuses distributions c) accélère automatiquement les scripts d) est obligatoire pour cron

## Section B — Questions ouvertes (6 points)

**Q1 (2 pts).** Expliquez pourquoi un script fonctionnant parfaitement en test manuel peut échouer silencieusement lorsqu'il est exécuté par cron, et citez deux bonnes pratiques pour l'éviter.

**Q2 (2 pts).** Expliquez le lien entre les directives `postrotate`/`copytruncate` de `logrotate` et le piège de rotation de logs déjà étudié en UE1 (fichier renommé, descripteur toujours ouvert).

**Q3 (2 pts).** Une entreprise découvre qu'une tâche de sauvegarde critique a échoué silencieusement pendant plusieurs mois sans qu'aucune alerte ne soit déclenchée. Proposez deux axes de correction distincts et complémentaires.

## Section C — Exercice technique (4 points)

Donnez la configuration `/etc/systemd/journald.conf` (section `[Journal]`) permettant d'activer la persistance du journal avec une limite de 1 Go, ainsi que la ligne cron permettant d'exécuter un script toutes les heures, à la minute 0.

## Section D — Problème de diagnostic (6 points)

Une tâche cron ajoutée récemment ne semble jamais s'exécuter, sans erreur visible dans la sortie habituelle.

**a) (2 pts)** Donnez la première vérification à effectuer concernant le service cron lui-même.

**b) (2 pts)** Donnez la commande permettant de consulter les logs système du démon cron pour vérifier si la tâche a seulement été tentée.

**c) (2 pts)** Si la tâche est bien déclenchée par cron mais échoue, proposez la méthodologie de diagnostic à appliquer, en citant le piège le plus probable.

## Grille de correction résumée

| Section | Points | Critères |
|---|---|---|
| A — QCM | /4 | Réponse exacte uniquement |
| B — Questions ouvertes | /6 | Précision technique, lien correct avec les modules précédents (UE1, UE2) |
| C — Exercice technique | /4 | Configuration journald et ligne cron exactes |
| D — Diagnostic | /6 | Démarche méthodique en 3 étapes, piège PATH correctement identifié |

---

# PARTIE II — MINI-PROJET (échelle module)

## Cahier des charges

**Titre :** Superviseur de journalisation et de tâches planifiées (Log & Schedule Guardian)

**Contexte :** Suite à l'étude de cas du cours (perte de données après l'échec silencieux et prolongé d'une sauvegarde cron), l'équipe système souhaite un outil unifié couvrant à la fois la santé de la journalisation et la fiabilité des tâches planifiées critiques du serveur.

### Objectifs du livrable

Développer un outil (Bash ou Python, au choix justifié) qui :
1. Vérifie la configuration de persistance de `journald` (`Storage=persistent`, présence d'une limite `SystemMaxUse`) et signale toute machine non conforme
2. Audite l'ensemble des tâches cron du système (crontabs utilisateur et `/etc/cron.d/`) et détecte les commandes invoquées sans chemin absolu (heuristique de détection du piège PATH), en distinguant clairement ce signal d'un vrai test d'exécution
3. Implémente un mécanisme de heartbeat générique et réutilisable pour une liste configurable de tâches critiques, avec stockage dans un emplacement **garanti persistant** (jamais `/tmp`), incluant une tolérance explicite pendant la fenêtre suivant un redémarrage système (correction directe du piège du TP)
4. Vérifie l'existence d'une configuration `logrotate` pour chaque application journalisant significativement dans `/var/log` (heuristique de taille de fichier sans rotation associée détectée)
5. Journalise chaque détection avec horodatage et niveau de gravité
6. Génère un rapport de synthèse périodique unifiant les deux volets (journalisation et tâches planifiées)

### Contraintes techniques
- Le mécanisme de heartbeat doit être démontrablement insensible à un redémarrage (test explicite requis, reproduisant et corrigeant le scénario du TP)
- La détection de commandes sans chemin absolu ne doit pas produire de faux positifs sur des constructions shell légitimes (variables, pipes internes) — à tester explicitement
- Gestion d'erreurs robuste (crontabs absentes, permissions insuffisantes sur certains fichiers)
- Configuration externalisée (liste des tâches critiques à surveiller, seuils, tolérance post-redémarrage)
- Documentation d'installation et d'utilisation (README)
- Tests démontrés sur au moins 3 scénarios : détection d'une commande cron sans chemin absolu, heartbeat survivant à un redémarrage simulé (ou réel si possible), configuration logrotate manquante détectée pour une application de test

### Livrables attendus
- Code source commenté
- README (installation, configuration, format du rapport, limites connues)
- Journal de tests montrant les 3 scénarios validés, avec captures
- Rapport court (1 page) sur les choix techniques, notamment la gestion de la fenêtre de tolérance post-redémarrage, et les limites connues (ex : l'audit de PATH sans chemin absolu reste une heuristique, pas une garantie formelle d'échec)

### Grille d'évaluation

| Critère | Pondération |
|---|---|
| Audit de la persistance journald | 15% |
| Détection des commandes cron sans chemin absolu (sans faux positif excessif) | 25% |
| Mécanisme de heartbeat robuste au redémarrage | 30% |
| Audit de couverture logrotate | 15% |
| Documentation et tests démontrés (3 scénarios) | 15% |

### Modalité
Travail individuel ou en binôme, durée indicative : 1 semaine. Cet outil complète la série d'audit système initiée aux modules 14 à 18 — l'ensemble converge vers une suite d'audit unifiée à mobiliser pour le projet intégrateur de fin d'UE3.

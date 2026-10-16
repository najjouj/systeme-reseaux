# UE1 — Module 5 : IPC, Synchronisation et Deadlocks
## Compléments : ÉVALUATION / PROJET

---

# PARTIE I — ÉVALUATION

**Durée :** 1h30
**Modalités :** documents non autorisés sauf `man` local si épreuve sur machine
**Barème total :** /20

## Section A — QCM (4 points, 0,5 pt/question)

1. Un appel système bascule le CPU :
 a) en mode utilisateur b) en mode noyau (ring 0) c) en mode veille d) en mode secours

2. Le mécanisme d'IPC qui ne nécessite aucune copie de données entre processus est :
 a) le pipe anonyme b) la file de messages c) la mémoire partagée d) le socket réseau

3. Un sémaphore initialisé à 1 se comporte fonctionnellement comme :
 a) un spinlock b) un mutex c) une variable de condition d) un pipe

4. Un spinlock, contrairement à un mutex, fait qu'une unité en attente :
 a) se bloque sans consommer de CPU b) boucle activement (busy-waiting) c) est automatiquement terminée d) libère tous ses verrous

5. L'algorithme du banquier appartient à la stratégie de :
 a) prévention b) évitement c) détection et rétablissement d) aucune des trois

6. Un segment de mémoire partagée System V créé par un processus qui se termine sans le libérer :
 a) est automatiquement détruit par le noyau b) persiste tant qu'il n'est pas explicitement supprimé c) devient inaccessible à tout futur processus d) se transforme en fichier régulier

7. Le mécanisme futex sous Linux permet, dans le cas non contesté (pas d'attente) :
 a) d'éviter tout appel système lors de l'acquisition d'un mutex b) de forcer systématiquement un changement de contexte c) de remplacer les sémaphores System V d) de garantir l'absence d'interblocage

8. Un ordre total d'acquisition des mutex, appliqué de façon cohérente dans tout le code, permet de :
 a) accélérer les appels système b) casser la condition d'attente circulaire et donc prévenir l'interblocage c) remplacer les sémaphores d) éviter toute race condition à elle seule

## Section B — Questions ouvertes (6 points)

**Q1 (2 pts).** Comparez pipe anonyme, FIFO nommée et mémoire partagée sur deux critères précis : la nécessité d'une relation de parenté entre processus, et la présence ou non d'une synchronisation automatique.

**Q2 (2 pts).** Expliquez pourquoi un ordre d'acquisition de mutex incohérent entre deux portions de code peut provoquer un interblocage, même si chaque portion de code est correcte lorsqu'elle est examinée isolément.

**Q3 (2 pts).** Expliquez la nuance entre "état non sûr" (unsafe state) selon l'algorithme du banquier et "interblocage certain" — ces deux notions sont-elles équivalentes ? Justifiez.

## Section C — Exercice technique (4 points)

On dispose de deux threads T1 et T2 devant tous deux acquérir deux mutex `lock_x` et `lock_y`. Écrivez, en pseudo-code ou en Python, une version de ces deux threads qui **évite structurellement** tout risque d'interblocage entre eux, en appliquant le principe vu en cours.

## Section D — Problème de diagnostic (6 points)

Un pipeline de traitement composé de 3 processus communiquant via des files de messages System V se fige totalement après plusieurs heures de fonctionnement, sans qu'aucun processus ne crashe.

**a) (2 pts)** Donnez la première commande à exécuter pour observer l'état des 3 processus, et précisez ce que chaque état possible (`S`, `D`, `R`) indiquerait dans ce contexte.

**b) (2 pts)** Donnez la commande permettant d'observer l'état des files de messages System V utilisées, et précisez ce qu'une file "pleine" pourrait indiquer, en le distinguant explicitement d'un interblocage classique par cycle de verrous.

**c) (2 pts)** Proposez une commande plus fine permettant de confirmer précisément sur quel appel système chacun des 3 processus est bloqué, et expliquez comment son résultat permettrait de trancher entre les deux hypothèses (interblocage classique vs saturation de capacité).

## Grille de correction résumée

| Section | Points | Critères |
|---|---|---|
| A — QCM | /4 | Réponse exacte uniquement |
| B — Questions ouvertes | /6 | Précision technique, distinction correcte entre les concepts proches (ex: unsafe state vs deadlock certain) |
| C — Exercice technique | /4 | Solution structurellement correcte (ordre total d'acquisition ou équivalent), applicable aux deux threads |
| D — Diagnostic | /6 | Démarche méthodique, distinction claire entre les deux hypothèses, commandes pertinentes |

---

# PARTIE II — MINI-PROJET (échelle module)

## Cahier des charges

**Titre :** Détecteur d'interblocage et de fuite IPC (Deadlock & IPC Leak Detector)

**Contexte :** Pour clore la série d'outils développés au fil de l'UE1 (Process Watchdog, Memory Drift Detector, Ghost Disk Space Auditor, Boot Health Supervisor), l'équipe DevOps souhaite un cinquième outil ciblant spécifiquement les blocages applicatifs liés à la synchronisation et aux ressources IPC — la catégorie d'incidents la plus difficile à diagnostiquer sans outillage dédié, comme l'ont montré le cours et le TP.

### Objectifs du livrable

Développer un outil (Bash ou Python, au choix justifié) qui :
1. Scanne périodiquement les ressources IPC System V actives (`ipcs -a`) et détecte celles sans processus attaché (`nattch=0`) depuis plus de N secondes configurables — en s'appuyant sur un parsing **robuste** (basé sur les en-têtes de colonnes, pas sur une position fixe, conformément à la leçon du troubleshooting du TP)
2. Pour une liste configurable de processus surveillés, détecte les threads/processus bloqués depuis plus de M secondes sur un appel système lié à la synchronisation (`futex`) ou aux IPC (`msgsnd`, `msgrcv`, `semop`), via `strace` en mode court ou lecture de `/proc/PID/status` et `/proc/PID/wchan` si disponible
3. Distingue explicitement, dans son rapport, une alerte de type "fuite de ressource IPC" d'une alerte de type "blocage prolongé en synchronisation", ces deux catégories ne nécessitant pas la même action corrective
4. Journalise chaque détection avec horodatage, PID/ressource concernée, durée du blocage ou de l'orphelinage
5. Ne prend aucune action corrective automatique (ne tue aucun processus, ne supprime aucune ressource IPC) — propose uniquement une commande suggérée (`ipcrm ...`, ou signalement du processus concerné)
6. Génère un rapport hebdomadaire de synthèse

### Contraintes techniques
- Parsing robuste et testé (reproduire explicitement le bug de fragilité de parsing rencontré au TP, dans un test automatisé, pour prouver que l'outil y résiste)
- Configuration externalisée (seuils, liste de processus/ressources à surveiller)
- Gestion d'erreurs robuste (`strace` non disponible ou permissions insuffisantes — dans ce cas, dégrader proprement vers `/proc/PID/status`/`wchan` plutôt que de planter)
- Documentation d'installation et d'utilisation (README)
- Tests démontrés sur au moins 2 scénarios : ressource IPC orpheline (comme au TP), et interblocage réel provoqué (script `deadlock_demo.py` du TP ou équivalent)

### Livrables attendus
- Code source commenté
- README (installation, configuration, format du rapport, limites connues)
- Journal de tests montrant les 2 scénarios validés, avec captures avant/après
- Rapport court (1 page) sur les choix techniques, notamment la stratégie de détection de blocage prolongé (approche par `strace` ponctuel vs lecture directe de `/proc`, avec justification du compromis retenu)

### Grille d'évaluation

| Critère | Pondération |
|---|---|
| Détection robuste de ressources IPC orphelines (résistant au piège de parsing) | 20% |
| Détection de blocage prolongé en synchronisation/IPC | 25% |
| Distinction claire des deux catégories d'alerte dans le rapport | 15% |
| Robustesse (dégradation propre si `strace` indisponible) | 15% |
| Documentation et rapport | 15% |
| Tests démontrés (2 scénarios) | 10% |

### Modalité
Travail individuel ou en binôme, durée indicative : 1 semaine.

**Clôture de la série d'outils UE1 :** avec ce cinquième outil, l'ensemble (CPU/processus, mémoire, disque, démarrage, synchronisation/IPC) couvre les grandes catégories d'incidents systèmes vues sur les 6 modules de l'unité. Un projet de synthèse consistant à assembler ces cinq outils en une suite de supervision unifiée, avec un tableau de bord commun, pourra être proposé comme projet de clôture d'UE1 une fois tous les modules validés.

### Modalité d'évaluation module 5 — pondération globale suggérée
Conformément à la stratégie d'évaluation générale de la formation (§17 du programme), pour ce module : 15% contrôle continu / 20% TD / 35% TP / 30% mini-projet.

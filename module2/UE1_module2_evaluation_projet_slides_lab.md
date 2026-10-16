# UE1 — Module 2 : Processus et Threads
## Compléments : ÉVALUATION / PROJET / SLIDES / LAB

---

# PARTIE I — ÉVALUATION

**Durée :** 1h30
**Modalités :** documents non autorisés sauf `man` local si épreuve sur machine
**Barème total :** /20

## Section A — QCM (4 points, 0,5 pt/question)

1. L'état `D` d'un processus sous Linux signifie :
 a) attente d'un signal b) attente E/S non interruptible c) prêt à être élu d) zombie

2. Le changement de contexte a un coût car :
 a) il libère de la mémoire b) il ne fait aucun travail utile pendant sa durée c) il termine un processus d) il crée un thread

3. Un thread, contrairement à un processus, partage avec les autres threads du même processus :
 a) uniquement le PID b) l'espace mémoire et les fichiers ouverts c) rien, ils sont isolés d) uniquement les signaux

4. Le Round-Robin avec quantum très long tend à se comporter comme :
 a) FIFO b) priorités c) MLFQ d) aucun algorithme connu

5. Le vieillissement (aging) en ordonnancement par priorités sert à éviter :
 a) l'interblocage b) la famine c) le context switch d) le zombie

6. Laquelle des 4 conditions de Coffman est cassée en imposant un ordre total d'acquisition des ressources ?
 a) exclusion mutuelle b) détention et attente c) non-préemption d) attente circulaire

7. `kill -9` envoie :
 a) SIGTERM, interceptable b) SIGKILL, non interceptable c) SIGSTOP d) SIGCONT

8. Un processus zombie disparaît lorsque :
 a) on le tue avec `kill -9` b) son parent appelle `wait()` (ou à défaut, `init` l'adopte et le nettoie) c) on relance le système immédiatement d) le CPU se libère

## Section B — Questions ouvertes (6 points)

**Q1 (2 pts).** Expliquez la différence entre un processus et un thread en trois critères précis (mémoire, coût de création, isolation en cas de panne). Donnez un exemple applicatif où le choix "plusieurs processus" est préférable à "plusieurs threads", et justifiez.

**Q2 (2 pts).** Deux threads incrémentent une variable partagée sans synchronisation. Décrivez précisément (au niveau instruction) comment une incrémentation peut être "perdue", et proposez une solution.

**Q3 (2 pts).** Citez les 4 conditions de Coffman nécessaires à un interblocage et donnez, pour chacune, un exemple concret de mesure permettant de la casser.

## Section C — Exercice technique (4 points)

Un script Bash doit lancer un traitement en arrière-plan, en attendre la fin, puis vérifier le code de retour et agir en conséquence (log de succès ou d'échec). Écrivez ce script en utilisant `&`, `$!` et `wait`, avec gestion explicite du code de sortie.

## Section D — Problème de diagnostic (6 points)

Un serveur applicatif présente les symptômes suivants :
- `top` : CPU à 15% d'utilisation
- Load average : 22 sur une machine à 4 cœurs
- `ps aux` : nombreux processus dans l'état `D`
- Les utilisateurs rapportent des délais de réponse de plusieurs secondes sur des opérations de lecture de fichiers

**a) (2 pts)** Le problème est-il lié à l'ordonnancement CPU ? Justifiez en vous appuyant sur les chiffres fournis.

**b) (2 pts)** Formulez deux hypothèses plausibles sur la cause racine, et pour chacune, indiquez une commande permettant de la confirmer ou l'infirmer.

**c) (2 pts)** Proposez une méthodologie de diagnostic en 4 étapes ordonnées, cohérente avec la méthodologie de troubleshooting vue en cours.

## Grille de correction résumée

| Section | Points | Critères |
|---|---|---|
| A — QCM | /4 | Réponse exacte uniquement |
| B — Questions ouvertes | /6 | Précision technique + justification argumentée |
| C — Exercice technique | /4 | Script fonctionnel, gestion correcte de `wait`/code retour |
| D — Diagnostic | /6 | Raisonnement méthodique, hypothèses testables, cohérence avec les données fournies |

---

# PARTIE II — MINI-PROJET (échelle module)

## Cahier des charges

**Titre :** Système de surveillance des anomalies de processus (Process Watchdog)

**Contexte :** Une petite équipe DevOps vous confie la création d'un outil léger de surveillance des processus sur un serveur Linux, en attendant la mise en place d'une solution de monitoring complète (Prometheus/Grafana, hors périmètre de ce projet).

### Objectifs du livrable
Développer un outil (Bash ou Python, au choix justifié) qui :
1. Détecte les processus zombies (`Z`)
2. Détecte les processus en attente I/O prolongée (`D` depuis plus de N secondes, N configurable)
3. Détecte les processus consommant plus de X% CPU pendant plus de Y secondes consécutives (X, Y configurables)
4. Journalise chaque anomalie avec horodatage, PID, commande, durée de l'anomalie
5. Génère un rapport de synthèse quotidien (nombre d'anomalies par type)
6. Peut être planifié via `cron`

### Contraintes techniques
- Le script doit gérer proprement les erreurs (`set -euo pipefail` en Bash, ou gestion d'exceptions en Python)
- Aucune dépendance externe non standard sans justification
- Documentation d'installation et d'utilisation (README)
- Tests manuels démontrés sur au moins un cas réel provoqué (zombie, forte charge CPU)

### Livrables attendus
- Code source commenté
- README (installation, configuration, utilisation, format du rapport)
- Journal de tests montrant au moins 3 scénarios validés
- Court rapport (1 page) expliquant les choix techniques et les limites connues de l'outil

### Grille d'évaluation

| Critère | Pondération |
|---|---|
| Fonctionnalités (détection zombie, D prolongé, CPU élevé) | 30% |
| Qualité et robustesse du code (gestion d'erreurs) | 20% |
| Journalisation et rapport de synthèse | 15% |
| Documentation (README) | 15% |
| Tests démontrés | 10% |
| Justification des choix techniques et limites identifiées | 10% |

### Modalité
Travail individuel ou en binôme, durée indicative : 1 semaine, à rendre avant le TP du module suivant (Mémoire virtuelle).

---

# PARTIE III — SLIDES (plan de présentation)

**Format :** ~20 slides pour une séance de 4h de CM (avec pauses actives)

1. **Titre** — Processus et Threads : cycle de vie, ordonnancement, concurrence
2. **Pourquoi ce module ?** — cas d'usage réels (serveur web, base de données, embarqué)
3. **Programme vs Processus** — distinction fondamentale, schéma
4. **Le cycle de vie d'un processus** — schéma des 5 états (New/Ready/Running/Waiting/Terminated)
5. *Slide interactive :* "Un processus débloqué passe-t-il directement à Running ?" (vote/discussion)
6. **Le PCB** — quels champs, pourquoi
7. **Le changement de contexte** — étapes, coût réel
8. **Processus vs Threads** — tableau comparatif
9. **Pourquoi choisir des threads ? Pourquoi des processus ?** — exemples (navigateur, serveur web)
10. **User-level vs Kernel-level threads** — schéma
11. **Ordonnancement — objectifs contradictoires** — débit vs latence vs équité
12. **FIFO** — schéma, limite (effet convoi)
13. **Round-Robin** — schéma, impact du quantum
14. **Priorités et vieillissement (aging)** — schéma, risque de famine
15. **MLFQ et CFS (Linux)** — vue d'ensemble
16. **Synchronisation — la race condition** — exemple pas à pas sur `compteur++`
17. **Interblocage — les 4 conditions de Coffman** — schéma du cycle d'attente
18. **Cas Linux : fork/exec/wait, task_struct** — schéma unifié processus/thread
19. **Étude de cas industrielle** — le serveur avec load average élevé et CPU faible (`D` states)
20. **Synthèse + questions ouvertes** — transition vers le TD/TP

*Note pédagogique :* prévoir un temps de pause active après la slide 8 (Processus vs Threads) en demandant aux étudiants de proposer, en binôme, un exemple d'application où ils choisiraient l'un plutôt que l'autre — reprise collective avant de continuer.

---

# PARTIE IV — LAB (environnement de laboratoire)

## Objectif du LAB
Fournir un environnement reproductible permettant d'observer et de manipuler processus/threads sans risque pour une machine de production, utilisable pour le TP et le mini-projet.

## Infrastructure

**Option recommandée : conteneur unique** (suffisant pour ce module — pas de besoin réseau multi-machines)

```
┌────────────────────────────────────┐
│   Hôte (poste étudiant / VM école)  │
│  ┌────────────────────────────────┐│
│  │  Conteneur Docker "lab-proc"    ││
│  │  Image: ubuntu:24.04             ││
│  │  Outils: ps, top, htop, strace,  ││
│  │  pstree, bash, python3           ││
│  └────────────────────────────────┘│
└────────────────────────────────────┘
```

### Dockerfile fourni aux étudiants

```dockerfile
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y \
    procps htop strace psmisc python3 python3-pip cron \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /lab
COPY scripts/ /lab/scripts/
CMD ["/bin/bash"]
```

### Lancement

```bash
docker build -t lab-proc .
docker run -it --rm --name lab-proc --cap-add=SYS_PTRACE lab-proc
```

*Note :* `--cap-add=SYS_PTRACE` est nécessaire pour que `strace` fonctionne correctement en conteneur (les capacités par défaut sont restreintes).

## Configuration
- Aucun réseau spécifique requis
- Prévoir un volume monté (`-v ./scripts:/lab/scripts`) pour que les étudiants éditent leurs scripts depuis l'hôte avec leur éditeur habituel
- Limiter les ressources du conteneur pour rendre l'exercice de charge CPU (étape 3 du TP) réaliste et sans danger pour l'hôte :
```bash
docker run -it --rm --cpus="1" --memory="256m" --cap-add=SYS_PTRACE lab-proc
```

## Scénarios préconfigurés pour le LAB

| Scénario | Script fourni | Objectif pédagogique |
|---|---|---|
| Zombie | `scripts/zombie_maker.sh` | Observer l'état `Z` |
| Forte charge CPU | `scripts/cpu_burner.sh` (boucle limitée par `timeout`) | Observer l'ordonnancement sous charge |
| Race condition | `scripts/race_condition.py` (deux threads sans lock) | Observer une incrémentation partagée incohérente |
| Interblocage | `scripts/deadlock_demo.py` (deux threads, deux locks, ordre inversé) | Observer et provoquer un interblocage volontairement, puis le corriger |

## Objectifs pédagogiques du LAB
- Manipuler les commandes d'observation en sécurité (sans risque pour une machine partagée)
- Provoquer volontairement les pathologies vues en cours pour mieux les reconnaître en contexte réel
- Préparer le terrain du mini-projet (Process Watchdog) sur un environnement identique pour tous les étudiants, garantissant l'équité de correction

## Points de vigilance pour l'enseignant
- Vérifier que `--cap-add=SYS_PTRACE` est bien accepté par l'infrastructure de l'établissement (certains environnements Docker managés le restreignent)
- Prévoir une limite de temps stricte sur les scripts de charge CPU (`timeout`) pour éviter la saturation des postes en salle TP
- Fournir une image Docker pré-construite (registry interne) si le temps de build en séance est un problème logistique

# UE1 — Module 2 : Processus et Threads

---

# PARTIE I — COURS

## 1. Titre du cours
Processus et Threads : cycle de vie, ordonnancement et concurrence

## 2. Niveau
Élève ingénieur — cycle ingénieur, tronc commun systèmes (S1)

## 3. Durée
CM : 4h — (rappel : le module complet UE1-M2 comprend en plus 3h de TD et 4h de TP détaillées en Parties II et III)

## 4. Prérequis
- Notions de programmation impérative (variables, boucles, fonctions)
- Module 1 de l'UE1 : architecture générale d'un OS, kernel vs user space

## 5. Objectifs pédagogiques
À la fin du cours, l'étudiant doit être capable de :
- Expliquer le cycle de vie d'un processus et les transitions entre états
- Décrire le rôle du PCB (Process Control Block) et du changement de contexte
- Comparer processus et threads, et justifier le choix de l'un ou l'autre selon un cas d'usage
- Expliquer au moins trois algorithmes d'ordonnancement et leurs compromis
- Identifier les conditions d'un interblocage (deadlock) et proposer une stratégie de prévention
- Utiliser les outils Linux (`ps`, `top`, `htop`, `/proc`) pour observer l'état réel des processus d'un système

## 6. Motivation

Un serveur web moderne traite des milliers de requêtes simultanées. Une base de données doit répondre à des centaines de clients en parallèle tout en garantissant la cohérence des données. Un système embarqué doit réagir à des événements en quelques millisecondes sans que ses tâches ne se bloquent mutuellement.

Dans tous ces cas, l'ingénieur système doit répondre à des questions concrètes : *combien de processus mon serveur peut-il supporter avant saturation ? Pourquoi mon application freeze-t-elle parfois sans erreur visible ? Pourquoi deux threads qui accèdent à la même ressource produisent-ils parfois un résultat incohérent ?*

Comprendre les processus et les threads n'est pas un exercice académique : c'est ce qui permet de diagnostiquer un serveur qui rame, de dimensionner une infrastructure, et d'éviter les bugs de concurrence les plus coûteux à corriger en production.

## 7. Concepts fondamentaux

### 7.1 Qu'est-ce qu'un processus ?

Un **processus** est une instance d'un programme en cours d'exécution. Il ne faut pas confondre :
- le **programme** : fichier statique sur disque (code binaire)
- le **processus** : entité dynamique, avec son propre espace mémoire, son état, ses ressources allouées

Chaque processus possède :
- un espace d'adressage virtuel propre (isolé des autres processus)
- un identifiant unique (PID)
- un état d'exécution
- des ressources (fichiers ouverts, sockets, mémoire)

**Pourquoi l'isolation existe-t-elle ?** Sans isolation mémoire entre processus, un bug dans une application pourrait corrompre la mémoire d'une autre. L'isolation est un mécanisme de **sûreté** autant que de **sécurité**.

### 7.2 Le cycle de vie d'un processus

Un processus transite entre plusieurs états au cours de son exécution :

```
        création
           │
           ▼
        ┌─────┐   élu par le scheduler   ┌────────┐
        │ Prêt │ ────────────────────────▶│ Élu    │
        │(Ready)│◀────────────────────────│(Running)│
        └─────┘   préemption / quantum    └────────┘
           ▲                                   │
           │ événement disponible              │ attente E/S,
           │                                   │ ressource
        ┌──────────┐                           │
        │ Bloqué   │◀──────────────────────────┘
        │ (Waiting)│
        └──────────┘
```

- **Nouveau (New)** : le processus est en cours de création
- **Prêt (Ready)** : le processus a toutes les ressources nécessaires sauf le CPU, il attend d'être élu
- **Élu (Running)** : le processus s'exécute sur le CPU
- **Bloqué (Waiting/Blocked)** : le processus attend un événement (I/O, verrou, signal)
- **Terminé (Terminated)** : le processus a fini son exécution ou a été tué

**Point clé pédagogique :** un processus ne passe *jamais* directement de Bloqué à Élu — il repasse obligatoirement par Prêt. C'est le scheduler qui décide *quand* un processus prêt devient élu, pas l'événement qui le débloque.

### 7.3 Le PCB (Process Control Block)

Le PCB est la structure de données maintenue par le noyau pour chaque processus. Il contient notamment :
- PID, PID du parent (PPID)
- État du processus
- Contexte CPU (valeurs des registres, compteur de programme)
- Informations de gestion mémoire (table des pages)
- Liste des fichiers ouverts
- Priorité d'ordonnancement
- Informations de comptabilité (temps CPU consommé)

Sous Linux, le PCB correspond à la structure noyau `task_struct`.

### 7.4 Le changement de contexte (context switch)

Lorsque le scheduler décide de retirer un processus du CPU pour en élire un autre, il doit :
1. Sauvegarder l'état du processus courant (registres, PC, pointeur de pile) dans son PCB
2. Charger l'état du nouveau processus élu depuis son PCB
3. Basculer la table de pages mémoire si nécessaire (TLB flush potentiel)

**Ce n'est pas gratuit.** Un changement de contexte coûte du temps CPU pur (aucun travail utile n'est fait pendant l'opération). C'est pourquoi un système avec trop de processus en compétition pour le CPU voit sa performance globale chuter — pas seulement à cause de la charge, mais à cause du *coût du changement de contexte lui-même*.

### 7.5 Processus vs Threads

Un **thread** (fil d'exécution) est une unité d'exécution *au sein* d'un processus. Plusieurs threads d'un même processus partagent :
- l'espace mémoire (segment de données, tas)
- les fichiers ouverts
- les variables globales

Mais chaque thread a son propre :
- pointeur de pile (stack)
- jeu de registres
- compteur de programme

| Aspect | Processus | Thread |
|---|---|---|
| Espace mémoire | Isolé | Partagé (au sein du même processus) |
| Coût de création | Élevé | Faible |
| Coût du changement de contexte | Élevé | Plus faible |
| Communication | IPC nécessaire (pipes, sockets, mémoire partagée) | Directe (mémoire partagée) |
| Panne d'une unité | N'affecte pas les autres processus | Un crash peut affecter tout le processus |

**Pourquoi choisir l'un ou l'autre ?** Les threads sont adaptés quand on veut de la concurrence *avec partage de données* et un coût de création faible (ex : serveur web multi-threadé). Les processus sont préférés quand on veut de l'**isolation** — un crash ou une faille de sécurité dans un composant ne doit pas affecter les autres (ex : navigateur web moderne qui isole chaque onglet dans un processus).

Il existe aussi des threads **user-level** (gérés par une bibliothèque en espace utilisateur, invisibles du noyau) et **kernel-level** (gérés directement par le noyau, ce qui permet un vrai parallélisme sur plusieurs cœurs). La plupart des OS modernes, dont Linux, utilisent un modèle 1:1 (chaque thread utilisateur correspond à un thread noyau).

### 7.6 Ordonnancement (Scheduling)

Le **scheduler** décide quel processus/thread prêt obtient le CPU, et pour combien de temps. Objectifs souvent contradictoires :
- Maximiser le débit (throughput)
- Minimiser le temps de réponse (latence)
- Garantir l'équité (fairness)
- Respecter des contraintes temps réel si applicable

**Algorithmes classiques :**

- **FIFO (First In First Out)** : le premier arrivé est exécuté jusqu'à sa fin. Simple, mais un processus long bloque tous les autres (effet convoi).
- **Round-Robin** : chaque processus reçoit un quantum de temps fixe, puis passe en fin de file. Bon compromis pour l'équité, mais le choix du quantum est critique : trop court → trop de changements de contexte ; trop long → dégénère vers du FIFO.
- **Priorités** : chaque processus a une priorité, le scheduler élit le plus prioritaire. Risque de famine (starvation) pour les processus de faible priorité — corrigé par le *vieillissement* (aging), qui augmente progressivement la priorité d'un processus qui attend trop longtemps.
- **Multilevel Feedback Queue** : combine plusieurs files avec des quantums différents, un processus peut changer de file selon son comportement (interactif vs calcul intensif). C'est proche de ce qu'utilisent les OS généralistes modernes (Linux utilise CFS — Completely Fair Scheduler — basé sur un principe différent, d'équité par temps virtuel).

### 7.7 Synchronisation et interblocage (Deadlock)

Quand plusieurs threads/processus partagent des ressources, il faut les synchroniser (verrous/mutex, sémaphores) pour éviter les **conditions de course** (race conditions) — un résultat qui dépend de l'ordre d'exécution imprévisible des threads.

Un **interblocage** survient quand un ensemble de processus attendent mutuellement des ressources détenues par les autres, sans qu'aucun ne puisse progresser. Les 4 conditions nécessaires (conditions de Coffman) sont :
1. **Exclusion mutuelle** — une ressource ne peut être utilisée que par un seul processus à la fois
2. **Détention et attente** — un processus détient une ressource en attendant d'en obtenir une autre
3. **Non-préemption** — une ressource ne peut être retirée de force à un processus
4. **Attente circulaire** — il existe un cycle de processus s'attendant mutuellement

Casser *une seule* de ces conditions suffit à empêcher l'interblocage (ex : imposer un ordre total d'acquisition des ressources casse l'attente circulaire).

## 8. Architecture

```
┌──────────────────────────────────────────────────────┐
│                     User Space                        │
│   Processus A          Processus B                    │
│  ┌──────────────┐     ┌──────────────┐                │
│  │ Thread 1     │     │ Thread 1     │                │
│  │ Thread 2     │     │              │                │
│  │ Mémoire      │     │ Mémoire      │                │
│  │ partagée A   │     │ partagée B   │                │
│  └──────┬───────┘     └──────┬───────┘                │
└─────────┼─────────────────────┼───────────────────────┘
          │  system calls       │
┌─────────▼─────────────────────▼───────────────────────┐
│                    Kernel Space                        │
│   ┌────────────────────────────────────────────────┐  │
│   │  Scheduler (ex: CFS sous Linux)                 │  │
│   └────────────────────────────────────────────────┘  │
│   PCB (task_struct) A.T1 │ A.T2 │ B.T1                 │
│   Gestion mémoire (tables de pages)                    │
└──────────────────────────────────────────────────────┘
```

## 9. Fonctionnement interne (cas Linux)

Sous Linux, il n'existe pas de distinction stricte au niveau noyau entre processus et thread : les deux sont représentés par une `task_struct`. Ce qui les différencie, c'est ce qu'ils **partagent** :
- `fork()` crée un nouveau processus avec un espace mémoire copié (copy-on-write)
- `clone()` (utilisé en interne pour créer des threads via des bibliothèques comme pthreads) crée une nouvelle `task_struct` mais avec des flags qui font *partager* l'espace mémoire, les fichiers, etc.

C'est pourquoi Linux parle parfois de "tâches" (tasks) de façon unifiée. Le scheduler CFS ordonnance des `task_struct`, sans distinguer fondamentalement processus et thread au niveau de la décision d'ordonnancement.

Cycle typique : `fork()` → `exec()` (remplace l'image mémoire par un nouveau programme) → exécution → `exit()` → le parent récupère le code de retour via `wait()`. Un processus terminé mais non "récupéré" par son parent devient un **zombie** (visible dans `ps` avec l'état `Z`).

## 10. Démonstration

```bash
# Observer l'arborescence de processus
pstree -p

# Voir l'état détaillé d'un processus
cat /proc/<PID>/status | grep State

# Provoquer et observer un processus zombie
sleep 100 &
PID=$!
kill -STOP $PID   # simule un blocage
ps -o pid,ppid,stat,cmd -p $PID
kill -CONT $PID
kill $PID
```

## 11. Commandes / outils

| Commande | Rôle | Exemple |
|---|---|---|
| `ps aux` | Lister tous les processus avec détails | `ps aux \| grep nginx` |
| `ps -eLf` | Lister processus **et threads** | `ps -eLf \| grep firefox` |
| `top` / `htop` | Vue dynamique CPU/mémoire par processus | `htop` |
| `pstree -p` | Arborescence parent/enfant | `pstree -p 1` |
| `kill -SIGNAL PID` | Envoyer un signal | `kill -9 1234` |
| `nice` / `renice` | Définir/modifier la priorité | `renice -n 5 -p 1234` |
| `strace -p PID` | Tracer les appels système d'un processus | `strace -p 1234` |
| `/proc/PID/status` | État détaillé (dont S/R/D/Z) | `cat /proc/1234/status` |

**Erreurs fréquentes :** confondre `kill -9` (SIGKILL, non interceptable, peut laisser des ressources non libérées) avec `kill -15` (SIGTERM, terminaison propre, interceptable par le processus) — toujours préférer SIGTERM en premier recours.

## 12–13. TD et TP
Voir Parties II et III de ce document.

## 14. Étude de cas industrielle

**Contexte :** un serveur d'API REST en production répond normalement, mais depuis ce matin, le temps de réponse moyen est passé de 50ms à 2s. `top` montre un load average élevé mais un CPU non saturé. `ps aux` révèle des centaines de processus en état `D` (uninterruptible sleep).

**Analyse attendue de l'étudiant :** l'état `D` signifie que les processus attendent une opération I/O (souvent disque). Un CPU non saturé mais un load average élevé avec beaucoup de `D` pointe vers un goulot d'étranglement disque/stockage, pas CPU. Pistes : disque saturé (I/O wait élevé via `vmstat` ou `iostat`), NFS distant qui répond mal, ou swap excessif (mémoire insuffisante forçant des accès disque).

## 15. Sécurité

- Un processus mal isolé (privilèges excessifs) peut compromettre tout le système en cas de faille — appliquer le principe de moindre privilège (ne jamais exécuter un service en root sans nécessité)
- Les threads partageant la mémoire d'un même processus : une faille de sécurité (ex: buffer overflow) dans un thread peut compromettre tout le processus, y compris les données des autres threads
- `strace`/`ptrace` sont puissants pour le diagnostic mais aussi pour l'attaque (un processus peut être tracé/inspecté par un utilisateur avec les bons droits) — d'où l'intérêt de restreindre `ptrace` en production (voir `/proc/sys/kernel/yama/ptrace_scope`)

## 16. Automatisation

Exemple : script Bash de surveillance de processus zombies avec alerte.

```bash
#!/usr/bin/env bash
# Détecte les processus zombies et journalise une alerte
set -euo pipefail

ZOMBIES=$(ps -eo pid,stat,cmd | awk '$2 ~ /^Z/ {print}')

if [[ -n "$ZOMBIES" ]]; then
    echo "$(date '+%F %T') - ALERTE: processus zombies détectés" | tee -a /var/log/zombie_watch.log
    echo "$ZOMBIES" | tee -a /var/log/zombie_watch.log
    exit 1
fi

echo "$(date '+%F %T') - OK: aucun zombie" >> /var/log/zombie_watch.log
exit 0
```

## 17. Troubleshooting (panne volontaire — voir TP pour la version guidée complète)

Scénario court : un processus consomme 100% CPU en continu sans raison apparente. Démarche attendue : `top` pour identifier le PID → `strace -p PID` pour voir s'il boucle sur un appel système précis → vérifier les logs applicatifs → `renice` en urgence pour limiter l'impact pendant l'investigation → correction du code ou redémarrage du service.

## 18. Synthèse

- Un processus = espace isolé ; un thread = unité d'exécution partageant l'espace de son processus
- Le PCB/task_struct est la mémoire du noyau sur chaque tâche ; le changement de contexte a un coût réel
- Le scheduler arbitre entre débit, latence et équité — aucun algorithme n'est optimal sur tous les critères
- L'interblocage nécessite 4 conditions simultanées ; en casser une suffit à le prévenir
- Sous Linux, processus et threads sont unifiés au niveau noyau (`task_struct`), différenciés par ce qu'ils partagent

## 19. Questions d'évaluation

**QCM**
1. Un processus en état `D` sous Linux attend :
 a) le CPU b) une opération I/O non interruptible c) un signal d) la fin d'un thread enfant

2. Le Round-Robin avec un quantum trop court entraîne principalement :
 a) la famine b) un excès de changements de contexte c) un interblocage d) une fuite mémoire

**Questions ouvertes**
- Expliquez pourquoi deux threads d'un même processus peuvent produire un résultat incohérent sur une variable partagée, même sans erreur de programmation apparente.
- Un processus enfant se termine mais son parent ne l'a jamais "attendu" (`wait()`). Que se passe-t-il et pourquoi est-ce potentiellement problématique à grande échelle ?

**Exercice technique**
- Donnez la séquence exacte d'appels système (`fork`, `exec`, `wait`, `exit`) pour un shell qui lance une commande externe et attend sa fin.

**Problème de diagnostic**
- Un serveur affiche un load average de 15 sur une machine à 4 cœurs, mais le CPU est utilisé à 20%. Formulez au moins deux hypothèses et la commande permettant de les tester.

## 20. Mini-projet

Développer un script (Bash ou Python) qui :
1. Liste tous les processus dans un état `D` ou `Z` depuis plus de 30 secondes
2. Journalise chaque détection avec horodatage
3. Envoie une alerte (email simulé ou simple `echo` structuré JSON) si plus de 3 processus zombies sont détectés simultanément
4. Fournit un rapport quotidien résumant le nombre d'alertes déclenchées

---

# PARTIE II — TD (Travaux Dirigés)

**Consigne générale :** cherchez à répondre par vous-même avant de consulter la correction en fin de document. Justifiez toujours vos réponses.

### Niveau 1 — Compréhension

**Exercice 1.1**
Un processus passe de l'état "Élu" à l'état "Bloqué". Quel type d'événement peut causer cette transition ? Donnez deux exemples concrets.

**Exercice 1.2**
Expliquez avec vos propres mots la différence entre un programme et un processus, en donnant un exemple où un même programme génère plusieurs processus simultanément.

**Exercice 1.3**
Que contient un PCB ? Pourquoi le noyau a-t-il besoin de sauvegarder les registres CPU dans le PCB lors d'un changement de contexte ?

### Niveau 2 — Application

**Exercice 2.1**
Vous exécutez `ps -eo pid,ppid,stat,cmd` et observez une ligne avec `STAT = Z`. Que représente cet état, et quelle commande permettrait normalement de le faire disparaître (indice : ce n'est pas `kill`) ?

**Exercice 2.2**
Écrivez la commande permettant d'observer, pour un PID donné, la liste des threads associés à ce processus.

**Exercice 2.3**
Un administrateur souhaite baisser la priorité d'un processus déjà en cours d'exécution (PID 4521) sans le redémarrer. Donnez la commande exacte.

### Niveau 3 — Analyse

**Exercice 3.1**
Deux threads T1 et T2 d'un même processus incrémentent tous deux une variable globale `compteur` (initialement à 0), chacun 100 000 fois, sans aucune synchronisation. Expliquez pourquoi le résultat final n'est presque jamais exactement 200 000. Illustrez avec un scénario d'entrelacement d'instructions.

**Exercice 3.2**
Un système présente un load average de 8 sur une machine à 2 cœurs uniquement en heures de pointe, avec un CPU utilisé à 95%. S'agit-il d'un problème d'ordonnancement, de dimensionnement, ou insuffisant pour conclure ? Justifiez.

### Niveau 4 — Problème complexe

**Exercice 4.1**
Concevez un scénario minimal (avec 2 processus P1 et P2 et 2 ressources R1 et R2) illustrant les 4 conditions de Coffman menant à un interblocage. Puis proposez une modification de ce scénario qui élimine le risque d'interblocage en cassant une seule des 4 conditions, en précisant laquelle.

---

## Correction du TD

**1.1** — Événements typiques : attente d'une E/S disque (lecture de fichier), attente d'une réponse réseau (socket bloquante), attente d'un verrou détenu par un autre processus. Le processus est retiré du CPU car il ne peut pas progresser tant que l'événement n'est pas survenu — le laisser sur le CPU serait un gaspillage de ressource.

**1.2** — Un programme est un fichier statique (ex: `/usr/bin/firefox`) ; un processus est une instance en mémoire de ce programme avec son propre état d'exécution. Exemple : lancer `firefox` plusieurs fois (ou plusieurs onglets isolés en processus séparés) crée plusieurs processus à partir du même programme, chacun avec son propre PID et espace mémoire.

**1.3** — Le PCB contient : PID/PPID, état, contexte CPU (registres, compteur de programme), informations mémoire, fichiers ouverts, priorité. Les registres doivent être sauvegardés car ils représentent l'état exact d'exécution du processus interrompu — sans cette sauvegarde, il serait impossible de reprendre l'exécution exactement là où elle a été interrompue lors de la réélection du processus.

**2.1** — `Z` = processus zombie : le processus a terminé son exécution mais son parent n'a pas encore appelé `wait()` pour récupérer son code de sortie. La commande qui le fait disparaître n'est pas un `kill` (un zombie n'a déjà plus de code en exécution) mais consiste à faire en sorte que le **processus parent** appelle `wait()` — en pratique, si le parent est bogué, il faut le corriger ou le redémarrer ; s'il meurt, `init`/PID 1 adopte le zombie et le nettoie.

**2.2** — `ps -eLf | grep <PID>` ou plus précisément `ps -T -p <PID>` qui liste les threads (LWP) du processus donné.

**2.3** — `renice -n <valeur> -p 4521` (par exemple `renice -n 10 -p 4521` pour baisser la priorité, une valeur "nice" plus élevée = priorité plus basse).

**3.1** — L'incrémentation `compteur++` n'est pas atomique : elle se décompose en (1) lire la valeur en registre, (2) l'incrémenter, (3) réécrire en mémoire. Si T1 lit la valeur (ex: 500), puis T2 lit la même valeur (500) avant que T1 n'ait réécrit sa version incrémentée, les deux threads vont écrire 501 au lieu d'obtenir 502 au total — une incrémentation est "perdue". Répété des milliers de fois, cela explique un résultat final inférieur à 200 000, de façon non déterministe (dépend du timing réel d'exécution).

**3.2** — Un CPU à 95% avec un load average de 8 sur 2 cœurs indique une **vraie saturation CPU** (contrairement au cas avec beaucoup de processus en `D`, ici le CPU est réellement occupé). C'est un problème de dimensionnement (sous-capacité CPU par rapport à la charge), pas d'ordonnancement — l'ordonnanceur fait déjà son travail correctement en répartissant équitablement une ressource insuffisante. La solution est d'augmenter les capacités (scaling vertical/horizontal) ou de réduire la charge, pas de changer d'algorithme d'ordonnancement.

**4.1** — Scénario : P1 détient R1 et demande R2 ; P2 détient R2 et demande R1. Les 4 conditions : (1) exclusion mutuelle — R1 et R2 ne sont utilisables que par un seul processus à la fois ; (2) détention et attente — chaque processus garde sa ressource en attendant l'autre ; (3) non-préemption — ni R1 ni R2 ne peuvent être retirés de force ; (4) attente circulaire — P1→R2→P2→R1→P1.
Correction possible : imposer un **ordre total d'acquisition** des ressources (ex : toujours acquérir R1 avant R2, pour tous les processus). Cela casse la condition (4) attente circulaire, car aucun processus ne pourra détenir R2 en attendant R1 dans ce nouvel ordre imposé.

---

# PARTIE III — TP (Travaux Pratiques)

## Contexte

Vous intervenez comme ingénieur système junior sur un serveur applicatif Linux. L'équipe rapporte des ralentissements intermittents et la présence suspecte de processus "fantômes" dans les rapports de monitoring. Votre mission : observer, diagnostiquer et automatiser la détection de ces situations.

## Architecture

Une seule machine Linux (VM ou conteneur avec accès aux commandes standard : `ps`, `top`, `kill`, `bash`). Aucune infrastructure réseau nécessaire pour ce TP.

## Objectifs
- Observer concrètement les états de processus vus en cours
- Provoquer, identifier et nettoyer un processus zombie
- Provoquer et diagnostiquer un cas de forte charge CPU
- Écrire un script d'automatisation de détection

## Préparation

```bash
# Vérifier les outils disponibles
which ps top htop pstree 2>/dev/null
```

## Étapes

### Étape 1 — Observer le cycle de vie normal

```bash
sleep 60 &
PID=$!
ps -o pid,ppid,stat,cmd -p $PID
```
*Résultat attendu :* état `S` (interruptible sleep).

### Étape 2 — Provoquer un processus zombie

```bash
cat << 'EOF' > /tmp/zombie_maker.sh
#!/bin/bash
sleep 2 &
CHILD=$!
sleep 10   # le parent ne fait PAS de wait avant de se terminer
EOF
chmod +x /tmp/zombie_maker.sh
/tmp/zombie_maker.sh &
sleep 3
ps -eo pid,ppid,stat,cmd | grep -i defunct
```
*Résultat attendu :* une ligne avec l'état `Z` (defunct) apparaît après quelques secondes, tant que le script parent est encore en vie (le zombie disparaît quand le parent se termine et que `init` l'adopte et le nettoie).

### Étape 3 — Provoquer une forte charge CPU et diagnostiquer

```bash
# Attention : limité à 10 secondes pour ne pas saturer durablement la machine
timeout 10 bash -c 'while true; do :; done' &
PID=$!
top -b -n 1 | head -15
ps -o pid,pcpu,cmd -p $PID
```
*Question :* quel est le %CPU observé pour ce processus ? Est-ce cohérent avec une boucle infinie sans I/O ?

### Étape 4 — Script d'automatisation

Reprendre et tester le script de détection de zombies fourni en section 16 du cours (`zombie_watch.sh`), en l'exécutant pendant que vous relancez `zombie_maker.sh`.

```bash
bash zombie_watch.sh
```

*Résultat attendu :* le script doit détecter et journaliser le zombie créé à l'étape 2.

## Commandes de référence pour ce TP
`ps`, `top`, `kill`, `sleep`, `timeout`, `pstree`

## Résultats attendus
- Capture (copier-coller) montrant un état `Z` observé
- Capture montrant un processus à ~100% CPU identifié via `top`/`ps`
- Fichier de log `/var/log/zombie_watch.log` (ou chemin local équivalent) contenant au moins une alerte

## Questions

1. Pourquoi le zombie de l'étape 2 finit-il par disparaître même sans intervention, si vous attendez que `zombie_maker.sh` se termine complètement ?
2. Dans l'étape 3, si vous aviez utilisé `sleep 999999 &` au lieu d'une boucle CPU, quel état afficherait `ps` pour ce processus, et pourquoi le %CPU serait-il proche de 0 ?
3. Quelle modification apporteriez-vous au script `zombie_watch.sh` pour qu'il envoie une vraie alerte email plutôt qu'un simple log ?

## Troubleshooting (panne à diagnostiquer)

**Situation fournie :** en exécutant `zombie_watch.sh` en boucle via `cron` toutes les minutes, l'étudiant constate que le fichier `/var/log/zombie_watch.log` grossit de façon anormale même en l'absence de zombies réels.

**Démarche attendue :**
1. Symptôme : croissance du log sans zombie apparent
2. Hypothèse : le filtre `awk` du script capture peut-être un faux positif (ex: le processus `grep`/`awk` lui-même apparaissant transitoirement dans `ps`)
3. Vérification : exécuter la commande `ps -eo pid,stat,cmd | awk '$2 ~ /^Z/'` manuellement et inspecter la sortie ligne par ligne
4. Isolation : vérifier si le script `ps` capture sa propre exécution ou celle d'un sous-processus temporaire
5. Correction : affiner le filtre `awk` ou exclure le PID du script lui-même
6. Vérification : relancer sur plusieurs cycles cron et confirmer l'absence de faux positifs
7. Documentation : noter la cause et le correctif dans le changelog du script

## Validation

Le TP est validé si l'étudiant fournit :
- Les 4 captures/résultats demandés
- Des réponses argumentées aux 3 questions
- Une analyse correcte du scénario de troubleshooting (identification de la cause du faux positif)

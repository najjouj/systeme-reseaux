# UE1 — Module 5 : I/O, Interruptions, System Calls, IPC, Synchronisation et Deadlocks

---

# PARTIE I — COURS

## 1. Titre du cours
Communication et synchronisation inter-processus : IPC, primitives de synchronisation et interblocages

## 2. Niveau
Élève ingénieur — cycle ingénieur, tronc commun systèmes (S1)

## 3. Durée
CM : 5h (module dense — +3h TD, +5h TP en Parties II et III)

## 4. Prérequis
- UE1-M2 : processus, threads, race condition, introduction aux 4 conditions de Coffman
- UE1-M4 : appels système, interruptions, chemin d'une E/S (rappels approfondis ici)

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Expliquer précisément le mécanisme d'un appel système (transition user space → kernel space)
- Choisir et justifier un mécanisme d'IPC (pipe, FIFO, mémoire partagée, file de messages, socket) selon un cas d'usage
- Utiliser correctement mutex, sémaphores et variables de condition pour synchroniser des threads/processus
- Distinguer prévention, évitement et détection d'interblocage, et appliquer l'algorithme du banquier sur un cas simple
- Diagnostiquer un interblocage ou une situation de contention avec les outils Linux

## 6. Motivation

Un pipeline de traitement de données fait collaborer plusieurs processus : l'un lit des fichiers, l'autre les transforme, un troisième les écrit en base. Comment ces processus, isolés par construction (rappel M3 : mémoire virtuelle), échangent-ils des données efficacement ? Un serveur multi-thread doit protéger l'accès à une structure partagée : comment éviter à la fois la corruption de données (race condition, vue en M2) et le blocage complet du système (interblocage) ?

Ce module complète directement M2 et M4 : après avoir vu *pourquoi* la synchronisation est nécessaire et *ce qu'est* un interblocage, nous voyons ici *comment* concrètement communiquer et se synchroniser correctement — et comment prévenir, détecter ou résoudre un interblocage en pratique, pas seulement le reconnaître en théorie.

## 7. Concepts fondamentaux

### 7.1 L'appel système en détail

Un **appel système (system call)** est le mécanisme contrôlé par lequel un processus en espace utilisateur demande un service au noyau (accès matériel, gestion mémoire, création de processus, IPC...). Le processus ne peut pas exécuter directement ces opérations privilégiées ; il déclenche une **interruption logicielle** (ou instruction dédiée, ex : `syscall` sur x86_64) qui bascule le CPU en mode noyau (`ring 0`), exécute la fonction noyau correspondante, puis restaure le mode utilisateur (`ring 3`) avant de retourner le résultat.

**Pourquoi cette indirection ?** Sans cette barrière contrôlée, un processus utilisateur pourrait accéder directement au matériel ou à la mémoire d'autres processus — l'appel système est donc autant un mécanisme de service qu'un mécanisme de **sécurité et d'isolation**, cohérent avec tout ce qui a été vu depuis M1.

### 7.2 IPC — Inter-Process Communication

Puisque les processus sont isolés en mémoire (M3), ils ont besoin de mécanismes explicites pour échanger des données : c'est l'**IPC**.

**Pipes (tubes anonymes)**
Canal de communication unidirectionnel entre processus apparentés (typiquement parent/enfant après `fork()`), FIFO par nature (premier écrit, premier lu). Créé par l'appel système `pipe()`, exposé sous forme de deux descripteurs de fichiers (lecture/écriture) — cohérent avec le modèle "tout est fichier" vu en M4.

**FIFO nommées (named pipes)**
Extension du pipe anonyme, mais accessible via un nom dans le système de fichiers (`mkfifo`), permettant la communication entre processus **non apparentés**.

**Mémoire partagée (shared memory)**
Mécanisme le plus performant : plusieurs processus mappent la **même** zone de mémoire physique dans leur espace d'adressage virtuel respectif (rappel M3 : plusieurs pages virtuelles différentes peuvent pointer vers le même cadre physique). Aucune copie de données n'est nécessaire entre processus — mais **aucune synchronisation n'est fournie automatiquement** : c'est à l'application de protéger les accès concurrents (d'où le lien fort avec la section 7.3).

**Files de messages (message queues)**
Permettent l'envoi de messages structurés et typés entre processus, avec un mécanisme de mise en file géré par le noyau (contrairement au flux d'octets brut d'un pipe).

**Sockets**
Mécanisme le plus général : communication entre processus sur la même machine (sockets Unix/`AF_UNIX`) ou sur des machines distinctes (sockets réseau/`AF_INET`, préfigurant l'UE2 — réseaux). C'est le seul mécanisme d'IPC natif qui franchit la frontière d'une seule machine.

| Mécanisme | Processus apparentés requis ? | Copie de données ? | Cas d'usage typique |
|---|---|---|---|
| Pipe anonyme | Oui | Oui (via le noyau) | Chaînage de commandes shell (`cmd1 \| cmd2`) |
| FIFO nommée | Non | Oui | Processus indépendants sur la même machine |
| Mémoire partagée | Non | Non (accès direct) | Échange de gros volumes de données, haute performance |
| File de messages | Non | Oui | Messages structurés, prioritaires |
| Socket | Non | Oui | Communication locale ou réseau, client/serveur |

### 7.3 Synchronisation

Dès que plusieurs threads/processus partagent une ressource (mémoire partagée, fichier, structure de données), il faut coordonner leurs accès pour éviter les races conditions (rappel M2).

**Mutex (mutual exclusion)**
Verrou binaire garantissant qu'une seule unité d'exécution accède à une ressource à un instant donné. Une unité qui tente d'acquérir un mutex déjà détenu est **bloquée** (mise en état d'attente, pas de gaspillage CPU) jusqu'à sa libération.

**Sémaphore**
Généralisation du mutex : un compteur entier associé à deux opérations atomiques, `wait()`/`P()` (décrémente, bloque si le compteur atteint zéro) et `signal()`/`V()` (incrémente, réveille éventuellement une unité en attente). Un sémaphore initialisé à 1 se comporte comme un mutex ; un sémaphore initialisé à N permet de limiter l'accès concurrent à N unités simultanées (ex : un pool de connexions limité).

**Spinlock**
Verrou où l'unité en attente **boucle activement** (busy-waiting) plutôt que de se bloquer, jusqu'à ce que le verrou se libère. Utile uniquement quand la durée d'attente attendue est très courte (souvent en contexte noyau, sur système multi-cœur) — sur un système mono-cœur ou pour une attente longue, un spinlock gaspille du CPU inutilement, contrairement à un mutex classique qui libère le CPU pendant l'attente (lien direct avec le coût du changement de contexte vu en M2 : le compromis n'est pas gratuit dans un sens ni dans l'autre).

**Variables de condition**
Permettent à une unité d'exécution d'attendre qu'une condition précise devienne vraie (ex : "la file n'est plus vide"), en étant explicitement réveillée par une autre unité (`signal`/`broadcast`) plutôt que de vérifier activement la condition en boucle. Toujours utilisées en combinaison avec un mutex, pour éviter une race condition entre la vérification de la condition et la mise en attente elle-même.

**Point clé pédagogique :** un verrou mal utilisé peut lui-même *causer* un interblocage — c'est le lien direct avec la section 7.4.

### 7.4 Interblocage — approfondissement (suite de M2)

M2 a introduit les 4 conditions de Coffman. Ce module aborde les **trois stratégies pratiques** pour y faire face :

**a) Prévention (deadlock prevention)**
Casser structurellement l'une des 4 conditions, de façon systématique et garantie. Exemple déjà vu en M2 : imposer un ordre total d'acquisition des ressources (casse l'attente circulaire).

**b) Évitement (deadlock avoidance) — l'algorithme du banquier**
Contrairement à la prévention (qui restreint *a priori* les possibilités), l'évitement analyse **dynamiquement**, avant chaque allocation de ressource, si accorder cette allocation pourrait mener à un état non sûr (unsafe state) — c'est-à-dire un état d'où aucune séquence d'exécution ne garantit que tous les processus pourront terminer sans interblocage. L'**algorithme du banquier** (Dijkstra) formalise ce calcul : il ne connaît pas seulement les ressources actuellement détenues, mais aussi les besoins **maximaux déclarés à l'avance** par chaque processus, ce qui lui permet de refuser une allocation risquée avant même qu'elle ne cause un problème.

*Principe simplifié :* pour chaque demande de ressource, le système simule l'octroi et vérifie s'il reste au moins une séquence d'exécution complète possible pour tous les processus avec les ressources restantes ; si oui, l'état est "sûr" et l'allocation est accordée ; sinon, elle est différée.

**c) Détection et rétablissement (deadlock detection and recovery)**
Ne pas empêcher l'interblocage à l'avance, mais le **détecter** périodiquement (via un graphe d'allocation de ressources et la recherche de cycles), puis le **résoudre** en sacrifiant un ou plusieurs processus (les terminer de force) ou en préemptant une ressource. Stratégie pertinente quand les interblocages sont rares et que le coût de prévention/évitement systématique serait disproportionné.

**Comparaison des trois approches :** la prévention est la plus sûre mais la plus restrictive (moins de parallélisme possible) ; l'évitement est plus flexible mais nécessite de connaître à l'avance les besoins maximaux (souvent irréaliste en pratique) ; la détection/rétablissement est la plus permissive mais accepte un risque réel et un coût de récupération. **En pratique, la plupart des systèmes d'exploitation généralistes (dont Linux) ne mettent en œuvre aucune prévention/évitement systématique au niveau noyau pour les verrous applicatifs — c'est à l'application (donc au développeur/ingénieur) d'appliquer ces principes**, le noyau se contentant d'offrir les primitives de base (mutex, sémaphores).

## 8. Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Processus A               Processus B                     │
│  ┌──────────┐              ┌──────────┐                    │
│  │ Thread 1  │              │ Thread 1  │                    │
│  └────┬─────┘              └────┬─────┘                    │
└───────┼───────────────────────────┼───────────────────────┘
        │ syscall (write/read/       │ syscall
        │ shmget/semop...)           │
┌───────▼───────────────────────────▼───────────────────────┐
│                        Noyau Linux                           │
│  ┌────────────┐  ┌───────────────┐  ┌────────────────┐     │
│  │  Pipes/FIFO │  │ Mémoire        │  │ Sémaphores/     │     │
│  │  (buffer     │  │ partagée       │  │ files de        │     │
│  │  noyau)      │  │ (mappée dans   │  │ messages         │     │
│  │              │  │ les 2 espaces) │  │ (System V/POSIX)│     │
│  └────────────┘  └───────────────┘  └────────────────┘     │
└───────────────────────────────────────────────────────────┘
```

## 9. Fonctionnement interne (cas Linux)

Linux propose deux familles historiques d'API IPC : **System V IPC** (`shmget`, `semget`, `msgget`, gérés via des identifiants numériques, visibles avec `ipcs`) et **POSIX IPC** (`shm_open`, `sem_open`, interfaces plus modernes basées sur des noms de type fichier, souvent exposées sous `/dev/shm`). Les deux coexistent sur un système Linux moderne ; POSIX est généralement recommandé pour du nouveau code pour sa cohérence avec le modèle "tout est fichier".

Pour les mutex/sémaphores en espace utilisateur (au sein d'un même processus multi-thread, via `pthreads`), Linux utilise en interne le mécanisme **futex** (Fast Userspace Mutex) : dans le cas non contesté (pas d'attente), l'acquisition/libération se fait **entièrement en espace utilisateur, sans appel système**, ce qui est extrêmement rapide. Ce n'est que lorsqu'une contention réelle survient (le verrou est déjà détenu) qu'un appel système est déclenché pour mettre le thread en attente au niveau noyau. C'est une optimisation directement héritée du constat vu en M2 : un appel système/changement de contexte a un coût, qu'il faut éviter quand ce n'est pas nécessaire.

## 10. Démonstration

```bash
# Créer et utiliser une FIFO nommée entre deux terminaux
mkfifo /tmp/mafifo
# Terminal A :
cat /tmp/mafifo
# Terminal B :
echo "message via FIFO" > /tmp/mafifo

# Observer les ressources IPC System V actives sur le système
ipcs -a
```

## 11. Commandes / outils

| Commande | Rôle | Exemple |
|---|---|---|
| `mkfifo` | Créer une FIFO nommée | `mkfifo /tmp/monpipe` |
| `ipcs` | Lister les ressources IPC System V (mémoire partagée, sémaphores, files) | `ipcs -a` |
| `ipcrm` | Supprimer une ressource IPC orpheline | `ipcrm -m <shmid>` |
| `ls /dev/shm` | Lister les objets de mémoire partagée POSIX | `ls -lh /dev/shm` |
| `strace -e trace=ipc,futex` | Tracer les appels système IPC/synchronisation d'un processus | `strace -e trace=futex -p 1234` |
| `lslocks` | Lister les verrous de fichiers actifs sur le système | `lslocks` |

**Erreur fréquente :** oublier de nettoyer une ressource IPC System V après la fin d'un programme (contrairement à la mémoire d'un processus, une ressource `shmget`/`semget` **survit à la terminaison du processus qui l'a créée** si elle n'est pas explicitement libérée) — cause classique de fuite de ressources IPC détectable via `ipcs -a` montrant des segments sans processus attaché.

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** un pipeline de traitement composé de 3 processus (lecteur, transformateur, écrivain), communiquant via une file de messages System V, se fige complètement après quelques heures de fonctionnement en production — aucun processus ne semble crasher, mais plus aucune donnée n'est traitée.

**Analyse attendue :** un blocage total sans crash est la signature classique d'un interblocage. Démarche : `ps -eo pid,stat,cmd` pour confirmer que les 3 processus sont bien en état `S` (sleeping, en attente) et non `R`/`D` ; `ipcs -a` pour vérifier l'état des ressources IPC (file de messages pleine ? sémaphore jamais libéré ?) ; reconstruire le graphe d'attente (qui attend quoi) pour confirmer un cycle. Cause probable typique : le transformateur attend un message que l'écrivain devrait consommer pour libérer de l'espace dans la file, mais l'écrivain attend lui-même un verrou détenu indirectement par le lecteur — un cycle d'attente classique, à corriger par un ordre d'acquisition cohérent ou une limite de capacité mieux dimensionnée (voir aussi la section 7.4 sur la prévention).

## 15. Sécurité

- Les ressources IPC System V (segments de mémoire partagée notamment) ont des **permissions** similaires aux fichiers (lecture/écriture par propriétaire/groupe/autres) — une mauvaise configuration peut exposer des données sensibles à d'autres utilisateurs du système
- Une **file de messages ou un segment de mémoire partagée non nettoyé** peut constituer une fuite d'information résiduelle si son contenu reste accessible après la fin logique du programme qui l'a utilisé
- Les **sockets Unix** (`AF_UNIX`) sont soumis aux permissions du système de fichiers (le socket apparaît comme un fichier spécial) — un mauvais réglage peut permettre à un processus non autorisé de s'y connecter
- Un interblocage provoqué délibérément (par un acteur malveillant exploitant un défaut de synchronisation) peut constituer une attaque de type **déni de service** applicatif, sans qu'aucune vulnérabilité mémoire classique ne soit nécessaire

## 16. Automatisation

Script Bash de détection de ressources IPC System V orphelines (sans processus attaché), pouvant indiquer une fuite de ressources.

```bash
#!/usr/bin/env bash
# Détecte les segments de mémoire partagée System V sans processus attaché (nattch=0)
set -euo pipefail

LOGFILE="/var/log/ipc_leak_watch.log"

ORPHELINS=$(ipcs -m | awk 'NR>3 && $6 == 0 {print}')

if [[ -n "$ORPHELINS" ]]; then
    echo "$(date '+%F %T') - ALERTE: segments de mémoire partagée orphelins détectés" \
        | tee -a "$LOGFILE"
    echo "$ORPHELINS" | tee -a "$LOGFILE"
    exit 1
fi

echo "$(date '+%F %T') - OK: aucun segment orphelin" >> "$LOGFILE"
exit 0
```

## 17. Troubleshooting

Scénario court : une application multi-thread se fige progressivement, de plus en plus de threads restant bloqués au fil du temps, jusqu'au blocage total. Démarche : `ps -T -p PID` pour lister tous les threads et leur état → `strace -e trace=futex -p <TID>` sur un thread bloqué pour confirmer l'attente sur un verrou → analyse du code source pour repérer un ordre d'acquisition de verrous incohérent entre différentes portions du programme (cause typique d'interblocage progressif : deux fonctions différentes acquièrent les deux mêmes mutex, mais dans un ordre inversé) → correction par un ordre d'acquisition unifié et documenté.

## 18. Synthèse

- Un appel système bascule le CPU en mode noyau de façon contrôlée — mécanisme central de service **et** de sécurité
- L'IPC permet à des processus isolés (M3) d'échanger des données : pipes/FIFO (flux), mémoire partagée (performance, sans synchronisation automatique), files de messages (structuré), sockets (local ou réseau)
- Mutex, sémaphores, spinlocks et variables de condition sont les primitives de base de la synchronisation, chacune avec un compromis coût/usage différent
- Trois stratégies existent face à l'interblocage : prévention (restrictive mais sûre), évitement (algorithme du banquier, nécessite de connaître les besoins maximaux), détection/rétablissement (permissive, accepte un risque)
- Linux n'impose aucune prévention/évitement systématique au niveau noyau pour les verrous applicatifs — c'est une responsabilité de conception logicielle
- Les ressources IPC System V survivent à la terminaison du processus créateur si elles ne sont pas explicitement libérées — source classique de fuite de ressources

## 19. Questions d'évaluation

**QCM**
1. La mémoire partagée, contrairement à un pipe, ne fournit pas nativement :
 a) un espace de communication b) une synchronisation automatique des accès c) une taille définie d) de nom

2. L'algorithme du banquier appartient à la catégorie :
 a) prévention b) évitement c) détection d) aucune des trois

**Questions ouvertes**
- Expliquez pourquoi un mutex acquis dans un ordre différent par deux portions de code distinctes peut provoquer un interblocage, même si chaque portion de code semble correcte isolément.
- Expliquez le mécanisme futex et pourquoi il est plus rapide qu'un appel système systématique pour un mutex non contesté.

**Exercice technique**
- Donnez la commande permettant d'identifier, parmi les ressources IPC System V actives sur un système, celles qui n'ont aucun processus attaché.

**Problème de diagnostic**
- Une application se fige totalement sans crasher. Donnez la première commande à exécuter pour orienter le diagnostic entre "interblocage" et "simple lenteur/attente légitime", et ce que son résultat permettrait de conclure.

## 20. Mini-projet

Développer un script (Bash ou Python) qui :
1. Scanne périodiquement les ressources IPC System V actives (`ipcs -a`)
2. Détecte les segments/sémaphores/files sans processus attaché depuis plus de N secondes (configurable)
3. Pour les sémaphores, tente d'identifier (via `/proc/PID/status` ou équivalent) des processus bloqués depuis un temps anormalement long en attente d'une ressource IPC (piste avancée, à documenter même si l'implémentation complète dépasse le cadre du module)
4. Journalise chaque détection avec horodatage et détails
5. Génère un rapport hebdomadaire de synthèse

---

# PARTIE II — TD

### Niveau 1 — Compréhension

**Exercice 1.1**
Expliquez ce qu'est un appel système et pourquoi un processus utilisateur ne peut pas accéder directement au matériel sans passer par ce mécanisme.

**Exercice 1.2**
Citez trois mécanismes d'IPC vus en cours et, pour chacun, une caractéristique qui le distingue clairement des deux autres.

**Exercice 1.3**
Quelle est la différence fondamentale entre un mutex et un spinlock en termes de comportement d'une unité d'exécution en attente ?

### Niveau 2 — Application

**Exercice 2.1**
Un sémaphore est initialisé à 3. Trois threads appellent `wait()` (P) sur ce sémaphore sans qu'aucun `signal()` (V) n'ait encore été appelé. Un quatrième thread appelle ensuite `wait()`. Que se passe-t-il pour ce quatrième thread ?

**Exercice 2.2**
Donnez la commande permettant de lister toutes les ressources IPC System V (mémoire partagée, sémaphores, files de messages) actuellement actives sur un système Linux.

**Exercice 2.3**
Un programme crée un segment de mémoire partagée System V puis se termine sans appeler la fonction de libération correspondante. Ce segment disparaît-il automatiquement ? Justifiez.

### Niveau 3 — Analyse

**Exercice 3.1**
Deux threads T1 et T2 doivent tous deux acquérir les mutex A et B pour effectuer leur traitement. T1 les acquiert dans l'ordre A puis B ; T2 les acquiert dans l'ordre B puis A. Expliquez précisément le scénario d'entrelacement qui mène à un interblocage, et proposez une correction respectant le principe de prévention vu en cours.

**Exercice 3.2**
Un système utilise l'algorithme du banquier pour gérer l'allocation de trois types de ressources entre plusieurs processus. Un processus demande une allocation supplémentaire qui, si elle était accordée, laisserait le système dans un état où aucune séquence d'exécution ne garantit la terminaison de tous les processus. Que doit faire le système face à cette demande, et cela signifie-t-il qu'un interblocage est certain si la demande est refusée ? Justifiez la nuance.

### Niveau 4 — Problème complexe

**Exercice 4.1**
Un pipeline de traitement de données est composé de 3 processus communiquant via une file de messages de capacité limitée (10 messages maximum) : un processus producteur, un processus filtre (lit un message, le transforme, écrit le résultat dans une seconde file), et un processus consommateur. Après plusieurs heures de fonctionnement, le pipeline se fige totalement. Concevez une démarche de diagnostic complète (au moins 5 étapes ordonnées, avec les commandes associées) permettant d'identifier la cause exacte, en envisageant explicitement au moins deux hypothèses distinctes (l'une liée à un interblocage classique par cycle d'attente, l'autre liée à une saturation de capacité de file sans réel cycle d'attente au sens de Coffman) et la façon de les distinguer l'une de l'autre.

---

## Correction du TD

**1.1** — Un appel système est le mécanisme par lequel un processus en espace utilisateur demande un service au noyau (accès matériel, création de processus, IPC...), en déclenchant une interruption logicielle qui bascule le CPU en mode noyau. Un processus utilisateur ne peut pas accéder directement au matériel car cela romprait l'isolation et la sécurité du système : n'importe quel programme pourrait alors manipuler directement le disque, la mémoire d'autres processus, ou le réseau sans aucun contrôle du noyau — l'appel système impose un point de passage unique et contrôlé.

**1.2** — Exemples possibles : le pipe anonyme est limité aux processus apparentés et ne survit pas à leur terminaison ; la mémoire partagée offre l'échange le plus rapide (pas de copie) mais ne fournit aucune synchronisation automatique ; le socket est le seul mécanisme capable de franchir la frontière d'une seule machine (communication réseau).

**1.3** — Une unité en attente d'un mutex est **bloquée** (état Waiting, transition vue en M2, ne consomme pas de CPU pendant l'attente) ; une unité en attente d'un spinlock **boucle activement** (busy-waiting), consommant du CPU en continu jusqu'à obtenir le verrou — adapté uniquement à des attentes très courtes, typiquement en contexte noyau multi-cœur.

**2.1** — Le sémaphore passe de 3 à 0 après les trois premiers `wait()`. Le quatrième thread appelant `wait()` sur un sémaphore à 0 est **bloqué** (mis en attente), jusqu'à ce qu'un `signal()` soit appelé par l'un des trois threads détenteurs, ce qui incrémentera le compteur et débloquera potentiellement le thread en attente.

**2.2** — `ipcs -a` (affiche l'ensemble des ressources IPC System V actives : mémoire partagée, sémaphores, files de messages).

**2.3** — Non, le segment ne disparaît pas automatiquement. Contrairement à la mémoire privée d'un processus (libérée par le noyau à sa terminaison), une ressource IPC System V (créée via `shmget`) **persiste** tant qu'elle n'est pas explicitement supprimée (via `shmctl` avec `IPC_RMID`, ou manuellement via `ipcrm`) — c'est une source classique de fuite de ressources détectable avec `ipcs -a` (segment visible avec `nattch=0`, aucun processus attaché).

**3.1** — Scénario : T1 acquiert A, puis T2 acquiert B (avant que T1 n'ait eu le temps d'acquérir B). T1 tente ensuite d'acquérir B (détenu par T2, T1 se bloque) ; T2 tente d'acquérir A (détenu par T1, T2 se bloque). Aucun des deux ne peut progresser : interblocage classique par attente circulaire (cycle T1→B→T2→A→T1), exactement la 4ᵉ condition de Coffman vue en M2. Correction respectant le principe de prévention : imposer un **ordre total d'acquisition** des mutex pour tout le code (par exemple, toujours acquérir A avant B, sans exception, y compris dans le code de T2) — cela casse la possibilité même du cycle d'attente circulaire.

**3.2** — Le système doit **refuser** la demande (ou la différer), car l'accorder mènerait à un état "non sûr" (unsafe state) selon la définition de l'algorithme du banquier. Cela ne signifie *pas* qu'un interblocage est certain si la demande est refusée : un état "non sûr" signifie seulement qu'*aucune garantie* ne peut être donnée qu'une séquence d'exécution complète existe pour tous les processus — un interblocage n'est pas certain dans un état non sûr, il devient seulement *possible* selon la façon dont les processus se comporteront réellement par la suite. L'algorithme du banquier est donc **conservateur par construction** : il peut refuser des allocations qui, en pratique, n'auraient pas causé de problème, au bénéfice d'une garantie totale d'absence d'interblocage tant qu'il accepte une demande.

**4.1** — Démarche possible :
1. `ps -eo pid,stat,cmd` sur les 3 processus du pipeline — confirme qu'ils sont bien en état `S` (sleeping/en attente) et non crashés ou en `D` (ce qui orienterait plutôt vers un problème disque, hors sujet ici)
2. `ipcs -q` (files de messages) — vérifie si l'une des deux files est pleine (proche de sa capacité maximale de 10 messages) : une file pleine bloquerait un producteur en écriture sans qu'il s'agisse nécessairement d'un cycle d'attente au sens strict de Coffman
3. Si une file est pleine et l'autre vide : hypothèse de **saturation de capacité asymétrique** plutôt qu'interblocage classique — par exemple le filtre est bloqué en écriture sur la seconde file (pleine, car le consommateur est plus lent), ce qui l'empêche à son tour de lire la première file (donc désormais pleine aussi côté producteur) : un blocage en cascade qui *ressemble* à un interblocage mais dont la cause racine est un déséquilibre de débit, pas un cycle de verrous applicatifs
4. `strace -p <PID>` sur chacun des 3 processus pour confirmer précisément sur quel appel système chacun est bloqué (`msgsnd` en écriture, `msgrcv` en lecture) — permet de distinguer un blocage IPC classique (capacité) d'un blocage sur mutex applicatif (auquel cas `strace -e trace=futex` serait plus pertinent)
5. Si l'hypothèse de saturation par déséquilibre de débit est confirmée : la correction n'est pas un simple réordonnancement de verrous (inutile ici, il n'y a pas de cycle Coffman classique entre mutex applicatifs) mais un redimensionnement de la capacité des files, ou une accélération du consommateur le plus lent, ou l'introduction d'un mécanisme de contrôle de flux (backpressure) explicite entre les étages du pipeline

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur système sur un pipeline de traitement multi-processus développé en interne. L'équipe rapporte des blocages intermittents et vous demande de construire un environnement de test reproductible pour comprendre, provoquer et corriger ces situations.

## Architecture

Une machine Linux unique (VM ou conteneur avec `python3`, `gcc` optionnel, `ipcs`/`ipcutils` installés). Aucune infrastructure réseau nécessaire.

## Objectifs
- Manipuler concrètement pipes, FIFO et mémoire partagée
- Provoquer et observer un interblocage classique par ordre de verrous inversé
- Observer et nettoyer une ressource IPC orpheline
- Écrire un script de détection de ressources IPC orphelines

## Préparation

```bash
which ipcs ipcrm mkfifo strace python3 2>/dev/null
```

## Étapes

### Étape 1 — Pipe et FIFO nommée

```bash
# Pipe anonyme (processus apparentés, via le shell)
echo "test pipe anonyme" | cat

# FIFO nommée (processus indépendants)
mkfifo /tmp/lab_fifo
(cat /tmp/lab_fifo &)
sleep 1
echo "message via FIFO nommée" > /tmp/lab_fifo
sleep 1
rm /tmp/lab_fifo
```
*Question :* le pipe anonyme aurait-il fonctionné entre deux terminaux totalement indépendants (pas de relation parent/enfant) ? Pourquoi la FIFO nommée le permet-elle ?

### Étape 2 — Ressource IPC orpheline

```bash
python3 -c "
import sysv_ipc
seg = sysv_ipc.SharedMemory(None, sysv_ipc.IPC_CREX, size=1024)
print('Segment créé, id:', seg.id)
" 2>/dev/null || echo "Module sysv_ipc absent, utiliser l'alternative ci-dessous"

# Alternative sans dépendance externe : utiliser ipcmk (souvent disponible via util-linux)
ipcmk -M 1024   # crée un segment de mémoire partagée de 1024 octets, affiche son id

ipcs -m   # observer le segment, colonne "nattch" à 0 (aucun processus attaché)
```
*Résultat attendu :* le segment apparaît dans `ipcs -m` avec `nattch=0`, preuve qu'il persiste sans processus attaché — exactement le scénario de fuite de ressources vu en cours.

Nettoyage :
```bash
ipcrm -m <shmid_observé>
ipcs -m   # confirmer la disparition
```

### Étape 3 — Provoquer un interblocage classique

```bash
cat << 'EOF' > /tmp/deadlock_demo.py
import threading
import time

lock_a = threading.Lock()
lock_b = threading.Lock()

def thread1():
    with lock_a:
        print("T1 a acquis A")
        time.sleep(1)
        print("T1 tente B...")
        with lock_b:
            print("T1 a acquis B")

def thread2():
    with lock_b:
        print("T2 a acquis B")
        time.sleep(1)
        print("T2 tente A...")
        with lock_a:
            print("T2 a acquis A")

t1 = threading.Thread(target=thread1)
t2 = threading.Thread(target=thread2)
t1.start()
t2.start()
t1.join(timeout=5)
t2.join(timeout=5)
print("Terminé (ou bloqué si vous ne voyez pas ce message après 5s)")
EOF
timeout 8 python3 /tmp/deadlock_demo.py
```
*Résultat attendu :* les deux threads affichent qu'ils tentent d'acquérir le second verrou, mais aucun message "a acquis B"/"a acquis A" final n'apparaît — le script se termine par le `timeout` de sécurité, preuve de l'interblocage.

### Étape 4 — Correction et script de détection

Corrigez le scénario de l'étape 3 en imposant un ordre total d'acquisition (les deux threads acquièrent toujours A avant B), puis relancez pour confirmer l'absence de blocage.

Ensuite, testez le script de détection de ressources IPC orphelines fourni en section 16 du cours, après avoir recréé un segment orphelin comme à l'étape 2 :

```bash
bash ipc_leak_watch.sh
```

## Commandes de référence
`mkfifo`, `ipcs`, `ipcrm`, `ipcmk`, `strace -e trace=futex`, `python3`

## Résultats attendus
- Capture montrant le fonctionnement de la FIFO nommée entre deux processus indépendants
- Capture de `ipcs -m` montrant un segment orphelin (`nattch=0`), puis sa suppression confirmée
- Capture montrant l'interblocage provoqué à l'étape 3 (arrêt par timeout)
- Capture montrant la version corrigée de l'étape 3 se terminant normalement
- Log d'alerte généré par le script de détection à l'étape 4

## Questions

1. Dans l'étape 3, pourquoi l'ajout d'un `time.sleep(1)` dans chaque fonction rend-il l'interblocage quasi certain à chaque exécution, alors que sans ce délai, l'interblocage pourrait ne se produire qu'occasionnellement (dépendant du timing réel d'exécution) ?
2. Si, à l'étape 4, vous n'aviez corrigé qu'un seul des deux threads (en gardant l'ordre inversé sur l'autre), l'interblocage serait-il toujours possible ? Justifiez.
3. En production, sur un système où vous ne pouvez pas modifier le code source d'une application tierce bloquée par interblocage, quelle serait votre seule option réaliste de rétablissement, en lien avec la stratégie "détection et rétablissement" vue en cours ?

## Troubleshooting (panne à diagnostiquer)

**Situation fournie :** le script de détection de l'étape 4, exécuté en continu via `cron`, ne signale jamais d'alerte alors qu'un développeur affirme avoir observé, via `ipcs -m` manuellement, un segment orphelin resté actif pendant plusieurs heures la veille.

**Démarche attendue :**
1. Symptôme : absence de détection malgré une fuite confirmée manuellement
2. Hypothèse : le script (section 16 du cours) filtre sur `$6 == 0` (colonne `nattch`), mais la structure de sortie de `ipcs -m` peut varier légèrement selon la version/distribution (nombre d'espaces, colonnes localisées) — le filtrage `awk` par position de colonne est fragile
3. Vérification : exécuter `ipcs -m` manuellement et compter précisément les colonnes réelles, comparer avec l'hypothèse du script (`NR>3` pour ignorer l'en-tête, `$6` pour `nattch`)
4. Isolation : reproduire un segment orphelin volontairement (comme à l'étape 2) et exécuter la ligne `awk` du script isolément pour voir si elle le détecte réellement
5. Correction : ajuster le script pour se baser sur les en-têtes de colonnes plutôt que sur une position fixe (ou utiliser une option de sortie plus stable si disponible), et ajouter un test automatisé reproduisant un segment orphelin connu pour valider la détection avant déploiement
6. Vérification : relancer le script corrigé sur le cas reproduit et confirmer la détection
7. Documentation : noter la fragilité du parsing par position de colonne comme point de vigilance récurrent en scripting système (déjà rencontré avec `lsof` au module 4) — leçon transversale à généraliser à tout script s'appuyant sur la sortie texte d'une commande système

## Validation

Le TP est validé si l'étudiant fournit :
- Les 5 captures/résultats demandés
- Des réponses argumentées aux 3 questions
- Une identification correcte de la fragilité de parsing dans le scénario de troubleshooting, avec une correction méthodologiquement solide

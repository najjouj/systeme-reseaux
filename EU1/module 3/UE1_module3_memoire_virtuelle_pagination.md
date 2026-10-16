# UE1 — Module 3 : Mémoire Virtuelle et Pagination

---

# PARTIE I — COURS

## 1. Titre du cours
Mémoire virtuelle et pagination : adressage, MMU et gestion des défauts de page

## 2. Niveau
Élève ingénieur — cycle ingénieur, tronc commun systèmes (S1)

## 3. Durée
CM : 4h (module complet UE1-M3 : +3h TD, +4h TP en Parties II et III)

## 4. Prérequis
- Représentation binaire/hexadécimale des nombres
- UE1-M2 : espace d'adressage d'un processus, PCB

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Expliquer pourquoi la mémoire virtuelle existe et quel(s) problème(s) elle résout
- Décrire le mécanisme de pagination (pages, cadres, table des pages, MMU, TLB)
- Différencier pagination et segmentation, et justifier le choix de Linux
- Expliquer le traitement d'un défaut de page et distinguer défaut mineur/majeur
- Diagnostiquer un problème mémoire (swap excessif, thrashing, fuite) avec les outils Linux
- Lire les informations mémoire d'un processus (`/proc/PID/status`, `smaps`, `pmap`)

## 6. Motivation

Un serveur exécute simultanément 200 processus. Chacun "croit" disposer d'un espace mémoire de plusieurs gigaoctets, alors que la machine physique n'a peut-être que 16 Go de RAM au total. Comment est-ce possible ? Comment le système empêche-t-il un processus de lire ou corrompre la mémoire d'un autre, sans que les développeurs n'aient à s'en soucier explicitement dans leur code ?

Et à l'inverse : pourquoi une application qui "fuit" de la mémoire finit-elle par rendre tout le serveur inutilisable, bien avant que la mémoire ne soit réellement épuisée ? Pourquoi un serveur avec "assez" de RAM peut-il devenir subitement inutilisable, avec un CPU à 100% alors qu'aucun calcul lourd n'est en cours ?

La réponse à toutes ces questions se trouve dans un seul mécanisme : la **mémoire virtuelle**, et son implémentation la plus répandue, la **pagination**. C'est un mécanisme invisible pour le développeur applicatif, mais absolument central pour l'ingénieur système : la moitié des incidents de performance en production ont une composante mémoire.

## 7. Concepts fondamentaux

### 7.1 Pourquoi la mémoire virtuelle ?

Sans mémoire virtuelle, chaque processus devrait connaître et utiliser directement des adresses physiques RAM. Cela poserait trois problèmes majeurs :

1. **Isolation** : rien n'empêcherait un processus de lire/écrire dans la mémoire physique d'un autre processus — catastrophe pour la sûreté et la sécurité.
2. **Fragmentation et allocation** : le système devrait garantir à chaque processus un bloc contigu de mémoire physique libre, ce qui devient vite impraticable à mesure que les processus se créent et se terminent (fragmentation externe).
3. **Sur-allocation impossible** : la somme des besoins mémoire de tous les processus dépasse très souvent la RAM physique réellement disponible — sans mécanisme d'indirection, ce serait un facteur limitant strict.

La **mémoire virtuelle** répond à ces trois problèmes en donnant à chaque processus l'**illusion** d'un espace d'adressage privé, continu, et potentiellement plus grand que la RAM physique, tout en isolant totalement les processus les uns des autres.

### 7.2 Adressage virtuel vs adressage physique

- **Adresse virtuelle** : adresse utilisée par le processus (dans son propre espace d'adressage, indépendant des autres processus)
- **Adresse physique** : adresse réelle en RAM

La traduction entre les deux est effectuée à chaque accès mémoire par un composant matériel dédié : la **MMU** (Memory Management Unit), à l'aide d'une structure gérée par le noyau : la **table des pages**.

**Point clé pédagogique :** deux processus différents peuvent utiliser la *même* adresse virtuelle (ex : 0x00400000) sans jamais se marcher dessus, car cette adresse virtuelle est traduite vers des adresses physiques totalement différentes selon la table des pages de chaque processus.

### 7.3 Pagination : pages et cadres

La pagination découpe :
- l'espace d'adressage **virtuel** en blocs de taille fixe appelés **pages** (typiquement 4 Ko sous Linux x86_64)
- la mémoire **physique** en blocs de même taille appelés **cadres de page** (page frames)

Une page virtuelle n'a pas besoin d'être associée à un cadre physique contigu par rapport aux pages voisines — c'est ce qui élimine la fragmentation externe. La **table des pages** associe chaque page virtuelle à son cadre physique (ou indique qu'elle n'est pas actuellement chargée en RAM).

```
Espace virtuel du processus         Mémoire physique (RAM)
┌────────────┐                      ┌────────────┐
│ Page 0      │──────────────────▶ │ Cadre 12    │
├────────────┤                      ├────────────┤
│ Page 1      │──────────────────▶ │ Cadre 3     │
├────────────┤                      ├────────────┤
│ Page 2      │── (non chargée) ──▶ │  (sur disque, swap)
├────────────┤                      ├────────────┤
│ Page 3      │──────────────────▶ │ Cadre 47    │
└────────────┘                      └────────────┘
```

### 7.4 MMU et TLB

À chaque accès mémoire, le CPU génère une adresse virtuelle, que la **MMU** doit traduire en adresse physique en consultant la table des pages. Mais consulter la table des pages en mémoire à *chaque* accès mémoire serait extrêmement coûteux (elle-même est en RAM, donc la traduction doublerait le nombre d'accès mémoire).

C'est pourquoi le CPU dispose d'un cache matériel dédié : le **TLB** (Translation Lookaside Buffer), qui mémorise les traductions récentes. Un accès dont la traduction est déjà en TLB (*TLB hit*) est quasi instantané ; un *TLB miss* nécessite de consulter la table des pages en mémoire, ce qui est nettement plus lent.

**Implication pratique :** un changement de contexte entre deux processus peut invalider tout ou partie du TLB (car les tables de pages sont différentes), ce qui explique en partie pourquoi les changements de contexte fréquents dégradent la performance globale au-delà du simple coût de sauvegarde/restauration des registres.

### 7.5 Défaut de page (page fault)

Un **défaut de page** survient quand le CPU tente d'accéder à une page virtuelle qui n'a pas de traduction valide immédiatement disponible. Le comportement diffère selon la cause :

- **Défaut mineur (minor page fault)** : la page existe déjà en mémoire physique mais n'est pas encore mappée dans la table des pages du processus (ex : mémoire partagée, copy-on-write après un `fork()`). Résolu rapidement, sans accès disque.
- **Défaut majeur (major page fault)** : la page doit être chargée depuis le disque (fichier exécutable, ou page swappée). Coûteux — implique une opération I/O.
- **Segmentation fault** : le processus accède à une adresse virtuelle qui n'a *aucune* correspondance valide et n'en aura jamais (accès hors des zones légitimement allouées) — le noyau termine le processus avec le signal `SIGSEGV`.

**Ce qui se passe lors d'un défaut de page valide :**
1. Le CPU déclenche une exception matérielle
2. Le noyau identifie la page manquante et sa localisation (fichier, swap)
3. Le noyau alloue un cadre physique libre (ou en libère un via une politique de remplacement, ex : LRU approximatif)
4. Le contenu est chargé depuis le disque si nécessaire (défaut majeur)
5. La table des pages est mise à jour
6. L'instruction qui a déclenché le défaut est ré-exécutée

### 7.6 Segmentation (comparaison)

La **segmentation** découpe la mémoire en blocs de taille *variable*, correspondant à des unités logiques du programme (code, données, pile). Contrairement à la pagination, elle ne résout pas intrinsèquement la fragmentation externe (les segments de taille variable peuvent laisser des trous inutilisables). Certains systèmes combinent historiquement les deux (segmentation paginée), mais les architectures modernes (dont x86_64 sous Linux) utilisent quasi exclusivement la **pagination pure**, la segmentation étant réduite à un rôle résiduel voire désactivée en mode 64 bits.

### 7.7 Swap et Thrashing

Le **swap** est un espace disque utilisé comme extension de la RAM : les pages peu utilisées peuvent être écrites sur disque pour libérer de la RAM physique pour d'autres pages plus actives.

Le **thrashing** est une pathologie survenant quand le système passe l'essentiel de son temps à échanger des pages entre RAM et disque plutôt qu'à exécuter du travail utile — typiquement quand la mémoire physique disponible est très inférieure au besoin réel de l'ensemble des processus actifs. Symptôme typique : CPU très occupé (en attente I/O, état `D`), système quasi figé, alors qu'aucun calcul intensif n'est visible.

**Diagnostic caractéristique du thrashing :** taux de défauts de page majeurs très élevé, activité disque (I/O wait) élevée, load average élevé avec CPU réel faible — signature très proche de celle vue en UE1-M2 pour les processus en état `D`, mais ici la cause racine est la pression mémoire, pas un disque lent en tant que tel.

## 8. Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    CPU                                    │
│   Instruction → adresse virtuelle                          │
│           │                                                │
│           ▼                                                │
│   ┌─────────────┐   TLB hit   ┌──────────────────────┐    │
│   │    TLB       │────────────▶│ adresse physique      │   │
│   └──────┬───────┘             └──────────────────────┘   │
│          │ TLB miss                                        │
│          ▼                                                 │
│   ┌─────────────┐                                          │
│   │     MMU      │── consulte → Table des pages (RAM)      │
│   └─────────────┘                                          │
└─────────────────────────────────────────────────────────┘
           │
           ▼ page absente → Défaut de page → noyau
┌─────────────────────────────────────────────────────────┐
│  Noyau : gestionnaire de mémoire                           │
│  - alloue un cadre physique libre                           │
│  - charge depuis disque si nécessaire (swap ou fichier)     │
│  - met à jour la table des pages                            │
└─────────────────────────────────────────────────────────┘
```

## 9. Fonctionnement interne (cas Linux)

Sous Linux, chaque processus dispose d'une structure `mm_struct` décrivant son espace d'adressage virtuel, incluant les zones mappées (`vm_area_struct`) : code, données, tas (heap), pile (stack), bibliothèques partagées, fichiers mappés en mémoire (`mmap`).

Linux utilise une politique d'**allocation paresseuse** (lazy allocation / overcommit) : lorsqu'un processus demande de la mémoire (via `malloc` → `brk`/`mmap`), le noyau réserve l'espace d'adressage virtuel mais n'alloue **pas immédiatement** de page physique. La page physique n'est réellement allouée qu'au premier accès effectif (premier défaut de page mineur sur cette zone). Cela explique pourquoi la mémoire "réservée" (VSZ) et la mémoire "réellement utilisée" (RSS) d'un processus peuvent différer fortement.

Quand la mémoire physique devient critique et qu'aucune page ne peut être libérée à temps, le noyau Linux déclenche l'**OOM Killer** (Out-Of-Memory Killer), qui sélectionne et termine de force un processus (selon un score `oom_score`) pour éviter le blocage total du système.

## 10. Démonstration

```bash
# Voir l'espace d'adressage virtuel d'un processus
cat /proc/self/maps

# Voir la mémoire virtuelle (VSZ) vs résidente (RSS)
ps -o pid,vsz,rss,cmd -p $$

# Observer les défauts de page d'un processus
/usr/bin/time -v sleep 1
# chercher "Minor (reclaiming a frame) page faults" et "Major"
```

## 11. Commandes / outils

| Commande | Rôle | Exemple |
|---|---|---|
| `free -h` | Vue globale RAM/swap | `free -h` |
| `vmstat 1` | Statistiques mémoire/swap en continu | `vmstat 1 5` |
| `ps -o vsz,rss` | Mémoire virtuelle vs résidente d'un processus | `ps -o pid,vsz,rss -p 1234` |
| `pmap -x PID` | Cartographie mémoire détaillée d'un processus | `pmap -x 1234` |
| `cat /proc/PID/smaps` | Détail par zone mémoire (partagée, privée, swappée) | `cat /proc/1234/smaps` |
| `/usr/bin/time -v` | Statistiques d'exécution incluant les défauts de page | `/usr/bin/time -v ./mon_prog` |
| `dmesg \| grep -i "out of memory"` | Vérifier une intervention de l'OOM Killer | `dmesg \| grep -i oom` |

**Erreur fréquente :** confondre VSZ (mémoire virtuelle réservée, souvent très supérieure au besoin réel à cause de l'allocation paresseuse et des bibliothèques partagées) avec RSS (mémoire physique réellement utilisée) — c'est RSS qui doit être surveillé pour évaluer la pression mémoire réelle d'un processus.

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** un service de traitement de données en production voit son temps de réponse se dégrader progressivement sur plusieurs heures, jusqu'à devenir inutilisable, sans pic de trafic particulier. `free -h` montre un swap presque plein. `vmstat 1` montre une colonne `si`/`so` (swap in/out) constamment non nulle.

**Analyse attendue :** ces symptômes sont caractéristiques d'un **thrashing** progressif — probablement causé par une fuite mémoire applicative qui consomme de plus en plus de RAM au fil du temps, forçant le noyau à swapper des pages de plus en plus activement pour libérer de l'espace, jusqu'à ce que le système passe l'essentiel de son temps à gérer des I/O de swap plutôt qu'à traiter les requêtes. Piste de résolution immédiate : redémarrer le service pour libérer la mémoire (traitement symptomatique) ; piste de fond : profiler l'application pour identifier la fuite (ex: objets non libérés, cache non borné).

## 15. Sécurité

- **ASLR (Address Space Layout Randomization)** : randomise les adresses virtuelles de base (pile, tas, bibliothèques) à chaque exécution, rendant plus difficile l'exploitation de failles mémoire (buffer overflow) qui dépendent d'adresses prévisibles.
- **Protections W^X (Write XOR Execute)** : une page mémoire ne doit jamais être à la fois inscriptible et exécutable, ce qui limite fortement l'exploitation de failles par injection de code exécutable dans une zone de données.
- L'isolation offerte par la mémoire virtuelle est elle-même une mesure de sécurité fondamentale : sans elle, aucune séparation de privilèges entre processus ne serait possible.
- Les attaques de type **buffer overflow** exploitent typiquement une écriture hors des limites d'une zone mémoire allouée — comprendre l'architecture mémoire (pile, tas, code) est un prérequis pour comprendre ce type de vulnérabilité (approfondi en UE8 — Sécurité).

## 16. Automatisation

Script Bash de détection de pression mémoire critique (swap actif + RAM faible), avec alerte.

```bash
#!/usr/bin/env bash
# Détecte une pression mémoire critique et journalise une alerte
set -euo pipefail

SEUIL_RAM_LIBRE_MO=200
LOGFILE="/var/log/mem_watch.log"

RAM_LIBRE=$(free -m | awk '/^Mem:/ {print $7}')   # colonne "available"
SWAP_UTILISE=$(free -m | awk '/^Swap:/ {print $3}')

if (( RAM_LIBRE < SEUIL_RAM_LIBRE_MO && SWAP_UTILISE > 0 )); then
    echo "$(date '+%F %T') - ALERTE: RAM libre=${RAM_LIBRE}Mo, Swap utilisé=${SWAP_UTILISE}Mo" \
        | tee -a "$LOGFILE"
    echo "--- Top 5 processus par RSS ---" >> "$LOGFILE"
    ps -eo pid,comm,rss --sort=-rss | head -6 >> "$LOGFILE"
    exit 1
fi

echo "$(date '+%F %T') - OK: RAM libre=${RAM_LIBRE}Mo" >> "$LOGFILE"
exit 0
```

## 17. Troubleshooting

Scénario court : un conteneur applicatif est tué de façon répétée sans message d'erreur applicatif explicite. Démarche : `dmesg | grep -i oom` pour confirmer une intervention de l'OOM Killer → identifier le processus visé et son `oom_score` → vérifier la limite mémoire du conteneur (`docker inspect`, cgroup memory.max) → comparer avec le RSS réel observé avant la mort du processus → ajuster la limite ou corriger la fuite mémoire applicative.

## 18. Synthèse

- La mémoire virtuelle résout l'isolation, la fragmentation et permet la sur-allocation contrôlée
- La pagination découpe l'espace en blocs fixes (pages/cadres), traduits par la MMU via la table des pages, accélérée par le TLB
- Un défaut de page peut être mineur (rapide) ou majeur (coûteux, accès disque) ; un accès invalide déclenche SIGSEGV
- Linux alloue la mémoire de façon paresseuse : VSZ (réservé) ≠ RSS (réellement utilisé)
- Le thrashing survient sous pression mémoire extrême et se signale par un CPU faible mais un système quasi figé, avec I/O de swap élevées
- L'OOM Killer intervient en dernier recours pour préserver la stabilité globale du système

## 19. Questions d'évaluation

**QCM**
1. Un défaut de page majeur implique :
 a) un accès disque b) uniquement une mise à jour de la table des pages c) toujours un SIGSEGV d) une invalidation du TLB uniquement

2. Le TLB sert à :
 a) stocker les pages swappées b) accélérer la traduction adresse virtuelle → physique c) remplacer la table des pages d) détecter les fuites mémoire

**Questions ouvertes**
- Expliquez pourquoi VSZ peut être très supérieur à RSS pour un même processus, en citant le mécanisme responsable.
- Pourquoi le thrashing peut-il faire apparaître un système comme "figé" alors que le CPU affiche une utilisation réelle très faible ?

**Exercice technique**
- À partir d'une sortie `free -h` fournie, calculez le pourcentage de swap utilisé et déterminez si la situation est préoccupante en justifiant un seuil raisonnable.

**Problème de diagnostic**
- Un conteneur redémarre en boucle sans log applicatif d'erreur. Donnez la première commande à exécuter et ce que vous en attendez.

## 20. Mini-projet

Développer un script (Bash ou Python) qui :
1. Surveille en continu (boucle ou cron) l'utilisation RAM/swap du système
2. Détecte une tendance de croissance continue du swap sur une fenêtre glissante (ex : 5 mesures consécutives en hausse) — signature potentielle de fuite mémoire ou de thrashing naissant
3. Identifie et journalise les 3 processus les plus consommateurs de RSS au moment de la détection
4. Produit un rapport quotidien de synthèse

---

# PARTIE II — TD

### Niveau 1 — Compréhension

**Exercice 1.1**
Expliquez, en une phrase pour chacun, la différence entre adresse virtuelle et adresse physique, et le rôle de la MMU dans cette relation.

**Exercice 1.2**
Qu'est-ce qu'un défaut de page mineur ? Donnez un exemple concret de situation qui en génère un.

**Exercice 1.3**
Pourquoi Linux privilégie-t-il la pagination pure plutôt que la segmentation ?

### Niveau 2 — Application

**Exercice 2.1**
Vous exécutez `ps -o pid,vsz,rss -p 1234` et obtenez `VSZ=800000` (Ko) et `RSS=45000` (Ko). Expliquez cet écart avec le mécanisme vu en cours.

**Exercice 2.2**
Donnez la commande permettant d'observer, pour le processus courant du shell, la liste des zones mémoire mappées (fichiers, bibliothèques, pile, tas).

**Exercice 2.3**
Donnez la commande permettant de vérifier si l'OOM Killer est intervenu récemment sur le système.

### Niveau 3 — Analyse

**Exercice 3.1**
Un système affiche : CPU à 8% d'utilisation, load average à 12, et `vmstat 1` montre des colonnes `si`/`so` non nulles en continu. Interprétez cette situation et distinguez-la du cas "processus en `D` à cause d'un disque lent" vu en UE1-M2 — quel indicateur permet de trancher entre les deux causes ?

**Exercice 3.2**
Expliquez pourquoi un système avec beaucoup de RAM libre en apparence (`free -h` affichant un chiffre élevé dans la colonne `available`) peut malgré tout déclencher l'OOM Killer sur un processus spécifique. (Indice : penser aux limites cgroups d'un conteneur.)

### Niveau 4 — Problème complexe

**Exercice 4.1**
Un service applicatif redémarre toutes les 6 heures environ, sans erreur applicative visible dans ses propres logs. Concevez une démarche de diagnostic complète (au moins 5 étapes ordonnées) permettant de confirmer ou d'infirmer l'hypothèse d'une intervention de l'OOM Killer, en précisant à chaque étape la commande utilisée et ce que son résultat permettrait de conclure.

---

## Correction du TD

**1.1** — L'adresse virtuelle est celle utilisée par le processus dans son propre espace d'adressage logique ; l'adresse physique est l'adresse réelle en RAM. La MMU traduit la première vers la seconde à chaque accès mémoire, en consultant la table des pages du processus courant.

**1.2** — Un défaut de page mineur survient quand la page demandée existe déjà en mémoire physique mais n'est pas encore mappée dans la table des pages du processus courant — aucun accès disque n'est nécessaire. Exemple : après un `fork()`, les pages du processus enfant pointent initialement (copy-on-write) vers les mêmes cadres physiques que le parent ; le premier accès en écriture déclenche un défaut mineur pour établir le mapping propre à l'enfant (avant la copie réelle).

**1.3** — La pagination utilise des blocs de taille fixe, ce qui élimine la fragmentation externe (contrairement à la segmentation, dont les blocs de taille variable peuvent laisser des trous inutilisables). Cette simplicité de gestion et l'efficacité matérielle associée (MMU/TLB optimisés pour des tailles de page fixes) expliquent pourquoi les architectures modernes, dont x86_64 sous Linux, privilégient la pagination pure.

**2.1** — L'écart s'explique par l'allocation paresseuse (lazy allocation) : VSZ représente tout l'espace d'adressage virtuel réservé (y compris bibliothèques partagées mappées mais non totalement utilisées, zones réservées par `malloc` mais jamais touchées), tandis que RSS ne compte que les pages physiquement chargées en RAM à cet instant. Un programme peut réserver beaucoup d'espace virtuel sans jamais y accéder entièrement.

**2.2** — `cat /proc/self/maps` (ou `cat /proc/$$/maps` en Bash pour cibler explicitement le shell courant).

**2.3** — `dmesg | grep -i "out of memory"` ou `dmesg | grep -i oom` ; on peut aussi consulter `/var/log/syslog` ou `journalctl -k | grep -i oom` selon la configuration de journalisation.

**3.1** — CPU faible + load average élevé + `si`/`so` actifs en continu indique un **thrashing** : le système passe son temps à échanger des pages entre RAM et swap plutôt qu'à exécuter du travail utile. L'indicateur qui permet de trancher avec le cas "disque lent classique" vu en UE1-M2 est précisément les colonnes `si`/`so` de `vmstat` (swap in/out) : si elles sont actives, la cause racine est la pression mémoire (pas un disque intrinsèquement lent sur des I/O applicatives classiques, bien que dans les deux cas le symptôme visible en `ps` puisse être des processus en état `D`).

**3.2** — Le `free -h` global du système peut afficher de la RAM disponible, mais un processus s'exécutant dans un **conteneur** (ou un cgroup) contraint par une limite mémoire spécifique peut être tué par l'OOM Killer dès qu'il dépasse *cette limite locale*, indépendamment de la RAM disponible au niveau de la machine hôte. C'est une cause très fréquente d'incidents en environnement conteneurisé (Docker/Kubernetes) mal dimensionné.

**4.1** — Démarche possible :
1. `dmesg | grep -i oom` (ou `journalctl -k | grep -i oom`) — confirme si l'OOM Killer est intervenu, et sur quel PID/nom de processus
2. Si confirmé : `docker inspect <conteneur> | grep -i memory` (ou consultation des limites cgroups) — permet de connaître la limite mémoire configurée
3. Comparaison de cette limite avec le RSS maximal observé avant chaque incident (via un monitoring historique ou en reproduisant la charge en test) — confirme si le processus dépasse effectivement la limite
4. Analyse de la tendance de RSS dans le temps (croissance continue ou pic ponctuel) — distingue une fuite mémoire progressive d'un pic de charge légitime
5. Selon le résultat : augmenter la limite mémoire (si pic légitime et occasionnel) ou profiler l'application pour corriger une fuite (si croissance continue)

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur système sur un serveur applicatif où l'équipe suspecte un problème de mémoire, sans certitude sur sa nature exacte (fuite, sous-dimensionnement, ou simple pic ponctuel). Vous devez observer le comportement réel du système, provoquer des situations contrôlées, et développer un outil de détection.

## Architecture

Une machine Linux unique (VM ou conteneur), avec un peu de swap configuré si possible (pour observer les colonnes `si`/`so` — sur un conteneur Docker standard sans swap dédié, l'exercice 3 peut être simulé côté hôte ou remplacé par une observation de la croissance RSS uniquement, voir note à l'étape 3).

## Objectifs
- Observer VSZ vs RSS sur un processus réel
- Provoquer et observer une croissance mémoire progressive (simulateur de fuite)
- Observer les défauts de page mineurs/majeurs
- Écrire un script de détection de tendance mémoire anormale

## Préparation

```bash
which free vmstat pmap ps python3 2>/dev/null
free -h
```

## Étapes

### Étape 1 — VSZ vs RSS

```bash
python3 -c "
import time
data = []
for i in range(5):
    data.append(bytearray(10 * 1024 * 1024))  # +10 Mo par itération
    time.sleep(2)
time.sleep(30)
" &
PID=$!
sleep 3
ps -o pid,vsz,rss,cmd -p $PID
```
*Résultat attendu :* RSS croît visiblement toutes les 2 secondes si vous relancez la commande `ps` plusieurs fois pendant l'exécution ; observez comment VSZ évolue en comparaison.

### Étape 2 — Défauts de page

```bash
/usr/bin/time -v python3 -c "
data = bytearray(50 * 1024 * 1024)
for i in range(0, len(data), 4096):
    data[i] = 1
" 2>&1 | grep -i "page faults"
```
*Question :* combien de défauts de page mineurs sont rapportés ? Pourquoi majoritairement mineurs et non majeurs dans ce scénario (aucun fichier ni swap impliqué) ?

### Étape 3 — Simuler une pression mémoire progressive (fuite simulée)

```bash
cat << 'EOF' > /tmp/leak_simulator.py
import time
chunks = []
for i in range(20):
    chunks.append(bytearray(20 * 1024 * 1024))  # +20 Mo toutes les 3s
    print(f"Itération {i+1}: ~{(i+1)*20} Mo alloués")
    time.sleep(3)
EOF
python3 /tmp/leak_simulator.py &
PID=$!

# Dans un autre terminal (ou en tâche de fond) :
watch -n 2 "ps -o pid,vsz,rss,cmd -p $PID"
```
*Note :* si le conteneur/VM ne dispose pas de swap configuré, remplacez l'observation de `vmstat` (`si`/`so`) par une simple observation de la croissance continue de RSS — l'objectif pédagogique (détection de tendance) reste identique.

### Étape 4 — Script de détection de tendance

Adapter et exécuter le script de la section 20 (mini-projet du cours) pour qu'il surveille spécifiquement le PID du simulateur de fuite lancé à l'étape 3, et qu'il déclenche une alerte après 3 mesures consécutives de RSS croissant.

## Commandes de référence
`ps`, `free`, `vmstat`, `pmap`, `/usr/bin/time -v`, `watch`

## Résultats attendus
- Capture montrant l'évolution de VSZ/RSS pendant l'étape 1
- Nombre de défauts de page mineurs/majeurs relevé à l'étape 2, avec interprétation écrite
- Capture montrant la croissance progressive de RSS à l'étape 3
- Log d'alerte généré par le script de détection à l'étape 4

## Questions

1. Pourquoi RSS croît-il progressivement dans l'étape 3 alors que le script alloue un bloc "d'un coup" (`bytearray(20*1024*1024)`) à chaque itération ? Reliez votre réponse au mécanisme d'allocation paresseuse.
2. Si vous relancez l'étape 2 en accédant à la mémoire dans l'ordre inverse (dernier octet d'abord), le nombre de défauts de page mineurs change-t-il ? Pourquoi ou pourquoi pas ?
3. Dans un environnement de production réel (pas simulé), quelle commande utiliseriez-vous en premier pour confirmer qu'un incident était bien dû à l'OOM Killer plutôt qu'à un crash applicatif classique ?

## Troubleshooting (panne à diagnostiquer)

**Situation fournie :** le script de détection de l'étape 4, exécuté en continu via `cron` toutes les minutes, déclenche des alertes en rafale même quand la mémoire du système est stable, dès qu'un autre processus ponctuel (ex : une sauvegarde `tar` nocturne) s'exécute brièvement.

**Démarche attendue :**
1. Symptôme : fausses alertes corrélées à l'exécution d'un processus tiers de courte durée
2. Hypothèse : le script suit peut-être un PID recyclé (le PID de l'ancien processus surveillé a été réattribué au nouveau processus `tar` par le système)
3. Vérification : comparer l'heure de démarrage du PID surveillé (`ps -o pid,lstart,cmd -p PID`) avec l'heure attendue du processus initialement ciblé
4. Isolation : confirmer que le script ne revérifie jamais que le PID correspond toujours à la bonne commande avant de comparer les mesures de RSS
5. Correction : ajouter une vérification du nom de commande (`comm`) ou de l'heure de démarrage en plus du PID avant chaque comparaison de tendance
6. Vérification : relancer sur plusieurs cycles avec un processus tiers actif et confirmer l'absence de fausse alerte
7. Documentation : consigner la cause (réutilisation de PID) dans le changelog du script, point de vigilance classique en scripting système

## Validation

Le TP est validé si l'étudiant fournit :
- Les 4 résultats/captures demandés
- Des réponses argumentées aux 3 questions, notamment un lien explicite avec l'allocation paresseuse
- Une identification correcte de la cause de réutilisation de PID dans le scénario de troubleshooting

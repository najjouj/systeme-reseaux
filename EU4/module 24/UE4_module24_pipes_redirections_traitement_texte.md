# UE4 — Module 24 : Pipes, Redirections et Traitement de Texte

---

# PARTIE I — COURS

## 1. Titre du cours
Chaîner et transformer : pipes, redirections, grep, sed, awk et l'écosystème du traitement de texte Unix

## 2. Niveau
Élève ingénieur — cycle ingénieur, UE4 automatisation (S2)

## 3. Durée
CM : 2h (module dense en outils, court en CM — +3h TD, +5h TP en Parties II et III)

## 4. Prérequis
- UE4-Module 23 : syntaxe Bash fondamentale, quotation
- UE1-Module 4 : descripteurs de fichiers (0/1/2 pour stdin/stdout/stderr, déjà mentionnés)

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Utiliser les redirections (`>`, `>>`, `<`, `2>`, `2>&1`, `&>`) en distinguant précisément stdout et stderr
- Chaîner des commandes avec des pipes pour construire des traitements complexes
- Écrire des expressions régulières de base et étendues avec `grep`
- Utiliser `sed` pour des substitutions et transformations de texte, y compris l'édition sur place
- Utiliser `awk` pour un traitement structuré par champs, avec motifs et actions
- Combiner `cut`, `sort`, `uniq`, `xargs` dans un pipeline efficace

## 6. Motivation

La philosophie Unix, déjà pressentie en UE1-M4 ("tout est fichier"), repose aussi sur un second principe fondamental : des **petits outils spécialisés**, chacun excellent dans une tâche précise, combinés entre eux via des pipes pour construire des traitements arbitrairement complexes — plutôt qu'un unique outil monolithique tentant de tout faire. `grep` cherche, `sed` transforme, `awk` structure, `sort` ordonne, `uniq` déduplique — et le pipe (`|`) est le ciment qui les assemble en une chaîne de traitement, sans jamais avoir besoin de fichier intermédiaire.

Ce module donne la maîtrise de cet écosystème, omniprésent dans l'administration système au quotidien : analyser un fichier de log volumineux, extraire une colonne précise d'une sortie de commande, transformer un format de données à la volée — des tâches que ces outils accomplissent en une ligne, là où un script plus verbeux dans un autre langage prendrait bien plus de lignes pour un résultat identique.

## 7. Concepts fondamentaux

### 7.1 Rappel et approfondissement des flux standards

Chaque processus dispose de trois flux standards (rappel UE1-M4/M5) : entrée standard (stdin, descripteur 0), sortie standard (stdout, descripteur 1), erreur standard (stderr, descripteur 2) — une distinction essentielle pour ce module, car les redirections manipulent précisément ces flux indépendamment les uns des autres.

### 7.2 Redirections

```bash
commande > fichier          # redirige stdout vers fichier (écrase le contenu existant)
commande >> fichier          # redirige stdout vers fichier (ajoute à la suite)
commande < fichier            # utilise fichier comme entrée standard
commande 2> erreurs.log        # redirige uniquement stderr
commande > sortie.log 2>&1       # redirige stdout ET stderr vers le même fichier (ordre important, voir ci-dessous)
commande &> tout.log              # équivalent Bash raccourci pour rediriger stdout et stderr ensemble
commande 2>/dev/null                # supprime silencieusement les messages d'erreur
```

**Pourquoi l'ordre compte dans `> sortie.log 2>&1` :** cette syntaxe se lit de gauche à droite — d'abord, stdout est redirigé vers `sortie.log` ; ensuite, `2>&1` fait pointer stderr vers **la même destination que stdout à cet instant précis** (déjà `sortie.log`). Écrire `2>&1 > sortie.log` (ordre inversé) produirait un résultat différent : stderr serait d'abord dupliqué vers la destination actuelle de stdout (le terminal, par défaut), **puis** seul stdout serait redirigé vers le fichier — stderr continuerait donc d'apparaître à l'écran, pas dans le fichier, un piège classique pour qui ne comprend pas cet ordre de traitement séquentiel.

### 7.3 Pipes

```bash
commande1 | commande2
```

Le pipe connecte la sortie standard de `commande1` directement à l'entrée standard de `commande2`, sans fichier intermédiaire — les deux commandes s'exécutent en parallèle, `commande2` consommant le flux au fur et à mesure que `commande1` le produit (un mécanisme de flux, pas une attente de complétion totale avant transmission).

**Note importante :** un pipe ne connecte que **stdout** par défaut, jamais stderr — les messages d'erreur d'une commande dans un pipeline continuent d'apparaître directement à l'écran, sauf redirection explicite préalable (`commande1 2>&1 | commande2` pour inclure également stderr dans le flux transmis).

### 7.4 `grep` — recherche par motif

```bash
grep "erreur" fichier.log              # lignes contenant "erreur"
grep -i "erreur" fichier.log             # insensible à la casse
grep -v "debug" fichier.log               # lignes NE contenant PAS "debug" (inversé)
grep -c "erreur" fichier.log               # compte les lignes correspondantes
grep -n "erreur" fichier.log                # affiche le numéro de ligne
grep -r "TODO" /projet/                      # recherche récursive dans une arborescence
grep -E "erreur|warning" fichier.log           # regex étendue (alternation)
```

**Expressions régulières de base :** `.` (un caractère quelconque), `*` (zéro ou plusieurs répétitions du caractère précédent), `^`/`$` (début/fin de ligne), `[...]` (classe de caractères). **Regex étendues (`-E`)** ajoutent `+` (une ou plusieurs répétitions), `?` (zéro ou une), `|` (alternation), `()` (groupement) — sans `-E`, ces caractères doivent être échappés (`\+`, `\|`...) pour conserver leur sens spécial en `grep` basique, une source fréquente de confusion.

### 7.5 `sed` — édition de flux (stream editor)

```bash
sed 's/ancien/nouveau/' fichier.txt          # remplace la première occurrence par ligne, affiche le résultat
sed 's/ancien/nouveau/g' fichier.txt           # remplace TOUTES les occurrences (flag global)
sed -i 's/ancien/nouveau/g' fichier.txt          # édition SUR PLACE, modifie directement le fichier
sed -n '5,10p' fichier.txt                        # affiche uniquement les lignes 5 à 10
sed '/pattern/d' fichier.txt                        # supprime les lignes correspondant au motif
```

**`sed -i` — précaution indispensable :** cette option modifie le fichier **directement et irréversiblement** (sauf sauvegarde explicite via `sed -i.bak 's/.../.../'`, qui conserve l'original avec l'extension `.bak`) — toujours tester une commande `sed` sans `-i` d'abord pour vérifier le résultat produit avant de l'appliquer réellement sur place, un principe de prudence directement analogue au `--dry-run` de `rsync` déjà rencontré en UE3-M21.

### 7.6 `awk` — traitement structuré par champs

`awk` traite chaque ligne d'entrée comme découpée en **champs** (par défaut, séparés par des espaces/tabulations), accessibles via `$1`, `$2`... (`$0` désignant la ligne entière) — une structure de programme `motif { action }` :

```bash
awk '{ print $1, $3 }' fichier.txt              # affiche les colonnes 1 et 3
awk -F: '{ print $1 }' /etc/passwd                # séparateur de champs personnalisé (":") pour ce fichier
awk '$3 > 100 { print $0 }' fichier.txt             # motif conditionnel : lignes où le 3ème champ dépasse 100
awk '{ somme += $2 } END { print somme }' fichier.txt  # accumulation avec bloc END (exécuté après toutes les lignes)
```

**`awk` vs `grep`/`sed` — quand choisir quoi ?** `grep` pour filtrer des lignes selon un motif ; `sed` pour transformer du texte ligne par ligne (substitutions simples) ; `awk` dès qu'un raisonnement par **champs structurés** (colonnes) ou un **calcul/accumulation** à travers plusieurs lignes est nécessaire — les trois se combinent fréquemment dans un même pipeline, chacun pour la partie du traitement où il excelle.

### 7.7 `cut`, `sort`, `uniq`, `xargs`

```bash
cut -d: -f1 /etc/passwd            # extrait le 1er champ, séparateur ":"
sort fichier.txt                     # tri alphabétique
sort -n fichier.txt                    # tri numérique
sort -k2 -n fichier.txt                  # tri numérique selon la 2ème colonne
uniq fichier_trie.txt                     # supprime les doublons consécutifs (nécessite un tri préalable pour un dédoublonnage complet)
uniq -c fichier_trie.txt                    # compte les occurrences de chaque ligne
sort fichier.txt | uniq -c | sort -rn         # pipeline classique : compter et classer les occurrences les plus fréquentes
xargs -I {} cp {} /destination/                # exécute une commande pour chaque élément reçu en entrée
find . -name "*.log" | xargs rm                  # applique rm à chaque résultat de find
```

**`uniq` nécessite un tri préalable :** contrairement à une intuition naturelle, `uniq` ne déduplique que les lignes **consécutives** identiques — un fichier non trié contenant des doublons non adjacents ne sera pas correctement dédupliqué sans un `sort` préalable dans le pipeline.

**`xargs` et le piège des espaces — lien direct avec le module 23 :** `xargs` découpe par défaut son entrée sur les espaces/retours à la ligne, exactement le même risque de découpage en mots déjà étudié au module 23 pour les noms de fichiers contenant des espaces — la combinaison robuste, déjà utilisée au module 23, est `find ... -print0 | xargs -0 ...` (séparateur nul, insensible aux espaces).

## 8. Architecture

```
┌────────────────────────────────────────────────────────────┐
│  commande1 ──stdout──▶ | ──stdin──▶ commande2 ──stdout──▶ | ──▶ commande3 │
│      │                                    │                        │
│    stderr                              stderr                    stderr    │
│  (terminal, sauf 2>&1)              (terminal, sauf 2>&1)     (terminal)     │
└────────────────────────────────────────────────────────────┘

  Exemple concret : journalctl -u nginx | grep "error" | awk '{print $1,$2}' | sort | uniq -c
```

## 9. Fonctionnement interne

Un pipe est implémenté au niveau noyau comme un tube de communication inter-processus (rappel direct UE1-M5, IPC) — un tampon circulaire en mémoire noyau, avec un descripteur en écriture pour le processus émetteur et un descripteur en lecture pour le processus récepteur ; le noyau gère automatiquement la synchronisation (le lecteur se bloque si le tampon est vide, l'écrivain se bloque si le tampon est plein), sans que ni `grep` ni `awk` n'aient besoin d'implémenter eux-mêmes une logique de synchronisation — exactement le même mécanisme que le pipe anonyme étudié pour l'IPC en UE1-M5, ici simplement exposé et combiné via la syntaxe shell `|`.

## 10. Démonstration

```bash
# Pipeline complet : trouver les 5 adresses IP les plus fréquentes dans un log d'accès web (format simplifié)
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -5

# Extraction et transformation combinées
cat /etc/passwd | grep -v "^#" | cut -d: -f1,3 | sed 's/:/ -> UID /'
```

## 11. Commandes / outils

| Commande | Rôle | Exemple |
|---|---|---|
| `grep -E`/`-i`/`-v`/`-r` | Recherche par motif | `grep -riE "erreur|warning" /var/log/` |
| `sed -i`/`-n`/`s///g` | Édition de flux, substitution | `sed -i 's/old/new/g' fichier` |
| `awk -F` / `{print}` | Traitement par champs | `awk -F, '{print $2}' data.csv` |
| `cut -d -f` | Extraction de champs simple | `cut -d: -f1 /etc/passwd` |
| `sort -n`/`-k`/`-r` | Tri | `sort -k2 -n fichier` |
| `uniq -c` | Comptage de doublons consécutifs | `sort f \| uniq -c` |
| `xargs -0`/`-I` | Application d'une commande à une liste | `find . -print0 \| xargs -0 rm` |

**Erreur fréquente :** oublier que `grep`/`sed` de base utilisent des regex **basiques** (BRE), où `+`, `?`, `|` n'ont pas leur sens étendu sans échappement ou sans l'option `-E` — un motif comme `erreur|avertissement` sans `-E` recherche littéralement la chaîne contenant un caractère `|`, pas une alternation, un piège fréquent pour qui découvre ces outils.

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** un ingénieur doit analyser en urgence un fichier de log de plusieurs gigaoctets pour identifier les adresses IP responsables du plus grand nombre de requêtes en erreur (code HTTP 500) durant un incident, afin de déterminer s'il s'agit d'un pic de trafic légitime ou d'une attaque ciblée.

**Analyse attendue :** ce cas illustre parfaitement la puissance du chaînage d'outils Unix pour un diagnostic rapide, sans avoir besoin d'écrire un script complexe dans un autre langage ni de charger l'intégralité du fichier en mémoire (les pipes traitent le flux progressivement, rappel section 9) :

```bash
grep " 500 " access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -10
```

Ce pipeline filtre d'abord les lignes contenant le code d'erreur recherché (`grep`), extrait l'adresse IP (`awk`, en supposant qu'elle constitue le premier champ du format de log), trie ces adresses pour regrouper les occurrences identiques (`sort`, prérequis de `uniq`), compte les occurrences de chaque adresse (`uniq -c`), puis trie ce résultat par fréquence décroissante (`sort -rn`) et affiche les 10 adresses les plus fréquentes (`head`). Un résultat où une poignée d'adresses concentrent une part disproportionnée des erreurs orienterait vers une attaque ciblée ou un client défaillant ; une distribution large et homogène orienterait plutôt vers un problème applicatif général affectant tous les utilisateurs de façon similaire.

## 15. Sécurité

- Une chaîne de commandes construite dynamiquement à partir d'une entrée externe non maîtrisée (rappel du risque d'injection déjà mentionné au module 23) s'applique également aux pipelines : un motif `grep` ou une expression `sed` construits à partir d'une entrée utilisateur non validée peut se comporter de façon inattendue voire dangereuse si des métacaractères ne sont pas correctement échappés
- `sed -i` sans sauvegarde (`.bak`) sur un fichier de configuration système sensible, exécuté par erreur avec un motif incorrect, peut corrompre irréversiblement une configuration critique — la prudence de test préalable (section 7.5) est autant une question de sécurité opérationnelle que de simple bonne pratique
- L'analyse de logs via ce type de pipeline (étude de cas) est elle-même un outil de sécurité de premier niveau, souvent le point de départ d'une investigation avant de mobiliser des outils plus spécialisés

## 16. Automatisation

Script Bash combinant plusieurs outils du module pour un rapport d'analyse de logs de connexion SSH (rappel direct UE3-M18, cette fois avec les outils de traitement de texte de ce module).

```bash
#!/usr/bin/env bash
# Analyse les tentatives de connexion SSH échouées et identifie les IP les plus actives
set -euo pipefail

LOG="${1:-/var/log/auth.log}"

if [[ ! -r "$LOG" ]]; then
    echo "Erreur: impossible de lire $LOG" >&2
    exit 1
fi

echo "=== Top 10 des IP avec le plus de tentatives échouées ==="
grep "Failed password" "$LOG" | \
    awk '{for(i=1;i<=NF;i++) if ($i=="from") print $(i+1)}' | \
    sort | uniq -c | sort -rn | head -10

echo ""
echo "=== Nombre total de tentatives échouées ==="
grep -c "Failed password" "$LOG" || echo 0
```

## 17. Troubleshooting

Scénario court : un pipeline `commande1 | commande2` ne produit aucune sortie visible, alors que `commande1` exécutée seule affiche bien des résultats à l'écran. Démarche : vérifier si `commande1` écrit son résultat sur **stderr** plutôt que stdout (rappel section 7.3 : un pipe ne transmet que stdout par défaut) — de nombreux outils écrivent certains messages sur stderr par convention (messages de progression, avertissements), invisibles pour `commande2` qui n'attend que sur stdin/stdout → corriger si nécessaire en ajoutant `2>&1` avant le pipe si l'intention est bien de transmettre également ce flux ; vérifier également que `commande2` lit effectivement depuis stdin et ne s'attend pas à un argument de fichier explicite à la place.

## 18. Synthèse

- Les redirections manipulent stdout (`>`), stderr (`2>`) et l'entrée (`<`) indépendamment ; l'ordre dans `2>&1` est significatif
- Un pipe ne connecte que stdout entre deux commandes, jamais stderr par défaut
- `grep` filtre par motif (regex basique par défaut, `-E` pour les regex étendues) ; `sed` transforme (avec `-i` à utiliser prudemment, toujours testé sans `-i` d'abord) ; `awk` structure par champs et permet accumulation/calcul
- `uniq` ne déduplique que les lignes consécutives — toujours précédé d'un `sort` pour un dédoublonnage complet
- `xargs` hérite du même risque de découpage en mots que les boucles Bash (module 23) — `find -print0 | xargs -0` pour la robustesse
- La combinaison de ces outils simples en pipeline permet des traitements de données puissants sans script complexe, en traitant le flux progressivement sans charger l'intégralité en mémoire

## 19. Questions d'évaluation

**QCM**
1. `commande > fichier 2>&1` redirige :
 a) uniquement stdout b) stdout et stderr vers le même fichier c) uniquement stderr d) l'entrée standard

2. `uniq` sans tri préalable :
 a) déduplique correctement toutes les occurrences b) ne déduplique que les lignes consécutives identiques c) trie automatiquement le fichier d) provoque une erreur

**Questions ouvertes**
- Expliquez pourquoi l'ordre compte dans `commande > fichier 2>&1` par rapport à `commande 2>&1 > fichier`.
- Expliquez pourquoi `sed -i` doit être utilisé avec prudence, et la pratique recommandée avant de l'appliquer réellement.

**Exercice technique**
- Écrivez un pipeline affichant les 5 lignes les plus fréquentes d'un fichier `donnees.txt`.

**Problème de diagnostic**
- Un pipeline `commande1 | commande2` ne produit aucune sortie alors que `commande1` seule fonctionne. Proposez la cause la plus probable.

## 20. Mini-projet

Développer un script (Bash) qui :
1. Analyse un fichier de log au format personnalisable (séparateur de champs configurable) et produit un résumé statistique par colonne choisie (top N des valeurs les plus fréquentes)
2. Combine `grep`, `awk`, `sort`, `uniq` dans un pipeline efficace, sans recourir à une boucle Bash explicite là où un pipeline suffit
3. Gère les erreurs (fichier absent, colonne invalide) selon les bonnes pratiques du module 23
4. Propose une option de filtrage temporel si le format de log contient un horodatage exploitable
5. Documente chaque pipeline utilisé avec un commentaire expliquant le rôle de chaque étape de la chaîne

---

# PARTIE II — TD

### Niveau 1 — Compréhension

**Exercice 1.1**
Expliquez la différence entre `>` et `>>`, et entre `2>` et `2>&1`.

**Exercice 1.2**
Expliquez pourquoi `uniq` doit généralement être précédé d'un `sort` pour dédupliquer correctement un fichier.

**Exercice 1.3**
Expliquez la différence de rôle entre `grep`, `sed` et `awk`, en une phrase pour chacun.

### Niveau 2 — Application

**Exercice 2.1**
Écrivez la commande `grep` permettant de compter le nombre de lignes contenant le mot "erreur" (insensible à la casse) dans un fichier `app.log`.

**Exercice 2.2**
Écrivez la commande `sed` permettant de remplacer, dans tout le fichier `config.txt` et directement sur place, toutes les occurrences de "http://" par "https://", avec une sauvegarde de l'original en `.bak`.

**Exercice 2.3**
Écrivez la commande `awk` permettant d'afficher la 2ème colonne d'un fichier `data.csv` séparé par des virgules.

### Niveau 3 — Analyse

**Exercice 3.1**
Un ingénieur exécute `grep "erreur|avertissement" fichier.log` (sans l'option `-E`) et n'obtient aucun résultat, alors que le fichier contient manifestement des lignes avec le mot "erreur". Expliquez précisément pourquoi cette commande échoue, et corrigez-la de deux façons différentes.

**Exercice 3.2**
Un script utilise `find . -name "*.tmp" | xargs rm` pour nettoyer des fichiers temporaires. Ce script fonctionne sur un répertoire de test mais provoque des erreurs inattendues sur un répertoire contenant des fichiers aux noms comportant des espaces. Expliquez le lien avec le module 23, et proposez la commande corrigée.

### Niveau 4 — Problème complexe

**Exercice 4.1**
Un ingénieur doit analyser un fichier de log d'accès web au format `IP - - [date] "requête" code_statut taille`, et identifier, pour chaque code de statut HTTP en erreur (4xx et 5xx), le nombre total d'occurrences, trié par fréquence décroissante. Construisez, étape par étape en expliquant le rôle de chaque commande, un pipeline complet accomplissant cette tâche, en supposant que le code de statut est le 9ème champ séparé par des espaces (en tenant compte du fait que le champ "requête" entre guillemets peut lui-même contenir des espaces, ce qui doit être pris en compte dans votre raisonnement sur le découpage en champs).

---

## Correction du TD

**1.1** — `>` redirige stdout vers un fichier en **écrasant** son contenu existant ; `>>` redirige stdout vers un fichier en **ajoutant** à la suite du contenu existant, sans l'écraser. `2>` redirige uniquement stderr vers une destination (fichier ou autre) ; `2>&1` fait pointer stderr vers la **même destination que stdout à cet instant précis** du traitement de la ligne de commande, un mécanisme différent d'une simple redirection vers un fichier nommé.

**1.2** — `uniq` ne compare que des lignes **consécutives** pour détecter des doublons — il ne maintient aucune mémoire des lignes déjà vues plus tôt dans le fichier au-delà de la ligne immédiatement précédente. Un fichier non trié peut contenir des doublons non adjacents (la même ligne apparaissant à des positions éloignées), qui ne seraient donc pas détectés par `uniq` seul ; un tri préalable (`sort`) regroupe toutes les occurrences identiques de façon consécutive, rendant la déduplication par `uniq` réellement complète.

**1.3** — `grep` : filtre les lignes d'un texte selon un motif (regex), affichant celles qui correspondent (ou ne correspondent pas, avec `-v`). `sed` : transforme un flux de texte ligne par ligne, typiquement via des substitutions, sans nécessairement filtrer sur un motif de correspondance de ligne entière. `awk` : traite un texte structuré en champs (colonnes), permettant des motifs conditionnels sophistiqués et des calculs/accumulations à travers plusieurs lignes.

**2.1** — `grep -ci "erreur" app.log`.

**2.2** — `sed -i.bak 's/http:\/\//https:\/\//g' config.txt` (les barres obliques du motif et du remplacement doivent être échappées avec un antislash puisqu'elles coïncident avec le délimiteur `/` utilisé par `sed`, sauf à choisir un délimiteur différent comme `sed -i.bak 's#http://#https://#g' config.txt`, tout aussi valide et plus lisible dans ce cas précis).

**2.3** — `awk -F, '{print $2}' data.csv`.

**3.1** — Sans l'option `-E`, `grep` utilise les expressions régulières **basiques** (BRE), où le caractère `|` n'a **pas** sa signification étendue d'alternation — il est interprété littéralement comme le caractère pipe lui-même. La commande `grep "erreur|avertissement" fichier.log` recherche donc, sans `-E`, la chaîne littérale "erreur|avertissement" (avec un caractère `|` au milieu), ce qui n'existe manifestement pas dans le fichier, d'où l'absence de résultat malgré la présence du mot "erreur" seul. Deux corrections possibles : (1) ajouter l'option `-E` pour activer les regex étendues : `grep -E "erreur|avertissement" fichier.log` ; (2) échapper le caractère `|` pour lui redonner son sens spécial même en regex basique (comportement supporté par certaines implémentations de `grep`, notamment GNU grep) : `grep "erreur\|avertissement" fichier.log`.

**3.2** — Ce problème est directement lié au piège du module 23 concernant le découpage en mots et à son extension à `xargs` (rappel section 7.7 de ce module) : `xargs`, comme une boucle `for` non protégée, découpe par défaut son entrée sur les espaces et retours à la ligne — un nom de fichier contenant un espace, transmis par `find` via un pipe classique, est donc fragmenté en plusieurs "arguments" distincts pour `rm`, provoquant des erreurs ("fichier introuvable" pour les fragments qui ne correspondent à aucun fichier réel) voire, dans des cas plus dangereux, la suppression accidentelle d'un fichier différent si un fragment correspond par coïncidence à un autre nom existant. Commande corrigée, utilisant un séparateur nul pour une robustesse totale : `find . -name "*.tmp" -print0 | xargs -0 rm`.

**4.1** — Construction du pipeline étape par étape :

1. **Extraction du champ de statut :** puisque le champ "requête" (contenant potentiellement des espaces à l'intérieur des guillemets) fausserait un découpage naïf par position de champ avec `awk` par défaut (séparateur espace simple), une approche plus robuste consiste à s'appuyer sur les guillemets eux-mêmes comme repère : après la fermeture du dernier guillemet de la requête, le code de statut est le premier champ qui suit. Une méthode pratique : utiliser `awk` avec le guillemet `"` comme séparateur de champs pour isoler la portion après la requête, puis extraire le premier champ numérique de cette portion :
```bash
awk -F'"' '{print $3}' access.log | awk '{print $1}'
```
Ici, le premier `awk -F'"'` découpe chaque ligne sur les guillemets ; le troisième champ résultant (`$3`) correspond à tout ce qui suit la requête entre guillemets (typiquement " 200 1234" avec un espace initial) ; le second `awk` (séparateur par défaut, espace) en extrait alors le premier élément, le code de statut lui-même.

2. **Filtrage des codes d'erreur (4xx et 5xx) :** ajouter un filtre sur le motif du code obtenu :
```bash
awk -F'"' '{print $3}' access.log | awk '{print $1}' | grep -E "^[45][0-9]{2}$"
```
Le motif `^[45][0-9]{2}$` (regex étendue) correspond à un code à 3 chiffres commençant par 4 ou 5.

3. **Comptage et tri par fréquence décroissante :**
```bash
awk -F'"' '{print $3}' access.log | awk '{print $1}' | grep -E "^[45][0-9]{2}$" | sort | uniq -c | sort -rn
```

Le pipeline complet : le premier `awk` isole la portion de ligne après la requête entre guillemets (contournant le problème des espaces internes à la requête) ; le second `awk` en extrait le code de statut ; `grep -E` filtre uniquement les codes 4xx/5xx ; `sort` regroupe les occurrences identiques de façon consécutive (prérequis pour `uniq`, rappel exercice 1.2) ; `uniq -c` compte chaque code distinct ; `sort -rn` trie enfin le résultat par fréquence numérique décroissante, répondant complètement à la demande initiale.

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur système junior pour analyser des fichiers de logs réalistes en construisant des pipelines de traitement de texte, et pour diagnostiquer un cas classique de redirection mal ordonnée.

## Architecture

Une machine Linux avec `grep`, `sed`, `awk`, `sort`, `uniq`, `xargs` disponibles (présents nativement sur toute distribution).

## Objectifs
- Construire des pipelines combinant plusieurs outils pour une analyse de log réaliste
- Manipuler les redirections stdout/stderr en observant précisément leur comportement
- Utiliser `sed -i` avec la prudence appropriée
- Diagnostiquer un cas de redirection mal ordonnée

## Préparation

```bash
which grep sed awk sort uniq xargs 2>/dev/null
```

## Étapes

### Étape 1 — Générer un log de test et construire un pipeline d'analyse

```bash
cat << 'EOF' > /tmp/access_test.log
192.168.1.10 - - [10/Jan/2024] "GET /api" 200 512
192.168.1.11 - - [10/Jan/2024] "GET /api/data avec espace" 404 128
192.168.1.10 - - [10/Jan/2024] "POST /api" 500 64
192.168.1.12 - - [10/Jan/2024] "GET /home" 200 2048
192.168.1.10 - - [10/Jan/2024] "GET /api" 500 64
192.168.1.11 - - [10/Jan/2024] "GET /api" 404 128
EOF

# Reprendre le pipeline complet construit à l'exercice 4.1 du TD
awk -F'"' '{print $3}' /tmp/access_test.log | awk '{print $1}' | grep -E "^[45][0-9]{2}$" | sort | uniq -c | sort -rn
```
*Question :* le résultat obtenu est-il cohérent avec un comptage manuel des codes 404/500 dans le fichier de test ? Vérifiez ligne par ligne.

### Étape 2 — Redirections stdout/stderr

```bash
# Une commande qui produit à la fois du stdout et du stderr
ls /tmp/access_test.log /tmp/fichier_inexistant.log

echo "=== stdout uniquement vers fichier ==="
ls /tmp/access_test.log /tmp/fichier_inexistant.log > /tmp/sortie.log
cat /tmp/sortie.log
echo "(l'erreur reste visible à l'écran, pas dans le fichier)"

echo "=== stdout et stderr ensemble (ordre correct) ==="
ls /tmp/access_test.log /tmp/fichier_inexistant.log > /tmp/tout.log 2>&1
cat /tmp/tout.log

echo "=== ordre incorrect (à titre de démonstration du piège) ==="
ls /tmp/access_test.log /tmp/fichier_inexistant.log 2>&1 > /tmp/tout_incorrect.log
cat /tmp/tout_incorrect.log
echo "(l'erreur devrait à nouveau être visible à l'écran, pas dans ce fichier)"
```
*Question :* confirmez la différence entre les deux dernières commandes. Expliquez précisément, en vous appuyant sur la section 7.2 du cours, pourquoi l'ordre inverse ne capture pas stderr dans le fichier.

### Étape 3 — `sed` avec prudence

```bash
cp /tmp/access_test.log /tmp/access_test_copie.log

# Toujours tester sans -i d'abord
sed 's/200/000/g' /tmp/access_test_copie.log

# Une fois le résultat validé visuellement, appliquer avec sauvegarde
sed -i.bak 's/200/000/g' /tmp/access_test_copie.log
diff /tmp/access_test_copie.log.bak /tmp/access_test_copie.log
```

### Étape 4 — `xargs` robuste face aux espaces

```bash
mkdir -p /tmp/lab_xargs
touch "/tmp/lab_xargs/fichier normal.tmp" "/tmp/lab_xargs/autre.tmp"

echo "=== Version fautive ==="
find /tmp/lab_xargs -name "*.tmp" | xargs -I {} echo "Trouvé: {}"

echo "=== Version robuste ==="
find /tmp/lab_xargs -name "*.tmp" -print0 | xargs -0 -I {} echo "Trouvé: {}"
```
*Question :* la version fautive traite-t-elle correctement "fichier normal.tmp" ? Comparez avec la version robuste.

## Commandes de référence
`grep -E`, `sed -i.bak`, `awk -F`, `sort`, `uniq -c`, `xargs -0`

## Résultats attendus
- Résultat du pipeline d'analyse de l'étape 1, avec vérification manuelle
- Comparaison des trois cas de redirection de l'étape 2
- Preuve de la modification `sed` avec sauvegarde `.bak` (étape 3)
- Comparaison version fautive/robuste de `xargs` (étape 4)

## Questions

1. Dans l'étape 1, pourquoi le pipeline utilise-t-il deux appels successifs à `awk` plutôt qu'un seul plus complexe ? Cette approche en plusieurs étapes simples est-elle une bonne pratique ou un signe de pipeline mal conçu ?
2. Si vous deviez analyser un fichier de log de plusieurs gigaoctets, le pipeline de l'étape 1 nécessiterait-il de charger l'intégralité du fichier en mémoire avant de produire un résultat ? Justifiez en vous appuyant sur la section 9 du cours (fonctionnement interne des pipes).
3. Dans l'étape 4, pourquoi la commande `xargs -I {}` nécessite-t-elle explicitement l'option `-0` en complément de `find -print0`, plutôt que de fonctionner automatiquement avec le séparateur nul dès que celui-ci est présent dans l'entrée ?

## Troubleshooting (panne à diagnostiquer)

**Situation fournie :** un étudiant tente de rediriger à la fois stdout et stderr d'un script vers un fichier de log unique, mais après avoir écrit `mon_script.sh 2>&1 > log_complet.txt`, il constate que seules les erreurs apparaissent encore à l'écran, alors que la sortie normale se retrouve bien dans le fichier.

**Démarche attendue :**
1. Symptôme : redirection partiellement fonctionnelle, stderr toujours visible à l'écran contrairement à l'intention
2. Hypothèse : ordre incorrect des redirections dans la commande, exactement le piège vu en cours (section 7.2) et reproduit à l'étape 2 du TP — `2>&1` a été placé **avant** `> log_complet.txt`, ce qui fait pointer stderr vers la destination de stdout **au moment où cette ligne est évaluée** (le terminal, valeur par défaut à cet instant), puis seul stdout est ensuite redirigé vers le fichier, laissant stderr continuer vers le terminal
3. Vérification : relire la commande exacte utilisée par l'étudiant et confirmer l'ordre des redirections
4. Isolation : reproduire volontairement le même ordre sur une commande de test simple générant à la fois stdout et stderr, confirmer que le comportement observé correspond bien à l'hypothèse
5. Correction : inverser l'ordre des redirections : `mon_script.sh > log_complet.txt 2>&1` (rediriger stdout en premier, puis faire pointer stderr vers cette même destination déjà établie)
6. Vérification : relancer le script avec l'ordre corrigé, confirmer que stdout et stderr apparaissent désormais tous deux dans le fichier, sans rien à l'écran
7. Documentation : noter cette règle d'ordre comme un point de vigilance particulièrement fréquent chez les débutants en scripting Bash, à vérifier systématiquement chaque fois qu'une redirection combinée de stdout et stderr est nécessaire

## Validation

Le TP est validé si l'étudiant fournit :
- Les 4 résultats/comparaisons demandés
- Des réponses argumentées aux 3 questions
- Une identification correcte de la cause du scénario de troubleshooting (ordre des redirections), avec la correction exacte

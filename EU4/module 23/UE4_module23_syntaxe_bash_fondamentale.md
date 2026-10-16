# UE4 — Module 23 : Syntaxe Bash Fondamentale

*Ce module formalise les règles précises du langage Bash déjà utilisé de façon intensive et pratique dans les UE1, UE2 et UE3 — il ne part pas de zéro, il consolide et rend explicite ce qui a été appliqué intuitivement jusqu'ici.*

---

# PARTIE I — COURS

## 1. Titre du cours
Bash en rigueur : variables, quotation, conditions, boucles et fonctions

## 2. Niveau
Élève ingénieur — cycle ingénieur, entrée en UE4 automatisation (S2)

## 3. Durée
CM : 2h (module dense mais volontairement court en CM — +2h TD, +5h TP en Parties II et III, l'essentiel de l'apprentissage se faisant par la pratique)

## 4. Prérequis
- L'ensemble des scripts Bash déjà écrits et utilisés dans les UE1, UE2 et UE3 (aucun prérequis théorique nouveau, mais une familiarité pratique déjà acquise)

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Expliquer précisément les règles de quotation Bash (guillemets simples, doubles, absence de guillemets) et leurs conséquences
- Utiliser correctement les variables, y compris les variables spéciales (`$?`, `$#`, `$@`, `$0`...) et les tableaux
- Écrire des conditions robustes avec `test`/`[ ]`/`[[ ]]`, en connaissant leurs différences
- Écrire des boucles `for`/`while`/`until` en évitant les pièges classiques (notamment la lecture ligne par ligne)
- Définir des fonctions avec des variables locales et une gestion de retour appropriée
- Gérer arguments de script et codes de sortie de façon rigoureuse

## 6. Motivation

Vous avez déjà écrit et utilisé des dizaines de scripts Bash au fil de l'UE1, l'UE2 et l'UE3 — des scripts qui fonctionnaient, parfois avec quelques ajustements empiriques. Ce module prend du recul : *pourquoi* certaines constructions fonctionnent-elles alors que d'autres, en apparence similaires, échouent silencieusement ou de façon inattendue ? La réponse tient presque toujours aux règles de quotation et d'expansion de Bash — un sujet rarement enseigné rigoureusement mais responsable d'une part disproportionnée des bugs de scripts en production, y compris chez des administrateurs système expérimentés.

Ce module ne réapprend pas Bash depuis zéro — il comble les angles morts qui, laissés inexplorés, produisent des scripts fragiles ressemblant à ceux déjà écrits mais cassant sur des cas limites (un nom de fichier contenant un espace, une variable vide, un argument absent).

## 7. Concepts fondamentaux

### 7.1 Variables — déclaration et expansion

```bash
nom="Alice"
echo "Bonjour $nom"        # expansion simple
echo "Bonjour ${nom}!"      # accolades : délimitent clairement la variable (indispensable si collée à du texte)
```

Bash ne type pas ses variables — tout est fondamentalement une chaîne de caractères, y compris ce qui ressemble à un nombre (les opérations arithmétiques utilisent un contexte spécial, voir 7.5).

### 7.2 Quotation — la règle la plus importante de ce module

**Sans guillemets :** Bash effectue le **découpage en mots** (word splitting, sur les espaces/tabulations) et l'**expansion des globs** (`*`, `?`) sur le résultat de l'expansion de variable.

```bash
fichier="mon document.txt"
ls $fichier      # ÉCHOUE : interprété comme deux arguments "mon" et "document.txt"
ls "$fichier"     # fonctionne : un seul argument "mon document.txt"
```

**Guillemets doubles `"..."` :** empêchent le découpage en mots et l'expansion de glob, mais autorisent toujours l'**expansion de variables** (`$var`) et de **commandes** (`` $(commande) ``).

**Guillemets simples `'...'` :** empêchent **toute** expansion — le contenu est traité littéralement, y compris les variables.

```bash
echo "Utilisateur: $USER"     # affiche la valeur de $USER
echo 'Utilisateur: $USER'      # affiche littéralement "Utilisateur: $USER"
```

**Règle pratique à retenir et appliquer systématiquement :** toujours entourer une variable de guillemets doubles (`"$variable"`) sauf besoin explicite et conscient du découpage en mots (rarissime en pratique fiable) — cette seule règle élimine une proportion très importante des bugs Bash rencontrés en production, y compris dans plusieurs scripts déjà écrits en UE1/UE2/UE3 où cette rigueur n'avait pas toujours été appliquée explicitement.

### 7.3 Variables spéciales

| Variable | Signification |
|---|---|
| `$0` | Nom du script lui-même |
| `$1`, `$2`... | Arguments positionnels du script (ou d'une fonction) |
| `$#` | Nombre d'arguments passés |
| `$@` | Tous les arguments, chacun traité individuellement (avec guillemets : `"$@"`) |
| `$*` | Tous les arguments, concaténés en une seule chaîne |
| `$?` | Code de sortie de la **dernière** commande exécutée |
| `$$` | PID du script en cours d'exécution |

**`"$@"` vs `"$*"` — une distinction cruciale :** `"$@"` préserve chaque argument comme un mot séparé même s'il contient des espaces (comportement presque toujours souhaité), tandis que `"$*"` les fusionne en une seule chaîne selon le séparateur `IFS` — pour transmettre fidèlement une liste d'arguments à une autre commande, `"$@"` est le choix quasi systématiquement correct.

### 7.4 Tests et conditions

```bash
if [ "$age" -ge 18 ]; then
    echo "Majeur"
fi

if [[ "$nom" == "Alice" && "$age" -gt 20 ]]; then
    echo "Condition composée"
fi
```

**`[ ]` (test POSIX) vs `[[ ]]` (extension Bash) :** `[[ ]]` est généralement préférable en Bash pur (pas nécessairement portable vers `sh` strict) car il évite plusieurs pièges de `[ ]` — notamment il n'effectue pas de découpage en mots sur les variables non quotées à l'intérieur, et supporte `&&`/`||` directement sans échappement supplémentaire, et le pattern matching (`==` avec des globs).

**Opérateurs de comparaison numérique (distincts des opérateurs de chaîne) :**

| Test numérique | Test de chaîne | Signification |
|---|---|---|
| `-eq` | `==` ou `=` | égal |
| `-ne` | `!=` | différent |
| `-lt` | (pas d'équivalent direct, comparaison lexicographique avec `<` dans `[[ ]]`) | inférieur |
| `-gt` | `>` (dans `[[ ]]`) | supérieur |
| `-ge` / `-le` | — | supérieur/inférieur ou égal |

**Erreur fréquente :** utiliser `==` pour comparer des nombres, ou `-eq` pour comparer des chaînes — Bash ne signale pas toujours cette confusion explicitement, produisant un résultat silencieusement incorrect plutôt qu'une erreur immédiatement visible.

**Tests de fichiers (déjà croisés dans plusieurs scripts des UE précédentes) :**
```bash
[ -f "$fichier" ]     # est un fichier régulier
[ -d "$repertoire" ]   # est un répertoire
[ -x "$script" ]        # est exécutable
[ -z "$variable" ]       # chaîne vide
[ -n "$variable" ]        # chaîne non vide
```

### 7.5 Arithmétique

```bash
resultat=$((5 + 3))
echo "$resultat"

compteur=0
((compteur++))          # syntaxe arithmétique dédiée, plus lisible que compteur=$((compteur+1))
```

Le contexte arithmétique `$(( ))`/`(( ))` traite le contenu comme une expression numérique, sans nécessiter le `$` devant les noms de variables à l'intérieur (bien que cela reste toléré).

### 7.6 Boucles — et leur piège le plus fréquent

```bash
for i in 1 2 3; do
    echo "Itération $i"
done

for fichier in *.txt; do
    echo "Fichier: $fichier"
done

while [ "$compteur" -lt 10 ]; do
    ((compteur++))
done
```

**Le piège classique de la lecture ligne par ligne :**
```bash
# INCORRECT en présence de noms de fichiers avec espaces, et modifie le comportement de $IFS
for ligne in $(cat fichier.txt); do
    echo "$ligne"     # découpe sur les espaces, pas sur les lignes !
done

# CORRECT
while IFS= read -r ligne; do
    echo "$ligne"
done < fichier.txt
```

**Pourquoi la version `while read` est-elle correcte et l'autre non ?** `for ligne in $(cat fichier.txt)` effectue d'abord une substitution de commande (sans guillemets, donc soumise au découpage en mots vu en 7.2) sur l'**intégralité** du contenu du fichier, puis boucle sur chaque **mot** ainsi obtenu, pas sur chaque ligne — un fichier contenant des espaces au sein d'une même ligne logique serait incorrectement scindé. `while IFS= read -r ligne` lit le flux d'entrée ligne par ligne nativement, `IFS=` empêchant le découpage des espaces en début/fin de ligne, et `-r` empêchant l'interprétation des antislashs comme caractères d'échappement — la combinaison correcte et robuste pour tout traitement ligne par ligne.

### 7.7 Fonctions

```bash
verifier_service() {
    local nom_service="$1"
    if systemctl is-active --quiet "$nom_service"; then
        return 0
    else
        return 1
    fi
}

if verifier_service "sshd"; then
    echo "sshd est actif"
fi
```

**`local` — une discipline essentielle :** sans `local`, une variable définie à l'intérieur d'une fonction est **globale** par défaut en Bash, un comportement source de bugs difficiles à tracer dans des scripts de taille significative (une fonction modifiant accidentellement une variable du script principal portant le même nom). Toujours déclarer `local` pour toute variable interne à une fonction qui n'a pas vocation à affecter le reste du script.

**Retourner une valeur depuis une fonction Bash :** contrairement à de nombreux langages, une fonction Bash ne "retourne" pas directement une valeur au sens d'un langage typé — `return` définit uniquement un **code de sortie** (0-255, convention Unix : 0 = succès), consultable via `$?`. Pour renvoyer une donnée (une chaîne, un nombre), la convention est d'utiliser `echo` en sortie standard, capturée par l'appelant via une substitution de commande :

```bash
obtenir_date_formattee() {
    echo "$(date +%Y-%m-%d)"
}

date_du_jour=$(obtenir_date_formattee)
```

### 7.8 Codes de sortie

Rappel direct des conventions déjà rencontrées dans de nombreux scripts des UE précédentes : `0` signifie succès, toute valeur non nulle (1-255) signifie un échec, dont le code précis peut porter une signification conventionnelle propre à chaque commande. `exit N` termine un script avec ce code ; `$?` consulte le code de sortie de la dernière commande exécutée — cette valeur est **écrasée** à chaque nouvelle commande, elle doit donc être consultée ou stockée immédiatement après la commande concernée.

```bash
grep "erreur" fichier.log
if [ $? -eq 0 ]; then
    echo "Erreur trouvée"
fi

# équivalent plus idiomatique, sans dépendre explicitement de $?
if grep -q "erreur" fichier.log; then
    echo "Erreur trouvée"
fi
```

## 8. Architecture

```
┌──────────────────────────────────────────────────────────┐
│  Script Bash                                                  │
│                                                                 │
│  $0, $1..$9, $@, $#  ──▶  arguments du script                    │
│                                                                 │
│  Variables (non typées, expansion selon quotation)                │
│    "$var"  → expansion protégée (recommandé par défaut)             │
│    $var    → soumis au word splitting + glob (rarement souhaité)    │
│    'texte' → aucune expansion                                       │
│                                                                 │
│  Fonctions ──▶ local pour l'isolation, echo pour retourner une donnée │
│           ──▶ return/$? pour le code de sortie uniquement            │
└──────────────────────────────────────────────────────────┘
```

## 9. Fonctionnement interne

Bash traite chaque ligne de commande en plusieurs phases successives (expansion des accolades, expansion du tilde, expansion de paramètres/variables, substitution de commandes, expansion arithmétique, découpage en mots, expansion de chemin/glob, suppression des quotes) — comprendre cet ordre explique pourquoi le guillemetage change fondamentalement le résultat : les guillemets doubles inhibent spécifiquement les phases de découpage en mots et d'expansion de glob, tout en laissant intactes les phases d'expansion de variable et de substitution de commande, qui interviennent **avant** dans l'ordre de traitement.

## 10. Démonstration

```bash
# Démonstration de l'importance de la quotation
fichiers=("rapport final.txt" "notes.txt")
for f in "${fichiers[@]}"; do
    echo "Fichier: $f"
done
# Comparer avec la version sans guillemets sur l'expansion du tableau (à éviter)

# Démonstration $@ vs $*
demo_args() {
    echo "Avec \"\$@\":"
    for a in "$@"; do echo "  [$a]"; done
    echo "Avec \"\$*\":"
    for a in "$*"; do echo "  [$a]"; done
}
demo_args "premier argument" "second"
```

## 11. Commandes / outils

| Élément | Rôle | Exemple |
|---|---|---|
| `"$var"` | Expansion protégée (à utiliser par défaut) | `echo "$nom"` |
| `[[ ]]` | Test Bash étendu, préféré à `[ ]` en Bash pur | `[[ "$a" == "$b" ]]` |
| `local` | Variable locale à une fonction | `local x="valeur"` |
| `while IFS= read -r` | Lecture ligne par ligne robuste | `while IFS= read -r l; do ...; done < fichier` |
| `shellcheck` | Analyseur statique détectant les erreurs de quotation/syntaxe courantes | `shellcheck monscript.sh` |

**Erreur fréquente :** oublier `-r` dans `read`, provoquant une interprétation indésirable des antislashs présents dans les lignes lues (par exemple des chemins Windows ou des expressions régulières contenant des antislashs, silencieusement altérés).

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** un script de nettoyage automatique, écrit rapidement et fonctionnant correctement en test, est déployé en production. Quelques semaines plus tard, il supprime accidentellement des fichiers qui n'auraient jamais dû être concernés, sur un serveur où certains noms de fichiers contiennent des espaces (des exports générés par une application tierce).

**Analyse attendue :** ce scénario est la conséquence directe et classique d'un oubli de quotation (section 7.2) : une ligne du type `for fichier in $(find /data -name "*.tmp"); do rm $fichier; done`, fonctionnant parfaitement tant qu'aucun nom de fichier ne contient d'espace, se comporte de façon totalement différente et dangereuse dès qu'un nom de fichier en contient un — le découpage en mots fragmente un unique nom de fichier en plusieurs "arguments" pour `rm`, potentiellement en faisant correspondre des fragments à d'autres fichiers existants par coïncidence de nommage. Correction : réécrire systématiquement avec des guillemets doubles autour de chaque variable, et préférer `find ... -print0 | while IFS= read -r -d '' fichier` (utilisant un séparateur nul, totalement insensible à la présence d'espaces ou même de retours à la ligne dans les noms de fichiers) pour un traitement de liste de fichiers réellement robuste. Mesure préventive généralisable : tout script destiné à la production, en particulier tout script effectuant des suppressions, devrait être passé au travers d'un analyseur statique comme `shellcheck` avant déploiement, capable de détecter automatiquement ce type d'oubli de quotation.

## 15. Sécurité

- Un script mal quotifié peut être vulnérable à une forme d'**injection de commande** si une variable dont le contenu n'est pas maîtrisé (entrée utilisateur, contenu d'un fichier externe) est interpolée sans quotation dans une commande construite dynamiquement — un risque directement analogue en esprit à l'injection SQL, transposé au shell
- La règle du moindre privilège s'applique également aux scripts eux-mêmes : un script exécuté avec des privilèges élevés (root, ou via un binaire SUID comme vu en UE3-M14) doit être audité avec une rigueur accrue sur sa gestion des entrées et sa quotation, une faille de script mal quotifié pouvant devenir un vecteur d'escalade de privilèges
- `shellcheck` détecte également plusieurs classes de vulnérabilités potentielles liées à une mauvaise quotation, au-delà des simples bugs fonctionnels — son usage systématique est autant une pratique de qualité que de sécurité

## 16. Automatisation

*Ce module étant lui-même consacré aux fondations du langage d'automatisation, la section suivante illustre une bonne pratique de structuration plutôt qu'un outil de supervision classique.*

```bash
#!/usr/bin/env bash
# Squelette de script robuste, illustrant les bonnes pratiques du module
set -euo pipefail   # anticipation directe du module 25 (gestion des erreurs)

verifier_argument() {
    local valeur="$1"
    if [[ -z "$valeur" ]]; then
        echo "Erreur: argument requis manquant" >&2
        return 1
    fi
    return 0
}

traiter_fichier() {
    local chemin="$1"
    if [[ ! -f "$chemin" ]]; then
        echo "Erreur: fichier introuvable: $chemin" >&2
        return 1
    fi
    while IFS= read -r ligne; do
        echo "Traitement: $ligne"
    done < "$chemin"
}

main() {
    if [[ $# -eq 0 ]]; then
        echo "Usage: $0 <fichier>" >&2
        exit 1
    fi
    verifier_argument "$1" || exit 1
    traiter_fichier "$1"
}

main "$@"
```

## 17. Troubleshooting

Scénario court : un script qui fonctionne parfaitement lorsqu'il est testé avec des noms de fichiers simples échoue de façon incohérente dès qu'il traite un répertoire contenant un fichier avec un espace dans son nom. Démarche : identifier chaque endroit du script où une variable représentant un nom de fichier est utilisée sans guillemets doubles (rappel section 7.2) → corriger systématiquement en ajoutant les guillemets → si le script itère sur une liste de fichiers obtenue par une commande externe, vérifier également que cette itération elle-même est robuste (préférer `find -print0`/`read -d ''` à une simple boucle `for` sur une substitution de commande non protégée) → valider avec `shellcheck` pour détecter d'éventuelles occurrences similaires non encore repérées manuellement dans le reste du script.

## 18. Synthèse

- La règle de quotation la plus importante : toujours `"$variable"`, jamais `$variable` sans guillemets, sauf besoin explicite et conscient
- `"$@"` (arguments préservés individuellement) est presque toujours préférable à `"$*"` (concaténation)
- `[[ ]]` est généralement préférable à `[ ]` en Bash pur ; les opérateurs numériques (`-eq`, `-lt`...) et de chaîne (`==`, `!=`) ne sont pas interchangeables
- La lecture ligne par ligne robuste utilise `while IFS= read -r ligne`, jamais une boucle `for` sur une substitution de commande non protégée
- `local` dans les fonctions évite la pollution de l'espace de noms global, une source de bugs difficiles à tracer
- Une fonction Bash "retourne" un code de sortie (0-255) via `return`/`$?` ; pour renvoyer une donnée, la convention est `echo` capturé par substitution de commande
- `shellcheck` est un outil de qualité et de sécurité à utiliser systématiquement avant tout déploiement de script en production

## 19. Questions d'évaluation

**QCM**
1. La commande `echo $nom` (sans guillemets), si `$nom` contient un espace, produit :
 a) toujours le même résultat qu'avec des guillemets b) un découpage en mots pouvant altérer le résultat attendu c) une erreur de syntaxe systématique d) l'affichage littéral de "$nom"

2. `"$@"`, comparé à `"$*"`, préserve :
 a) rien de particulier, ils sont identiques b) chaque argument individuellement, même s'il contient des espaces c) uniquement le premier argument d) uniquement le nombre d'arguments

**Questions ouvertes**
- Expliquez pourquoi `for ligne in $(cat fichier.txt)` est une méthode incorrecte pour traiter un fichier ligne par ligne, et donnez la méthode correcte.
- Expliquez pourquoi `local` est important dans une fonction Bash, avec un exemple de bug que son absence pourrait provoquer.

**Exercice technique**
- Réécrivez la ligne suivante en corrigeant le problème de quotation : `rm $fichier_a_supprimer`.

**Problème de diagnostic**
- Un script de suppression automatique supprime parfois des fichiers inattendus sur un système où certains noms contiennent des espaces. Proposez la cause la plus probable et la correction appropriée.

## 20. Mini-projet

Réviser un script déjà écrit dans une UE précédente (au choix : un script de l'UE1, l'UE2 ou l'UE3) en appliquant systématiquement les bonnes pratiques de ce module :
1. Auditer chaque variable pour vérifier sa quotation correcte
2. Remplacer toute boucle `for` sur une substitution de commande non protégée par un `while IFS= read -r` approprié
3. Ajouter `local` à toute variable interne aux fonctions qui n'en disposait pas
4. Exécuter `shellcheck` sur le script révisé et corriger tous les avertissements signalés
5. Documenter, dans un court rapport, les problèmes identifiés dans la version originale et les corrections apportées

---

# PARTIE II — TD

### Niveau 1 — Compréhension

**Exercice 1.1**
Expliquez la différence de comportement entre `echo $var`, `echo "$var"` et `echo '$var'`.

**Exercice 1.2**
Expliquez la différence entre `$@` et `$*` dans le contexte d'un script recevant plusieurs arguments.

**Exercice 1.3**
Expliquez pourquoi une fonction Bash ne peut pas directement "retourner" une chaîne de caractères comme dans un langage typé, et quelle convention est utilisée à la place.

### Niveau 2 — Application

**Exercice 2.1**
Écrivez une condition Bash testant si la variable `$fichier` correspond à un fichier régulier existant.

**Exercice 2.2**
Écrivez une boucle `while` robuste lisant chaque ligne du fichier `/etc/hosts` et l'affichant, en utilisant la méthode correcte vue en cours.

**Exercice 2.3**
Écrivez une fonction Bash `est_pair` prenant un nombre en argument et retournant un code de sortie 0 s'il est pair, 1 sinon.

### Niveau 3 — Analyse

**Exercice 3.1**
Un script contient la ligne suivante : `if [ $age -eq "adulte" ]; then`. Identifiez les deux erreurs présentes dans cette ligne (une de quotation, une de choix d'opérateur), et corrigez-la.

**Exercice 3.2**
Une fonction Bash définit une variable `resultat` sans `local`, puis le script principal, après avoir appelé cette fonction, utilise également une variable nommée `resultat` à ses propres fins, avec un contenu totalement différent. Décrivez le bug probable qui en résulterait, et la correction à apporter.

### Niveau 4 — Problème complexe

**Exercice 4.1**
Un script destiné à archiver puis supprimer tous les fichiers `.tmp` d'un répertoire est écrit ainsi :
```bash
for f in $(find /data -name "*.tmp"); do
    tar -rf archive.tar "$f"
    rm $f
done
```
Ce script fonctionne correctement en test sur un répertoire ne contenant que des fichiers aux noms simples, mais provoque des dégâts inattendus une fois déployé sur un répertoire réel contenant des fichiers aux noms plus variés (certains avec des espaces). Identifiez précisément **toutes** les occurrences problématiques de ce script (il y en a plusieurs, de nature différente), expliquez pour chacune le mécanisme exact du problème, et proposez une version entièrement corrigée et robuste.

---

## Correction du TD

**1.1** — `echo $var` (sans guillemets) : la valeur de `$var` est soumise au découpage en mots et à l'expansion de glob ; si elle contient des espaces ou des caractères spéciaux de glob, le résultat peut être fragmenté ou modifié de façon inattendue. `echo "$var"` (guillemets doubles) : l'expansion de la variable a bien lieu, mais le résultat est préservé comme un seul mot, sans découpage ni expansion de glob — le comportement à privilégier systématiquement. `echo '$var'` (guillemets simples) : aucune expansion n'a lieu, le texte `$var` est affiché littéralement, caractère par caractère.

**1.2** — `$@` (utilisé avec des guillemets, `"$@"`) préserve chaque argument individuellement, même s'il contient des espaces — équivalent à autant de mots distincts qu'il y a d'arguments réellement passés au script. `$*` (même avec des guillemets, `"$*"`) concatène tous les arguments en une seule chaîne unique, séparée par le premier caractère de la variable `IFS` (un espace par défaut) — perdant la distinction entre arguments individuels. Pour transmettre fidèlement une liste d'arguments à une autre commande, `"$@"` est presque toujours le choix correct.

**1.3** — Bash ne dispose pas d'un système de types au sens d'un langage de programmation classique ; le mécanisme `return` d'une fonction Bash est hérité de la logique des codes de sortie de processus Unix (rappel UE1) : il ne peut renvoyer qu'un entier compris entre 0 et 255, conventionnellement interprété comme un indicateur de succès (0) ou d'échec (valeur non nulle), jamais une donnée arbitraire comme une chaîne de caractères. La convention utilisée pour renvoyer une donnée réelle est d'utiliser `echo` (ou `printf`) pour écrire cette donnée sur la sortie standard de la fonction, que l'appelant capture ensuite via une substitution de commande (`resultat=$(ma_fonction)`).

**2.1** — `if [ -f "$fichier" ]; then`.

**2.2** — 
```bash
while IFS= read -r ligne; do
    echo "$ligne"
done < /etc/hosts
```

**2.3** — 
```bash
est_pair() {
    local nombre="$1"
    if (( nombre % 2 == 0 )); then
        return 0
    else
        return 1
    fi
}
```

**3.1** — Deux erreurs : (1) `$age` n'est pas quotifié (`$age` au lieu de `"$age"`), ce qui pourrait poser problème si la variable était vide ou contenait des espaces (bonne pratique systématique, même si dans ce cas précis l'impact immédiat dépend du contenu réel de la variable) ; (2) `-eq` est un opérateur de comparaison **numérique**, utilisé ici pour comparer avec la chaîne `"adulte"`, ce qui est incohérent — soit la comparaison porte sur un âge numérique (auquel cas comparer avec la chaîne "adulte" n'a pas de sens et devrait être `[ "$age" -ge 18 ]` par exemple), soit elle porte sur une catégorie textuelle (auquel cas l'opérateur correct serait `==` : `[ "$age" == "adulte" ]`). Correction cohérente selon l'intention probable : `if [ "$age" -ge 18 ]; then` (si `$age` est numérique et représente un âge en années).

**3.2** — Sans `local`, la variable `resultat` définie à l'intérieur de la fonction est **globale** par défaut en Bash — après l'exécution de la fonction, cette variable globale `resultat` conserve la valeur qui lui a été assignée par la fonction, écrasant silencieusement toute valeur que le script principal aurait pu lui attribuer précédemment sous ce même nom, ou étant elle-même écrasée si le script principal l'utilise après l'appel de fonction sans se douter qu'elle a été modifiée par cette dernière. Le bug résultant serait une valeur de `resultat` incohérente et inattendue dans le script principal après l'appel de la fonction, potentiellement très difficile à tracer si le script est de taille conséquente et que cette collision de nommage n'est pas immédiatement évidente à la lecture. Correction : ajouter `local resultat="..."` à l'intérieur de la définition de la fonction, isolant cette variable de l'espace de noms global du script.

**4.1** — Occurrences problématiques identifiées :

1. **`for f in $(find /data -name "*.tmp")`** : la substitution de commande `$(find ...)` n'est pas quotifiée, son résultat est donc soumis au découpage en mots — chaque **espace** dans un nom de fichier fragmente ce nom en plusieurs éléments distincts de la boucle `for`, corrompant totalement la liste de fichiers réellement traités.

2. **`rm $f`** : la variable `$f` (elle-même déjà potentiellement corrompue par le problème précédent) n'est pas quotifiée non plus lors de son utilisation avec `rm`, un second niveau de risque même si le premier problème était par hypothèse corrigé isolément — un fragment de nom de fichier contenant par coïncidence un glob ou correspondant accidentellement à un autre fichier existant pourrait être supprimé de façon totalement inattendue et potentiellement dangereuse.

Version corrigée et robuste, utilisant un séparateur nul pour une insensibilité totale aux espaces et autres caractères spéciaux dans les noms de fichiers :

```bash
find /data -name "*.tmp" -print0 | while IFS= read -r -d '' f; do
    tar -rf archive.tar "$f"
    rm "$f"
done
```

Cette version corrige les deux occurrences identifiées : `-print0` associé à `read -r -d ''` traite chaque nom de fichier comme une unité complète et atomique, insensible à la présence d'espaces (ou même de retours à la ligne, cas extrême mais possible) dans les noms, et chaque utilisation ultérieure de `"$f"` est correctement quotifiée pour préserver cette intégrité jusqu'à son usage final dans `tar` et `rm`.

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur système junior pour auditer et corriger un ensemble de scripts existants présentant des problèmes de robustesse liés à la quotation et à la gestion des arguments, en vous appuyant sur `shellcheck` comme outil de validation systématique.

## Architecture

Une machine Linux avec Bash et `shellcheck` installés (`sudo apt install shellcheck` ou équivalent selon la distribution, rappel UE3-M20).

## Objectifs
- Diagnostiquer et corriger des scripts fautifs en matière de quotation
- Utiliser `shellcheck` comme outil de validation systématique
- Écrire une fonction robuste avec gestion d'arguments et de code de sortie
- Reproduire concrètement le piège de la boucle `for` sur substitution de commande non protégée

## Préparation

```bash
which bash shellcheck 2>/dev/null
bash --version
```

## Étapes

### Étape 1 — Reproduire le piège de la quotation sur noms de fichiers

```bash
mkdir -p /tmp/lab_bash/donnees
touch "/tmp/lab_bash/donnees/rapport final.txt"
touch "/tmp/lab_bash/donnees/notes.txt"
cd /tmp/lab_bash/donnees

echo "=== Version fautive (sans guillemets) ==="
for f in $(ls *.txt); do
    echo "Traitement de: $f"
done

echo "=== Version corrigée (avec guillemets et méthode robuste) ==="
for f in *.txt; do
    echo "Traitement de: $f"
done
```
*Question :* combien d'itérations la version fautive produit-elle pour le fichier "rapport final.txt" ? Combien la version corrigée produit-elle pour ce même fichier ? Expliquez précisément la différence en vous appuyant sur le mécanisme de découpage en mots.

### Étape 2 — Audit avec shellcheck

```bash
cat << 'EOF' > /tmp/lab_bash/script_fautif.sh
#!/bin/bash
fichier=$1
if [ $fichier == "test" ]; then
    echo trouve
fi
for f in $(find . -name "*.log"); do
    cat $f
done
EOF

shellcheck /tmp/lab_bash/script_fautif.sh
```
*Question :* combien d'avertissements `shellcheck` signale-t-il ? Pour chacun, identifiez à quelle règle du cours (quotation, opérateur de comparaison, méthode de boucle) il correspond.

Corrigez le script en tenant compte de tous les avertissements, puis validez :
```bash
# Après correction dans un nouveau fichier ou en éditant le fichier existant
shellcheck /tmp/lab_bash/script_corrige.sh   # doit ne produire aucun avertissement (ou les avertissements restants justifiés)
```

### Étape 3 — Fonction robuste avec gestion d'arguments

```bash
cat << 'EOF' > /tmp/lab_bash/verif_service.sh
#!/usr/bin/env bash
set -euo pipefail

verifier_service() {
    local nom_service="$1"
    if systemctl is-active --quiet "$nom_service" 2>/dev/null; then
        echo "actif"
        return 0
    else
        echo "inactif"
        return 1
    fi
}

main() {
    if [[ $# -eq 0 ]]; then
        echo "Usage: $0 <nom_du_service>" >&2
        exit 1
    fi
    local statut
    statut=$(verifier_service "$1") || true
    echo "Le service '$1' est: $statut"
}

main "$@"
EOF
chmod +x /tmp/lab_bash/verif_service.sh

/tmp/lab_bash/verif_service.sh sshd
/tmp/lab_bash/verif_service.sh service_inexistant
/tmp/lab_bash/verif_service.sh   # sans argument, doit afficher l'usage
```
*Question :* pourquoi la ligne `statut=$(verifier_service "$1") || true` inclut-elle `|| true` ? Que se passerait-il, avec `set -euo pipefail` actif, si cette précaution était omise et que la fonction retournait un code de sortie non nul (service inactif) ?

### Étape 4 — Reproduire et corriger le scénario complet de l'exercice 4.1 du TD

```bash
mkdir -p /tmp/lab_bash/archive_test
cd /tmp/lab_bash/archive_test
touch "fichier avec espace.tmp" "fichier_simple.tmp"

cat << 'EOF' > /tmp/lab_bash/script_archivage_fautif.sh
#!/bin/bash
cd /tmp/lab_bash/archive_test
for f in $(find . -name "*.tmp"); do
    echo "Archiverait: $f"
    # rm $f   # commenté par sécurité pour ce TP, mais illustrerait le risque réel
done
EOF
chmod +x /tmp/lab_bash/script_archivage_fautif.sh
/tmp/lab_bash/script_archivage_fautif.sh
```
*Résultat attendu :* observez comment "fichier avec espace.tmp" est traité de façon incorrecte (fragmenté en plusieurs éléments).

Corrigez selon la méthode vue en cours et en TD (exercice 4.1), puis vérifiez :
```bash
cat << 'EOF' > /tmp/lab_bash/script_archivage_corrige.sh
#!/usr/bin/env bash
set -euo pipefail
cd /tmp/lab_bash/archive_test
find . -name "*.tmp" -print0 | while IFS= read -r -d '' f; do
    echo "Archiverait correctement: $f"
done
EOF
chmod +x /tmp/lab_bash/script_archivage_corrige.sh
/tmp/lab_bash/script_archivage_corrige.sh
shellcheck /tmp/lab_bash/script_archivage_corrige.sh
```

## Commandes de référence
`shellcheck`, `while IFS= read -r`, `find -print0`, `set -euo pipefail`

## Résultats attendus
- Comparaison des deux versions de l'étape 1, avec explication du mécanisme
- Liste des avertissements `shellcheck` de l'étape 2, avec leur correction
- Sorties du script de l'étape 3 pour les trois cas de test
- Comparaison avant/après correction du scénario d'archivage (étape 4)

## Questions

1. Dans l'étape 1, pourquoi la version corrigée utilise-t-elle directement `for f in *.txt` plutôt que `for f in "$(ls *.txt)"` (avec guillemets ajoutés autour de la substitution de commande) ? Ce dernier serait-il également correct ? Justifiez en réfléchissant à ce qui se passerait avec plusieurs fichiers.
2. `shellcheck` est-il capable de détecter toutes les erreurs logiques possibles d'un script, ou seulement certaines catégories de problèmes ? Donnez un exemple d'erreur logique qui échapperait probablement à son analyse.
3. Dans l'étape 4, si le nom de fichier avait contenu un retour à la ligne littéral (cas extrême mais possible sous Linux), la méthode `find -print0`/`read -d ''` resterait-elle robuste, contrairement à une hypothétique méthode basée sur `read` sans séparateur nul ? Justifiez votre réponse.

## Troubleshooting (panne à diagnostiquer)

**Situation fournie :** un étudiant corrige son script de l'étape 3 en ajoutant `set -euo pipefail`, mais le script se termine immédiatement avec une erreur dès qu'il teste un service inactif, sans jamais afficher le message "Le service '...' est: inactif" attendu.

**Démarche attendue :**
1. Symptôme : le script s'arrête prématurément avec `set -euo pipefail` actif, alors qu'il fonctionnait avant l'ajout de cette option
2. Hypothèse : `set -e` fait terminer immédiatement le script dès qu'une commande retourne un code de sortie non nul, **y compris** une fonction appelée dans une substitution de commande — si la ligne `statut=$(verifier_service "$1")` ne comporte pas le `|| true` vu à l'étape 3, un service inactif (faisant retourner 1 par la fonction) déclencherait l'arrêt immédiat du script sous `set -e`, avant même d'atteindre la ligne d'affichage du statut
3. Vérification : relire précisément le script de l'étudiant pour confirmer l'absence du `|| true` (ou d'une construction équivalente) après l'appel de la fonction dans un contexte où son échec attendu et normal (service inactif) ne doit pas interrompre le script
4. Isolation : tester la fonction `verifier_service` isolément avec un service sciemment inactif, en observant si son code de sortie non nul provoque effectivement l'arrêt du script sous `set -e` dans ce contexte précis d'appel
5. Correction : ajouter `|| true` (ou une structure `if`/`else` plus explicite selon la préférence de style) après l'appel de fonction dont l'échec est un résultat normal et attendu à traiter, pas une erreur imprévue devant interrompre le script
6. Vérification : relancer le script sur un service inactif, confirmer qu'il affiche désormais correctement le message de statut sans s'interrompre prématurément
7. Documentation : noter cette interaction entre `set -e` et les fonctions retournant un code de sortie non nul comme un point de vigilance central du prochain module (M25, gestion des erreurs) — un aperçu direct de la nuance que `set -e` traite l'échec d'une commande comme une erreur fatale par défaut, ce qui nécessite une gestion explicite pour tout échec qui fait partie de la logique normale du script plutôt que d'une véritable erreur

## Validation

Le TP est validé si l'étudiant fournit :
- Les 4 résultats/comparaisons demandés
- Des réponses argumentées aux 3 questions
- Une identification correcte de la cause du scénario de troubleshooting (interaction `set -e` et code de sortie de fonction), avec une correction cohérente et une préfiguration explicite du module suivant

# UE4 — Module 25 : Gestion des Erreurs et Bonnes Pratiques

*Ce module reprend et explique en détail `set -euo pipefail`, déjà utilisé sans justification approfondie dans la quasi-totalité des scripts des UE1, UE2 et UE3.*

---

# PARTIE I — COURS

## 1. Titre du cours
Scripts Bash robustes : `set -e`/`-u`/`-o pipefail`, trap et débogage professionnel

## 2. Niveau
Élève ingénieur — cycle ingénieur, UE4 automatisation (S2)

## 3. Durée
CM : 2h (module dense en compréhension fine — +3h TD, +5h TP en Parties II et III)

## 4. Prérequis
- UE4-Module 23 : syntaxe Bash, codes de sortie, fonctions
- UE4-Module 24 : pipes et redirections (indispensable pour comprendre `pipefail`)
- Le TP du module 23 a déjà amorcé ce module via le piège `set -e` et fonction retournant un code non nul

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Expliquer précisément le comportement de `set -e`, ses subtilités et ses limites
- Expliquer `set -u` et le risque des variables non définies
- Expliquer `set -o pipefail` et pourquoi son absence masque des échecs dans un pipeline
- Utiliser `trap` pour garantir un nettoyage même en cas d'erreur ou d'interruption
- Déboguer un script avec `set -x`/`bash -x`
- Structurer un script professionnel : messages d'erreur vers stderr, codes de sortie cohérents, validation des entrées

## 6. Motivation

`set -euo pipefail` est apparu en tête de la quasi-totalité des scripts rencontrés depuis l'UE1 — présenté comme une bonne pratique à appliquer, sans toujours en détailler le mécanisme exact. Ce module comble cette lacune volontaire : comprendre précisément ce que fait chacune de ces trois options, leurs interactions parfois contre-intuitives (déjà entrevues au TP du module 23), et leurs limites réelles — car `set -e` en particulier a des subtilités qui, mal comprises, peuvent donner une fausse impression de sécurité.

Un script qui échoue bruyamment et immédiatement à la première erreur est infiniment préférable à un script qui continue silencieusement après un échec partiel, produisant un résultat incorrect sans qu'aucune alerte ne soit levée — exactement le type d'incident déjà rencontré à plusieurs reprises dans les UE précédentes (tâche cron silencieusement défaillante, sauvegarde qui échoue sans le signaler). Ce module donne les outils pour qu'un script échoue de façon **visible et immédiate**, plutôt que silencieuse et tardive.

## 7. Concepts fondamentaux

### 7.1 `set -e` — arrêt à la première erreur

`set -e` (ou `set -o errexit`) fait terminer immédiatement le script dès qu'une commande simple retourne un code de sortie non nul, sans qu'une gestion explicite (`if`, `&&`, `||`) n'intercepte cet échec.

```bash
set -e
echo "Début"
false            # commande qui échoue toujours (code de sortie 1)
echo "Cette ligne ne s'exécute jamais"
```

**Les exceptions importantes à connaître (subtilités fréquemment ignorées) :**
- Une commande faisant partie d'une condition (`if commande; then`) n'interrompt **pas** le script même si elle échoue — c'est précisément le comportement attendu et recherché dans ce contexte
- Une commande précédée de `!` n'interrompt pas le script
- Une commande dans une liste `&&`/`||`, sauf la toute dernière de la chaîne, n'interrompt pas le script de la même façon
- **Le piège déjà rencontré au TP du module 23 :** `variable=$(fonction_qui_peut_echouer)` — si cette ligne constitue une commande simple en position d'affectation, l'échec de la substitution de commande **interrompt** le script sous `set -e`, sauf si explicitement neutralisé (`|| true`, ou une structure conditionnelle)

### 7.2 `set -u` — variables non définies

`set -u` (ou `set -o nounset`) fait échouer le script dès qu'une variable **non définie** est utilisée, plutôt que de la traiter silencieusement comme une chaîne vide (comportement par défaut de Bash sans cette option).

```bash
set -u
echo "$variable_jamais_definie"    # erreur immédiate : "unbound variable"
```

**Pourquoi cette option est précieuse :** une faute de frappe dans un nom de variable (`$repertoir` au lieu de `$repertoire`) serait normalement silencieusement traitée comme une chaîne vide, produisant un comportement incorrect sans aucun message d'erreur — `set -u` transforme cette classe entière d'erreurs de frappe en un échec immédiat et explicite, au lieu d'un bug latent et difficile à diagnostiquer.

**Piège associé :** les paramètres positionnels non fournis (`$1` sans argument correspondant) sont également considérés comme non définis sous `set -u` — un script doit donc vérifier `$#` avant d'accéder à `$1` si l'argument est censé être optionnel, ou fournir une valeur par défaut avec la syntaxe `${1:-valeur_par_defaut}`.

### 7.3 `set -o pipefail` — le maillon faible d'un pipeline

Par défaut en Bash, le code de sortie d'un pipeline (`commande1 | commande2`) est celui de la **dernière** commande de la chaîne, indépendamment du succès ou de l'échec des commandes précédentes.

```bash
false | echo "ce message s'affiche"
echo $?     # affiche 0 (succès), car echo a réussi -- l'échec de "false" est masqué !
```

`set -o pipefail` change ce comportement : le code de sortie du pipeline devient celui de la **première** commande qui échoue dans la chaîne (ou 0 si toutes réussissent).

```bash
set -o pipefail
false | echo "ce message s'affiche"
echo $?     # affiche désormais 1, reflétant l'échec de "false"
```

**Pourquoi cette option est cruciale, notamment après le module 24 :** un pipeline `grep "erreur" fichier.log | mail -s "Alerte" admin@exemple.com` dont le fichier n'existe pas (`grep` échoue) enverrait néanmoins un email "réussi" (vide) sans `pipefail`, masquant totalement l'échec réel de `grep` — exactement le type de défaillance silencieuse déjà dénoncée dans plusieurs études de cas des UE précédentes (UE3-M19, M21).

### 7.4 `set -euo pipefail` — la combinaison recommandée par défaut

```bash
#!/usr/bin/env bash
set -euo pipefail
```

Cette combinaison, déjà utilisée sans justification approfondie dans de nombreux scripts précédents, active simultanément l'arrêt à la première erreur, la détection de variable non définie, et la propagation correcte de l'échec dans un pipeline — un point de départ recommandé pour la quasi-totalité des scripts destinés à un usage réel, bien au-delà d'un simple exercice ponctuel.

**Limite importante à connaître :** `set -e` ne protège pas contre **toutes** les formes d'erreur — une commande qui "réussit" au sens de son code de sortie (0) mais produit un résultat incorrect ou inattendu (par exemple, une commande qui ne trouve aucun résultat mais retourne malgré tout 0 selon sa convention propre) ne sera jamais interceptée par `set -e` seul. Une validation explicite du résultat métier reste nécessaire au-delà de la simple vérification du code de sortie technique.

### 7.5 `trap` — garantir un nettoyage

`trap` permet d'exécuter une commande ou une fonction lors de la réception d'un signal (rappel UE1-M2) ou lors de conditions particulières du script (sortie normale ou anormale) :

```bash
FICHIER_TEMP=$(mktemp)

nettoyer() {
    rm -f "$FICHIER_TEMP"
    echo "Nettoyage effectué"
}

trap nettoyer EXIT
```

`trap ... EXIT` garantit que la fonction `nettoyer` s'exécute **quelle que soit la façon dont le script se termine** — succès normal, erreur sous `set -e`, ou interruption par signal (`Ctrl+C`, `SIGTERM`) — un mécanisme essentiel pour garantir qu'un fichier temporaire est toujours supprimé, qu'un verrou est toujours libéré, ou qu'une notification est toujours envoyée, indépendamment du chemin exact par lequel le script se termine.

```bash
trap 'echo "Interruption reçue, nettoyage en cours..."; nettoyer; exit 130' INT TERM
```

**Application directe à un piège déjà rencontré en UE3 :** un script de sauvegarde qui échoue silencieusement (UE3-M19, M21) aurait pu utiliser `trap` pour garantir qu'une notification d'échec soit envoyée même si le script se termine prématurément à un endroit imprévu, plutôt que de dépendre d'une vérification explicite placée uniquement à la toute fin du script (qui ne serait jamais atteinte en cas d'arrêt prématuré sous `set -e`).

### 7.6 Débogage avec `set -x`

```bash
set -x    # active le mode trace : chaque commande est affichée avant son exécution, avec ses expansions
# ... code à déboguer ...
set +x    # désactive le mode trace
```

Alternative sans modifier le script : `bash -x monscript.sh` — utile pour observer précisément la valeur réelle des variables après expansion à chaque étape, un outil de diagnostic bien plus rigoureux qu'une multiplication de `echo` de débogage temporaires disséminés dans le code.

### 7.7 Structure professionnelle d'un script

**Messages d'erreur vers stderr, jamais stdout :**
```bash
echo "Erreur: fichier introuvable" >&2
```
Séparer stdout (résultat normal du script, potentiellement capturé ou traité par un autre programme) de stderr (messages de diagnostic destinés à l'humain ou aux logs) est une discipline essentielle, notamment pour tout script destiné à être utilisé dans un pipeline (rappel module 24) où stdout doit rester "propre", ne contenant que la donnée réellement produite.

**Codes de sortie cohérents et documentés :**
```bash
readonly ERREUR_ARGUMENT_MANQUANT=1
readonly ERREUR_FICHIER_INTROUVABLE=2
readonly ERREUR_PERMISSION=3
```
Documenter la signification de chaque code de sortie non nul facilite grandement l'intégration du script dans un système plus large (supervision, autres scripts appelants) qui pourrait avoir besoin de réagir différemment selon la nature précise de l'échec.

**Validation systématique des entrées, avant tout traitement :**
```bash
main() {
    if [[ $# -lt 1 ]]; then
        echo "Usage: $0 <fichier>" >&2
        exit "$ERREUR_ARGUMENT_MANQUANT"
    fi
    if [[ ! -r "$1" ]]; then
        echo "Erreur: fichier illisible: $1" >&2
        exit "$ERREUR_FICHIER_INTROUVABLE"
    fi
    # ... traitement principal ...
}
main "$@"
```

## 8. Architecture

```
┌────────────────────────────────────────────────────────────┐
│  set -e   : arrêt immédiat sur échec d'une commande simple      │
│  set -u   : arrêt immédiat sur variable non définie               │
│  set -o pipefail : propage l'échec du maillon faible d'un pipeline  │
│                                                                    │
│  trap nettoyer EXIT  ──▶ garantit le nettoyage quel que soit         │
│                          le chemin de sortie (succès, erreur,          │
│                          interruption par signal)                       │
└────────────────────────────────────────────────────────────┘
```

## 9. Fonctionnement interne

`set -e` s'appuie sur le mécanisme de code de sortie de processus déjà étudié en UE1-M2 : après chaque commande simple exécutée par l'interpréteur Bash, celui-ci consulte le code de sortie retourné et, si `errexit` est actif et qu'aucune des exceptions de la section 7.1 ne s'applique, déclenche immédiatement la terminaison du script (équivalent interne à un `exit` avec ce même code). `trap` s'appuie sur le mécanisme de signal du noyau (rappel UE1-M2) : Bash intercepte la réception d'un signal (`SIGINT`, `SIGTERM`) ou la condition interne de fin de script (`EXIT`, un pseudo-signal propre à Bash) et exécute la commande associée avant de terminer réellement le processus.

## 10. Démonstration

```bash
# Démonstration du piège pipefail
bash -c 'set -o pipefail; false | true; echo "Code: $?"'
bash -c 'false | true; echo "Code: $?"'   # sans pipefail, pour comparaison

# Démonstration trap
bash -c 'trap "echo Nettoyage" EXIT; echo "Travail en cours"; exit 1'
```

## 11. Commandes / outils

| Élément | Rôle | Exemple |
|---|---|---|
| `set -e` | Arrêt à la première erreur | en tête de script |
| `set -u` | Détection de variable non définie | en tête de script |
| `set -o pipefail` | Propagation d'échec dans un pipeline | en tête de script |
| `trap ... EXIT` | Nettoyage garanti | `trap nettoyer EXIT` |
| `set -x` / `bash -x` | Mode trace pour débogage | `bash -x script.sh` |
| `${var:-defaut}` | Valeur par défaut si variable non définie/vide | `echo "${1:-inconnu}"` |

**Erreur fréquente :** activer `set -e` en pensant qu'il protège contre **toute** forme d'erreur logique, alors qu'il ne détecte que l'échec au sens du code de sortie d'une commande — une commande "réussie" produisant un résultat métier incorrect (par exemple zéro ligne extraite d'un `grep` alors qu'un résultat était attendu) doit être validée explicitement, `set -e` ne le fera jamais à sa place.

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** un script de sauvegarde utilise le pipeline `tar -czf - /data | ssh serveur_distant "cat > /backup/archive.tar.gz"` sans `set -o pipefail`. Un problème réseau interrompt la connexion SSH en cours de transfert, mais le script se termine avec un code de sortie 0 (succès), car `cat` côté distant retourne un succès trivial, masquant l'échec réel du transfert de `tar`/`ssh`. L'équipe découvre, des semaines plus tard, que plusieurs sauvegardes nocturnes récentes sont en réalité tronquées et inutilisables.

**Analyse attendue :** ce scénario est une démonstration directe et grave du piège de la section 7.3 : sans `pipefail`, le code de sortie du pipeline reflète uniquement le succès de la dernière commande (`cat`, trivialement toujours réussie si elle reçoit la connexion), masquant totalement l'échec réel du transfert survenu plus tôt dans la chaîne. Ce cas rejoint directement les études de cas déjà rencontrées en UE3 (M19, M21) sur les échecs silencieux de sauvegarde, ici avec une cause technique précise et évitable. Correction : ajouter `set -o pipefail` (voire `set -euo pipefail` complet) en tête du script, garantissant que tout échec dans la chaîne de transfert soit correctement reflété dans le code de sortie final, permettant sa détection par un mécanisme de supervision (rappel direct du heartbeat déjà étudié en UE3-M19/M21). Mesure préventive généralisable : `set -euo pipefail` devrait être une clause d'ouverture systématique et non négociable pour tout script de production, en particulier tout script impliquant une chaîne de traitement via pipeline.

## 15. Sécurité

- Un script exécuté avec des privilèges élevés (root, ou invoqué par un service systemd comme vu en UE3-M16) qui continue silencieusement après un échec partiel peut laisser le système dans un état incohérent ou partiellement configuré, potentiellement exploitable — la rigueur de `set -euo pipefail` est autant une question de fiabilité que de sécurité opérationnelle
- `trap` peut également être utilisé pour garantir la suppression sécurisée d'informations sensibles temporaires (un mot de passe déchiffré en mémoire dans une variable, un fichier de clé temporaire) même en cas d'interruption anormale du script, évitant qu'une donnée sensible ne persiste au-delà de son usage légitime
- Un script mal protégé contre les variables non définies (`set -u` absent) pourrait, dans certains contextes, interpoler une chaîne vide inattendue dans une commande sensible (un chemin de fichier, une condition de permission), avec des conséquences potentiellement dangereuses selon le contexte

## 16. Automatisation

Script Bash illustrant l'ensemble des bonnes pratiques du module, appliqué à une tâche de sauvegarde robuste avec nettoyage garanti.

```bash
#!/usr/bin/env bash
set -euo pipefail

readonly ERREUR_ARGUMENT=1
readonly ERREUR_SOURCE_INTROUVABLE=2
readonly ERREUR_TRANSFERT=3

FICHIER_TEMP=""

nettoyer() {
    local code_sortie=$?
    if [[ -n "$FICHIER_TEMP" && -f "$FICHIER_TEMP" ]]; then
        rm -f "$FICHIER_TEMP"
    fi
    if [[ $code_sortie -ne 0 ]]; then
        echo "Script terminé avec une erreur (code: $code_sortie)" >&2
    fi
    exit "$code_sortie"
}
trap nettoyer EXIT

main() {
    if [[ $# -lt 1 ]]; then
        echo "Usage: $0 <repertoire_source>" >&2
        exit "$ERREUR_ARGUMENT"
    fi

    local source="$1"
    if [[ ! -d "$source" ]]; then
        echo "Erreur: répertoire introuvable: $source" >&2
        exit "$ERREUR_SOURCE_INTROUVABLE"
    fi

    FICHIER_TEMP=$(mktemp)
    if ! tar -czf "$FICHIER_TEMP" "$source"; then
        echo "Erreur: échec de la création de l'archive" >&2
        exit "$ERREUR_TRANSFERT"
    fi

    echo "Archive créée avec succès: $FICHIER_TEMP"
}

main "$@"
```

## 17. Troubleshooting

Scénario court : un script sous `set -euo pipefail` se termine prématurément à une ligne qui, en apparence, ne devrait pas échouer — une simple affectation `nombre_lignes=$(wc -l < fichier_vide.txt)` sur un fichier vide. Démarche : tester isolément la commande en question (`wc -l < fichier_vide.txt; echo $?`) pour vérifier son code de sortie réel sur ce cas limite précis → si une commande apparemment anodine retourne parfois un code non nul dans certaines conditions limites (fichier vide, résultat nul), cela interrompra le script sous `set -e` de façon peu intuitive → décider consciemment si ce cas doit interrompre le script (comportement peut-être réellement souhaité) ou doit être traité comme un cas normal à gérer explicitement (`|| nombre_lignes=0`, ou une structure conditionnelle appropriée), plutôt que de subir un arrêt non anticipé sur un cas limite qui n'avait pas été envisagé lors de l'écriture initiale du script.

## 18. Synthèse

- `set -e` arrête le script à la première commande simple en échec, avec des exceptions importantes (conditions, listes `&&`/`||`) à bien connaître
- `set -u` détecte les variables non définies, transformant les fautes de frappe en erreurs immédiates plutôt qu'en bugs silencieux
- `set -o pipefail` propage l'échec du maillon faible d'un pipeline, sans quoi seul le code de sortie de la dernière commande compte — un piège potentiellement grave, illustré par l'étude de cas de ce module
- `set -euo pipefail` est une clause d'ouverture recommandée par défaut, mais ne remplace jamais une validation explicite du résultat métier d'une commande
- `trap ... EXIT` garantit un nettoyage quel que soit le chemin de sortie du script, y compris une interruption par signal
- Un script professionnel sépare stdout (résultat) de stderr (diagnostic), documente ses codes de sortie, et valide systématiquement ses entrées avant tout traitement

## 19. Questions d'évaluation

**QCM**
1. Sans `set -o pipefail`, le code de sortie d'un pipeline reflète :
 a) le succès global de toute la chaîne b) uniquement le code de sortie de la dernière commande c) toujours 0 d) toujours 1

2. `set -u` provoque une erreur lors de :
 a) l'utilisation d'une variable non définie b) l'utilisation d'une variable trop longue c) toute utilisation de guillemets d) l'appel d'une fonction

**Questions ouvertes**
- Expliquez pourquoi l'absence de `set -o pipefail` peut masquer un échec grave dans un script de sauvegarde utilisant un pipeline.
- Expliquez le rôle de `trap ... EXIT` et donnez un exemple de situation où son absence pourrait laisser un système dans un état incohérent.

**Exercice technique**
- Écrivez la ligne de code permettant d'activer simultanément `errexit`, `nounset` et `pipefail` en tête d'un script.

**Problème de diagnostic**
- Un script sous `set -euo pipefail` se termine prématurément à une ligne en apparence anodine. Proposez une démarche pour identifier précisément la cause.

## 20. Mini-projet

Reprendre le script du mini-projet du module 24 (Pipeline Log Analyzer) et le robustifier intégralement selon les pratiques de ce module :
1. Ajouter `set -euo pipefail` et vérifier que le script continue de fonctionner correctement sur tous les cas déjà testés au module 24
2. Ajouter un `trap` de nettoyage garantissant la suppression de tout fichier temporaire créé pendant l'exécution
3. Documenter des codes de sortie cohérents pour chaque catégorie d'erreur possible (argument manquant, fichier introuvable, option invalide)
4. Séparer systématiquement les messages d'erreur (stderr) de la sortie d'analyse normale (stdout)
5. Tester explicitement au moins un cas limite qui aurait pu échapper à la version précédente (fichier vide, pipeline dont un élément intermédiaire échoue), et documenter comment `set -euo pipefail` modifie (ou ne modifie pas) le comportement observé sur ce cas

---

# PARTIE II — TD

### Niveau 1 — Compréhension

**Exercice 1.1**
Expliquez le comportement de `set -e`, avec un exemple de commande qui interrompt le script et un exemple de commande qui ne l'interrompt pas malgré un échec.

**Exercice 1.2**
Expliquez ce que `set -u` détecte et pourquoi c'est utile pour repérer des fautes de frappe dans les noms de variables.

**Exercice 1.3**
Expliquez ce que change `set -o pipefail` par rapport au comportement par défaut de Bash concernant le code de sortie d'un pipeline.

### Niveau 2 — Application

**Exercice 2.1**
Écrivez la ligne de code activant simultanément les trois options `errexit`, `nounset` et `pipefail`.

**Exercice 2.2**
Écrivez un `trap` garantissant qu'un message "Script terminé" s'affiche quel que soit le mode de sortie du script (normal ou erreur).

**Exercice 2.3**
Donnez la commande permettant d'exécuter un script `mon_script.sh` en mode trace, sans modifier le fichier du script lui-même.

### Niveau 3 — Analyse

**Exercice 3.1**
Un script utilise `resultat=$(commande_qui_echoue)` sous `set -e`, sans aucune gestion d'erreur supplémentaire sur cette ligne. Le script s'arrête immédiatement à cette ligne. Un développeur, surpris, s'attendait à ce que le script continue avec `resultat` simplement vide. Expliquez pourquoi son intuition était incorrecte sous `set -e`, et proposez une modification de cette ligne permettant réellement d'obtenir le comportement qu'il attendait initialement (continuer avec une valeur vide en cas d'échec).

**Exercice 3.2**
Un script effectue `mkdir /tmp/travail_temporaire`, réalise plusieurs opérations dans ce répertoire, puis le supprime à la toute dernière ligne du script avec `rm -rf /tmp/travail_temporaire`. Si une erreur survient au milieu du script (sous `set -e`), ce répertoire temporaire sera-t-il supprimé ? Proposez une modification garantissant sa suppression quel que soit le point d'échec du script.

### Niveau 4 — Problème complexe

**Exercice 4.1**
Un script de sauvegarde utilise le pipeline suivant, sans `set -o pipefail` : `tar -czf - /data | ssh backup-serveur "cat > /backup/archive_$(date +%Y%m%d).tar.gz"`. Une équipe constate, plusieurs semaines après la mise en production de ce script, que certaines archives de sauvegarde sont tronquées et inutilisables, sans qu'aucune alerte n'ait jamais été déclenchée par le script lui-même (qui rapporte systématiquement un succès). En vous appuyant sur les mécanismes vus en cours, expliquez précisément pourquoi ce script peut échouer silencieusement de cette façon, identifiez à quel moment exact dans la chaîne un problème réseau interromprait le transfert sans que cela ne soit détecté, et proposez une version corrigée du script incluant à la fois la correction de fond (`pipefail`) et une vérification complémentaire de l'intégrité de l'archive obtenue.

---

## Correction du TD

**1.1** — `set -e` fait terminer immédiatement le script dès qu'une commande simple (non protégée par une structure conditionnelle) retourne un code de sortie non nul. Exemple qui interrompt le script : une ligne isolée `false` (ou toute commande échouant seule, en dehors de toute condition). Exemple qui n'interrompt PAS le script malgré un échec : `if false; then echo "jamais affiché"; fi` — l'échec de `false` dans le contexte d'une condition `if` est un comportement normal et attendu, `set -e` ne considère pas cela comme une erreur fatale.

**1.2** — `set -u` détecte l'utilisation de toute variable qui n'a jamais été définie (ou qui a été explicitement désactivée avec `unset`), provoquant une erreur immédiate ("unbound variable") plutôt que de traiter silencieusement cette variable comme une chaîne vide (comportement par défaut de Bash sans cette option). C'est utile pour repérer des fautes de frappe dans les noms de variables : une variable mal orthographiée (`$repertoir` au lieu de `$repertoire`) serait normalement traitée silencieusement comme vide, produisant potentiellement un comportement incorrect du script sans aucun signal d'erreur — `set -u` transforme immédiatement cette classe d'erreur de frappe en un échec explicite et facilement diagnosticable.

**1.3** — Par défaut, le code de sortie d'un pipeline (`cmd1 | cmd2 | cmd3`) est celui de la **dernière** commande de la chaîne uniquement, indépendamment du succès ou de l'échec des commandes précédentes — un échec de `cmd1` ou `cmd2` peut donc être totalement masqué si `cmd3` réussit malgré tout. `set -o pipefail` change ce comportement : le code de sortie du pipeline devient celui de la **première** commande qui échoue dans la chaîne (ou 0 si toutes réussissent), garantissant qu'un échec intermédiaire dans un pipeline soit correctement reflété et détectable.

**2.1** — `set -euo pipefail`.

**2.2** — `trap 'echo "Script terminé"' EXIT`.

**2.3** — `bash -x mon_script.sh`.

**3.1** — L'intuition du développeur était incorrecte car `set -e` considère l'échec de la commande substituée (`commande_qui_echoue`, dont le code de sortie non nul est propagé à la ligne d'affectation entière) comme une erreur fatale au même titre que toute autre commande simple en échec, interrompant immédiatement le script — le comportement "continuer avec une valeur vide" ne correspond pas au comportement par défaut de `set -e` face à un échec de commande, y compris dans un contexte d'affectation de variable. Pour obtenir réellement le comportement souhaité (continuer avec `resultat` vide en cas d'échec, sans interrompre le script), il faut neutraliser explicitement l'échec potentiel sur cette ligne précise, par exemple : `resultat=$(commande_qui_echoue) || resultat=""` (ou plus simplement `resultat=$(commande_qui_echoue || true)`, selon la nuance exacte souhaitée sur la valeur capturée en cas d'échec).

**3.2** — Non, ce répertoire ne sera **pas** supprimé si une erreur survient avant d'atteindre la dernière ligne du script sous `set -e` — le script s'interrompt immédiatement à l'endroit de l'erreur, sans jamais exécuter les lignes suivantes, y compris la ligne de nettoyage final prévue. Modification garantissant la suppression quel que soit le point d'échec : utiliser un `trap` associé au pseudo-signal `EXIT`, exécuté systématiquement à la terminaison du script quelle qu'en soit la cause : `trap 'rm -rf /tmp/travail_temporaire' EXIT`, placé immédiatement après la création du répertoire (ou en tout début de script si le répertoire est créé conditionnellement plus tard, avec une vérification appropriée dans la fonction de nettoyage elle-même).

**4.1** — Ce script peut échouer silencieusement de cette façon car, sans `set -o pipefail`, le code de sortie global du pipeline `tar | ssh "cat > ..."` est déterminé uniquement par le succès de la **dernière** commande de la chaîne — ici, la commande `cat` exécutée côté serveur distant via SSH, qui retourne un succès trivial dès lors qu'elle a pu s'exécuter et écrire sur la sortie qui lui est demandée, indépendamment de la question de savoir si le flux de données qu'elle a reçu était complet ou tronqué. Un problème réseau interrompant la connexion SSH **en cours de transfert** (après l'établissement initial de la connexion, mais avant la complétion totale du transfert de `tar`) provoquerait un échec réel de `tar` et/ou de `ssh` lui-même, mais ce point précis de la chaîne n'est jamais consulté pour déterminer le code de sortie final du pipeline sans `pipefail` — `cat`, ayant reçu un flux partiel avant l'interruption, l'écrit simplement tel quel et se termine normalement, masquant totalement l'incident survenu en amont dans la chaîne. Version corrigée :

```bash
#!/usr/bin/env bash
set -euo pipefail

ARCHIVE_DISTANTE="/backup/archive_$(date +%Y%m%d).tar.gz"

tar -czf - /data | ssh backup-serveur "cat > $ARCHIVE_DISTANTE"

# Vérification complémentaire de l'intégrité, au-delà du seul code de sortie du pipeline
TAILLE_DISTANTE=$(ssh backup-serveur "stat -c%s '$ARCHIVE_DISTANTE'" 2>/dev/null || echo 0)
if [[ "$TAILLE_DISTANTE" -lt 1000 ]]; then
    echo "Erreur: archive distante suspicieusement petite ou absente ($TAILLE_DISTANTE octets)" >&2
    exit 1
fi

echo "Sauvegarde complétée et vérifiée: $ARCHIVE_DISTANTE ($TAILLE_DISTANTE octets)"
```

Cette version corrige la cause de fond (`pipefail`, garantissant que tout échec dans la chaîne `tar`/`ssh` interrompt effectivement le script avec un code d'erreur explicite) et ajoute une vérification complémentaire indépendante du code de sortie technique du pipeline lui-même (une simple absence d'erreur de code de sortie ne garantissant pas nécessairement, dans l'absolu, l'intégrité complète du contenu transféré) — une taille de fichier anormalement faible du côté distant est un signal supplémentaire de transfert incomplet, cohérent avec la limite déjà soulignée en section 7.4 du cours : `set -e`/`pipefail` détectent des échecs au sens du code de sortie, pas nécessairement toute anomalie de résultat métier, d'où l'intérêt de cette vérification complémentaire explicite.

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur système junior pour robustifier un ensemble de scripts fragiles en appliquant systématiquement `set -euo pipefail`, `trap`, et une structure professionnelle de gestion d'erreurs, en observant concrètement l'effet de chaque option.

## Architecture

Une machine Linux avec Bash.

## Objectifs
- Observer concrètement le comportement de `set -e`, `set -u` et `set -o pipefail` isolément et combinés
- Utiliser `trap` pour garantir un nettoyage
- Déboguer un script avec `set -x`
- Reproduire et corriger le scénario de sauvegarde silencieusement défaillante du cours

## Préparation

```bash
bash --version
```

## Étapes

### Étape 1 — Observer `set -e`, `set -u`, `set -o pipefail` isolément

```bash
echo "=== Sans set -e ==="
bash -c 'false; echo "Cette ligne s affiche quand meme"'

echo "=== Avec set -e ==="
bash -c 'set -e; false; echo "Cette ligne ne devrait jamais s afficher"' || echo "Script interrompu comme attendu, code: $?"

echo "=== Sans set -u ==="
bash -c 'echo "Valeur: $variable_inexistante"'

echo "=== Avec set -u ==="
bash -c 'set -u; echo "Valeur: $variable_inexistante"' || echo "Erreur détectée comme attendu"

echo "=== Sans pipefail ==="
bash -c 'false | true; echo "Code sans pipefail: $?"'

echo "=== Avec pipefail ==="
bash -c 'set -o pipefail; false | true; echo "Code avec pipefail: $?"'
```
*Question :* pour chacun des trois tests, notez le comportement observé et confirmez qu'il correspond exactement à la description du cours.

### Étape 2 — `trap` pour un nettoyage garanti

```bash
cat << 'EOF' > /tmp/lab_trap.sh
#!/usr/bin/env bash
set -euo pipefail

FICHIER_TEMP=$(mktemp)
echo "Fichier temporaire créé: $FICHIER_TEMP"

nettoyer() {
    echo "Nettoyage: suppression de $FICHIER_TEMP"
    rm -f "$FICHIER_TEMP"
}
trap nettoyer EXIT

echo "Travail en cours..."
echo "test" > "$FICHIER_TEMP"

if [[ "${1:-}" == "echec" ]]; then
    echo "Simulation d'un échec"
    false
fi

echo "Travail terminé avec succès"
EOF
chmod +x /tmp/lab_trap.sh

echo "=== Exécution normale ==="
/tmp/lab_trap.sh
ls /tmp/tmp.* 2>/dev/null || echo "Aucun fichier temporaire résiduel, comme attendu"

echo "=== Exécution avec échec simulé ==="
/tmp/lab_trap.sh echec || echo "Script terminé en erreur (attendu), code: $?"
ls /tmp/tmp.* 2>/dev/null || echo "Aucun fichier temporaire résiduel malgré l'échec, grâce au trap"
```
*Question :* le fichier temporaire est-il bien supprimé dans les deux cas (succès et échec) ? Qu'est-ce que cela démontre concrètement sur la garantie offerte par `trap ... EXIT` ?

### Étape 3 — Débogage avec `set -x`

```bash
cat << 'EOF' > /tmp/lab_debug.sh
#!/usr/bin/env bash
set -euo pipefail

calculer() {
    local a="$1"
    local b="$2"
    echo $((a + b))
}

resultat=$(calculer 5 3)
echo "Résultat: $resultat"
EOF
chmod +x /tmp/lab_debug.sh

echo "=== Exécution normale ==="
/tmp/lab_debug.sh

echo "=== Exécution en mode trace ==="
bash -x /tmp/lab_debug.sh
```
*Question :* que révèle précisément le mode trace sur l'ordre d'évaluation des expansions de variables et l'appel de la fonction, comparé à une exécution normale sans visibilité sur ces étapes intermédiaires ?

### Étape 4 — Reproduire et corriger le scénario de sauvegarde silencieuse

```bash
mkdir -p /tmp/lab_source_data
echo "donnee importante" > /tmp/lab_source_data/fichier.txt

cat << 'EOF' > /tmp/lab_backup_fautif.sh
#!/usr/bin/env bash
# Version SANS pipefail, reproduisant le piège du cours
tar -czf - /tmp/lab_source_data 2>/dev/null | (sleep 1; exit 1) | cat > /tmp/lab_archive_fautive.tar.gz
echo "Code de sortie du pipeline: $?"
echo "Sauvegarde signalée comme réussie (à tort)"
EOF
chmod +x /tmp/lab_backup_fautif.sh
/tmp/lab_backup_fautif.sh
ls -la /tmp/lab_archive_fautive.tar.gz
```
*Résultat attendu :* le script rapporte un succès (code 0) malgré l'échec volontairement introduit au milieu du pipeline (`exit 1`).

```bash
cat << 'EOF' > /tmp/lab_backup_corrige.sh
#!/usr/bin/env bash
set -euo pipefail
tar -czf - /tmp/lab_source_data 2>/dev/null | (sleep 1; exit 1) | cat > /tmp/lab_archive_corrigee.tar.gz
echo "Cette ligne ne devrait jamais s'afficher"
EOF
chmod +x /tmp/lab_backup_corrige.sh
/tmp/lab_backup_corrige.sh || echo "Échec correctement détecté, code: $?"
```

## Commandes de référence
`set -e`/`-u`/`-o pipefail`, `trap ... EXIT`, `bash -x`

## Résultats attendus
- Les six comportements observés à l'étape 1, confirmés conformes au cours
- Preuve que le fichier temporaire est supprimé dans les deux cas à l'étape 2
- Sortie du mode trace de l'étape 3 avec commentaire
- Comparaison script fautif/corrigé de l'étape 4, avec le code de sortie de chacun

## Questions

1. Dans l'étape 1, pourquoi le test `set -u` échoue-t-il même si la variable inexistante est utilisée dans un simple `echo`, une commande qui normalement ne devrait jamais "échouer" au sens propre ? Le mécanisme de détection de `set -u` intervient-il au même niveau que celui de `set -e` ?
2. Dans l'étape 2, si vous aviez supprimé la ligne `trap nettoyer EXIT` du script, et testé uniquement le cas d'échec (`echec`), le fichier temporaire aurait-il été supprimé ? Pourquoi ce comportement serait-il problématique dans un vrai script de production exécuté fréquemment ?
3. Dans l'étape 4, pourquoi le script fautif affiche-t-il malgré tout "Code de sortie du pipeline: $?" avec une valeur qui ne reflète pas l'échec réel ? Ce comportement aurait-il été différent si l'affichage de `$?` avait été fait immédiatement après une commande simple plutôt qu'après un pipeline complexe ?

## Troubleshooting (panne à diagnostiquer)

**Situation fournie :** après avoir ajouté `set -euo pipefail` à un script existant qui fonctionnait auparavant, un étudiant constate que le script s'arrête immédiatement dès sa première ligne utile, avec une erreur mentionnant une variable non liée (`unbound variable`), alors que cette même variable était utilisée sans problème apparent avant l'ajout de ces options.

**Démarche attendue :**
1. Symptôme : arrêt immédiat avec erreur de variable non définie, uniquement après ajout de `set -euo pipefail`
2. Hypothèse : le script utilise probablement un paramètre positionnel (`$1`, `$2`...) sans argument correspondant réellement fourni lors de ce test précis, ou une variable d'environnement supposée définie par le contexte d'exécution habituel mais absente dans ce test isolé — un cas exactement décrit en section 7.2 du cours (piège des paramètres positionnels non fournis sous `set -u`)
3. Vérification : identifier précisément quelle variable est signalée dans le message d'erreur, et vérifier si le script est exécuté avec le nombre d'arguments qu'il attend normalement en usage réel
4. Isolation : relancer le script avec les arguments complets habituellement fournis en production, et confirmer si l'erreur disparaît dans ce contexte correctement paramétré
5. Correction : si l'argument est réellement optionnel dans la logique du script, utiliser la syntaxe de valeur par défaut (`${1:-valeur_par_defaut}`) plutôt que d'accéder directement à `$1` ; si l'argument est réellement obligatoire, ajouter une vérification explicite de `$#` avant tout accès aux paramètres positionnels, avec un message d'erreur clair plutôt que de laisser `set -u` produire un message technique moins immédiatement compréhensible pour l'utilisateur final du script
6. Vérification : relancer le script à la fois avec et sans l'argument concerné (selon qu'il est censé être obligatoire ou optionnel), confirmant un comportement approprié dans les deux cas
7. Documentation : noter cette interaction comme un rappel que l'ajout de `set -euo pipefail` à un script existant révèle souvent des hypothèses implicites non vérifiées (variables supposées toujours définies, arguments supposés toujours fournis) qui fonctionnaient auparavant par simple absence de vérification — l'apparition de telles erreurs après l'ajout de ces options est un signe que le script gagne en rigueur, pas un signe que ces options sont elles-mêmes défectueuses

## Validation

Le TP est validé si l'étudiant fournit :
- Les 4 résultats/comparaisons demandés
- Des réponses argumentées aux 3 questions
- Une identification correcte de la cause du scénario de troubleshooting (hypothèse implicite révélée par set -u), avec une correction appropriée selon le caractère obligatoire ou optionnel de l'argument concerné

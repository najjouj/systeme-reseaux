# UE5 — Module 32 : PowerShell Fondamentaux

*Module pivot de l'UE5 — compare systématiquement et en profondeur PowerShell à Bash déjà maîtrisé (UE4), autour de la différence philosophique centrale : pipeline d'objets vs flux de texte.*

---

# PARTIE I — COURS

## 1. Titre du cours
PowerShell en profondeur : cmdlets, objets et le pipeline — une philosophie différente de Bash

## 2. Niveau
Élève ingénieur — cycle ingénieur, UE5 administration Windows (S2)

## 3. Durée
CM : 4h (module pivot — +3h TD, +6h TP en Parties II et III, volume TP élevé car module fondateur pour la suite de l'UE5)

## 4. Prérequis
- UE4 complète (Modules 23-26) : syntaxe Bash, pipelines texte, robustesse — mobilisée en comparaison systématique
- L'ensemble des commandes PowerShell déjà utilisées de façon empirique dans les modules 27 à 31 (sans explication syntaxique approfondie jusqu'ici)

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Expliquer la convention de nommage Verbe-Nom des cmdlets PowerShell
- Expliquer la différence fondamentale entre le pipeline PowerShell (objets) et le pipeline Bash (texte), avec ses implications pratiques
- Utiliser variables, conditions, boucles et fonctions PowerShell avec une syntaxe rigoureuse
- Manipuler des objets dans le pipeline (`Select-Object`, `Where-Object`, `Sort-Object`, `ForEach-Object`)
- Écrire une fonction PowerShell avec paramètres typés et validation
- Comparer explicitement une tâche réalisée en Bash (UE4) et son équivalent PowerShell, en identifiant où chaque approche excelle

## 6. Motivation

Vous avez déjà utilisé PowerShell de façon intensive dans les modules 27 à 31 sans que sa syntaxe ne soit formellement expliquée — un choix pédagogique assumé, à l'image de la façon dont Bash a été pratiqué abondamment dans les UE1-UE3 avant d'être formalisé en UE4. Ce module comble maintenant cette lacune, avec un objectif clairement annoncé dès le programme général : **comparer systématiquement Bash et PowerShell**, pour faire ressortir une différence philosophique fondamentale entre les deux mondes.

Bash (UE4) manipule des **flux de texte** : chaque commande produit du texte, que la suivante doit reparser (avec `awk`, `cut`, `grep`, déjà étudiés en UE4-M24) pour en extraire une information structurée. PowerShell manipule des **objets** : chaque cmdlet produit des objets structurés et typés, dont les propriétés sont directement accessibles par la commande suivante dans le pipeline, sans jamais avoir besoin de reparser du texte. Cette différence n'est pas un détail syntaxique — c'est un choix de conception qui change fondamentalement la façon d'écrire un script, avec des avantages et des inconvénients réels de chaque côté, qu'un ingénieur évoluant entre les deux écosystèmes doit comprendre en profondeur.

## 7. Concepts fondamentaux

### 7.1 Cmdlets — convention Verbe-Nom

Une **cmdlet** (prononcé "command-let") suit systématiquement une convention de nommage **Verbe-Nom** (`Get-Service`, `Start-Process`, `New-Item`, `Remove-Item`) — une régularité qui n'existe pas dans l'écosystème Unix, où les noms de commandes (`ls`, `grep`, `awk`) sont historiques et hétérogènes, sans convention unifiée.

**Verbes standardisés courants :** `Get` (obtenir/lire), `Set` (modifier), `New` (créer), `Remove` (supprimer), `Start`/`Stop` (démarrer/arrêter), `Add`/`Remove` (ajouter/retirer d'une collection) — cette régularité rend PowerShell plus **prévisible et découvrable** qu'un écosystème de commandes Unix hétérogènes : face à un nouveau domaine (module ActiveDirectory, DhcpServer, DnsServer, déjà rencontrés), on peut raisonnablement deviner qu'il existe une cmdlet `Get-ADUser`, `New-DhcpServerv4Scope`, etc., sans avoir à mémoriser un nom de commande totalement arbitraire.

```powershell
Get-Command -Verb Get -Noun *Service*   # découvrir les cmdlets disponibles par convention
Get-Help Get-Service -Examples            # aide intégrée, équivalent enrichi de man sous Linux
```

### 7.2 Le pipeline d'objets — la différence fondamentale avec Bash

**Rappel direct UE4-M24 :** un pipe Bash (`cmd1 | cmd2`) transmet un **flux de texte brut** — `cmd2` doit reparser ce texte (souvent avec `awk`/`grep`/`cut`) pour en extraire l'information structurée dont elle a besoin, une opération fragile si le format de sortie de `cmd1` change ou contient des cas limites imprévus (rappel des nombreux pièges de parsing déjà rencontrés tout au long du programme : `ps`, `ip neigh`, `lsof`, `dig`...).

**Le pipeline PowerShell transmet des objets .NET structurés**, pas du texte : chaque propriété d'un objet reste directement accessible par son nom, sans aucun reparsing nécessaire.

```powershell
# PowerShell : accès direct à la propriété, sans reparsing
Get-Process | Where-Object {$_.CPU -gt 100} | Select-Object Name, CPU

# Équivalent Bash conceptuel (UE4) : nécessiterait de parser la sortie texte de `ps`
ps aux | awk '$3 > 100 {print $11, $3}'
```

**`$_` — la variable courante du pipeline :** à l'intérieur d'un bloc de script (`{...}`) traitant chaque élément du pipeline un par un, `$_` représente l'objet actuellement traité — un concept sans équivalent syntaxique direct en Bash, où chaque élément d'un flux de texte n'est qu'une ligne à reparser, pas un objet dont on accède aux propriétés nommées.

**Avantages du pipeline d'objets (par rapport au texte Bash) :**
- Aucun risque de "piège de parsing" (rappel des nombreux exemples rencontrés dans tout le programme) — une propriété nommée reste accessible quelle que soit sa représentation textuelle éventuelle
- Manipulation directement typée (un nombre reste un nombre, une date reste une date, sans conversion de chaîne nécessaire)

**Avantages du pipeline de texte Bash (par rapport aux objets PowerShell) :**
- Universalité : n'importe quel programme Unix, même très ancien, produit et consomme du texte, permettant une interopérabilité totale entre outils sans jamais se soucier de compatibilité de type d'objet
- Légèreté et rapidité pour des traitements simples, sans le surcoût de sérialisation/désérialisation d'objets structurés
- Lisibilité directe du flux intermédiaire (un flux de texte peut être inspecté visuellement à toute étape d'un pipeline, un objet PowerShell nécessite un affichage formaté pour être lisible)

**Ce n'est pas un jugement de supériorité d'un système sur l'autre** — les deux philosophies répondent à des besoins et contextes historiques différents, et un ingénieur compétent doit savoir choisir l'outil adapté à chaque contexte plutôt que de considérer l'un comme universellement meilleur.

### 7.3 Variables et types

```powershell
$nom = "Alice"                    # chaîne
$age = 30                          # entier
$actif = $true                      # booléen
$liste = @("a", "b", "c")            # tableau
$dictionnaire = @{cle1="valeur1"; cle2="valeur2"}   # table de hachage

Write-Output "Bonjour $nom, vous avez $age ans"   # interpolation directe dans une chaîne entre guillemets doubles
```

**Différence avec Bash (rappel UE4-M23) :** PowerShell est **typé** — une variable contenant un entier reste un entier utilisable directement dans une opération arithmétique, sans le contexte spécial `$(( ))` nécessaire en Bash où tout est fondamentalement une chaîne de caractères. Cette différence de typage est directement liée à la philosophie d'objets du pipeline (7.2) : PowerShell manipule des types de données réels de bout en bout, Bash manipule uniformément du texte.

### 7.4 Conditions et boucles

```powershell
if ($age -ge 18) {
    Write-Output "Majeur"
} elseif ($age -ge 13) {
    Write-Output "Adolescent"
} else {
    Write-Output "Enfant"
}

foreach ($service in Get-Service) {
    if ($service.Status -eq "Stopped") {
        Write-Output "$($service.Name) est arrêté"
    }
}

$i = 0
while ($i -lt 5) {
    Write-Output "Itération $i"
    $i++
}
```

**Opérateurs de comparaison — une différence syntaxique notable avec Bash :** PowerShell utilise des opérateurs textuels (`-eq`, `-ne`, `-gt`, `-lt`, `-ge`, `-le`) de façon **unifiée pour tous les types** (nombres et chaînes), contrairement à Bash qui distingue strictement `-eq` (numérique) de `==`/`=` (chaîne), une source d'erreur fréquente déjà signalée en UE4-M23 — PowerShell élimine cette classe d'erreur en unifiant la syntaxe, bien que le comportement exact dépende du type réel des opérandes comparés.

### 7.5 Fonctions PowerShell — paramètres typés

```powershell
function Verifier-Service {
    param(
        [Parameter(Mandatory=$true)]
        [string]$NomService,

        [int]$SeuilTentatives = 3
    )

    $service = Get-Service -Name $NomService -ErrorAction SilentlyContinue
    if (-not $service) {
        Write-Error "Service introuvable: $NomService"
        return $false
    }
    return $service.Status -eq "Running"
}

$resultat = Verifier-Service -NomService "Spooler"
```

**Comparaison directe avec les fonctions Bash (UE4-M23) :** PowerShell offre nativement des paramètres **nommés**, **typés**, et potentiellement **obligatoires** (`Mandatory=$true`) avec validation intégrée — là où Bash accède aux arguments uniquement par position (`$1`, `$2`), sans validation de type native, nécessitant une vérification manuelle explicite (rappel des bonnes pratiques de validation d'arguments déjà enseignées en UE4-M23/M25). Ce n'est pas seulement une commodité syntaxique : elle réduit structurellement une classe d'erreurs (argument manquant, type incorrect) que Bash laisse à la responsabilité du développeur de vérifier lui-même.

**Retour de valeur — autre différence notable :** une fonction PowerShell peut `return` directement une valeur de **n'importe quel type** (pas seulement un code de sortie 0-255 comme rappelé pour Bash en UE4-M23) — une conséquence directe de la philosophie d'objets déjà soulignée en 7.2, éliminant le besoin de la convention Bash consistant à capturer une sortie `echo` via une substitution de commande pour simuler un retour de valeur typée.

### 7.6 Manipulation du pipeline — cmdlets essentielles

```powershell
Get-Process | Where-Object {$_.CPU -gt 50}                  # filtrage (équivalent conceptuel de grep/awk)
Get-Service | Sort-Object -Property Status -Descending          # tri (équivalent conceptuel de sort)
Get-Process | Select-Object Name, Id, CPU                          # sélection de propriétés (équivalent conceptuel de cut/awk)
Get-Service | ForEach-Object { Write-Output "$($_.Name): $($_.Status)" }   # itération explicite
Get-Process | Group-Object -Property Company                          # regroupement (équivalent conceptuel de uniq -c après sort)
Get-Process | Measure-Object -Property CPU -Sum -Average                # agrégation (équivalent conceptuel des accumulations awk)
```

**Tableau de correspondance conceptuelle avec Bash (UE4-M24) :**

| Besoin | Bash (texte) | PowerShell (objets) |
|---|---|---|
| Filtrer | `grep`, `awk '$3 > 100'` | `Where-Object` |
| Trier | `sort` | `Sort-Object` |
| Extraire des colonnes | `cut`, `awk '{print $1}'` | `Select-Object` |
| Dédupliquer/compter | `sort \| uniq -c` | `Group-Object` |
| Traiter élément par élément | `while read -r` | `ForEach-Object` |

## 8. Architecture

```
┌────────────────────────────────────────────────────────────┐
│  Bash (UE4)                    PowerShell (UE5)                   │
│                                                                    │
│  cmd1 ──texte──▶ | ──▶ cmd2       Get-X ──objets──▶ | ──▶ Where-Object │
│  (reparsing nécessaire            (propriétés directement          │
│   via awk/grep/cut)                accessibles par nom)              │
│                                                                    │
│  $1, $2, $@ (positionnel,        param() (nommé, typé,               │
│  non typé)                        validation intégrée)                │
│                                                                    │
│  return = code 0-255 uniquement   return = n'importe quel type        │
│  (echo pour "retourner" une        directement                        │
│  donnée réelle)                                                     │
└────────────────────────────────────────────────────────────┘
```

## 9. Fonctionnement interne

PowerShell est construit sur le framework .NET — chaque objet transitant dans le pipeline est une véritable instance d'objet .NET, avec ses propres méthodes et propriétés définies par le système de types .NET sous-jacent, pas une simple structure de données ad hoc propre à PowerShell. C'est cette fondation qui explique la richesse et la cohérence du système d'objets : `Get-Process` retourne des objets `System.Diagnostics.Process`, les mêmes objets que manipulerait un développeur .NET écrivant du code C#, offrant une continuité directe entre l'administration système en ligne de commande et le développement logiciel dans l'écosystème Microsoft — une différence architecturale profonde avec Bash, qui n'a aucune notion de système de types structuré sous-jacent à ses flux de texte.

## 10. Démonstration

```powershell
# Découvrir la structure d'un objet PowerShell
Get-Process | Get-Member | Select-Object -First 10

# Comparer visuellement flux d'objets et affichage textuel final
Get-Service | Where-Object {$_.Status -eq "Running"} | Measure-Object
```

## 11. Commandes / outils

| Cmdlet | Rôle | Équivalent conceptuel Bash |
|---|---|---|
| `Get-Member` | Découvrir les propriétés/méthodes d'un objet | (aucun équivalent direct, `type`/`man` partiellement) |
| `Where-Object` | Filtrer le pipeline | `grep`/`awk` conditionnel |
| `Select-Object` | Sélectionner des propriétés | `cut`/`awk '{print}'` |
| `Sort-Object` | Trier | `sort` |
| `ForEach-Object` | Itérer élément par élément | `while read -r` |
| `Group-Object` | Regrouper par valeur de propriété | `sort \| uniq -c` |
| `Measure-Object` | Statistiques (somme, moyenne, compte) | `awk` avec accumulation |

**Erreur fréquente :** tenter de traiter la **sortie affichée** d'une cmdlet comme du texte à reparser (par exemple avec `-match` sur une représentation textuelle formatée), alors que la bonne pratique PowerShell consiste à accéder directement aux propriétés de l'objet sous-jacent (rappel section 7.2) — un réflexe Bash qui, transposé sans adaptation à PowerShell, ignore l'avantage principal du pipeline d'objets.

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** un ingénieur, expérimenté en Bash mais débutant en PowerShell, écrit un script PowerShell pour identifier les 5 processus consommant le plus de mémoire, en réutilisant ses réflexes Bash : il exécute `Get-Process`, capture la sortie affichée comme texte, puis tente de la reparser avec des expressions régulières PowerShell (`-match`) pour extraire les valeurs numériques de mémoire — un script fragile, sensible au moindre changement de format d'affichage, et significativement plus complexe que nécessaire.

**Analyse attendue :** ce scénario illustre directement le piège pédagogique central de ce module (section 7.2) : appliquer une habitude Bash (traiter une sortie comme du texte à reparser) dans un contexte PowerShell où elle est structurellement inadaptée et contre-productive. La solution PowerShell idiomatique, bien plus courte, robuste et lisible, exploite directement le pipeline d'objets :

```powershell
Get-Process | Sort-Object -Property WorkingSet -Descending | Select-Object -First 5 Name, WorkingSet
```

Cette unique ligne accède directement à la propriété numérique `WorkingSet` (mémoire de travail) de chaque objet processus, la trie, et sélectionne les 5 premiers résultats avec leurs propriétés pertinentes — sans aucun risque de piège de parsing, contrairement à l'approche initiale de l'ingénieur. Cette étude de cas illustre une leçon transférable au-delà de PowerShell lui-même : maîtriser un nouvel écosystème technique implique de comprendre et d'adopter sa philosophie propre, pas seulement de traduire syntaxiquement des réflexes acquis dans un système différent — une compétence de transposition consciente, plutôt qu'une simple traduction mécanique.

## 15. Sécurité

- PowerShell dispose d'une politique d'exécution (`Execution Policy`) contrôlant si et comment des scripts peuvent être exécutés sur une machine — un mécanisme de sécurité sans équivalent direct et natif en Bash (où un script exécutable s'exécute simplement s'il en a la permission, rappel UE3-M14), reflétant une préoccupation historique différente sur la prolifération de scripts malveillants dans l'écosystème Windows
- La richesse du système d'objets PowerShell, combinée à un accès profond à .NET et à l'API Windows, en fait un outil puissant mais aussi potentiellement dangereux entre de mauvaises mains — PowerShell est fréquemment utilisé comme vecteur d'attaque dans des scénarios de post-exploitation, justifiant une vigilance accrue sur les scripts PowerShell exécutés dans un environnement de production (journalisation renforcée, restriction de l'Execution Policy, approfondi en UE8)
- Comme pour Bash (rappel UE4-M23/M25), la validation rigoureuse des paramètres d'entrée d'une fonction PowerShell (section 7.5, paramètres typés et obligatoires) est autant une question de robustesse que de sécurité — un paramètre non validé peut être un vecteur d'injection ou de comportement inattendu, quel que soit le langage de script utilisé

## 16. Automatisation

Script PowerShell illustrant la philosophie objet appliquée à un besoin déjà rencontré en Bash (UE3-M16, Service Health Guardian) — comparaison directe des deux approches pour un même besoin fonctionnel.

```powershell
function Get-ServicesEnEchec {
    param(
        [string[]]$ServicesCritiques = @("Spooler", "DHCPServer", "DNS")
    )

    Get-Service -Name $ServicesCritiques -ErrorAction SilentlyContinue |
        Where-Object { $_.Status -ne "Running" } |
        Select-Object Name, Status, StartType |
        ForEach-Object {
            [PSCustomObject]@{
                Service   = $_.Name
                Etat      = $_.Status
                Attendu   = "Running"
                Categorie = if ($_.StartType -eq "Automatic") { "CRITIQUE" } else { "INFO" }
            }
        }
}

$resultats = Get-ServicesEnEchec
if ($resultats) {
    $resultats | Format-Table -AutoSize
} else {
    Write-Output "Tous les services critiques sont actifs"
}
```

*Note pédagogique :* `[PSCustomObject]` permet de créer un objet structuré personnalisé à la volée, une construction sans équivalent direct en Bash — en Bash, un "objet" équivalent serait au mieux une ligne de texte formatée (CSV, ou une structure ad hoc), toujours soumise au risque de reparsing déjà discuté.

## 17. Troubleshooting

Scénario court : un script PowerShell utilisant `Where-Object {$_.Name -eq "MonService"}` ne trouve jamais le service recherché, alors que celui-ci apparaît bien listé par `Get-Service` seul. Démarche : vérifier la valeur exacte de la propriété `Name` de l'objet (souvent différente du nom d'affichage visible, rappel du piège déjà signalé au module 27 entre nom d'affichage et `SamAccountName`/nom technique — un piège directement comparable ici entre le nom affiché et la propriété réelle `Name`/`DisplayName` d'un service) → utiliser `Get-Member` ou simplement afficher l'objet complet pour identifier la propriété exacte et sa valeur précise → corriger la condition de filtrage pour cibler la bonne propriété.

## 18. Synthèse

- Les cmdlets suivent une convention Verbe-Nom régulière, rendant PowerShell plus découvrable qu'un écosystème de commandes Unix hétérogènes
- Le pipeline PowerShell transmet des objets structurés, éliminant le risque de piège de parsing omniprésent en Bash, au prix d'une moindre universalité et d'un surcoût de sérialisation
- PowerShell est typé nativement ; les opérateurs de comparaison sont unifiés (`-eq`, `-gt`...) pour tous les types, contrairement à la distinction stricte Bash entre opérateurs numériques et de chaîne
- Les fonctions PowerShell offrent des paramètres nommés, typés et potentiellement obligatoires nativement, réduisant une classe d'erreurs que Bash laisse à la vigilance du développeur
- `Where-Object`, `Select-Object`, `Sort-Object`, `ForEach-Object`, `Group-Object` sont les équivalents conceptuels directs de `grep`/`awk`, `cut`/`awk`, `sort`, `while read`, `sort | uniq -c` déjà maîtrisés en Bash
- Transposer un réflexe Bash sans adaptation dans un contexte PowerShell (traiter la sortie comme du texte à reparser) est une erreur d'apprentissage classique à éviter consciemment

## 19. Questions d'évaluation

**QCM**
1. La convention de nommage des cmdlets PowerShell est :
 a) totalement libre b) Verbe-Nom (ex: Get-Service) c) Nom-Verbe d) uniquement des abréviations

2. Le pipeline PowerShell transmet :
 a) du texte brut, comme Bash b) des objets structurés et typés c) uniquement des nombres d) rien, PowerShell n'a pas de pipeline

**Questions ouvertes**
- Expliquez la différence fondamentale entre le pipeline Bash et le pipeline PowerShell, avec un exemple illustrant l'avantage du second pour éviter un piège de parsing déjà rencontré dans le programme.
- Expliquez pourquoi une fonction PowerShell peut retourner directement une donnée de n'importe quel type, contrairement à une fonction Bash limitée à un code de sortie 0-255.

**Exercice technique**
- Écrivez la commande PowerShell permettant d'afficher les 3 services consommant le plus de mémoire (propriété `WorkingSet`), triés par ordre décroissant.

**Problème de diagnostic**
- Un filtre `Where-Object` sur la propriété `Name` d'un objet ne trouve jamais le résultat attendu, alors que l'objet apparaît bien listé sans filtre. Proposez la démarche de diagnostic.

## 20. Mini-projet

Développer un script PowerShell qui reprend, en l'adaptant à la philosophie objet plutôt qu'en la traduisant mécaniquement, un script Bash déjà écrit dans le programme (au choix de l'étudiant, parmi les outils d'audit développés en UE1-UE4) :
1. Identifier les endroits où le script Bash original s'appuie sur un pipeline de traitement de texte (`grep`/`awk`/`sort`/`uniq`)
2. Réécrire chaque étape en exploitant le pipeline d'objets PowerShell (`Where-Object`/`Select-Object`/`Sort-Object`/`Group-Object`), sans reparsing de texte
3. Comparer la robustesse et la lisibilité des deux versions, en identifiant explicitement où la version PowerShell élimine un risque de piège de parsing présent dans l'original Bash
4. Documenter cette comparaison comme livrable pédagogique, illustrant concrètement la transposition consciente entre les deux philosophies

---

# PARTIE II — TD

### Niveau 1 — Compréhension

**Exercice 1.1**
Expliquez la convention de nommage Verbe-Nom des cmdlets PowerShell et son avantage par rapport aux noms de commandes Unix.

**Exercice 1.2**
Expliquez la différence fondamentale entre le pipeline Bash (UE4) et le pipeline PowerShell.

**Exercice 1.3**
Citez trois cmdlets PowerShell de manipulation du pipeline et leur équivalent conceptuel Bash correspondant.

### Niveau 2 — Application

**Exercice 2.1**
Écrivez la commande PowerShell permettant de lister uniquement les processus dont le nom contient "chrome" (insensible à la casse par défaut en PowerShell).

**Exercice 2.2**
Écrivez une fonction PowerShell `Est-Pair` prenant un paramètre entier obligatoire `Nombre` et retournant `$true` s'il est pair, `$false` sinon.

**Exercice 2.3**
Écrivez la commande PowerShell permettant de compter le nombre de services actuellement à l'état "Running".

### Niveau 3 — Analyse

**Exercice 3.1**
Un ingénieur habitué à Bash écrit `Get-Process | Out-String | Select-String "chrome"` pour trouver un processus, capturant la sortie en texte avant de la filtrer avec une recherche de motif. Expliquez pourquoi cette approche, bien que fonctionnelle, n'exploite pas la philosophie PowerShell, et proposez la commande idiomatique équivalente.

**Exercice 3.2**
Une fonction PowerShell définit un paramètre sans `[Parameter(Mandatory=$true)]` ni valeur par défaut. Un utilisateur appelle cette fonction sans fournir ce paramètre. Quel sera le comportement, et en quoi diffère-t-il de la vérification manuelle de `$#`/`$1` habituellement nécessaire en Bash (rappel UE4-M23) ?

### Niveau 4 — Problème complexe

**Exercice 4.1**
Reprenez le pipeline Bash suivant, déjà construit et expliqué en UE4-Module 24 (étude de cas du cours de ce module) pour identifier les adresses IP responsables du plus grand nombre d'erreurs HTTP 500 dans un log d'accès web :
```bash
grep " 500 " access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -10
```
En supposant que ce même log soit accessible sous forme d'objets structurés PowerShell (chaque ligne représentée par un objet avec des propriétés `IP`, `StatusCode`, `Timestamp`, par exemple après import via une cmdlet appropriée), réécrivez ce pipeline en PowerShell idiomatique, en expliquant pour chaque étape la cmdlet utilisée et son équivalent conceptuel dans le pipeline Bash d'origine.

---

## Correction du TD

**1.1** — La convention Verbe-Nom (ex : `Get-Service`, `New-Item`) impose une structure prévisible à toutes les cmdlets PowerShell, quel que soit le module ou domaine concerné. Cet avantage est direct par rapport aux noms de commandes Unix (`ls`, `grep`, `awk`), historiquement hétérogènes et sans convention unifiée : face à un nouveau domaine (par exemple le module DhcpServer déjà rencontré en UE5-M29), on peut raisonnablement anticiper l'existence d'une cmdlet `Get-DhcpServerv4Scope` ou similaire, une découvrabilité que l'écosystème Unix, malgré ses qualités propres, n'offre pas nativement de la même façon.

**1.2** — Le pipeline Bash transmet un flux de **texte brut** entre commandes, que la commande réceptrice doit reparser (souvent avec `awk`/`grep`/`cut`) pour en extraire une information structurée — une opération potentiellement fragile face à des cas limites du format textuel. Le pipeline PowerShell transmet des **objets structurés et typés** (instances .NET), dont les propriétés restent directement accessibles par leur nom à chaque étape, sans jamais nécessiter de reparsing textuel — éliminant structurellement toute une classe de pièges déjà rencontrés à de multiples reprises dans le programme avec les outils Unix.

**1.3** — `Where-Object` (filtrage, équivalent conceptuel de `grep`/`awk` conditionnel) ; `Select-Object` (sélection de propriétés, équivalent conceptuel de `cut`/`awk '{print}'`) ; `Sort-Object` (tri, équivalent conceptuel de `sort`) — d'autres réponses valides incluraient `ForEach-Object` (équivalent de `while read -r`) ou `Group-Object` (équivalent de `sort | uniq -c`).

**2.1** — `Get-Process | Where-Object {$_.Name -like "*chrome*"}`.

**2.2** — 
```powershell
function Est-Pair {
    param(
        [Parameter(Mandatory=$true)]
        [int]$Nombre
    )
    return ($Nombre % 2 -eq 0)
}
```

**2.3** — `(Get-Service | Where-Object {$_.Status -eq "Running"}).Count` (ou `Get-Service | Where-Object {$_.Status -eq "Running"} | Measure-Object | Select-Object -ExpandProperty Count`).

**3.1** — Cette approche n'exploite pas la philosophie PowerShell car elle convertit délibérément des objets structurés (`Get-Process` produit des objets `System.Diagnostics.Process` avec une propriété `Name` directement accessible) en texte brut (`Out-String`), pour ensuite appliquer une recherche de motif textuel (`Select-String`) — reproduisant artificiellement le comportement d'un pipeline Bash, avec le même risque de fragilité face à un format d'affichage textuel qui pourrait varier, alors que la propriété `Name` de l'objet reste, elle, garantie stable et directement accessible sans conversion. Commande idiomatique équivalente : `Get-Process | Where-Object {$_.Name -like "*chrome*"}`, qui filtre directement sur la propriété structurée de l'objet, sans jamais passer par une représentation textuelle intermédiaire.

**3.2** — Sans `Mandatory=$true` ni valeur par défaut, le paramètre non fourni prendra simplement la valeur `$null` (l'équivalent PowerShell d'une absence de valeur) — la fonction s'exécutera sans erreur immédiate, mais pourrait produire un comportement incorrect ou une erreur ultérieure si le code de la fonction utilise ce paramètre sans vérifier au préalable qu'il n'est pas `$null`. Ce comportement diffère de la vérification manuelle nécessaire en Bash (`if [[ $# -lt 1 ]]`, rappel UE4-M23) principalement par le fait que PowerShell offre une solution **native et déclarative** (l'attribut `Mandatory=$true`) pour rendre un paramètre réellement obligatoire, provoquant une erreur immédiate et un message clair si l'appelant omet de le fournir — alors que Bash nécessite systématiquement une vérification manuelle et explicite écrite par le développeur lui-même pour obtenir un comportement de robustesse équivalent, une différence de philosophie directement liée au système de paramètres typés déjà souligné en section 7.5 du cours.

**4.1** — Réécriture PowerShell idiomatique du pipeline, en supposant des objets `LogEntry` avec propriétés `IP`, `StatusCode`, `Timestamp` :

```powershell
$entreesLog | Where-Object {$_.StatusCode -eq 500} |
    Group-Object -Property IP |
    Sort-Object -Property Count -Descending |
    Select-Object -First 10 Name, Count
```

Correspondance étape par étape avec le pipeline Bash d'origine :
- `Where-Object {$_.StatusCode -eq 500}` correspond à `grep " 500 "` : filtre les entrées correspondant au code d'erreur recherché — mais ici en comparant directement une propriété numérique structurée, sans risque d'ambiguïté de correspondance textuelle (contrairement à `grep " 500 "`, qui pourrait potentiellement correspondre à une occurrence non désirée de la chaîne "500" ailleurs dans la ligne, un piège de robustesse que la comparaison structurée élimine nativement)
- `Group-Object -Property IP` correspond à la combinaison `sort | uniq -c` : regroupe et compte automatiquement les occurrences par valeur de propriété, en une seule cmdlet plutôt que deux commandes Bash chaînées (le tri intermédiaire nécessaire à `uniq` en Bash, rappel UE4-M24, n'est ici plus nécessaire, `Group-Object` ne nécessitant aucun tri préalable pour regrouper correctement, contrairement à la contrainte structurelle de `uniq`)
- `Sort-Object -Property Count -Descending` correspond à `sort -rn` : trie les groupes obtenus par fréquence décroissante, en utilisant directement la propriété numérique `Count` générée par `Group-Object`, sans nécessiter de spécifier une colonne par position comme le ferait `sort` sur du texte
- `Select-Object -First 10 Name, Count` correspond à `head -10` combiné implicitement à la sélection des colonnes pertinentes : limite le résultat aux 10 premiers éléments et n'affiche que les propriétés utiles (`Name` correspondant à la valeur d'IP du regroupement, `Count` au nombre d'occurrences)

Cette réécriture illustre concrètement, sur un cas réel déjà résolu en Bash, comment le pipeline d'objets PowerShell simplifie certaines étapes (regroupement/comptage en une seule cmdlet plutôt que deux commandes chaînées) tout en éliminant les risques de correspondance textuelle imprécise déjà identifiés comme des pièges classiques tout au long du programme.

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur système junior pour pratiquer intensivement la philosophie objet de PowerShell, en la comparant systématiquement à des tâches déjà réalisées en Bash, puis pour diagnostiquer un cas de filtrage incorrect lié à une confusion entre nom d'affichage et propriété réelle.

## Architecture

Une machine Windows (Server ou poste de travail) avec PowerShell disponible.

## Objectifs
- Explorer la structure d'objets PowerShell avec `Get-Member`
- Construire des pipelines d'objets équivalents à des pipelines Bash déjà maîtrisés
- Écrire une fonction PowerShell avec paramètres typés et validation
- Diagnostiquer un cas de confusion entre propriété affichée et propriété réelle

## Préparation

```powershell
$PSVersionTable.PSVersion
```

## Étapes

### Étape 1 — Explorer la structure d'un objet

```powershell
Get-Process | Select-Object -First 1 | Get-Member -MemberType Property

Get-Service | Select-Object -First 1 | Get-Member -MemberType Property
```
*Question :* combien de propriétés au total possède un objet `Process` retourné par `Get-Process` ? Comparez ce niveau de structuration à ce qu'offrirait la sortie texte brute de la commande `ps` sous Linux (rappel UE1-M2) pour un besoin de traitement équivalent.

### Étape 2 — Construire des pipelines équivalents à des pipelines Bash

```powershell
# Équivalent PowerShell de : ps aux | sort -k4 -rn | head -5  (tri par mémoire, rappel UE1/UE3)
Get-Process | Sort-Object -Property WorkingSet -Descending | Select-Object -First 5 Name, WorkingSet

# Équivalent PowerShell de : df -h | awk '$5+0 > 80'  (disques proches de saturation, rappel UE3-M15/M21)
Get-Volume | Where-Object { $_.SizeRemaining -lt ($_.Size * 0.2) } | Select-Object DriveLetter, Size, SizeRemaining

# Équivalent PowerShell de : sort fichier.txt | uniq -c | sort -rn  (comptage de fréquences, rappel UE4-M24)
Get-Process | Group-Object -Property Company | Sort-Object -Property Count -Descending | Select-Object -First 5 Name, Count
```
*Question :* pour chacun des trois pipelines, identifiez précisément quelle cmdlet PowerShell remplace quelle combinaison de commandes Bash, en vous appuyant sur le tableau de correspondance de la section 7.6 du cours.

### Étape 3 — Fonction avec paramètres typés

```powershell
function Test-SeuilDisque {
    param(
        [Parameter(Mandatory=$true)]
        [string]$LettreLecteur,

        [ValidateRange(0,100)]
        [int]$SeuilPourcent = 80
    )

    $volume = Get-Volume -DriveLetter $LettreLecteur -ErrorAction SilentlyContinue
    if (-not $volume) {
        Write-Error "Lecteur $LettreLecteur introuvable"
        return
    }

    $pourcentUtilise = [math]::Round((($volume.Size - $volume.SizeRemaining) / $volume.Size) * 100, 1)

    [PSCustomObject]@{
        Lecteur         = $LettreLecteur
        PourcentUtilise = $pourcentUtilise
        Seuil           = $SeuilPourcent
        Alerte          = $pourcentUtilise -ge $SeuilPourcent
    }
}

Test-SeuilDisque -LettreLecteur "C" -SeuilPourcent 50
Test-SeuilDisque -LettreLecteur "Z"   # lecteur probablement inexistant, pour tester la gestion d'erreur
```
*Question :* que se passe-t-il si vous appelez `Test-SeuilDisque -LettreLecteur "C" -SeuilPourcent 150` ? Quel mécanisme de la fonction empêche cet appel, et à quel principe de robustesse déjà étudié en Bash (UE4-M25, validation d'entrée) cela correspond-il ?

### Étape 4 — Diagnostic d'un filtrage incorrect

```powershell
# Provoquer volontairement le piège nom d'affichage vs propriété réelle
Get-Service | Where-Object {$_.DisplayName -eq "Spouleur d'impression"} 2>$null
Get-Service | Where-Object {$_.Name -eq "Spouleur d'impression"} 2>$null   # échoue si le nom technique diffère

Get-Service -Name "Spooler" | Select-Object Name, DisplayName
```
*À vous de jouer :* expliquez pourquoi la première commande peut échouer à trouver le service même s'il existe bien, selon la langue d'affichage du système, et pourquoi filtrer sur `Name` est structurellement plus fiable que filtrer sur `DisplayName`.

## Commandes de référence
`Get-Member`, `Where-Object`, `Select-Object`, `Sort-Object`, `Group-Object`, `param()`, `[PSCustomObject]`

## Résultats attendus
- Nombre de propriétés d'un objet `Process`, avec comparaison argumentée à `ps` (étape 1)
- Correspondance identifiée pour chacun des 3 pipelines (étape 2)
- Comportement observé avec `-SeuilPourcent 150`, mécanisme identifié (étape 3)
- Explication du piège nom d'affichage vs propriété technique (étape 4)

## Questions

1. Dans l'étape 1, un objet PowerShell possède généralement bien plus de propriétés que ce qu'un simple `ps aux` affiche par défaut sous Linux. Est-ce un avantage systématique de PowerShell, ou cela peut-il aussi présenter un inconvénient (indice : réfléchissez à la lisibilité et à la performance) ?
2. Dans l'étape 2, le second pipeline (volumes proches de saturation) illustre-t-il un cas où PowerShell simplifie réellement par rapport à l'équivalent Bash, ou s'agit-il d'une complexité comparable exprimée différemment ? Justifiez votre avis.
3. Dans l'étape 3, pourquoi la validation `[ValidateRange(0,100)]` est-elle préférable à une vérification manuelle écrite dans le corps de la fonction (`if ($SeuilPourcent -gt 100) {...}`) ? Y a-t-il malgré tout un scénario où une vérification manuelle resterait nécessaire en complément ?

## Troubleshooting (panne à diagnostiquer)

**Situation fournie :** un étudiant écrit un script PowerShell filtrant les processus avec `Where-Object {$_.CPU -gt "50"}` (en comparant à la chaîne `"50"` entre guillemets plutôt qu'au nombre `50`), et constate un comportement de filtrage incohérent et inattendu par rapport à ce qu'il visait.

**Démarche attendue :**
1. Symptôme : filtrage incohérent malgré une condition apparemment simple et correcte en apparence
2. Hypothèse : bien que PowerShell soit typé (rappel section 7.3 du cours) et que ses opérateurs de comparaison soient unifiés syntaxiquement pour tous les types (contrairement à Bash), la comparaison d'une valeur numérique réelle (`$_.CPU`, un nombre) avec une chaîne de caractères (`"50"`) peut entraîner une conversion de type implicite dont le résultat n'est pas toujours celui intuitivement attendu, notamment pour des comparaisons d'ordre (supérieur/inférieur) plutôt que d'égalité stricte
3. Vérification : tester isolément le type réel de `$_.CPU` pour un processus donné (`(Get-Process)[0].CPU.GetType()`) et comparer avec le type de la chaîne `"50"` (`"50".GetType()`), pour confirmer la nature de l'incohérence
4. Isolation : reproduire le problème sur un cas simple et contrôlé (comparer explicitement une valeur numérique connue avec sa représentation en chaîne, dans les deux sens de comparaison) pour bien cerner le comportement réel de conversion de type de PowerShell dans ce contexte précis
5. Correction : retirer les guillemets autour de la valeur numérique de comparaison (`Where-Object {$_.CPU -gt 50}`), laissant PowerShell comparer deux valeurs du même type numérique sans ambiguïté de conversion
6. Vérification : relancer le filtrage corrigé et confirmer un comportement cohérent avec l'intention initiale
7. Documentation : noter cette expérience comme un rappel que, malgré la robustesse générale du typage PowerShell (largement supérieure à l'absence de typage de Bash), une vigilance reste nécessaire sur la cohérence des types manipulés de part et d'autre d'une comparaison — le typage réduit une classe d'erreurs sans l'éliminer totalement, une nuance importante à ne pas negliger même dans un écosystème structurellement plus robuste que Bash sur ce point précis

## Validation

Le TP est validé si l'étudiant fournit :
- Les 4 résultats/observations demandés
- Des réponses argumentées aux 3 questions
- Une identification correcte de la cause du scénario de troubleshooting (comparaison numérique vs chaîne), avec une correction appropriée

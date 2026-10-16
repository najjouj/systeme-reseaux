# UE4 — Module 24 : Pipes, Redirections et Traitement de Texte
## Compléments : ÉVALUATION / PROJET

---

# PARTIE I — ÉVALUATION

**Durée :** 1h30
**Modalités :** documents non autorisés sauf `man`/`help` local si épreuve sur machine
**Barème total :** /20

## Section A — QCM (4 points, 0,5 pt/question)

1. `commande > fichier 2>&1` :
 a) redirige uniquement stdout b) redirige stdout et stderr vers le même fichier c) redirige uniquement stderr d) ne redirige rien

2. `commande1 2>&1 > fichier` (ordre inversé) :
 a) produit le même résultat que `> fichier 2>&1` b) laisse stderr apparaître à l'écran, seul stdout va dans le fichier c) provoque une erreur de syntaxe d) redirige uniquement stderr dans le fichier

3. Un pipe (`|`) connecte par défaut :
 a) stdout et stderr des deux commandes b) uniquement stdout de la première vers stdin de la seconde c) uniquement stderr d) l'entrée standard des deux commandes entre elles

4. `uniq` sans tri préalable :
 a) déduplique correctement toutes les occurrences, même non consécutives b) ne déduplique que les lignes consécutives identiques c) trie automatiquement le fichier d) provoque systématiquement une erreur

5. Sans l'option `-E`, `grep "a|b"` recherche :
 a) les lignes contenant "a" OU "b" b) la chaîne littérale "a|b" c) uniquement les lignes commençant par "a" d) une erreur de syntaxe

6. `sed -i` sur un fichier :
 a) affiche le résultat sans modifier le fichier b) modifie le fichier directement et irréversiblement, sauf sauvegarde explicite c) fonctionne uniquement en lecture seule d) nécessite obligatoirement `-E`

7. `awk -F, '{print $2}'` sur un fichier CSV :
 a) affiche la première colonne b) affiche la deuxième colonne, séparateur virgule c) affiche toutes les colonnes d) provoque une erreur si le fichier contient des virgules

8. `find . -print0 | xargs -0 rm` comparé à `find . | xargs rm` :
 a) sont strictement équivalents dans tous les cas b) la première est robuste face aux noms de fichiers contenant des espaces, contrairement à la seconde c) la seconde est plus rapide sans différence de robustesse d) aucune des deux ne fonctionne avec `rm`

## Section B — Questions ouvertes (6 points)

**Q1 (2 pts).** Expliquez pourquoi l'ordre compte dans une redirection combinée de stdout et stderr, avec un exemple concret des deux ordres et leur résultat respectif.

**Q2 (2 pts).** Expliquez pourquoi `xargs` hérite du même risque que les boucles Bash étudiées au module précédent, et la solution robuste pour l'éviter.

**Q3 (2 pts).** Expliquez la différence de rôle entre `grep`, `sed` et `awk`, avec un exemple de tâche où `awk` serait clairement le choix le plus adapté des trois.

## Section C — Exercice technique (4 points)

Construisez un pipeline (une seule ligne de commande, en chaînant les outils nécessaires) permettant d'afficher les 3 mots les plus fréquents d'un fichier texte `article.txt`, en supposant que chaque mot est déjà séparé par un espace sur chaque ligne.

## Section D — Problème de diagnostic (6 points)

Un script exécute `mon_script.sh 2>&1 > journal.log` dans l'intention de capturer à la fois la sortie normale et les erreurs dans le fichier `journal.log`, mais les erreurs continuent d'apparaître à l'écran.

**a) (2 pts)** Expliquez précisément pourquoi cet ordre de redirection ne produit pas le résultat attendu.

**b) (2 pts)** Donnez la commande corrigée.

**c) (2 pts)** Proposez une méthode de test rapide permettant de vérifier, sur une commande simple, qu'une redirection combinée fonctionne comme attendu avant de l'appliquer à un script plus complexe.

## Grille de correction résumée

| Section | Points | Critères |
|---|---|---|
| A — QCM | /4 | Réponse exacte uniquement |
| B — Questions ouvertes | /6 | Précision technique, mécanisme de redirection/pipe correctement expliqué |
| C — Exercice technique | /4 | Pipeline fonctionnel et correctement ordonné |
| D — Diagnostic | /6 | Explication exacte de l'ordre des redirections, correction exacte, méthode de test pertinente |

---

# PARTIE II — MINI-PROJET (échelle module)

## Cahier des charges

**Titre :** Analyseur de logs en pipeline (Pipeline Log Analyzer)

**Contexte :** Suite à l'étude de cas du cours (identification rapide des IP responsables d'erreurs HTTP 500 lors d'un incident), l'équipe système souhaite un outil généralisant cette approche de pipeline pour analyser rapidement différents types de logs, sans avoir à reconstruire manuellement un pipeline à chaque nouvel incident.

### Objectifs du livrable

Développer un script (Bash) qui, pour un fichier de log donné en argument :
1. Propose plusieurs modes d'analyse activables via des options (`--top-ip`, `--top-status`, `--errors-only`, `--time-range`), chacun construit comme un pipeline combinant `grep`/`awk`/`sort`/`uniq` de façon appropriée
2. Gère correctement les cas où un champ du log peut contenir des espaces internes (rappel direct de l'exercice 4.1 du TD, requête HTTP entre guillemets), en utilisant une approche robuste de découpage par guillemets plutôt qu'un découpage naïf par espace
3. Utilise systématiquement des redirections correctement ordonnées pour séparer les messages d'erreur du script lui-même (fichier introuvable, option invalide) de la sortie d'analyse normale
4. Applique les bonnes pratiques du module 23 (quotation systématique, gestion d'arguments robuste, codes de sortie appropriés)
5. Produit une sortie lisible, avec un en-tête clair indiquant le mode d'analyse utilisé et le fichier traité

### Contraintes techniques
- Chaque mode d'analyse doit être implémenté comme un pipeline réel combinant les outils du module, pas une réimplémentation en boucles Bash explicites là où un pipeline suffit
- Le script doit correctement traiter un fichier de log contenant des champs avec espaces internes (test explicite requis)
- Gestion d'erreurs robuste (fichier absent, option invalide, fichier vide)
- Documentation d'installation et d'utilisation (README), avec un exemple pour chaque mode d'analyse
- Tests démontrés sur au moins 3 scénarios : mode `--top-ip` sur un log de test, mode `--top-status` avec un champ requête contenant des espaces, gestion d'une erreur (fichier absent) avec message clair

### Livrables attendus
- Code source commenté, avec chaque pipeline expliqué par un commentaire précédent
- README (installation, utilisation, modes disponibles, format de log attendu, limites connues)
- Journal de tests montrant les 3 scénarios validés, avec captures
- Rapport court (1 page) sur les choix de conception des pipelines, notamment la gestion des champs avec espaces internes, et les limites connues (ex : format de log supposé relativement standard, ne gère pas nécessairement tous les formats exotiques)

### Grille d'évaluation

| Critère | Pondération |
|---|---|
| Pipelines fonctionnels pour chaque mode d'analyse | 35% |
| Gestion robuste des champs avec espaces internes | 25% |
| Redirections correctement ordonnées, séparation erreurs/résultats | 15% |
| Application des bonnes pratiques du module 23 (quotation, arguments) | 15% |
| Documentation et tests démontrés (3 scénarios) | 10% |

### Modalité
Travail individuel ou en binôme, durée indicative : 3-4 jours. Cet outil pourra être réutilisé et enrichi au module suivant (M25, gestion des erreurs) pour renforcer sa robustesse face aux cas limites, puis au module 26 (automatisation appliquée) comme brique d'un outil de supervision plus large.

# UE4 — Module 25 : Gestion des Erreurs et Bonnes Pratiques
## Compléments : ÉVALUATION / PROJET

---

# PARTIE I — ÉVALUATION

**Durée :** 1h30
**Modalités :** documents non autorisés sauf `man`/`help` local si épreuve sur machine
**Barème total :** /20

## Section A — QCM (4 points, 0,5 pt/question)

1. `set -e` fait terminer un script :
 a) à chaque commande, quel que soit son code de sortie b) dès qu'une commande simple non protégée retourne un code de sortie non nul c) uniquement en cas d'erreur de syntaxe d) jamais automatiquement

2. Une commande dans un `if commande; then` qui échoue, sous `set -e` :
 a) interrompt systématiquement le script b) n'interrompt pas le script, ce comportement étant normal dans ce contexte c) provoque une erreur de syntaxe d) redémarre le script

3. `set -u` détecte :
 a) les erreurs de syntaxe b) l'utilisation d'une variable non définie c) les boucles infinies d) les pipelines mal formés

4. Sans `set -o pipefail`, le code de sortie d'un pipeline reflète :
 a) le succès global de la chaîne b) uniquement le code de sortie de la dernière commande c) toujours 0 d) la somme des codes de sortie

5. `trap nettoyer EXIT` garantit l'exécution de `nettoyer` :
 a) uniquement en cas de succès du script b) quel que soit le mode de sortie du script (succès, erreur, interruption) c) uniquement sur réception de SIGKILL d) jamais si `set -e` est actif

6. Un script de sauvegarde utilisant un pipeline sans `pipefail`, dont un maillon intermédiaire échoue silencieusement :
 a) rapportera toujours un échec clair b) peut rapporter un succès trompeur, masquant l'échec réel c) ne peut jamais se produire d) est automatiquement détecté par `set -e` seul

7. `set -e` seul suffit-il à détecter qu'une commande a réussi (code 0) mais produit un résultat métier incorrect ?
 a) oui, systématiquement b) non, une validation explicite du résultat reste nécessaire c) oui, mais uniquement pour les pipelines d) non, aucun mécanisme Bash ne peut le faire

8. `bash -x monscript.sh` sert à :
 a) exécuter le script plus rapidement b) afficher chaque commande et ses expansions avant exécution, pour le débogage c) corriger automatiquement les erreurs d) chiffrer le script

## Section B — Questions ouvertes (6 points)

**Q1 (2 pts).** Expliquez pourquoi l'absence de `set -o pipefail` peut masquer un échec grave dans un pipeline de sauvegarde, avec un exemple concret.

**Q2 (2 pts).** Expliquez le rôle de `trap ... EXIT` et donnez un exemple de ressource (fichier temporaire, verrou) dont la libération devrait être garantie par ce mécanisme plutôt que par une simple ligne en fin de script.

**Q3 (2 pts).** Expliquez pourquoi `set -e` ne protège pas contre toute forme d'erreur, en distinguant échec technique (code de sortie) et erreur de résultat métier.

## Section C — Exercice technique (4 points)

Écrivez un script minimal illustrant : l'activation de `set -euo pipefail`, la création d'un fichier temporaire, un `trap` garantissant sa suppression en fin de script, et une validation d'argument avec message d'erreur envoyé sur stderr.

## Section D — Problème de diagnostic (6 points)

Un script de sauvegarde utilisant `tar -czf - /data | ssh serveur "cat > backup.tar.gz"` rapporte systématiquement un succès, mais certaines sauvegardes récentes se révèlent tronquées.

**a) (2 pts)** Expliquez précisément le mécanisme qui permet à ce script de rapporter un succès malgré un échec réel du transfert.

**b) (2 pts)** Proposez la correction de fond avec la ligne de code exacte.

**c) (2 pts)** Proposez une vérification complémentaire, indépendante du code de sortie du pipeline, permettant de détecter une archive incomplète même dans un cas où `pipefail` seul ne suffirait pas à le révéler.

## Grille de correction résumée

| Section | Points | Critères |
|---|---|---|
| A — QCM | /4 | Réponse exacte uniquement |
| B — Questions ouvertes | /6 | Précision technique, distinction correcte échec technique/résultat métier |
| C — Exercice technique | /4 | Script complet et fonctionnel avec les 4 éléments demandés |
| D — Diagnostic | /6 | Mécanisme correctement expliqué, correction exacte, vérification complémentaire pertinente |

---

# PARTIE II — MINI-PROJET (échelle module)

## Cahier des charges

**Titre :** Bibliothèque de robustesse Bash (Bash Hardening Toolkit)

**Contexte :** Suite à l'étude de cas du cours et à sa reprise au TP (sauvegarde silencieusement tronquée), l'équipe système souhaite une bibliothèque de fonctions Bash réutilisables, à inclure (`source`) dans tout script de production, standardisant les bonnes pratiques du module plutôt que de les réécrire à chaque fois.

### Objectifs du livrable

Développer une bibliothèque Bash (fichier `.sh` destiné à être sourcé, pas exécuté directement) qui fournit :
1. Une fonction d'initialisation standard activant `set -euo pipefail` et configurant un `trap` de nettoyage générique, avec un mécanisme permettant à chaque script appelant d'enregistrer ses propres ressources à nettoyer (fichiers temporaires, verrous) sans avoir à réécrire le `trap` lui-même
2. Des fonctions de validation d'arguments courantes (argument requis présent, fichier existant et lisible, répertoire existant et accessible en écriture), chacune avec un message d'erreur clair sur stderr et un code de sortie documenté et cohérent
3. Une fonction de journalisation standardisée distinguant niveaux d'information et d'erreur, écrivant respectivement sur stdout et stderr
4. Une fonction utilitaire pour exécuter un pipeline avec vérification explicite de chaque maillon (au-delà de ce que `pipefail` seul peut indiquer), utile pour le cas de l'étude du cours (vérification de taille de fichier après transfert, ou équivalent générique)
5. Une documentation claire de chaque fonction (paramètres attendus, valeur de retour, effets de bord)

### Contraintes techniques
- La bibliothèque doit être conçue pour être sourcée (`source bash_hardening.sh`) sans effets de bord indésirables au moment du chargement lui-même (pas d'exécution immédiate de logique métier)
- Le mécanisme d'enregistrement de ressources à nettoyer (objectif 1) doit permettre l'ajout de plusieurs ressources par un même script, toutes nettoyées correctement à la sortie
- Documentation d'installation et d'utilisation (README), avec un exemple de script complet utilisant la bibliothèque
- Tests démontrés sur au moins 3 scénarios : un script utilisant la bibliothèque qui réussit normalement (toutes les ressources enregistrées sont nettoyées), un script qui échoue en cours d'exécution (nettoyage tout de même garanti), un script qui valide un argument invalide (message d'erreur clair et code de sortie approprié)

### Livrables attendus
- Code source de la bibliothèque, commenté
- README (installation, utilisation, documentation de chaque fonction fournie)
- Un script d'exemple complet démontrant l'usage de la bibliothèque (idéalement une version robustifiée d'un script déjà écrit dans un module précédent de l'UE4)
- Journal de tests montrant les 3 scénarios validés, avec captures
- Rapport court (1 page) sur les choix de conception, notamment le mécanisme d'enregistrement de ressources à nettoyer, et les limites connues

### Grille d'évaluation

| Critère | Pondération |
|---|---|
| Initialisation standard (set -euo pipefail + trap générique) | 25% |
| Fonctions de validation d'arguments réutilisables | 20% |
| Journalisation standardisée stdout/stderr | 15% |
| Fonction de vérification de pipeline complémentaire à pipefail | 20% |
| Documentation et tests démontrés (3 scénarios) | 20% |

### Modalité
Travail individuel ou en binôme, durée indicative : 3-4 jours. Cette bibliothèque pourra être réutilisée dans le module suivant (M26, automatisation appliquée) et dans le projet intégrateur de fin d'UE4, comme fondation commune à tous les scripts développés.

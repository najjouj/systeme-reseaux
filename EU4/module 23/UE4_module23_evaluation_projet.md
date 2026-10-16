# UE4 — Module 23 : Syntaxe Bash Fondamentale
## Compléments : ÉVALUATION / PROJET

---

# PARTIE I — ÉVALUATION

**Durée :** 1h30
**Modalités :** documents non autorisés sauf `man`/`help` local si épreuve sur machine
**Barème total :** /20

## Section A — QCM (4 points, 0,5 pt/question)

1. `echo $nom` (sans guillemets), si `$nom` contient un espace, peut provoquer :
 a) toujours le même résultat qu'avec guillemets b) un découpage en mots altérant le résultat attendu c) une erreur de syntaxe systématique d) l'affichage littéral de "$nom"

2. `"$@"`, comparé à `"$*"` :
 a) sont strictement identiques dans tous les cas b) préserve chaque argument individuellement, même avec des espaces c) ne fonctionne qu'avec un seul argument d) supprime les espaces de chaque argument

3. L'opérateur `-eq` en Bash sert à comparer :
 a) des chaînes de caractères b) des valeurs numériques c) des noms de fichiers d) des tableaux

4. `for ligne in $(cat fichier.txt)` pour traiter un fichier ligne par ligne :
 a) est la méthode recommandée et robuste b) est incorrecte, car elle découpe sur les espaces, pas sur les lignes c) fonctionne uniquement sur macOS d) nécessite obligatoirement `sudo`

5. Sans `local` dans une fonction Bash, une variable qui y est définie :
 a) reste locale par défaut b) devient globale par défaut, risquant de collisionner avec le reste du script c) provoque une erreur de syntaxe d) est automatiquement effacée après l'appel

6. Une fonction Bash peut directement "retourner" :
 a) n'importe quel type de donnée comme une chaîne complexe b) uniquement un code de sortie entier entre 0 et 255 c) un tableau associatif directement d) rien du tout

7. `shellcheck` sert à :
 a) exécuter un script plus rapidement b) analyser statiquement un script pour détecter des erreurs courantes, notamment de quotation c) chiffrer un script d) remplacer Bash par un autre langage

8. `while IFS= read -r ligne; do ... done < fichier` est préférable à une boucle `for` sur substitution de commande car :
 a) elle est plus courte à écrire b) elle traite chaque ligne de façon robuste, insensible aux espaces internes c) elle ne fonctionne qu'avec des fichiers vides d) elle est obsolète

## Section B — Questions ouvertes (6 points)

**Q1 (2 pts).** Expliquez la différence de comportement entre `"$variable"`, `$variable` et `'$variable'`, avec un exemple où l'absence de guillemets doubles provoquerait un bug concret.

**Q2 (2 pts).** Expliquez pourquoi une fonction Bash ne peut pas retourner directement une chaîne de caractères comme dans un langage typé, et la convention utilisée à la place.

**Q3 (2 pts).** Expliquez le piège de `for f in $(find ... )` pour traiter une liste de fichiers, et donnez la méthode robuste alternative.

## Section C — Exercice technique (4 points)

Corrigez le script suivant en appliquant les bonnes pratiques du cours (quotation, opérateur de comparaison, boucle robuste) :
```bash
#!/bin/bash
rep=$1
if [ $rep = "" ]; then
    echo "manque argument"
fi
for f in $(ls $rep); do
    echo traitement $f
done
```

## Section D — Problème de diagnostic (6 points)

Un script d'archivage supprime des fichiers inattendus sur un serveur de production, alors qu'il fonctionnait normalement en test sur un jeu de fichiers aux noms simples.

**a) (2 pts)** Formulez l'hypothèse la plus probable concernant la cause de ce comportement.

**b) (2 pts)** Identifiez, dans un script hypothétique du type `for f in $(find /data -name "*.old"); do rm $f; done`, les deux occurrences distinctes contribuant au problème.

**c) (2 pts)** Proposez une version corrigée et robuste de cette ligne de script.

## Grille de correction résumée

| Section | Points | Critères |
|---|---|---|
| A — QCM | /4 | Réponse exacte uniquement |
| B — Questions ouvertes | /6 | Précision technique, mécanisme de quotation/expansion correctement expliqué |
| C — Exercice technique | /4 | Toutes les corrections appliquées (guillemets, opérateur, boucle robuste) |
| D — Diagnostic | /6 | Hypothèse correcte, deux occurrences identifiées, correction robuste complète |

---

# PARTIE II — MINI-PROJET (échelle module)

## Cahier des charges

**Titre :** Auditeur et correcteur de scripts Bash (Bash Script Linter & Fixer)

**Contexte :** Suite à l'étude de cas du cours (script de nettoyage ayant supprimé des fichiers inattendus faute de quotation), l'équipe système souhaite un outil de première ligne facilitant l'audit systématique des scripts Bash existants avant tout déploiement en production, en s'appuyant sur `shellcheck` et en ajoutant une couche d'analyse pédagogique.

### Objectifs du livrable

Développer un outil (Bash ou Python, au choix justifié) qui, pour un répertoire de scripts Bash fourni :
1. Exécute `shellcheck` sur chaque script trouvé et agrège les résultats
2. Catégorise les avertissements détectés selon les thèmes du cours (quotation, opérateur de comparaison incorrect, boucle non robuste, absence de `local`), en s'appuyant sur les codes d'avertissement `shellcheck` correspondants
3. Produit, pour chaque script analysé, un score de risque simple basé sur le nombre et la gravité des avertissements détectés
4. Identifie spécifiquement les scripts contenant des opérations potentiellement destructrices (`rm`, `mv`, redirections d'écrasement `>`) combinées à des avertissements de quotation, comme catégorie de risque prioritaire (rappel direct de l'étude de cas du cours)
5. Génère un rapport de synthèse classant les scripts par ordre de risque décroissant, avec pour chacun un résumé pédagogique des corrections recommandées en langage clair (pas seulement les codes d'avertissement bruts de shellcheck)
6. Documente l'exécution avec horodatage

### Contraintes techniques
- L'outil doit utiliser réellement `shellcheck` comme moteur d'analyse, pas réimplémenter sa propre analyse syntaxique
- La catégorisation par thème doit être fiable et testée sur des cas représentatifs de chaque catégorie du cours
- Gestion d'erreurs robuste (`shellcheck` non installé, répertoire vide ou inexistant, scripts non exécutables/non trouvés)
- Documentation d'installation et d'utilisation (README)
- Tests démontrés sur au moins 3 scripts de test représentant chacun une catégorie de problème distincte (quotation, opérateur, boucle non robuste), plus un script propre ne devant déclencher aucune alerte de risque prioritaire

### Livrables attendus
- Code source commenté
- README (installation, utilisation, catégories de risque détectées, limites connues)
- Journal de tests montrant les 4 scripts de test analysés avec leurs rapports respectifs
- Rapport court (1 page) sur les choix techniques, notamment la méthode de catégorisation par thème, et les limites connues (ex : `shellcheck` lui-même ne détecte pas toutes les erreurs logiques, l'outil hérite de cette limite)

### Grille d'évaluation

| Critère | Pondération |
|---|---|
| Intégration correcte et fonctionnelle de `shellcheck` | 20% |
| Catégorisation fiable par thème du cours | 30% |
| Détection prioritaire des opérations destructrices mal quotifiées | 25% |
| Clarté pédagogique du rapport (pas seulement les codes bruts) | 15% |
| Documentation et tests démontrés (4 scripts) | 10% |

### Modalité
Travail individuel ou en binôme, durée indicative : 3-4 jours (module court, projet proportionné). Cet outil pourra être appliqué en tout début du module suivant sur l'ensemble des scripts déjà produits au fil des UE1, UE2 et UE3 — une occasion concrète de mesurer leur robustesse réelle à la lumière de ce qui vient d'être appris.

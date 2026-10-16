# UE3 — Module 21 : Supervision, Performance et Sauvegarde
## Compléments : ÉVALUATION / PROJET

---

# PARTIE I — ÉVALUATION

**Durée :** 1h30
**Modalités :** documents non autorisés sauf `man` local si épreuve sur machine
**Barème total :** /20

## Section A — QCM (4 points, 0,5 pt/question)

1. `iostat -x` permet notamment d'observer :
 a) uniquement l'usage CPU b) le taux d'occupation (`%util`) et le temps d'attente (`await`) des disques c) uniquement la mémoire d) le trafic réseau exclusivement

2. Un load average élevé avec CPU peu sollicité et processus en état `D` oriente vers :
 a) une charge CPU réelle b) une attente disque c) toujours un problème réseau d) un problème purement applicatif certain

3. Une sauvegarde incrémentale copie :
 a) l'intégralité des données à chaque fois b) uniquement ce qui a changé depuis la dernière sauvegarde, quelle qu'elle soit c) uniquement ce qui a changé depuis la dernière complète d) rien, elle référence uniquement la complète

4. La règle 3-2-1 recommande :
 a) 3 sites, 2 copies, 1 support b) 3 copies, 2 supports différents, 1 copie hors site c) 1 copie, 2 sites, 3 supports d) aucune règle chiffrée standard

5. `rsync --delete` :
 a) supprime uniquement les fichiers corrompus b) supprime côté destination les fichiers absents côté source, à utiliser avec prudence c) n'a aucun effet réel d) chiffre automatiquement les données

6. Une sauvegarde jamais testée en restauration :
 a) peut être considérée comme fiable si le script ne rapporte aucune erreur b) ne peut pas être considérée comme réellement fiable c) est automatiquement plus rapide à restaurer d) n'a pas besoin de documentation

7. Diagnostiquer une lenteur système uniquement à partir du load average, sans corrélation avec d'autres outils :
 a) est une méthodologie rigoureuse et suffisante b) risque de mener à une conclusion prématurée et incorrecte c) est la seule méthode recommandée d) élimine le besoin de tout autre outil

8. Le RTO (Recovery Time Objective) désigne :
 a) le volume de données sauvegardées b) le temps réellement nécessaire pour restaurer un système après incident c) la fréquence des sauvegardes d) le nombre de copies recommandées

## Section B — Questions ouvertes (6 points)

**Q1 (2 pts).** Expliquez pourquoi un load average élevé ne permet pas, à lui seul, de conclure à une cause précise, et décrivez la démarche de corrélation appropriée.

**Q2 (2 pts).** Comparez sauvegarde incrémentale et différentielle sur la complexité et la robustesse de la chaîne de restauration.

**Q3 (2 pts).** Expliquez pourquoi une sauvegarde jamais testée en restauration ne peut pas être considérée comme fiable, avec un exemple concret de ce que ce test pourrait révéler.

## Section C — Exercice technique (4 points)

Donnez la commande `vmstat` permettant d'observer 5 échantillons toutes les 2 secondes, la commande `iostat` équivalente en statistiques étendues, et la commande `rsync` permettant de simuler (sans exécuter) une synchronisation avec suppression des fichiers obsolètes côté destination.

## Section D — Problème de diagnostic (6 points)

Un audit révèle qu'un espace de sauvegarde ne contient que des sauvegardes vieilles de plusieurs semaines, alors que la tâche planifiée associée s'exécute chaque nuit sans erreur rapportée par le script lui-même.

**a) (2 pts)** Formulez l'hypothèse la plus probable, en citant le principe déjà rencontré dans un module précédent qui s'applique directement ici.

**b) (2 pts)** Proposez deux vérifications concrètes permettant d'identifier la cause technique précise de l'échec.

**c) (2 pts)** Proposez une correction en deux volets : correction du script lui-même, et mécanisme de détection indépendant de tout futur échec.

## Grille de correction résumée

| Section | Points | Critères |
|---|---|---|
| A — QCM | /4 | Réponse exacte uniquement |
| B — Questions ouvertes | /6 | Précision technique, méthodologie de corrélation correctement décrite |
| C — Exercice technique | /4 | 3 commandes exactes |
| D — Diagnostic | /6 | Hypothèse cohérente (échec silencieux non vérifié), vérifications pertinentes, correction en deux volets distincts |

---

# PARTIE II — MINI-PROJET (échelle module)

## Cahier des charges

**Titre :** Superviseur de performance et de sauvegarde unifié (Performance & Backup Health Guardian)

**Contexte :** Ce module clôt la série d'outils d'audit système de l'UE3 avant le module de synthèse (troubleshooting méthodique). L'équipe souhaite un outil combinant diagnostic de performance corrélé et vérification active de la fiabilité des sauvegardes, incluant explicitement un rappel de test de restauration périodique.

### Objectifs du livrable

Développer un outil (Bash ou Python, au choix justifié) qui :
1. Collecte périodiquement des métriques corrélées (CPU, mémoire/swap via `vmstat`, disque via `iostat -x`) et applique automatiquement la logique de diagnostic du tableau de la section 7.2 du cours pour proposer une hypothèse de cause (charge CPU réelle, attente disque, thrashing mémoire), plutôt que de se limiter à un indicateur brut isolé
2. Exécute une sauvegarde avec rotation complète/incrémentale (reprise de l'outil du cours), avec vérification systématique et explicite du code de retour de chaque opération critique
3. Intègre un mécanisme de heartbeat indépendant (rappel module 19) pour la tâche de sauvegarde, stocké dans un emplacement garanti persistant
4. Maintient un registre des tests de restauration effectués (date, durée mesurée, résultat), et alerte si aucun test n'a été documenté depuis une durée configurable (par défaut, un seuil raisonnable comme 90 jours)
5. Journalise chaque détection (performance et sauvegarde) avec horodatage et catégorie claire
6. Génère un rapport hebdomadaire unifié, structuré en sections distinctes : diagnostic de performance, fiabilité de sauvegarde, statut des tests de restauration

### Contraintes techniques
- La logique de diagnostic de performance (objectif 1) doit être testée sur au moins deux scénarios distincts et produire des hypothèses différentes et correctes (reproduction des scénarios A/B du TP)
- Le script de sauvegarde ne doit jamais continuer silencieusement après l'échec d'une opération critique (`set -euo pipefail` ou équivalent rigoureux)
- Gestion d'erreurs robuste (`sysstat` non installé, destination de sauvegarde injoignable)
- Configuration externalisée (seuils de diagnostic, emplacement des sauvegardes, seuil d'alerte pour test de restauration obsolète)
- Documentation d'installation et d'utilisation (README)
- Tests démontrés sur au moins 3 scénarios : diagnostic correct d'une charge CPU réelle vs une attente disque (deux sous-cas), échec de sauvegarde correctement détecté et signalé (reproduction du scénario de troubleshooting du cours), alerte de test de restauration obsolète

### Livrables attendus
- Code source commenté
- README (installation, configuration, format du rapport en sections distinctes, limites connues)
- Journal de tests montrant les 3 scénarios validés, avec captures
- Rapport court (1 page) sur les choix techniques, notamment la logique de corrélation retenue pour le diagnostic de performance, et les limites connues (ex : les hypothèses de cause restent des heuristiques, pas des certitudes absolues, un avis humain reste nécessaire pour un diagnostic définitif)

### Grille d'évaluation

| Critère | Pondération |
|---|---|
| Diagnostic de performance corrélé (pas un indicateur isolé) | 25% |
| Sauvegarde avec vérification stricte des codes de retour | 20% |
| Heartbeat indépendant et persistant | 15% |
| Registre et alerte de test de restauration obsolète | 20% |
| Documentation et tests démontrés (3 scénarios) | 20% |

### Modalité
Travail individuel ou en binôme, durée indicative : 1,5 semaine (volume plus important, module de synthèse partielle). Cet outil clôt la série d'audit système initiée aux modules 14 à 20 — l'ensemble des huit outils développés au fil de l'UE3 pourra être mobilisé comme brique de supervision dans le projet intégrateur de fin d'UE3 (Projet 1 : déployer et sécuriser une infrastructure Linux).

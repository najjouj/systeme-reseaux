# UE5 — Module 27 : Architecture Windows, Services et Registre
## Compléments : ÉVALUATION / PROJET

---

# PARTIE I — ÉVALUATION

**Durée :** 1h30
**Modalités :** documents non autorisés sauf aide PowerShell locale (`Get-Help`) si épreuve sur machine
**Barème total :** /20

## Section A — QCM (4 points, 0,5 pt/question)

1. L'équivalent fonctionnel de systemd sous Windows est :
 a) le registre b) le Service Control Manager (SCM) c) l'Observateur d'événements d) le Gestionnaire des tâches

2. La définition d'un service Windows se trouve principalement dans :
 a) un fichier texte sous `C:\etc` b) le registre, sous HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services c) l'Observateur d'événements d) un fichier `.service`

3. La HAL (Hardware Abstraction Layer) sert à :
 a) chiffrer les données b) faciliter la portabilité de Windows entre architectures matérielles c) gérer les services d) remplacer le registre

4. Un service en type de démarrage "Automatique (différé)" :
 a) ne démarre jamais automatiquement b) démarre automatiquement, mais après les services automatiques standards c) nécessite une intervention manuelle systématique d) est toujours désactivé

5. Contrairement à `journalctl -u <service>`, le Journal des événements Windows :
 a) offre un filtrage par service tout aussi direct et natif b) organise ses journaux par catégories logiques, nécessitant d'identifier le bon journal et la bonne source c) n'existe pas d) est identique en tout point à journald

6. `svchost.exe` illustre :
 a) qu'un service Windows dispose toujours de son propre processus dédié b) que plusieurs services légers peuvent partager un même processus hôte c) l'équivalent de PID 1 sous Linux d) un mécanisme de chiffrement des services

7. Une modification incorrecte du registre :
 a) n'a jamais de conséquence grave b) peut empêcher un service ou le système de démarrer correctement, comme un fichier `/etc` corrompu sous Linux c) est automatiquement annulée par Windows d) ne concerne jamais les services

8. WSL (Windows Subsystem for Linux) illustre :
 a) une incompatibilité totale entre Windows et Linux b) un pont direct permettant d'exécuter un environnement Linux quasi natif sous Windows c) le remplacement de PowerShell d) une ancienne technologie abandonnée

## Section B — Questions ouvertes (6 points)

**Q1 (2 pts).** Comparez la philosophie de configuration système entre Linux (fichiers texte sous `/etc`) et Windows (registre centralisé), avec un avantage de chaque approche.

**Q2 (2 pts).** Expliquez pourquoi il n'existe pas d'équivalent direct à `journalctl -u <service>` sous Windows, et décrivez la démarche alternative à suivre.

**Q3 (2 pts).** Expliquez la différence entre les types de démarrage "Automatique" et "Automatique (différé)", avec un exemple de service pour lequel chacun serait approprié.

## Section C — Exercice technique (4 points)

Donnez la commande PowerShell permettant d'afficher l'état d'un service nommé "BITS", celle permettant de lister tous les services actuellement arrêtés dont le type de démarrage est Automatique, et le chemin de registre où se trouve la définition de ce service.

## Section D — Problème de diagnostic (6 points)

Un service Windows configuré en démarrage automatique n'est pas actif après un redémarrage planifié du serveur, sans qu'aucune modification de configuration n'ait été rapportée par l'équipe.

**a) (2 pts)** Donnez la première commande à exécuter, et ce que vous en attendez.

**b) (2 pts)** En supposant que le service est bien à l'état arrêté, proposez la démarche pour rechercher un événement d'erreur associé, en tenant compte de la difficulté déjà identifiée en cours de ne pas savoir a priori dans quel journal chercher.

**c) (2 pts)** Proposez une hypothèse alternative non liée à une erreur applicative, cohérente avec les mécanismes de dépendances de service vus en cours.

## Grille de correction résumée

| Section | Points | Critères |
|---|---|---|
| A — QCM | /4 | Réponse exacte uniquement |
| B — Questions ouvertes | /6 | Précision technique, comparaison Linux/Windows correctement articulée |
| C — Exercice technique | /4 | 3 éléments exacts |
| D — Diagnostic | /6 | Démarche méthodique cohérente avec la difficulté de recherche dans le journal, hypothèse de dépendance pertinente |

---

# PARTIE II — MINI-PROJET (échelle module)

## Cahier des charges

**Titre :** Auditeur de services Windows (Windows Service Health Auditor)

**Contexte :** En écho direct à la série d'outils d'audit système développée en Bash tout au long des UE1 à UE4, l'équipe système souhaite un premier outil PowerShell équivalent pour l'infrastructure Windows Server, posant les bases d'une suite de supervision qui sera enrichie aux modules suivants de l'UE5.

### Objectifs du livrable

Développer un script PowerShell qui :
1. Audite l'ensemble des services configurés en démarrage Automatique (et Automatique différé) et signale ceux actuellement arrêtés (reprise et extension du script du cours)
2. Pour chaque service problématique détecté, tente de localiser un événement d'erreur associé en recherchant dans plusieurs journaux courants (Système, Application) avec une fenêtre temporelle raisonnable autour du dernier démarrage du système
3. Consulte la définition registre de chaque service problématique pour en extraire le chemin de l'exécutable et les dépendances déclarées (`DependOnService`), et vérifie si une dépendance elle-même n'est pas à l'origine du problème
4. Produit un rapport structuré distinguant : services correctement actifs, services arrêtés avec cause probable identifiée (dépendance en échec, erreur applicative trouvée), services arrêtés sans cause identifiable (nécessitant une investigation manuelle)
5. Documente l'exécution avec horodatage

### Contraintes techniques
- Le script doit gérer le cas où aucun événement pertinent n'est trouvé sans échouer bruyamment (reprise de la leçon du TP : recherche élargie en cas d'échec de la recherche ciblée initiale)
- Gestion d'erreurs appropriée (accès insuffisant au registre ou aux journaux, service inexistant)
- Documentation d'installation et d'utilisation (README)
- Tests démontrés sur au moins 2 scénarios : un service arrêté à cause d'une dépendance en échec (reproductible en créant un service de test avec une fausse dépendance), un service arrêté sans cause facilement identifiable

### Livrables attendus
- Code source commenté
- README (installation, utilisation, format du rapport, limites connues)
- Journal de tests montrant les 2 scénarios validés, avec captures
- Rapport court (1 page) sur les choix techniques et le parallèle explicite avec les outils Bash équivalents déjà développés en UE1-UE4 (Service Health Guardian de l'UE3-M16 notamment)

### Grille d'évaluation

| Critère | Pondération |
|---|---|
| Détection correcte des services problématiques | 25% |
| Recherche d'événements d'erreur associés (multi-journaux) | 25% |
| Analyse des dépendances via le registre | 25% |
| Documentation et tests démontrés (2 scénarios) | 15% |
| Parallèle explicite avec les outils Bash équivalents | 10% |

### Modalité
Travail individuel ou en binôme, durée indicative : 1 semaine. Ce premier outil PowerShell de l'UE5 sera enrichi aux modules suivants (Active Directory, DNS/DHCP, GPO, NTFS) pour constituer, à terme, une suite d'audit Windows comparable à celle déjà construite en Bash pour Linux.

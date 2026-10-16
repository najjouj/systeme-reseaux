# UE3 — Module 21 : Supervision, Performance et Sauvegarde

---

# PARTIE I — COURS

## 1. Titre du cours
Supervision système, analyse de performance et stratégies de sauvegarde

## 2. Niveau
Élève ingénieur — cycle ingénieur, UE3 administration Linux (S2)

## 3. Durée
CM : 4h (module double thématique — +3h TD, +5h TP en Parties II et III)

## 4. Prérequis
- UE1-Module 2 et 3 : processus, mémoire virtuelle (pour interpréter les métriques de performance)
- UE3-Module 15 : stockage (pour la destination des sauvegardes)
- UE3-Module 19 : tâches planifiées (pour l'automatisation des sauvegardes)

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Utiliser une boîte à outils de supervision de performance au-delà de `top`/`htop` (`vmstat`, `iostat`, `sar`)
- Appliquer une méthodologie de diagnostic de performance corrélant CPU, mémoire, disque et réseau
- Différencier sauvegarde complète, incrémentale et différentielle, et choisir une stratégie adaptée
- Utiliser `rsync` et `tar` efficacement pour des sauvegardes fiables et performantes
- Appliquer la règle 3-2-1 de sauvegarde et en justifier chaque composante
- Concevoir et tester une procédure de restauration, pas seulement une procédure de sauvegarde

## 6. Motivation

"Le serveur est lent" est l'une des plaintes les plus fréquentes et les plus vagues qu'un ingénieur système puisse recevoir — et l'une des plus mal diagnostiquées si elle n'est pas abordée avec une méthodologie rigoureuse plutôt qu'une intuition. Ce module fournit les outils et la démarche pour transformer "c'est lent" en un diagnostic précis : CPU saturé ? Mémoire insuffisante provoquant du swap ? Disque en attente d'E/S ? Réseau congestionné ? Chacune de ces causes a sa propre signature, déjà en partie étudiée en UE1, mobilisée ici de façon opérationnelle et outillée.

La seconde moitié du module aborde un sujet dont l'importance n'est souvent comprise qu'après un incident : une sauvegarde qui n'a jamais été testée en restauration n'est pas une sauvegarde fiable, seulement une supposition non vérifiée. Ce module enseigne à concevoir des stratégies de sauvegarde robustes, et surtout à les valider.

## 7. Concepts fondamentaux

### 7.1 Au-delà de `top`/`htop` — la boîte à outils de supervision

- **`vmstat`** (déjà entrevu en UE1-M3) : vue synthétique et périodique de la mémoire, du swap, des processus et du CPU en une seule commande, idéale pour un premier coup d'œil rapide
- **`iostat`** (déjà entrevu en UE1-M4) : statistiques détaillées d'utilisation des disques (débit, temps d'attente, utilisation en pourcentage) — indispensable pour distinguer un disque réellement saturé d'un simple pic de trafic passager
- **`sar`** (System Activity Reporter, du paquet `sysstat`) : collecte **historique** de métriques système (CPU, mémoire, disque, réseau), permettant d'analyser une période passée, pas seulement l'instant présent — crucial pour diagnostiquer un incident survenu la nuit précédente sans supervision active en temps réel
- **`dstat`** : combine plusieurs sources de métriques (CPU, disque, réseau, mémoire) dans un affichage unifié et lisible en temps réel

```bash
vmstat 2 5        # 5 échantillons, toutes les 2 secondes
iostat -x 2 5      # statistiques disque étendues
sar -u 1 5          # utilisation CPU historique/temps réel
sar -r -f /var/log/sysstat/sa15   # mémoire, données du 15 du mois (historique)
```

### 7.2 Méthodologie de diagnostic de performance corrélé

Rappel direct et application pratique des symptômes déjà étudiés séparément en UE1 :

| Symptôme observé | Outil de confirmation | Cause probable (rappel UE1) |
|---|---|---|
| CPU élevé, load average cohérent avec le CPU | `top`, `vmstat` (colonne `us`/`sy`) | Charge de calcul réelle (UE1-M2) |
| CPU faible, load average élevé, processus en `D` | `iostat -x` (colonne `%util`, `await`) | Attente disque (UE1-M2/M4) |
| CPU faible, load average élevé, `si`/`so` actifs | `vmstat` (colonnes `si`/`so`) | Thrashing mémoire (UE1-M3) |
| Débit réseau saturé | `iftop`, `nload`, ou `sar -n DEV` | Congestion réseau (UE2) |

**Point clé pédagogique :** la vraie compétence de ce module n'est pas de mémoriser des commandes isolément, mais de savoir **corréler plusieurs outils** pour confirmer ou infirmer une hypothèse — un load average élevé seul ne dit rien sur sa cause réelle, il faut systématiquement croiser avec `iostat`/`vmstat` pour trancher entre les hypothèses déjà posées en UE1-M2 et M3.

### 7.3 `iostat` en détail

```bash
iostat -x 2
```

Colonnes clés : `%util` (pourcentage de temps où le disque est occupé à traiter des requêtes — proche de 100% signale une saturation), `await` (temps d'attente moyen d'une requête, incluant le temps de traitement, en millisecondes — une valeur élevée signale un disque en difficulté à suivre la charge), `r/s`/`w/s` (opérations de lecture/écriture par seconde).

**Nuance importante :** un `%util` élevé n'est pas nécessairement problématique en soi si `await` reste bas (un disque rapide peut être très sollicité tout en répondant vite) — c'est la combinaison des deux indicateurs, pas un seul isolément, qui permet un diagnostic fiable.

### 7.4 Sauvegarde complète, incrémentale, différentielle

- **Sauvegarde complète (full)** : copie l'intégralité des données à chaque exécution — simple à restaurer (une seule source), mais coûteuse en temps et en espace si répétée fréquemment
- **Sauvegarde incrémentale** : copie uniquement les données modifiées depuis la **dernière sauvegarde, quelle qu'elle soit** (complète ou incrémentale précédente) — rapide et économe en espace, mais une restauration nécessite de rejouer la sauvegarde complète **et toutes** les incrémentales suivantes dans l'ordre, un processus plus long et plus fragile (une seule incrémentale corrompue ou manquante casse la chaîne de restauration)
- **Sauvegarde différentielle** : copie toutes les données modifiées depuis la **dernière sauvegarde complète** (pas depuis la dernière différentielle) — un compromis entre les deux : plus volumineuse qu'une incrémentale au fil du temps (chaque différentielle recopie tout ce qui a changé depuis la complète, y compris ce que la différentielle précédente contenait déjà), mais une restauration ne nécessite que la complète **et la dernière** différentielle, plus simple et plus robuste qu'une longue chaîne d'incrémentales

**Tableau récapitulatif :**

| Type | Volume par sauvegarde | Complexité de restauration | Robustesse de la chaîne |
|---|---|---|---|
| Complète | Élevé, constant | Minimale (une seule source) | Maximale |
| Incrémentale | Faible | Élevée (rejouer toute la chaîne) | Fragile (un maillon manquant casse tout) |
| Différentielle | Croissant entre deux complètes | Modérée (complète + dernière différentielle) | Bonne (un seul maillon après la complète) |

### 7.5 La règle 3-2-1

Une pratique de sauvegarde robuste, largement reconnue dans l'industrie, se résume par la règle **3-2-1** :
- **3** copies des données au minimum (l'original + 2 sauvegardes)
- **2** supports de stockage différents (ex : disque local + stockage réseau, réduisant le risque qu'une même panne matérielle affecte toutes les copies)
- **1** copie hors site (offsite), protégeant contre un sinistre localisé affectant l'intégralité d'un site physique (incendie, inondation, vol, ou plus prosaïquement une erreur de configuration touchant toute l'infrastructure locale)

**Chaque composante de la règle répond à un risque distinct :** le nombre de copies protège contre la corruption d'une copie individuelle ; la diversité de support protège contre une panne matérielle systémique d'un type de support ; la localisation hors site protège contre un sinistre physique localisé — omettre l'une de ces trois dimensions laisse une catégorie de risque entière non couverte, quelle que soit la rigueur apportée aux deux autres.

### 7.6 `rsync` — synchronisation efficace

```bash
rsync -avz --delete /data/ utilisateur@serveur_backup:/backups/data/
```

- `-a` (archive) : préserve permissions, propriétaires, horodatages, liens symboliques — combinaison de plusieurs options utiles pour une sauvegarde fidèle
- `-v` : verbeux
- `-z` : compression pendant le transfert (utile sur un lien réseau limité, moins pertinent en local)
- `--delete` : supprime côté destination les fichiers qui n'existent plus côté source, gardant les deux emplacements synchronisés — **à utiliser avec prudence**, une suppression accidentelle côté source serait alors répliquée côté sauvegarde, annulant la protection recherchée si aucune rétention d'historique n'existe par ailleurs

**Pourquoi `rsync` est-il particulièrement adapté aux sauvegardes incrémentales ?** Il transfère uniquement les **différences** au niveau des blocs d'un fichier modifié (pas nécessairement le fichier entier), une efficacité qui rejoint directement, à un niveau différent, l'esprit d'une sauvegarde incrémentale — un fichier volumineux dont seule une petite portion a changé n'est pas retransmis intégralement.

### 7.7 `tar` — archivage

```bash
tar -czvf sauvegarde_$(date +%Y%m%d).tar.gz /data/
tar -xzvf sauvegarde_20240115.tar.gz -C /destination/
```

`tar` (tape archive, nom hérité de son usage historique sur bande magnétique) regroupe une arborescence en une seule archive, généralement combinée à une compression (`-z` pour gzip, ou `-j` pour bzip2, `-J` pour xz selon le compromis vitesse/taux de compression recherché). Contrairement à `rsync`, `tar` produit un **fichier unique** autonome, pratique pour un archivage daté et transportable, mais sans le mécanisme différentiel efficace de `rsync` pour des sauvegardes répétées fréquentes.

### 7.8 Tester la restauration — l'étape la plus souvent négligée

**Une sauvegarde jamais testée en restauration n'est pas une sauvegarde fiable.** De nombreux incidents révèlent, au pire moment possible, qu'une sauvegarde était corrompue, incomplète, ou que la procédure de restauration elle-même n'avait jamais été documentée ni pratiquée par l'équipe censée l'exécuter en urgence.

**Bonnes pratiques :**
- Tester périodiquement une restauration complète sur un environnement isolé (jamais directement en production), pour valider l'intégrité réelle des données sauvegardées
- Documenter précisément la procédure de restauration, avec les commandes exactes — une procédure connue uniquement "de tête" par une seule personne est un risque organisationnel (rappel indirect du principe de documentation déjà souligné à plusieurs reprises)
- Mesurer le temps réel nécessaire à une restauration complète (RTO — Recovery Time Objective), une donnée souvent découverte pour la première fois pendant un incident réel si elle n'a jamais été mesurée à l'avance, avec des conséquences potentiellement lourdes si ce délai s'avère incompatible avec les exigences métier

## 8. Architecture

```
┌────────────────────────────────────────────────────────────┐
│  Diagnostic de performance corrélé                                │
│                                                                    │
│  top/vmstat ──▶ CPU/mémoire globale                                │
│  iostat -x  ──▶ disque (%util, await)                              │
│  sar        ──▶ historique multi-ressources                          │
│                                                                    │
│  Corrélation ──▶ hypothèse confirmée (CPU / mémoire / disque / réseau) │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│  Règle 3-2-1                                                      │
│  Original ──▶ Copie 1 (support local, ex: LVM snapshot + tar)         │
│           ──▶ Copie 2 (support différent, ex: NAS réseau via rsync)    │
│           ──▶ Copie 3 (hors site, ex: stockage cloud/site distant)      │
└────────────────────────────────────────────────────────────┘
```

## 9. Fonctionnement interne

`iostat`/`vmstat`/`sar` s'appuient sur les compteurs statistiques exposés par le noyau via `/proc` (rappel UE1-M1 sur `procfs`) — ces outils ne mesurent rien "activement" au sens d'une sonde intrusive, ils lisent et présentent lisiblement des compteurs déjà maintenus en continu par le noyau lui-même à chaque événement pertinent (E/S disque, changement de contexte, allocation mémoire), un mécanisme à très faible coût, cohérent avec la conception de `procfs` déjà étudiée. `rsync` calcule des sommes de contrôle par bloc pour chaque fichier des deux côtés de la synchronisation (source et destination), lui permettant de déterminer précisément quels blocs ont changé sans avoir à retransmettre un fichier entier — un algorithme (rsync algorithm) particulièrement efficace sur un lien réseau à bande passante limitée.

## 10. Démonstration

```bash
# Vue combinée rapide
vmstat 1 3
iostat -x 1 3

# Historique de performance (nécessite le service sysstat actif)
sar -u 1 3

# Sauvegarde incrémentale efficace avec rsync
rsync -avz --dry-run /data/ /backup/data/   # simulation avant exécution réelle
```

## 11. Commandes / outils

| Commande | Rôle | Exemple |
|---|---|---|
| `vmstat` | Vue synthétique CPU/mémoire/swap | `vmstat 2 5` |
| `iostat -x` | Statistiques disque détaillées | `iostat -x 2 5` |
| `sar` | Historique de métriques système | `sar -u 1 5` |
| `rsync -avz --dry-run` | Simuler une synchronisation avant exécution | `rsync -avz --dry-run src/ dst/` |
| `tar -czvf` / `-xzvf` | Créer/extraire une archive compressée | `tar -czvf backup.tar.gz /data` |
| `du -sh` | Taille réelle d'un répertoire (rappel UE1-M4) | `du -sh /data` |

**Erreur fréquente :** utiliser `rsync --delete` sans avoir d'abord testé avec `--dry-run` — une erreur de chemin source/destination inversés combinée à `--delete` peut supprimer irrémédiablement des données du côté qu'on pensait protéger, un risque particulièrement élevé et sous-estimé pour cette option précise.

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** un serveur de base de données est signalé "lent" par les équipes applicatives. Un ingénieur junior consulte `top`, observe un load average élevé mais un CPU peu sollicité, et conclut hâtivement à un problème de base de données mal optimisée, sans investigation plus poussée.

**Analyse attendue :** cette conclusion est prématurée — un load average élevé avec CPU peu sollicité est, comme vu en UE1-M2 et rappelé dans le tableau de la section 7.2 de ce module, la signature possible d'une attente disque (processus en `D`) ou d'un thrashing mémoire, deux causes très différentes d'un problème purement applicatif de requêtes mal optimisées. La méthodologie correcte impose de corréler avec `iostat -x` (vérifier `%util`/`await` pour confirmer ou infirmer une saturation disque) et `vmstat` (vérifier les colonnes `si`/`so` pour du swap actif) **avant** de conclure à une cause précise — un diagnostic prématuré fondé sur un seul indicateur (load average) risque de faire perdre un temps précieux à optimiser des requêtes SQL alors que la cause réelle serait, par exemple, un disque de stockage sous-dimensionné pour la charge d'E/S réelle de la base de données.

## 15. Sécurité

- Une sauvegarde constitue elle-même une copie complète (ou partielle) de données potentiellement sensibles — elle doit être protégée avec les mêmes exigences de confidentialité que les données originales (chiffrement au repos, permissions restreintes, rappel des principes déjà vus au module 14)
- La copie hors site de la règle 3-2-1 introduit un nouveau périmètre de sécurité à sécuriser (accès réseau, authentification vers le stockage distant) — une sauvegarde mal protégée peut devenir elle-même une vulnérabilité, un moyen de fuite de données pour un attaquant ciblant spécifiquement les sauvegardes plutôt que le système de production principal, une stratégie effectivement observée dans certaines attaques par rançongiciel sophistiquées
- Un outil de supervision (métriques de performance) peut lui-même révéler des informations sensibles sur l'activité d'un système (ex : des pics de charge à des horaires révélant des habitudes d'exploitation) — un accès restreint aux outils de supervision fait partie d'une politique de sécurité globale cohérente
- Tester une restauration en environnement isolé (rappel section 7.8) est aussi une précaution de sécurité : restaurer directement en production sans validation préalable pourrait réintroduire des données déjà corrompues, obsolètes, ou potentiellement compromises

## 16. Automatisation

Script Bash de sauvegarde avec rotation intégrée (complète hebdomadaire, incrémentale quotidienne), suivi d'une vérification d'intégrité de base.

```bash
#!/usr/bin/env bash
# Sauvegarde avec rotation : complète le dimanche, incrémentale les autres jours
set -euo pipefail

SOURCE="/data"
DESTINATION="/backup"
JOUR=$(date +%u)   # 1=lundi ... 7=dimanche
DATE=$(date +%Y%m%d)
LOGFILE="/var/log/backup.log"

if [[ "$JOUR" -eq 7 ]]; then
    echo "$(date '+%F %T') - Sauvegarde COMPLETE démarrée" | tee -a "$LOGFILE"
    rsync -a --delete "$SOURCE/" "$DESTINATION/complete_$DATE/"
    ln -sfn "$DESTINATION/complete_$DATE" "$DESTINATION/derniere_complete"
else
    echo "$(date '+%F %T') - Sauvegarde INCREMENTALE démarrée" | tee -a "$LOGFILE"
    rsync -a --link-dest="$DESTINATION/derniere_complete/" "$SOURCE/" "$DESTINATION/incr_$DATE/"
fi

# Vérification d'intégrité basique : comparer le nombre de fichiers source/destination
NB_SOURCE=$(find "$SOURCE" -type f | wc -l)
NB_DEST=$(find "$DESTINATION/derniere_complete" -type f 2>/dev/null | wc -l || echo 0)
echo "$(date '+%F %T') - Fichiers source: $NB_SOURCE, dans la dernière complète: $NB_DEST" >> "$LOGFILE"

echo "$(date '+%F %T') - Sauvegarde terminée" | tee -a "$LOGFILE"
```

*Note pédagogique :* `--link-dest` (option `rsync`) crée des liens physiques vers les fichiers inchangés de la référence indiquée, économisant l'espace disque tout en présentant chaque sauvegarde comme une arborescence complète et directement navigable — une technique combinant les avantages d'une sauvegarde complète en apparence avec l'efficacité de stockage d'une incrémentale.

## 17. Troubleshooting

Scénario court : un espace de sauvegarde configuré depuis plusieurs mois se révèle, lors d'un audit, contenir uniquement des sauvegardes anciennes de plusieurs semaines, alors que la tâche planifiée censée les générer chaque nuit apparaît correctement configurée et le script ne signale aucune erreur explicite dans ses propres logs. Démarche : appliquer directement la méthodologie de la section 7.5/7.8 — vérifier d'abord si un mécanisme de heartbeat ou de vérification active existe réellement pour cette tâche (rappel direct du module 19 : une tâche planifiée sans vérification active peut échouer silencieusement pendant des mois) → si aucun heartbeat n'existe, c'est la cause racine organisationnelle du problème, indépendamment de la cause technique précise de l'échec lui-même → une fois cette lacune de supervision corrigée, investiguer la cause technique réelle de l'échec (espace disque de destination saturé, ce qui aurait dû lui-même être surveillé selon les modules précédents ; identifiants réseau expirés vers un stockage distant ; ou toute autre cause) avec les outils appropriés selon la nature exacte de l'échec.

## 18. Synthèse

- `vmstat`, `iostat`, `sar` complètent `top`/`htop` pour un diagnostic de performance rigoureux, historique et corrélé
- Un load average élevé ne dit rien seul sur sa cause — toujours corréler CPU, mémoire (swap) et disque (`%util`/`await`) avant de conclure
- Sauvegarde complète (simple à restaurer, coûteuse), incrémentale (économe, chaîne fragile), différentielle (compromis) — le choix dépend de la fréquence et de la criticité de restauration recherchée
- La règle 3-2-1 (3 copies, 2 supports, 1 hors site) couvre trois catégories de risque distinctes, chacune nécessaire
- `rsync` transfère efficacement les différences au niveau bloc ; `--delete` doit toujours être testé avec `--dry-run` avant exécution réelle
- Une sauvegarde jamais testée en restauration n'est pas fiable — la validation de restauration est aussi importante que la sauvegarde elle-même, un principe trop souvent négligé jusqu'à un incident

## 19. Questions d'évaluation

**QCM**
1. Un load average élevé avec un CPU peu sollicité et des processus en état `D` oriente le diagnostic vers :
 a) un problème CPU b) une attente disque c) toujours une cause applicative d) un problème réseau exclusivement

2. La règle 3-2-1 de sauvegarde recommande :
 a) 1 copie, 2 supports, 3 sites b) 3 copies, 2 supports différents, 1 copie hors site c) 3 sites, 2 copies, 1 support d) aucune règle chiffrée précise

**Questions ouvertes**
- Expliquez la différence entre sauvegarde incrémentale et différentielle, en particulier sur la complexité de restauration.
- Expliquez pourquoi une sauvegarde jamais testée en restauration ne peut pas être considérée comme fiable.

**Exercice technique**
- Donnez la commande `iostat` permettant d'observer les statistiques disque étendues, avec un intervalle de 2 secondes et 5 échantillons.

**Problème de diagnostic**
- Un serveur affiche un load average élevé avec un CPU peu sollicité. Proposez deux hypothèses distinctes et les commandes permettant de les départager.

## 20. Mini-projet

Développer un script (Bash ou Python) qui :
1. Collecte périodiquement des métriques de performance corrélées (CPU, mémoire/swap, disque via `iostat`) et les journalise de façon structurée
2. Détecte automatiquement, à partir de ces métriques croisées, la signature de chacune des causes du tableau de la section 7.2 (charge CPU réelle, attente disque, thrashing mémoire), plutôt que de se fier à un seul indicateur isolé
3. Exécute une sauvegarde avec rotation complète/incrémentale (reprise et extension du script de la section 16), avec vérification d'intégrité basique
4. Intègre un mécanisme de heartbeat (rappel module 19) pour cette tâche de sauvegarde elle-même
5. Génère un rapport hebdomadaire combinant performance et état des sauvegardes, avec une section dédiée recommandant, le cas échéant, un test de restauration si aucun n'a été documenté depuis une durée configurable

---

# PARTIE II — TD

### Niveau 1 — Compréhension

**Exercice 1.1**
Citez trois outils de supervision de performance vus en cours au-delà de `top`/`htop`, et l'usage principal de chacun.

**Exercice 1.2**
Expliquez la différence entre sauvegarde complète et sauvegarde incrémentale.

**Exercice 1.3**
Décrivez les trois composantes de la règle 3-2-1 et le risque distinct que chacune protège.

### Niveau 2 — Application

**Exercice 2.1**
Donnez la commande permettant d'observer, avec `iostat`, les statistiques disque étendues toutes les 2 secondes.

**Exercice 2.2**
Donnez la commande `rsync` permettant de simuler une synchronisation (sans l'exécuter réellement) entre `/data/` et `/backup/`.

**Exercice 2.3**
Donnez la commande `tar` permettant de créer une archive compressée gzip du répertoire `/etc` nommée `etc_backup.tar.gz`.

### Niveau 3 — Analyse

**Exercice 3.1**
Un serveur affiche, via `vmstat`, des colonnes `si`/`so` actives en continu, avec un CPU peu sollicité. Un load average élevé est également observé. Quelle cause cela suggère-t-il, et quelle commande complémentaire permettrait de la confirmer plus précisément ?

**Exercice 3.2**
Une entreprise choisit une stratégie de sauvegarde purement incrémentale pour un serveur critique, sans jamais réaliser de nouvelle sauvegarde complète après la première. Après six mois, un incident nécessite une restauration complète. Quel risque cette stratégie fait-elle courir, indépendamment de toute panne matérielle, et pourquoi une sauvegarde différentielle ou des sauvegardes complètes périodiques auraient réduit ce risque ?

### Niveau 4 — Problème complexe

**Exercice 4.1**
Un ingénieur junior diagnostique un serveur "lent" en consultant uniquement `top`, observe un load average élevé avec un CPU peu sollicité, et conclut immédiatement à un problème de base de données mal optimisée, engageant une optimisation de requêtes SQL qui prend plusieurs jours sans amélioration mesurable. En vous appuyant sur la méthodologie de corrélation vue en cours, expliquez l'erreur méthodologique commise, proposez la démarche correcte qui aurait dû être suivie dès le départ, et formulez au moins deux hypothèses alternatives que cette démarche aurait permis d'évaluer avant d'engager un travail d'optimisation applicative coûteux en temps.

---

## Correction du TD

**1.1** — `vmstat` : vue synthétique et périodique de la mémoire, du swap, du CPU et des processus, utile pour un premier coup d'œil rapide. `iostat` : statistiques détaillées d'utilisation des disques (débit, temps d'attente, taux d'occupation), indispensable pour diagnostiquer un problème d'E/S. `sar` : collecte historique de métriques système, permettant d'analyser une période passée, utile pour un incident déjà survenu sans supervision active au moment des faits.

**1.2** — Une sauvegarde complète copie l'intégralité des données à chaque exécution, simple à restaurer mais coûteuse en temps et en espace si répétée fréquemment. Une sauvegarde incrémentale ne copie que les données modifiées depuis la dernière sauvegarde (complète ou incrémentale précédente), rapide et économe en espace, mais nécessitant de rejouer toute la chaîne de sauvegardes (la complète et toutes les incrémentales suivantes, dans l'ordre) pour une restauration complète.

**1.3** — Le nombre de copies (3 au minimum) protège contre la corruption ou la perte d'une copie individuelle. La diversité de support (2 supports différents) protège contre une panne matérielle systémique affectant un type de support particulier (par exemple, tous les disques d'un même modèle défaillant simultanément). La localisation hors site (1 copie) protège contre un sinistre physique localisé affectant l'intégralité d'un site (incendie, inondation, vol, ou une erreur de configuration touchant toute l'infrastructure locale).

**2.1** — `iostat -x 2`.

**2.2** — `rsync -avz --dry-run /data/ /backup/`.

**2.3** — `tar -czvf etc_backup.tar.gz /etc`.

**3.1** — Les colonnes `si`/`so` actives en continu dans `vmstat`, combinées à un CPU peu sollicité et un load average élevé, suggèrent un **thrashing mémoire** (rappel UE1-M3) : le système passe une part significative de son temps à échanger des pages entre RAM et swap plutôt qu'à exécuter du travail utile. La commande complémentaire permettant de confirmer plus précisément cette hypothèse est `free -h` (pour observer l'état global de la mémoire disponible et du swap utilisé) et/ou `ps -eo pid,comm,rss --sort=-rss` pour identifier les processus les plus consommateurs de mémoire résidente, potentiellement responsables de cette pression mémoire.

**3.2** — Le risque principal est celui d'une **chaîne de restauration extrêmement longue et fragile** : après six mois de sauvegardes purement incrémentales sans jamais de nouvelle complète, une restauration complète nécessiterait de rejouer la sauvegarde complète initiale **et l'intégralité des mois d'incrémentales suivantes**, dans l'ordre exact, sans qu'aucun maillon ne manque ni ne soit corrompu — un processus potentiellement très long (bien au-delà d'un objectif de temps de restauration raisonnable) et à haut risque d'échec (une seule incrémentale corrompue ou manquante sur toute cette période casserait irrémédiablement la chaîne de restauration à partir de ce point). Une sauvegarde différentielle, ou des sauvegardes complètes périodiques régulières, auraient réduit ce risque en limitant le nombre de maillons nécessaires à une restauration complète (la dernière complète plus une seule différentielle, ou simplement la dernière complète si suffisamment récente), rendant la chaîne de restauration structurellement plus courte, plus rapide à exécuter, et plus robuste face à la corruption ou la perte d'un maillon individuel.

**4.1** — L'erreur méthodologique commise est de tirer une conclusion définitive à partir d'un **seul indicateur** (`top`, load average élevé) sans corrélation avec d'autres outils, alors que ce même symptôme (load average élevé, CPU peu sollicité) peut avoir plusieurs causes racines très différentes, comme rappelé dans le tableau de la section 7.2 du cours — un load average élevé ne désigne jamais, à lui seul, une cause applicative de façon certaine. La démarche correcte aurait dû inclure, avant tout engagement dans une optimisation applicative coûteuse, une vérification systématique avec `iostat -x` (pour observer `%util` et `await`, confirmant ou infirmant une saturation disque) et `vmstat` (pour observer les colonnes `si`/`so`, confirmant ou infirmant un thrashing mémoire). Deux hypothèses alternatives que cette démarche aurait permis d'évaluer avant d'engager le travail d'optimisation SQL : (1) une **saturation disque** (le stockage sous-jacent à la base de données étant sous-dimensionné pour le volume d'E/S réel généré par la charge actuelle, indépendamment de la qualité des requêtes SQL elles-mêmes) ; (2) un **thrashing mémoire** (la base de données, ou d'autres processus du même serveur, consommant plus de mémoire que la RAM physique disponible, forçant un swap excessif qui dégraderait les performances de façon similaire, sans aucun rapport avec l'optimisation des requêtes). Dans les deux cas, l'optimisation SQL engagée n'aurait eu aucun effet mesurable sur la performance réelle, exactement le résultat décrit dans l'énoncé — un temps et un effort d'ingénierie significatifs auraient pu être économisés par une démarche de diagnostic corrélé appliquée dès le départ.

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur système junior pour diagnostiquer un serveur signalé comme "lent" avec une méthodologie rigoureuse, puis pour concevoir et **tester réellement** une stratégie de sauvegarde avec restauration.

## Architecture

Une VM Linux avec `sysstat` installé (`sar`, `iostat`), `rsync`, `tar` disponibles.

## Objectifs
- Appliquer la méthodologie de diagnostic corrélé de performance sur un cas simulé
- Concevoir une sauvegarde avec rotation complète/incrémentale efficace
- Tester une restauration complète et en mesurer la durée
- Diagnostiquer un cas de sauvegarde silencieusement défaillante

## Préparation

```bash
which vmstat iostat sar rsync tar 2>/dev/null
sudo apt install sysstat -y 2>/dev/null || echo "adapter selon votre gestionnaire de paquets (module 20)"
```

## Étapes

### Étape 1 — Provoquer et diagnostiquer une charge CPU réelle vs une attente disque

```bash
# Scénario A : charge CPU réelle
timeout 10 bash -c 'while true; do :; done' &
sleep 2
vmstat 1 3
iostat -x 1 3
wait

# Scénario B : attente disque simulée (écriture intensive)
timeout 10 dd if=/dev/zero of=/tmp/gros_fichier bs=1M count=500 oflag=direct 2>&1 &
sleep 2
vmstat 1 3
iostat -x 1 3
wait
rm -f /tmp/gros_fichier
```
*Question :* comparez les colonnes `us` (vmstat) et `%util`/`await` (iostat) entre les deux scénarios. Quelle signature distincte confirme, pour chacun, la cause réelle (CPU vs disque) conformément au tableau de la section 7.2 du cours ?

### Étape 2 — Sauvegarde avec rotation complète/incrémentale

```bash
mkdir -p /tmp/lab_data /tmp/lab_backup
echo "Fichier initial" > /tmp/lab_data/fichier1.txt
echo "Autre fichier" > /tmp/lab_data/fichier2.txt

# Adapter le script de la section 16 du cours (backup_rotation.sh) aux chemins de laboratoire
# puis l'exécuter une première fois (simuler un dimanche pour la complète)
bash backup_rotation.sh   # avec SOURCE=/tmp/lab_data DESTINATION=/tmp/lab_backup en variables adaptées

ls -l /tmp/lab_backup/

# Modifier les données puis relancer (simuler un jour de semaine pour l'incrémentale)
echo "Nouvelle ligne" >> /tmp/lab_data/fichier1.txt
echo "Fichier ajouté" > /tmp/lab_data/fichier3.txt
```

### Étape 3 — Tester une restauration complète et en mesurer la durée

```bash
mkdir -p /tmp/lab_restauration
DEBUT=$(date +%s)

rsync -a /tmp/lab_backup/derniere_complete/ /tmp/lab_restauration/

FIN=$(date +%s)
echo "Durée de restauration: $((FIN - DEBUT)) secondes"

diff -r /tmp/lab_data /tmp/lab_restauration && echo "Restauration identique à la source actuelle" || echo "Différences détectées (attendu si des incrémentales n'ont pas été appliquées)"
```
*Question :* la restauration à partir de la seule "dernière complète" reflète-t-elle les modifications faites à l'étape 2 après cette sauvegarde complète ? Qu'aurait-il fallu faire en plus pour une restauration réellement à jour, conformément à la section 7.4 du cours ?

### Étape 4 — Diagnostiquer une sauvegarde silencieusement défaillante

```bash
# Simuler une destination saturée (scénario volontaire)
DESTINATION_PLEINE="/tmp/lab_backup_pleine"
mkdir -p "$DESTINATION_PLEINE"
# Simuler l'espace insuffisant en limitant artificiellement via un petit tmpfs si possible,
# sinon documenter en réflexion l'effet attendu :
rsync -a /tmp/lab_data/ "$DESTINATION_PLEINE/" 2>&1 | tee /tmp/lab_backup_test.log
echo "Code de retour: $?"
```
*Question :* si cette commande avait échoué à cause d'un espace disque insuffisant, le code de retour de `rsync` aurait-il été différent de 0 ? Pourquoi un script de sauvegarde qui ignore ce code de retour (comme dans le scénario de troubleshooting du cours) peut-il rester silencieux malgré un échec réel ?

## Commandes de référence
`vmstat`, `iostat -x`, `rsync -a`, `tar -czvf`, `diff -r`

## Résultats attendus
- Comparaison argumentée des deux scénarios de l'étape 1 (charge CPU vs disque)
- Structure de sauvegarde complète/incrémentale générée (étape 2)
- Mesure de durée de restauration et résultat de la comparaison `diff` (étape 3)
- Réflexion argumentée sur le code de retour et la détection de défaillance (étape 4)

## Questions

1. Dans l'étape 1, si vous n'aviez utilisé que `top` sans `iostat`, auriez-vous pu distinguer avec certitude les deux scénarios ? Pourquoi la corrélation avec `iostat` était-elle nécessaire pour un diagnostic rigoureux plutôt qu'une simple supposition ?
2. Pourquoi la mesure de durée de restauration réalisée à l'étape 3, même sur un petit jeu de données de laboratoire, est-elle une pratique transférable et importante à l'échelle d'un vrai système de production (rappel section 7.8 du cours, notion de RTO) ?
3. Le script de sauvegarde de la section 16 du cours contient-il, tel quel, un mécanisme de vérification active de son succès (au sens du module 19, heartbeat) ? Si non, quelle serait la modification minimale à y apporter pour qu'il en dispose ?

## Troubleshooting (panne à diagnostiquer)

**Situation fournie :** un audit révèle qu'un espace de sauvegarde ne contient que des sauvegardes vieilles de plusieurs semaines, alors que la tâche planifiée associée (rappel module 19) apparaît correctement configurée, s'exécute bien chaque nuit selon les logs du démon cron, et le script de sauvegarde lui-même ne rapporte aucune erreur explicite dans sa propre sortie.

**Démarche attendue :**
1. Symptôme : sauvegardes anciennes malgré une tâche apparemment fonctionnelle et sans erreur rapportée
2. Hypothèse : le script s'exécute bien et se termine sans planter (aucune erreur "explicite" au sens d'un plantage), mais échoue silencieusement sur une opération précise dont le code de retour n'est jamais vérifié par le script lui-même — rappel direct de l'étape 4 du TP et du principe déjà établi en UE1/UE3 (M4, M16, M19) qu'une absence d'erreur visible ne garantit pas un succès réel si aucune vérification active du résultat n'est effectuée
3. Vérification : examiner précisément le script de sauvegarde utilisé en production pour repérer si les codes de retour de `rsync`/`tar`/toute commande critique sont effectivement vérifiés (`$?`, ou `set -e` couplé à une gestion d'erreur appropriée), ou si le script continue silencieusement son exécution même en cas d'échec d'une étape individuelle
4. Isolation : vérifier l'état de la destination de sauvegarde au moment probable des échecs (espace disque disponible via `df -h`, rappel module 15 ; accessibilité réseau si la destination est distante) pour identifier la cause technique précise de l'échec silencieux
5. Correction : réécrire le script pour vérifier systématiquement le code de retour de chaque opération critique, avec `set -euo pipefail` en tête de script (bonne pratique déjà recommandée dans plusieurs scripts du programme) et une notification explicite en cas d'échec (email, entrée `journald` via `logger`, ou déclenchement d'une alerte de supervision) ; ajouter un mécanisme de heartbeat (rappel module 19) totalement indépendant du script lui-même pour détecter tout futur échec, quelle qu'en soit la cause, sans dépendre uniquement de la correction du bug actuellement identifié
6. Vérification : reproduire volontairement une condition d'échec (par exemple une destination temporairement saturée ou injoignable) et confirmer que le script corrigé la détecte et la signale correctement, plutôt que de continuer silencieusement
7. Documentation : noter ce cas comme une synthèse directe de plusieurs leçons déjà rencontrées séparément dans le programme (vérification active de résultat, mécanisme de heartbeat indépendant, `set -euo pipefail` systématique) — appliquées ici spécifiquement au domaine critique de la sauvegarde, où un échec silencieux prolongé peut avoir des conséquences irréversibles en cas d'incident survenant précisément durant la période non couverte

## Validation

Le TP est validé si l'étudiant fournit :
- Les 4 résultats/réflexions demandés
- Des réponses argumentées aux 3 questions
- Une identification correcte de la cause du scénario de troubleshooting (échec silencieux non vérifié), avec une correction synthétisant les bonnes pratiques déjà rencontrées dans le programme

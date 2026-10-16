# UE1 — Module 4 : Systèmes de Fichiers et I/O
## Compléments : ÉVALUATION / PROJET / SLIDES / LAB

---

# PARTIE I — ÉVALUATION

**Durée :** 1h30
**Modalités :** documents non autorisés sauf `man` local si épreuve sur machine
**Barème total :** /20

## Section A — QCM (4 points, 0,5 pt/question)

1. L'inode d'un fichier contient :
 a) son nom b) ses métadonnées et pointeurs vers les blocs de données c) uniquement son contenu d) le chemin absolu du fichier

2. La VFS sert principalement à :
 a) accélérer le CPU b) présenter une interface uniforme indépendante du système de fichiers réel c) remplacer le cache de pages d) gérer les threads

3. Un lien symbolique cassé se produit quand :
 a) le compteur de liens de l'inode cible tombe à zéro b) le fichier cible est supprimé ou déplacé c) le disque est plein d) le processus est un zombie

4. Un `write()` qui retourne un succès garantit que :
 a) les données sont physiquement sur le disque b) les données sont dans le cache de pages, pas nécessairement encore écrites physiquement c) le fichier est immédiatement fermé d) aucune interruption ne sera générée

5. Le modèle d'E/S le plus adapté à un serveur gérant des dizaines de milliers de connexions simultanées est :
 a) bloquant, un thread par connexion b) non bloquant/asynchrone (`epoll`) c) synchrone avec `sleep` d) aucune différence significative

6. Un fichier supprimé avec `rm` mais toujours ouvert par un processus :
 a) libère immédiatement son espace disque b) continue d'occuper l'espace disque jusqu'à la fermeture du descripteur c) est automatiquement recréé d) provoque un crash du processus

7. Le rôle d'une interruption matérielle lors d'une E/S disque est de :
 a) forcer le processus à rester en busy-waiting b) notifier le noyau que l'opération est terminée, sans attente active du CPU c) déclencher un défaut de page d) invalider le TLB

8. L'option `copytruncate` de `logrotate` évite un problème de fichier de logs qui grossit indéfiniment car :
 a) elle supprime le service concerné b) elle tronque le fichier d'origine sur place, préservant l'inode déjà ouvert par le service c) elle désactive le cache de pages d) elle crée un lien symbolique automatiquement

## Section B — Questions ouvertes (6 points)

**Q1 (2 pts).** Expliquez pourquoi un nom de fichier "n'appartient pas" au fichier lui-même, en vous appuyant sur la relation entre répertoire, inode et compteur de liens.

**Q2 (2 pts).** Un développeur constate qu'après avoir appelé `write()` sur un gros fichier, le programme continue immédiatement sans attendre, mais une coupure d'alimentation survenue juste après entraîne la perte de certaines données récemment écrites. Expliquez ce phénomène et citez le mécanisme qui permettrait de le garantir contre (au prix de la performance).

**Q3 (2 pts).** Un service de logs continue d'écrire dans un fichier renommé par un outil de rotation, sans qu'aucun signal ne lui ait été envoyé. Expliquez précisément, en termes d'inode et de descripteur de fichier, où sont réellement écrites les données, et pourquoi le nouveau fichier créé sous l'ancien nom reste vide.

## Section C — Exercice technique (4 points)

Écrivez une séquence de commandes Bash permettant de : créer un fichier, créer un lien physique vers ce fichier, vérifier que le compteur de liens vaut 2, supprimer le fichier original, et prouver que le contenu reste accessible via le lien physique.

## Section D — Problème de diagnostic (6 points)

`df -h` indique 97% d'utilisation sur `/var`, mais un audit avec `du -sh /var/*` ne comptabilise qu'environ 60% de l'espace annoncé occupé.

**a) (2 pts)** Formulez l'hypothèse la plus probable au vu des mécanismes vus en cours, et justifiez pourquoi `du` ne la révèle pas.

**b) (2 pts)** Donnez la commande précise permettant de confirmer cette hypothèse, et précisez ce que vous attendez d'observer dans sa sortie.

**c) (2 pts)** Une fois la cause confirmée, proposez une correction immédiate et une mesure préventive de fond pour éviter la récurrence.

## Grille de correction résumée

| Section | Points | Critères |
|---|---|---|
| A — QCM | /4 | Réponse exacte uniquement |
| B — Questions ouvertes | /6 | Précision technique, mécanisme correctement expliqué (inode/descripteur/cache) |
| C — Exercice technique | /4 | Séquence de commandes correcte et complète |
| D — Diagnostic | /6 | Hypothèse cohérente, commande de vérification pertinente, corrections distinctes et justifiées |

---

# PARTIE II — MINI-PROJET (échelle module)

## Cahier des charges

**Titre :** Auditeur d'espace disque fantôme (Ghost Disk Space Auditor)

**Contexte :** À la suite des deux outils précédents (Process Watchdog, Memory Drift Detector), l'équipe DevOps souhaite un troisième outil ciblant spécifiquement les incidents de saturation disque causés par des fichiers supprimés mais toujours ouverts — un scénario récurrent identifié en cours et en TP.

### Objectifs du livrable

Développer un outil (Bash ou Python, au choix justifié) qui :
1. Scanne périodiquement les fichiers supprimés mais toujours maintenus ouverts par un processus (`lsof` ou équivalent), au-delà d'un seuil de taille configurable
2. Pour chaque détection, identifie : PID, nom de la commande, taille estimée occupée, ancienneté du processus
3. Calcule et journalise l'espace disque "fantôme" total (somme de toutes les détections) par point de montage
4. Compare cet espace fantôme à l'espace réellement rapporté disponible par `df`, et signale si sa libération résoudrait une alerte de saturation existante
5. Ne prend **aucune action corrective automatique** (ne tue ni ne redémarre aucun processus) — propose uniquement une commande suggérée en sortie
6. Génère un rapport hebdomadaire de synthèse

### Contraintes techniques
- Gestion robuste des erreurs (`lsof` absent, permissions insuffisantes sur certains PIDs)
- Configuration externalisée (seuils, points de montage à surveiller)
- Documentation d'installation et d'utilisation (README)
- Tests démontrés sur au moins 2 scénarios : fichier supprimé simple (comme au TP), et cas simulé de rotation de logs sans signal (fichier renommé, ancien descripteur toujours actif)

### Livrables attendus
- Code source commenté
- README (installation, configuration, format du rapport)
- Journal de tests montrant les 2 scénarios validés
- Rapport court (1 page) sur les choix techniques, notamment la robustesse du parsing `lsof` (voir note du cours sur `lsof -F`), et les limites connues (ex : ne détecte pas les fichiers dont l'espace est "gaspillé" pour d'autres raisons, comme des trous non alloués)

### Grille d'évaluation

| Critère | Pondération |
|---|---|
| Détection correcte des fichiers supprimés ouverts | 25% |
| Calcul et agrégation de l'espace fantôme par point de montage | 20% |
| Comparaison avec `df` et pertinence du signalement | 15% |
| Robustesse (gestion d'erreurs, parsing fiable) | 15% |
| Documentation et rapport | 15% |
| Tests démontrés (2 scénarios) | 10% |

### Modalité
Travail individuel ou en binôme, durée indicative : 1 semaine. Les trois outils du module 2, 3 et 4 (Process Watchdog, Memory Drift Detector, Ghost Disk Space Auditor) pourront être présentés en fin d'UE1 comme les briques d'une suite de supervision unifiée — anticiper cette convergence dans les choix d'architecture est valorisé mais non obligatoire à ce stade.

---

# PARTIE III — SLIDES (plan de présentation)

**Format :** ~20 slides pour une séance de 4h de CM

1. **Titre** — Systèmes de fichiers et E/S
2. **Pourquoi ce module ?** — cas d'usage réels (base de données, serveur web, sauvegarde)
3. **"Tout est fichier"** — le modèle Unix, exemples (périphériques, pipes, `/proc`)
4. **Inode vs nom de fichier** — schéma répertoire → inode → blocs de données
5. *Slide interactive :* "Le nom du fichier appartient-il au fichier ?" (vote/discussion avant révélation)
6. **Lien physique vs lien symbolique** — schéma comparatif, cas de rupture
7. **La VFS** — schéma de l'abstraction multi-FS
8. **Montage** — modèle arborescence unique vs lettres de lecteur
9. **Le cache de pages** — schéma, pourquoi RAM "utilisée" ≠ RAM "indisponible"
10. **Pages sales et writeback** — le piège du `write()` "réussi"
11. **E/S bloquantes** — schéma, coût en threads (rappel M2)
12. **E/S non bloquantes** — `select`/`poll`/`epoll`, schéma de principe
13. **E/S asynchrones** — notification sans attente active
14. *Slide interactive :* "Quel modèle d'E/S pour un serveur à 50 000 connexions simultanées ?" (lien direct avec le coût des threads vu en M2)
15. **Interruptions** — pourquoi pas de busy-waiting, schéma du réveil du processus
16. **Chemin complet d'un `read()`** — schéma en 8 étapes (cf. cours §7.7)
17. **Descripteurs de fichiers** — table par processus, partage après `fork()`
18. **Étude de cas industrielle** — le `rm` qui ne libère pas l'espace disque
19. **Sécurité** — permissions, symlink attacks, lien avec W^X (rappel M3)
20. **Synthèse + questions ouvertes** — transition vers TD/TP

*Note pédagogique :* la slide 14 est le point de convergence pédagogique du cours — elle relie explicitement le coût du changement de contexte et de la création de threads (M2) au choix architectural des serveurs à haute performance. Prendre le temps d'un vrai échange ici plutôt que de l'expédier.

---

# PARTIE IV — LAB (environnement de laboratoire)

## Objectif du LAB
Fournir un environnement reproductible permettant de manipuler inodes, liens, descripteurs de fichiers et cache de pages, et de provoquer en toute sécurité le scénario "fichier supprimé toujours ouvert" ainsi qu'un scénario de rotation de logs cassée.

## Infrastructure

**Option recommandée : conteneur unique avec volume dédié** (permet d'observer `df`/`du` sur un système de fichiers isolé de l'hôte, sans risque)

```
┌────────────────────────────────────┐
│   Hôte (poste étudiant / VM école)  │
│  ┌────────────────────────────────┐│
│  │  Conteneur Docker "lab-fs"       ││
│  │  Image: ubuntu:24.04              ││
│  │  Volume dédié: /data (ext4 simulé)││
│  │  Outils: lsof, du, df, stat,      ││
│  │  logrotate, python3               ││
│  └────────────────────────────────┘│
└────────────────────────────────────┘
```

### Dockerfile fourni aux étudiants

```dockerfile
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y \
    lsof coreutils util-linux logrotate python3 procps \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /lab
COPY scripts/ /lab/scripts/
CMD ["/bin/bash"]
```

### Lancement

```bash
docker build -t lab-fs .
docker run -it --rm --name lab-fs \
    -v ./scripts:/lab/scripts \
    -v lab-fs-data:/data \
    lab-fs
```

*Note :* l'utilisation d'un volume nommé (`lab-fs-data`) plutôt qu'un simple répertoire du conteneur permet d'observer un comportement `df`/`du` cohérent même après redémarrage du conteneur, utile pour des scénarios de rotation de logs étalés sur plusieurs sessions de TP.

## Configuration
- Prévoir `/data` comme répertoire de travail principal pour tous les scénarios (plutôt que `/tmp`, pour rester cohérent avec un usage type "partition de données" en production)
- Volume monté pour édition des scripts depuis l'hôte
- `logrotate` préinstallé pour permettre le scénario de rotation cassée en conditions quasi réelles

## Scénarios préconfigurés pour le LAB

| Scénario | Script/config fourni | Objectif pédagogique |
|---|---|---|
| Liens physiques/symboliques | `scripts/links_demo.sh` | Observer inode partagé, lien cassé |
| Fichier supprimé toujours ouvert | `scripts/deleted_open_demo.py` | Reproduire le cas de l'étude de cas industrielle |
| Cache de pages | `scripts/page_cache_demo.sh` (utilise `dd` + `sync` + `free`) | Observer buff/cache et writeback |
| Rotation de logs cassée | `scripts/fake_service.py` + `configs/logrotate_broken.conf` | Reproduire le cas TD 4.1 / troubleshooting TP |

**`configs/logrotate_broken.conf` (exemple fourni, volontairement mal configuré) :**
```
/data/app.log {
    daily
    rotate 7
    missingok
    notifempty
    # volontairement absent : postrotate / copytruncate
}
```

**Déroulé du scénario rotation cassée :**
```bash
python3 scripts/fake_service.py &   # écrit en continu dans /data/app.log
PID=$!
sleep 5
logrotate -f configs/logrotate_broken.conf   # force une rotation immédiate
sleep 5
ls -lh /data/                # app.log existe, mais quasi vide
lsof -p $PID | grep deleted  # révèle l'ancien fichier renommé, toujours ouvert et grossissant
```

## Objectifs pédagogiques du LAB
- Manipuler concrètement les commandes d'observation (`lsof`, `stat`, `du`, `df`) en toute sécurité
- Reproduire fidèlement, en quelques commandes, un incident classique et coûteux en production (rotation de logs cassée)
- Préparer le mini-projet (Ghost Disk Space Auditor) sur un environnement identique pour tous les étudiants

## Points de vigilance pour l'enseignant
- Vérifier que `logrotate` est exécutable sans droits root complets dans le conteneur (peut nécessiter un ajustement des permissions sur `/data` et les fichiers d'état de `logrotate`, ex : `--state` pointant vers un chemin accessible en écriture)
- Le volume nommé Docker persiste entre les runs `--rm` du conteneur — prévoir une commande de nettoyage (`docker volume rm lab-fs-data`) à communiquer aux étudiants en fin de séance pour repartir d'un état propre
- Insister sur le fait que `configs/logrotate_broken.conf` est un exemple pédagogique de mauvaise pratique — ne pas laisser les étudiants penser qu'il s'agit d'une configuration recommandée

# UE1 — Module 3 : Mémoire Virtuelle et Pagination
## Compléments : ÉVALUATION / PROJET / SLIDES / LAB

---

# PARTIE I — ÉVALUATION

**Durée :** 1h30
**Modalités :** documents non autorisés sauf `man` local si épreuve sur machine
**Barème total :** /20

## Section A — QCM (4 points, 0,5 pt/question)

1. La mémoire virtuelle résout principalement :
 a) la lenteur du CPU b) l'isolation entre processus et la fragmentation c) le nombre de threads disponibles d) la vitesse du réseau

2. La MMU utilise pour traduire une adresse virtuelle :
 a) le cache L1 uniquement b) la table des pages (accélérée par le TLB) c) le PCB d) le scheduler

3. Un défaut de page mineur :
 a) implique toujours un accès disque b) ne nécessite pas d'accès disque c) termine le processus d) invalide tout le TLB

4. VSZ et RSS diffèrent typiquement à cause de :
 a) la segmentation b) l'allocation paresseuse (lazy allocation) c) le swap uniquement d) l'ASLR

5. Le thrashing se caractérise par :
 a) CPU saturé, swap inactif b) CPU faible, activité swap élevée, système quasi figé c) RAM totalement libre d) absence de défauts de page

6. L'OOM Killer intervient :
 a) à chaque défaut de page b) quand la pression mémoire devient critique et qu'aucune page ne peut être libérée à temps c) uniquement au démarrage du système d) pour terminer les processus zombies

7. ASLR sert principalement à :
 a) accélérer les accès mémoire b) randomiser les adresses pour compliquer l'exploitation de failles mémoire c) réduire la taille des pages d) éviter le swap

8. Sous Linux, un processus dans un conteneur peut être tué par l'OOM Killer même si `free -h` sur l'hôte montre de la RAM disponible, car :
 a) c'est un bug connu b) une limite mémoire cgroup locale au conteneur peut être atteinte indépendamment de l'hôte c) les conteneurs n'ont pas de mémoire virtuelle d) le TLB est partagé entre conteneurs

## Section B — Questions ouvertes (6 points)

**Q1 (2 pts).** Expliquez le rôle du TLB et pourquoi son existence est nécessaire en complément de la table des pages, en évoquant explicitement le coût qu'il permet d'éviter.

**Q2 (2 pts).** Un processus affiche VSZ=2 Go et RSS=80 Mo juste après son démarrage. Ce chiffre est-il anormal ? Expliquez le mécanisme qui justifie un tel écart.

**Q3 (2 pts).** Distinguez précisément un cas de "processus en attente I/O disque classique" (vu en UE1-M2) d'un cas de "thrashing mémoire" (UE1-M3), en précisant l'indicateur `vmstat` qui permet de trancher entre les deux.

## Section C — Exercice technique (4 points)

Écrivez un script Bash qui, à partir de la sortie de `free -m`, calcule le pourcentage de swap utilisé, et affiche un message différent selon 3 seuils : `<10%` (OK), `10-50%` (surveillance), `>50%` (alerte critique).

## Section D — Problème de diagnostic (6 points)

Un service applicatif conteneurisé redémarre 3 à 4 fois par jour, toujours après plusieurs heures de fonctionnement, sans erreur applicative dans ses propres logs.

**a) (2 pts)** Formulez l'hypothèse la plus probable au vu des symptômes, et justifiez-la.

**b) (2 pts)** Donnez la séquence de 3 commandes permettant de confirmer ou infirmer cette hypothèse, en précisant ce que chaque commande doit révéler.

**c) (2 pts)** Si l'hypothèse est confirmée, proposez deux pistes de correction distinctes (une immédiate/symptomatique, une de fond), en expliquant leur différence.

## Grille de correction résumée

| Section | Points | Critères |
|---|---|---|
| A — QCM | /4 | Réponse exacte uniquement |
| B — Questions ouvertes | /6 | Précision technique + justification du mécanisme sous-jacent |
| C — Exercice technique | /4 | Script fonctionnel, seuils correctement implémentés |
| D — Diagnostic | /6 | Hypothèse cohérente avec les symptômes, commandes pertinentes, pistes de correction distinctes et justifiées |

---

# PARTIE II — MINI-PROJET (échelle module)

## Cahier des charges

**Titre :** Détecteur de dérive mémoire (Memory Drift Detector)

**Contexte :** Suite au Process Watchdog développé au module précédent, l'équipe DevOps souhaite un second outil, complémentaire, focalisé spécifiquement sur les tendances mémoire — capable de détecter une fuite avant qu'elle ne dégénère en incident de production (OOM Kill ou thrashing).

### Objectifs du livrable

Développer un outil (Bash ou Python, au choix justifié) qui :
1. Échantillonne périodiquement (intervalle configurable) le RSS des processus surveillés (liste de PIDs ou de noms de processus configurable)
2. Détecte une **tendance de croissance continue** du RSS sur une fenêtre glissante (ex : N mesures consécutives strictement croissantes, N configurable) — signature de fuite potentielle
3. Détecte une pression mémoire globale critique (swap actif + RAM disponible sous un seuil configurable)
4. Journalise chaque anomalie avec horodatage, PID/nom, valeurs mesurées, type d'anomalie
5. Vérifie explicitement, avant chaque comparaison, que le PID surveillé correspond toujours à la commande attendue (protection contre la réutilisation de PID vue en TP)
6. Génère un rapport de synthèse quotidien

### Contraintes techniques
- Gestion robuste des erreurs (processus disparu en cours de surveillance, PID invalide)
- Configuration externalisée (fichier de config ou arguments en ligne de commande, pas de valeurs codées en dur)
- Documentation d'installation et d'utilisation (README)
- Tests démontrés sur au moins 2 scénarios : fuite simulée (module précédent) et pression mémoire globale

### Livrables attendus
- Code source commenté
- README (installation, configuration, format du rapport)
- Journal de tests montrant au moins 2 scénarios validés, dont un avec réutilisation de PID délibérément provoquée pour prouver la protection
- Rapport court (1 page) sur les choix techniques et les limites connues

### Grille d'évaluation

| Critère | Pondération |
|---|---|
| Détection de tendance RSS (fenêtre glissante) | 25% |
| Détection de pression mémoire globale | 15% |
| Protection contre la réutilisation de PID | 15% |
| Robustesse (gestion d'erreurs, config externalisée) | 15% |
| Documentation et rapport | 15% |
| Tests démontrés | 15% |

### Modalité
Travail individuel ou en binôme, durée indicative : 1 semaine. Réutilisation autorisée et encouragée du code du Process Watchdog (Module 2) si l'architecture s'y prête — l'étudiant doit alors justifier en une phrase le choix de fusionner ou séparer les deux outils.

---

# PARTIE III — SLIDES (plan de présentation)

**Format :** ~20 slides pour une séance de 4h de CM

1. **Titre** — Mémoire virtuelle et pagination
2. **Pourquoi ce module ?** — l'illusion de mémoire infinie, cas d'usage réel (serveur à 200 processus, 16 Go de RAM)
3. **Les 3 problèmes sans mémoire virtuelle** — isolation, fragmentation, sur-allocation
4. **Adresse virtuelle vs adresse physique** — schéma, exemple de deux processus utilisant la même adresse virtuelle
5. **Pagination : pages et cadres** — schéma pages/cadres
6. **La table des pages** — rôle, structure simplifiée
7. **MMU** — traduction d'adresse à chaque accès mémoire
8. **TLB** — pourquoi un cache dédié, TLB hit vs TLB miss
9. *Slide interactive :* "Pourquoi un changement de contexte peut-il ralentir les accès mémoire suivants ?" (lien avec le TLB)
10. **Défaut de page — vue d'ensemble** — schéma des 6 étapes de traitement
11. **Défaut mineur vs majeur vs SIGSEGV** — tableau comparatif avec exemples
12. **Segmentation vs pagination** — comparaison, pourquoi Linux privilégie la pagination
13. **Swap** — principe, quand et pourquoi
14. **Thrashing** — schéma, signature symptomatique (CPU faible, si/so élevés)
15. **Cas Linux : allocation paresseuse** — VSZ vs RSS, schéma d'un `malloc` non encore accédé
16. **L'OOM Killer** — rôle, quand il intervient, `oom_score`
17. **Sécurité : ASLR et W^X** — vue conceptuelle, lien avec les buffer overflows (annonce UE8)
18. **Étude de cas industrielle** — le service qui se dégrade progressivement (swap croissant)
19. **Comparaison M2 vs M3** — tableau récapitulatif : état `D` classique (I/O disque) vs thrashing (pression mémoire), même symptôme apparent, causes différentes
20. **Synthèse + questions ouvertes** — transition vers TD/TP

*Note pédagogique :* prévoir une pause active après la slide 11 (défaut mineur vs majeur) : demander aux étudiants d'estimer, à main levée, si une opération donnée (accès `fork()`+écriture, ouverture d'un gros fichier, accès à une variable locale déjà utilisée) déclenche un défaut mineur, majeur, ou aucun défaut — correction collective immédiate.

---

# PARTIE IV — LAB (environnement de laboratoire)

## Objectif du LAB
Fournir un environnement reproductible permettant d'observer VSZ/RSS, de provoquer des défauts de page et une pression mémoire contrôlée, sans risque pour une machine partagée, et de servir de socle au TP et au mini-projet.

## Infrastructure

**Option recommandée : conteneur unique avec limite mémoire explicite** (permet de provoquer un OOM Kill de façon sûre et reproductible, sans affecter l'hôte)

```
┌────────────────────────────────────┐
│   Hôte (poste étudiant / VM école)  │
│  ┌────────────────────────────────┐│
│  │  Conteneur Docker "lab-mem"      ││
│  │  Image: python:3.12-slim         ││
│  │  Limite mémoire: 256 Mo          ││
│  │  Swap conteneur: 128 Mo (option) ││
│  │  Outils: free, vmstat, pmap,     ││
│  │  procps, time                    ││
│  └────────────────────────────────┘│
└────────────────────────────────────┘
```

### Dockerfile fourni aux étudiants

```dockerfile
FROM python:3.12-slim
RUN apt-get update && apt-get install -y \
    procps time util-linux \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /lab
COPY scripts/ /lab/scripts/
CMD ["/bin/bash"]
```

### Lancement (avec limite mémoire pour provoquer un OOM Kill contrôlé)

```bash
docker build -t lab-mem .
docker run -it --rm --name lab-mem \
    --memory="256m" --memory-swap="384m" \
    -v ./scripts:/lab/scripts \
    lab-mem
```

*Note :* `--memory-swap="384m"` avec `--memory="256m"` autorise 128 Mo de swap conteneur (384-256) en plus de la limite RAM — utile pour observer un comportement de swap avant l'OOM Kill final. Mettre `--memory-swap` égal à `--memory` pour désactiver le swap et observer un OOM Kill plus direct.

## Configuration
- Limite mémoire volontairement basse (256 Mo) pour que les scénarios de fuite/pression se déclenchent en quelques dizaines de secondes, sans attente excessive en séance
- Volume monté pour édition des scripts depuis l'hôte
- Prévoir une image pré-construite en registry interne si le temps de build en séance pose problème logistique

## Scénarios préconfigurés pour le LAB

| Scénario | Script fourni | Objectif pédagogique |
|---|---|---|
| VSZ vs RSS | `scripts/vsz_rss_demo.py` | Observer l'écart et l'allocation paresseuse |
| Défauts de page | `scripts/page_fault_demo.py` | Compter défauts mineurs via `/usr/bin/time -v` |
| Fuite simulée | `scripts/leak_simulator.py` | Observer la croissance progressive de RSS |
| OOM Kill contrôlé | `scripts/oom_trigger.py` (allocation rapide dépassant la limite du conteneur) | Observer un OOM Kill réel et l'analyser via `dmesg` côté hôte |

**Script `oom_trigger.py` (exemple fourni) :**
```python
import time
data = []
while True:
    data.append(bytearray(10 * 1024 * 1024))  # +10 Mo toutes les 0.5s
    print(f"Alloué: ~{len(data)*10} Mo")
    time.sleep(0.5)
```

## Objectifs pédagogiques du LAB
- Observer en toute sécurité un OOM Kill réel et son diagnostic (`dmesg`, `docker inspect`), sans risquer l'hôte grâce à la limite cgroup du conteneur
- Relier concrètement la théorie (allocation paresseuse, thrashing, OOM Killer) à des commandes et sorties réelles
- Préparer le mini-projet (Memory Drift Detector) sur un environnement identique pour tous les étudiants

## Points de vigilance pour l'enseignant
- Vérifier que la version de Docker utilisée en salle applique correctement les limites `--memory`/`--memory-swap` (certains environnements Docker Desktop nécessitent une configuration additionnelle du daemon)
- Le message d'OOM Kill du conteneur apparaît côté hôte via `dmesg` ou `docker inspect <container> --format='{{.State.OOMKilled}}'` — le communiquer explicitement aux étudiants, car ce n'est pas toujours visible dans les logs du conteneur lui-même
- Limiter la durée du scénario `oom_trigger.py` (le conteneur `--rm` se nettoie automatiquement après l'OOM Kill, mais prévoir un timeout de sécurité côté script si l'établissement a des contraintes de ressources partagées strictes)

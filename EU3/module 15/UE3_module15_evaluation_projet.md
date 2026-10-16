# UE3 — Module 15 : Filesystems, Mount, LVM et Stockage
## Compléments : ÉVALUATION / PROJET

---

# PARTIE I — ÉVALUATION

**Durée :** 1h30
**Modalités :** documents non autorisés sauf `man` local si épreuve sur machine
**Barème total :** /20

## Section A — QCM (4 points, 0,5 pt/question)

1. L'ordre logique correct des opérations pour rendre un nouveau disque utilisable est :
 a) mount → mkfs → partition b) partition → mkfs → mount c) mkfs → partition → mount d) mount → partition → mkfs

2. Dans l'architecture LVM, l'ordre des couches, du matériel vers l'usage final, est :
 a) VG → PV → LV b) PV → VG → LV c) LV → PV → VG d) PV → LV → VG

3. `/etc/fstab` devrait référencer un système de fichiers par :
 a) son nom de périphérique (`/dev/sdb1`) b) son UUID c) sa date de création d) son propriétaire

4. Après `lvextend -L +10G` sur un volume ext4, l'espace supplémentaire est utilisable par les applications :
 a) immédiatement, sans autre action b) seulement après exécution de `resize2fs` sur ce volume c) seulement après un redémarrage complet d) jamais, LVM ne permet pas l'extension

5. XFS, comparé à ext4, présente la particularité de :
 a) ne jamais pouvoir être étendu b) ne supporter aucune réduction de taille, contrairement à ext4 c) ne pas être compatible avec LVM d) nécessiter systématiquement un fsck au démarrage

6. Un snapshot LVM utilise un mécanisme de :
 a) duplication complète immédiate de toutes les données b) copie sur écriture (copy-on-write), n'occupant de l'espace que pour les blocs modifiés c) compression systématique d) chiffrement automatique

7. Exécuter `fsck` sur un système de fichiers monté et actif :
 a) est une pratique recommandée et sans risque b) est une pratique dangereuse pouvant provoquer une corruption de données c) est obligatoire avant chaque redémarrage d) accélère automatiquement le système

8. Une saturation disque peut avoir une conséquence de sécurité indirecte en :
 a) chiffrant automatiquement les données b) désactivant silencieusement la journalisation faute d'espace disponible c) désactivant le pare-feu d) supprimant automatiquement les comptes utilisateurs

## Section B — Questions ouvertes (6 points)

**Q1 (2 pts).** Expliquez pourquoi étendre un volume logique LVM (`lvextend`) ne suffit pas à rendre l'espace supplémentaire immédiatement utilisable, et quelle commande complète l'opération pour un système de fichiers ext4.

**Q2 (2 pts).** Expliquez l'intérêt de l'architecture LVM (PV/VG/LV) par rapport à un partitionnement classique, en particulier pour la gestion d'une croissance de besoin en stockage dans le temps.

**Q3 (2 pts).** Expliquez pourquoi utiliser l'UUID plutôt que le nom de périphérique dans `/etc/fstab` est une bonne pratique, avec un exemple concret de scénario où cela évite un incident.

## Section C — Exercice technique (4 points)

Donnez la séquence complète de commandes permettant de : créer un PV sur `/dev/sdc`, créer un VG nommé `vg_app` à partir de ce PV, créer un LV de 20 Go nommé `lv_data` dans ce VG, puis y appliquer un système de fichiers ext4.

## Section D — Problème de diagnostic (6 points)

Un volume logique LVM a été étendu de 15 Go (confirmé par `lvdisplay`), mais `df -h` continue d'afficher l'ancienne taille du système de fichiers plusieurs jours après l'opération, sans qu'aucune erreur n'ait été signalée lors de l'extension.

**a) (2 pts)** Formulez l'hypothèse la plus probable, en citant précisément l'étape manquante.

**b) (2 pts)** Donnez la commande de correction adaptée si le système de fichiers est ext4, et celle adaptée s'il est XFS.

**c) (2 pts)** Proposez une mesure organisationnelle (pas technique) permettant d'éviter que ce type d'oubli ne se reproduise à l'avenir.

## Grille de correction résumée

| Section | Points | Critères |
|---|---|---|
| A — QCM | /4 | Réponse exacte uniquement |
| B — Questions ouvertes | /6 | Précision technique, mécanisme correctement expliqué (extension en deux étapes, UUID) |
| C — Exercice technique | /4 | Séquence de commandes complète et dans le bon ordre |
| D — Diagnostic | /6 | Hypothèse exacte (extension FS oubliée), commandes correctes pour les deux FS, mesure organisationnelle pertinente |

---

# PARTIE II — MINI-PROJET (échelle module)

## Cahier des charges

**Titre :** Gestionnaire de capacité de stockage LVM (LVM Capacity Manager)

**Contexte :** Suite à l'outil d'audit de comptes et permissions du module 14, l'équipe système souhaite un second outil dédié à la gestion proactive de la capacité de stockage — capable d'anticiper une saturation et de suggérer (sans jamais exécuter automatiquement) les commandes d'extension appropriées, en tenant compte des spécificités LVM et des différences ext4/XFS.

### Objectifs du livrable

Développer un outil (Bash ou Python, au choix justifié) qui :
1. Surveille l'espace disque de l'ensemble des systèmes de fichiers montés et détecte ceux approchant un seuil critique configurable
2. Pour chaque système de fichiers en tension identifié comme volume LVM, interroge le VG parent pour déterminer l'espace libre immédiatement disponible pour une extension
3. Détecte et signale explicitement le cas d'une **extension LVM incomplète** (LV plus grand que le système de fichiers qu'il contient, reproduisant le scénario du TP) — un cas distinct d'une vraie saturation, ne nécessitant qu'un simple `resize2fs`/`xfs_growfs`, pas un ajout d'espace
4. Adapte sa suggestion de commande de redimensionnement selon le type réel de système de fichiers détecté (ext4 vs XFS), sans jamais proposer une commande incompatible avec le type détecté
5. Si le VG ne dispose d'aucun espace libre, signale une alerte de priorité plus élevée nécessitant une action matérielle, et vérifie l'existence de snapshots LVM potentiellement récupérables (piste de gestion de crise vue au TD, sans suppression automatique)
6. Journalise chaque détection avec horodatage et génère un rapport de synthèse périodique

### Contraintes techniques
- Ne jamais exécuter automatiquement une commande d'extension, de suppression de snapshot, ou de redimensionnement — l'outil recommande, il n'agit jamais de façon autonome sur le stockage
- Distinction stricte et fiable entre "extension LVM incomplète" (correction simple) et "vraie saturation nécessitant plus d'espace" (deux catégories de rapport différentes)
- Gestion d'erreurs robuste (commandes LVM non disponibles sur un système sans LVM, permissions insuffisantes)
- Configuration externalisée (seuils d'alerte, liste des points de montage à exclure de la surveillance si nécessaire)
- Documentation d'installation et d'utilisation (README)
- Tests démontrés sur au moins 3 scénarios : extension LVM incomplète (comme au TP), saturation avec espace VG disponible (suggestion d'extension), saturation sans espace VG disponible (alerte de priorité haute avec vérification des snapshots)

### Livrables attendus
- Code source commenté
- README (installation, configuration, format du rapport, limites connues)
- Journal de tests montrant les 3 scénarios validés, avec captures
- Rapport court (1 page) sur les choix techniques et les limites connues (ex : ne peut pas garantir que la suppression d'un snapshot ancien est réellement sans risque, se contente de le signaler pour décision humaine)

### Grille d'évaluation

| Critère | Pondération |
|---|---|
| Détection correcte de saturation et d'espace VG disponible | 25% |
| Distinction extension incomplète / vraie saturation | 25% |
| Adaptation correcte des commandes suggérées selon le type de FS (ext4/XFS) | 20% |
| Aucune action automatique sur le stockage (recommandation uniquement) | 15% |
| Documentation et tests démontrés (3 scénarios) | 15% |

### Modalité
Travail individuel ou en binôme, durée indicative : 1 semaine. Cet outil complète la série d'audit système initiée au module 14 — une intégration commune (format de rapport partagé) est valorisée mais non obligatoire.

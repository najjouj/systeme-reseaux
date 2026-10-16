# UE3 — Module 16 : Processus, Services et systemd en Pratique
## Compléments : ÉVALUATION / PROJET

---

# PARTIE I — ÉVALUATION

**Durée :** 1h30
**Modalités :** documents non autorisés sauf `man` local si épreuve sur machine
**Barème total :** /20

## Section A — QCM (4 points, 0,5 pt/question)

1. Après modification d'un fichier d'unité systemd, avant tout redémarrage du service, il faut exécuter :
 a) `systemctl restart` uniquement b) `systemctl daemon-reload` c) `systemctl enable` d) rien, c'est automatique

2. `Requires=` dans la section `[Unit]` exprime :
 a) une dépendance faible, sans effet en cas d'échec b) une dépendance forte : un échec de l'unité requise fait échouer celle qui la requiert c) une simple suggestion sans effet réel d) l'ordre d'affichage dans `systemctl status`

3. `StartLimitBurst` sert à :
 a) accélérer le démarrage b) limiter le nombre de tentatives de redémarrage automatique dans une fenêtre de temps c) augmenter automatiquement la mémoire allouée d) supprimer le service après un échec

4. `systemctl edit` sur une unité fournie par un paquet système :
 a) modifie directement le fichier d'origine b) crée un fichier de surcharge séparé, préservé lors des mises à jour du paquet c) désactive l'unité définitivement d) n'a aucun effet

5. Un service bloqué en état `failed` après épuisement de `StartLimitBurst` nécessite, après correction de la cause, la commande :
 a) `systemctl daemon-reload` seule b) `systemctl reset-failed` avant de pouvoir le redémarrer c) `systemctl mask` d) une réinstallation complète

6. Les cgroups sous-jacents à systemd permettent notamment de :
 a) accélérer le réseau b) suivre et arrêter fiablement l'ensemble des processus d'un service, y compris ses enfants indirects c) chiffrer automatiquement les données d) remplacer journald

7. `journalctl -u monapp -p err` affiche :
 a) tous les logs de monapp sans filtre b) uniquement les messages de sévérité erreur ou plus grave pour monapp c) uniquement les messages liés au réseau d) les logs de tous les services sauf monapp

8. Un timer systemd avec `Persistent=true` garantit que :
 a) la tâche s'exécute deux fois par sécurité b) si la machine était éteinte au moment prévu, la tâche s'exécute au prochain démarrage c) le timer ne peut jamais être désactivé d) la tâche s'exécute en boucle continue

## Section B — Questions ouvertes (6 points)

**Q1 (2 pts).** Expliquez pourquoi `systemctl edit` est préférable à une modification directe d'un fichier d'unité fourni par un paquet système, avec un exemple concret de conséquence d'une modification directe.

**Q2 (2 pts).** Décrivez la chaîne d'événements par laquelle l'absence de `StartLimitBurst` sur un service en boucle de crash peut provoquer un incident secondaire de saturation disque.

**Q3 (2 pts).** Expliquez la différence entre `Requires=` et `Wants=`, avec un exemple de situation où utiliser `Requires=` par erreur serait problématique.

## Section C — Exercice technique (4 points)

Rédigez un extrait de fichier d'unité systemd (section `[Service]` uniquement) pour un service qui doit : s'exécuter sous l'utilisateur `svc_app`, redémarrer automatiquement en cas d'échec avec un délai de 5 secondes entre tentatives, et cesser de retenter après 3 échecs en 30 secondes.

## Section D — Problème de diagnostic (6 points)

Un administrateur modifie un fichier d'unité pour changer la commande de démarrage d'un service, exécute `systemctl restart`, mais l'ancienne commande continue apparemment de s'exécuter.

**a) (2 pts)** Formulez l'hypothèse la plus probable, en citant l'étape manquante précise.

**b) (2 pts)** Donnez la commande permettant de confirmer cette hypothèse en comparant la définition réellement chargée par systemd au contenu du fichier sur le disque.

**c) (2 pts)** Donnez la séquence de commandes de correction complète.

## Grille de correction résumée

| Section | Points | Critères |
|---|---|---|
| A — QCM | /4 | Réponse exacte uniquement |
| B — Questions ouvertes | /6 | Précision technique, chaîne causale correctement reconstituée |
| C — Exercice technique | /4 | Directives `User=`, `Restart=`, `RestartSec=`, `StartLimitIntervalSec=`/`StartLimitBurst=` correctes |
| D — Diagnostic | /6 | Hypothèse exacte (daemon-reload oublié), commande de vérification pertinente, séquence de correction complète |

---

# PARTIE II — MINI-PROJET (échelle module)

## Cahier des charges

**Titre :** Superviseur de services applicatifs (Service Health Guardian)

**Contexte :** Suite aux outils d'audit développés aux modules 14 et 15, l'équipe système souhaite un outil dédié à la santé des services applicatifs déployés via systemd — capable de détecter une boucle de crash avant qu'elle ne dégénère en incident secondaire (saturation du journal, comme démontré au TD), et de vérifier la cohérence entre la configuration sur disque et celle réellement chargée par systemd.

### Objectifs du livrable

Développer un outil (Bash ou Python, au choix justifié), packagé lui-même comme unité systemd (timer + service `oneshot`, sur le modèle du module 6 de l'UE1), qui :
1. Liste l'ensemble des services en état `failed` et, pour chacun, résume les dernières lignes de journal pertinentes
2. Détecte un **rythme de redémarrage anormalement élevé** sur une fenêtre glissante pour une liste configurable de services critiques, **avant** qu'ils n'atteignent l'état `failed` définitif (détection précoce, pas seulement réactive)
3. Vérifie, pour chaque service critique surveillé, la cohérence entre le fichier d'unité sur disque et la définition effectivement chargée par systemd (détection de l'oubli de `daemon-reload`, cf. module et TD)
4. Vérifie que chaque service critique surveillé dispose bien de garde-fous de redémarrage configurés (`StartLimitBurst` non vide), et signale ceux qui en sont dépourvus comme risque potentiel
5. Journalise chaque détection avec horodatage et niveau de gravité
6. Génère un rapport quotidien de synthèse

### Contraintes techniques
- La détection de désynchronisation disque/mémoire (`daemon-reload` oublié) doit être fiable et testée explicitement (reproduction du scénario du TP)
- Gestion d'erreurs robuste (service inexistant, permissions insuffisantes)
- Configuration externalisée (liste des services critiques à surveiller, seuils de détection de boucle de crash)
- Documentation d'installation et d'utilisation (README), incluant son propre fichier d'unité et timer de déploiement
- Tests démontrés sur au moins 3 scénarios : service en échec simple (rapport avec résumé de logs), boucle de crash naissante détectée avant l'état `failed` définitif, désynchronisation disque/mémoire reproduite volontairement (modification sans `daemon-reload`)

### Livrables attendus
- Code source commenté
- Fichiers d'unité systemd (service + timer) pour le déploiement de l'outil lui-même
- README (installation, configuration, format du rapport, limites connues)
- Journal de tests montrant les 3 scénarios validés, avec captures
- Rapport court (1 page) sur les choix techniques, notamment la méthode de détection de désynchronisation disque/mémoire, et les limites connues

### Grille d'évaluation

| Critère | Pondération |
|---|---|
| Détection des services en échec avec résumé de logs pertinent | 20% |
| Détection précoce de boucle de crash (avant l'état failed définitif) | 25% |
| Détection de désynchronisation disque/mémoire (daemon-reload oublié) | 25% |
| Vérification de la présence de garde-fous de redémarrage | 15% |
| Documentation et tests démontrés (3 scénarios) | 15% |

### Modalité
Travail individuel ou en binôme, durée indicative : 1 semaine. Cet outil complète la série d'audit système initiée aux modules 14 et 15 — l'ensemble pourra converger vers une suite unifiée à la clôture de l'UE3.

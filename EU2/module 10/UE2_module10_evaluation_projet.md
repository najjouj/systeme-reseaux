# UE2 — Module 10 : TCP en Profondeur
## Compléments : ÉVALUATION / PROJET

---

# PARTIE I — ÉVALUATION

**Durée :** 1h30
**Modalités :** documents non autorisés sauf `man` local si épreuve sur machine
**Barème total :** /20

## Section A — QCM (4 points, 0,5 pt/question)

1. La troisième étape du three-way handshake (ACK final du client) sert à :
 a) transmettre les premières données applicatives b) confirmer au serveur la bonne réception de son SYN-ACK c) fermer la connexion d) demander une retransmission

2. Le champ ACK d'un segment TCP indique :
 a) le dernier octet reçu b) le prochain octet attendu c) la taille totale du fichier transféré d) le numéro de port du récepteur

3. Le contrôle de flux (rwnd) protège :
 a) le réseau contre la saturation collective b) le récepteur contre un tampon saturé c) l'émetteur contre ses propres pertes d) uniquement les connexions UDP

4. Après une perte de segment détectée, la fenêtre de congestion :
 a) reste inchangée b) est réduite significativement, puis reconstruite progressivement c) double immédiatement d) est ignorée par le protocole

5. L'état TIME_WAIT existe notamment pour :
 a) accélérer la fermeture de connexion b) garantir la bonne réception du dernier ACK et éviter la confusion avec une future connexion c) économiser de la bande passante d) chiffrer les données restantes

6. Un grand nombre de connexions en TIME_WAIT sur un serveur à fort trafic sortant est :
 a) toujours le signe d'une attaque b) un comportement normal, problématique seulement s'il mène à un épuisement réel de ports c) impossible en pratique d) réservé aux connexions UDP

7. Un nombre anormalement élevé de connexions en état SYN_RECV oriente le diagnostic vers :
 a) un problème de couche 2 uniquement b) une possible saturation de la file d'établissement de connexions (charge ou SYN flood) c) un problème de DNS d) une erreur de sous-réseau

8. Choisir un seuil d'alerte arbitraire pour le nombre de connexions TIME_WAIT, sans le rapporter à la plage de ports réellement disponible, risque de produire :
 a) une détection parfaitement fiable dans tous les cas b) des alertes non pertinentes, déconnectées du risque réel c) une amélioration automatique des performances TCP d) la désactivation automatique du contrôle de congestion

## Section B — Questions ouvertes (6 points)

**Q1 (2 pts).** Décrivez le three-way handshake TCP étape par étape, avec le rôle précis de chaque flag/numéro échangé.

**Q2 (2 pts).** Expliquez pourquoi un taux de perte de paquets apparemment faible peut provoquer une chute de débit disproportionnée sur un transfert TCP, en vous appuyant sur le comportement de la fenêtre de congestion.

**Q3 (2 pts).** Un administrateur constate une accumulation de connexions en SYN_RECV corrélée à des pics de charge, sans impact sur les connexions déjà établies. Expliquez pourquoi ce symptôme n'affecte que l'établissement de nouvelles connexions, en vous appuyant sur la machine à états TCP.

## Section C — Exercice technique (4 points)

Donnez la commande permettant de compter le nombre de connexions TCP par état (ESTABLISHED, TIME_WAIT, SYN_RECV, etc.) actuellement actives sur une machine Linux, sous une forme agrégée (nombre par état).

## Section D — Problème de diagnostic (6 points)

Un service accumule un nombre de connexions TIME_WAIT en croissance continue, jusqu'à provoquer des échecs d'établissement de nouvelles connexions sortantes ("impossible d'allouer un port").

**a) (2 pts)** Donnez la commande permettant de quantifier précisément ce nombre, et celle permettant de connaître la plage de ports éphémères disponible sur le système.

**b) (2 pts)** Proposez une cause probable liée à la conception applicative (pas à la configuration réseau), en la justifiant par le mécanisme TIME_WAIT vu en cours.

**c) (2 pts)** Proposez une correction de fond, en expliquant pourquoi un simple ajustement de paramètre noyau ne résout pas la cause racine du problème.

## Grille de correction résumée

| Section | Points | Critères |
|---|---|---|
| A — QCM | /4 | Réponse exacte uniquement |
| B — Questions ouvertes | /6 | Précision technique, mécanisme correctement expliqué (contrôle de congestion, machine à états) |
| C — Exercice technique | /4 | Commande correcte et agrégation par état |
| D — Diagnostic | /6 | Diagnostic quantifié, cause applicative correctement identifiée, correction de fond distincte d'un simple réglage |

---

# PARTIE II — MINI-PROJET (échelle module)

## Cahier des charges

**Titre :** Superviseur d'états TCP (TCP State Health Monitor)

**Contexte :** Suite aux outils réseau développés aux modules 7 et 8 (sonde multi-couches, surveillant ARP), l'équipe souhaite un troisième outil dédié spécifiquement à la santé des connexions TCP — capable de distinguer un état TIME_WAIT élevé mais bénin d'un véritable risque d'épuisement de ports, en évitant le piège du seuil arbitraire non calibré identifié en TP.

### Objectifs du livrable

Développer un outil (Bash ou Python, au choix justifié) qui :
1. Surveille en continu la répartition des connexions TCP par état (`ss -tan`)
2. Calcule un seuil d'alerte TIME_WAIT **dynamique**, basé sur un pourcentage configurable de la plage de ports éphémères réellement disponible sur le système (`net.ipv4.ip_local_port_range`), plutôt qu'une valeur arbitraire fixe
3. Détecte un nombre anormalement élevé de connexions en SYN_RECV, avec un seuil distinct et une alerte de nature différente (risque de saturation d'établissement ou SYN flood, pas un risque d'épuisement de ports)
4. Journalise chaque détection avec horodatage, état concerné, valeur observée, seuil calculé au moment de la détection
5. Produit un historique exploitable (fenêtre glissante) permettant de distinguer une croissance ponctuelle (pic de charge normal) d'une croissance continue et anormale sur plusieurs mesures consécutives
6. Génère un rapport quotidien de synthèse incluant la répartition complète par état et l'évolution du seuil dynamique calculé

### Contraintes techniques
- Le seuil TIME_WAIT ne doit jamais être une valeur codée en dur — toujours recalculé à partir de la plage de ports réelle du système
- Gestion d'erreurs robuste (`ss` absent, permissions insuffisantes)
- Configuration externalisée (pourcentage de seuil, intervalle de surveillance, seuil SYN_RECV)
- Documentation d'installation et d'utilisation (README)
- Tests démontrés sur au moins 2 scénarios : reproduction de l'accumulation TIME_WAIT du TP (script `many_connections.py`), et un cas où le nombre de TIME_WAIT est élevé en valeur absolue mais reste sous le seuil dynamique calculé (doit **ne pas** déclencher d'alerte, contrairement à un seuil fixe naïf)

### Livrables attendus
- Code source commenté
- README (installation, configuration, format du rapport, limites connues)
- Journal de tests montrant les 2 scénarios validés, avec preuve explicite que le second scénario ne déclenche pas de fausse alerte (contrairement à ce qu'un seuil fixe aurait produit)
- Rapport court (1 page) sur les choix techniques, notamment le calcul du seuil dynamique, et les limites connues (ex : ne tient pas compte d'un système utilisant déjà une grande partie de ses ports pour d'autres usages que TCP sortant)

### Grille d'évaluation

| Critère | Pondération |
|---|---|
| Calcul correct du seuil dynamique basé sur la plage de ports réelle | 25% |
| Détection distincte TIME_WAIT / SYN_RECV avec logiques séparées | 20% |
| Distinction pic ponctuel / croissance continue anormale | 20% |
| Robustesse et configuration externalisée | 15% |
| Documentation et rapport | 10% |
| Tests démontrés (2 scénarios, preuve d'absence de faux positif) | 10% |

### Modalité
Travail individuel ou en binôme, durée indicative : 1 semaine. Cet outil complète la série réseau initiée aux modules 7 et 8 — une intégration commune (format de rapport, configuration partagée) avec les outils précédents est valorisée mais non obligatoire.

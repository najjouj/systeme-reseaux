# UE2 — Module 10 : TCP en Profondeur

---

# PARTIE I — COURS

## 1. Titre du cours
TCP en profondeur : établissement de connexion, fiabilité, contrôle de flux et de congestion

## 2. Niveau
Élève ingénieur — cycle ingénieur, UE2 réseaux (S1)

## 3. Durée
CM : 5h (module dense — +4h TD, +4h TP en Parties II et III)

## 4. Prérequis
- UE2-Module 7 : modèle OSI/TCP-IP, encapsulation, ports (couche transport)
- UE2-Module 9 : adressage IP (les segments TCP voyagent dans des paquets IP)
- UE1-Module 2 : états d'un processus (utile pour l'analogie avec les états TCP)

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Décrire précisément le three-way handshake (SYN, SYN-ACK, ACK) et son objectif
- Expliquer le rôle des numéros de séquence et d'acquittement dans la fiabilité de TCP
- Expliquer le mécanisme de fenêtre glissante (sliding window) et son rôle dans le contrôle de flux
- Différencier contrôle de flux (protection du récepteur) et contrôle de congestion (protection du réseau)
- Décrire la terminaison d'une connexion TCP et expliquer l'état TIME_WAIT
- Lire une capture Wireshark d'un échange TCP complet et identifier chaque phase
- Diagnostiquer un problème de connexion TCP (latence, perte, connexions bloquées) avec les outils Linux

## 6. Motivation

TCP est le protocole sur lequel repose la quasi-totalité du trafic applicatif fiable d'Internet — HTTP, SSH, la plupart des bases de données. Comprendre son fonctionnement interne n'est pas optionnel pour un ingénieur système/réseau : c'est ce qui permet de répondre à des questions très concrètes en production. Pourquoi une connexion met-elle plusieurs secondes à s'établir alors que le réseau semble rapide ? Pourquoi un serveur très sollicité accumule-t-il des milliers de connexions en état `TIME_WAIT` ? Pourquoi le débit d'un transfert de fichier chute-t-il brutalement dès qu'une perte de paquet survient, même minime ?

Ce module descend au niveau des mécanismes internes de TCP — déjà entrevus en survol au module 2 de l'UE2 — pour donner à l'ingénieur la capacité de lire une capture réseau comme un vrai diagnostic, pas comme une boîte noire.

## 7. Concepts fondamentaux

### 7.1 Pourquoi TCP ? Rappel du problème résolu

IP (couche 3) offre un service de livraison **best-effort** : aucune garantie qu'un paquet arrive, arrive une seule fois, ou arrive dans l'ordre d'envoi. TCP (couche 4) construit, par-dessus IP, un service de **flux fiable, ordonné et orienté connexion** : les données arrivent complètes, dans l'ordre, sans duplication — au prix d'un surcoût de gestion (en-têtes, accusés de réception, retransmissions) qu'UDP (vu au module suivant) n'impose pas.

### 7.2 Établissement de connexion : le three-way handshake

Avant tout échange de données, TCP établit une connexion via un échange en trois étapes :

```
Client                          Serveur
  │                                 │
  │────────── SYN (seq=x) ────────▶│   1. Le client propose un numéro de séquence initial x
  │                                 │
  │◀──── SYN-ACK (seq=y, ack=x+1) ──│   2. Le serveur accepte, propose son propre seq y,
  │                                 │      acquitte le SYN du client (x+1)
  │────────── ACK (ack=y+1) ──────▶│   3. Le client acquitte le SYN du serveur (y+1)
  │                                 │
  │      Connexion établie            │
```

**Pourquoi trois étapes et pas deux ?** Chaque extrémité doit à la fois **proposer** son propre numéro de séquence initial et **confirmer** avoir reçu celui de l'autre. Un simple échange à deux temps (SYN puis ACK) ne permettrait pas au client de savoir si son propre SYN a bien été reçu — la troisième étape (ACK final) confirme cette réception au serveur, complétant la synchronisation bidirectionnelle nécessaire aux deux sens de communication indépendants de la connexion.

**Sécurité — SYN flood (aperçu, approfondi en UE8) :** un attaquant peut envoyer un grand nombre de SYN sans jamais compléter la troisième étape, forçant le serveur à maintenir de nombreuses connexions "à moitié ouvertes" (half-open), épuisant ses ressources — une des attaques par déni de service les plus classiques, directement compréhensible à partir de ce mécanisme.

### 7.3 Numéros de séquence et d'acquittement

Chaque octet transmis dans un flux TCP est numéroté (numéro de séquence). Le récepteur accuse réception en indiquant, dans le champ **ACK**, le numéro de séquence du **prochain octet attendu** — pas le dernier reçu. Ce mécanisme permet :
- de détecter une perte (l'émetteur ne reçoit pas d'accusé de réception dans un délai donné → retransmission)
- de détecter et éliminer les duplications (un segment déjà reçu, identifié par son numéro de séquence, est ignoré silencieusement au niveau applicatif)
- de réordonner les segments arrivés dans le désordre avant de les livrer à l'application dans le bon ordre

### 7.4 Fenêtre glissante (sliding window) et contrôle de flux

TCP n'attend pas un accusé de réception après chaque segment individuel (ce qui serait extrêmement lent) — il peut envoyer plusieurs segments "en vol" avant de recevoir leur accusé de réception, dans la limite d'une **fenêtre** de taille donnée.

La **fenêtre de réception (receive window, rwnd)**, annoncée par le récepteur à chaque accusé de réception, indique combien d'octets supplémentaires il peut encore accepter dans son tampon de réception à cet instant — c'est le mécanisme du **contrôle de flux** : protéger le **récepteur** contre un émetteur trop rapide qui saturerait sa mémoire tampon. Une fenêtre annoncée à 0 (zero window) indique au récepteur qu'aucune donnée supplémentaire ne doit être envoyée tant que de l'espace ne s'est pas libéré (l'application réceptrice doit d'abord consommer les données déjà reçues).

**La fenêtre "glisse"** au fur et à mesure que des accusés de réception arrivent : les octets confirmés sortent de la fenêtre côté émetteur, et de nouveaux octets peuvent alors être envoyés — d'où le nom.

### 7.5 Contrôle de congestion

Le contrôle de flux (7.4) protège le récepteur ; le **contrôle de congestion** protège le **réseau lui-même** contre une saturation collective (trop d'émetteurs envoyant trop de données simultanément sur un chemin partagé). TCP maintient, en plus de la fenêtre de réception, une **fenêtre de congestion (cwnd)**, gérée dynamiquement par l'émetteur lui-même en fonction des signaux de congestion observés (pertes de segments, principalement). La taille effective d'envoi à tout instant est le **minimum** entre `rwnd` et `cwnd`.

**Principe général (algorithmes classiques, ex : TCP Reno/Cubic) :**
- **Démarrage lent (slow start)** : la fenêtre de congestion démarre petite et **double** à chaque cycle d'aller-retour réussi (croissance exponentielle), jusqu'à atteindre un seuil ou détecter une perte
- **Évitement de congestion (congestion avoidance)** : passé ce seuil, la croissance devient linéaire (plus prudente), pour approcher progressivement la capacité réelle du chemin sans la dépasser brutalement
- **Réaction à la perte** : une perte de segment est interprétée comme un signal de congestion — la fenêtre de congestion est réduite significativement (division par deux dans les implémentations classiques, ou retour à une fenêtre minimale selon le mécanisme de détection : timeout vs retransmissions rapides/duplicate ACKs)

**Pourquoi ce comportement explique une chute de débit disproportionnée après une perte minime ?** Une seule perte de segment peut déclencher une réduction significative et immédiate de la fenêtre de congestion, alors que sa reconstruction (croissance lente puis linéaire) prend un temps proportionnellement bien plus long — d'où l'impression, très concrète en production sur des liens avec un taux de perte même faible, qu'"une petite perte coûte cher" en débit réel observé.

### 7.6 Terminaison de connexion

TCP ferme une connexion via un échange généralement en quatre étapes (FIN de chaque côté, indépendamment, car TCP est **full-duplex** : chaque sens de communication doit être fermé séparément) :

```
Client                          Serveur
  │────────── FIN ─────────────────▶│
  │◀───────── ACK ───────────────────│
  │◀───────── FIN ────────────────────│
  │────────── ACK ─────────────────▶│
```

### 7.7 L'état TIME_WAIT

Après avoir envoyé le dernier ACK de fermeture, l'extrémité qui a initié la fermeture entre dans un état **TIME_WAIT**, où elle conserve les informations de la connexion pendant une durée définie (souvent 2 × MSL — Maximum Segment Lifetime, soit typiquement plusieurs dizaines de secondes à quelques minutes selon la configuration système) avant de libérer définitivement les ressources.

**Pourquoi cet état existe-t-il ?** Deux raisons principales : (1) garantir que le dernier ACK envoyé a bien été reçu — s'il se perdait, l'autre extrémité retransmettrait son FIN, et l'extrémité en TIME_WAIT doit pouvoir répondre correctement à cette retransmission ; (2) empêcher que des segments retardés d'une ancienne connexion (encore en transit sur le réseau) ne soient accidentellement interprétés comme appartenant à une **nouvelle** connexion qui réutiliserait par coïncidence le même couple (IP, port).

**Implication pratique en production :** un serveur qui initie de très nombreuses fermetures de connexions sortantes (ex : un proxy ou un load balancer) peut accumuler un grand nombre de connexions en `TIME_WAIT`, potentiellement jusqu'à épuiser la plage de ports disponibles pour de nouvelles connexions — un problème de dimensionnement classique, directement lié à ce mécanisme.

### 7.8 Les états TCP — vue d'ensemble

| État | Signification |
|---|---|
| `LISTEN` | Le serveur attend des connexions entrantes |
| `SYN_SENT` | Le client a envoyé un SYN, attend le SYN-ACK |
| `SYN_RECEIVED` | Le serveur a reçu un SYN, a répondu, attend l'ACK final |
| `ESTABLISHED` | Connexion active, échange de données possible |
| `FIN_WAIT_1` / `FIN_WAIT_2` | Fermeture initiée localement, en cours |
| `CLOSE_WAIT` | L'autre extrémité a initié la fermeture, en attente de fermeture locale |
| `TIME_WAIT` | Fermeture terminée localement, attente de sécurité avant libération complète |
| `CLOSED` | Aucune connexion (état final ou initial) |

**Lien direct avec UE1-M2 :** cette machine à états, bien que spécifique à TCP, suit la même logique générale que le cycle de vie d'un processus vu en UE1 — un ensemble d'états discrets et de transitions précisément définies, où comprendre l'état courant oriente immédiatement le diagnostic.

## 8. Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Émetteur                                                    │
│  ┌──────────────────────────────────────────────┐          │
│  │ Fenêtre d'envoi = min(cwnd, rwnd)                 │          │
│  │  [octets confirmés][octets en vol][octets à venir] │          │
│  └──────────────────────────────────────────────┘          │
│         │ segments avec numéro de séquence                    │
└─────────┼─────────────────────────────────────────────────┘
          ▼
┌─────────────────────────────────────────────────────────┐
│  Récepteur                                                    │
│  ┌──────────────────────────────────────────────┐          │
│  │ Tampon de réception, annonce rwnd à chaque ACK    │          │
│  └──────────────────────────────────────────────┘          │
│         │ ACK (numéro du prochain octet attendu)               │
└─────────┴─────────────────────────────────────────────────┘
```

## 9. Fonctionnement interne (cas Linux)

Le noyau Linux implémente TCP directement dans sa pile réseau ; chaque connexion est représentée par une structure interne maintenant l'état (machine à états §7.8), les numéros de séquence, les tampons d'envoi/réception, et les paramètres de congestion. Linux propose plusieurs **algorithmes de contrôle de congestion** interchangeables (`cubic` par défaut sur la plupart des distributions modernes, `reno` plus ancien, `bbr` plus récent avec une approche différente basée sur l'estimation de bande passante plutôt que uniquement sur la perte) — consultable et modifiable via `sysctl net.ipv4.tcp_congestion_control`. Ce choix d'algorithme peut avoir un impact significatif sur la performance observée selon le type de réseau (latence élevée, pertes fréquentes, etc.), un sujet d'optimisation avancé en administration système/réseau.

## 10. Démonstration

```bash
# Observer un handshake complet en capture
sudo tcpdump -i any -nn 'tcp[tcpflags] & (tcp-syn|tcp-fin) != 0' -c 10 &
curl -s http://example.com > /dev/null
wait

# Observer les états TCP actifs sur le système
ss -tan

# Observer l'algorithme de congestion actif
sysctl net.ipv4.tcp_congestion_control
```

## 11. Commandes / outils

| Commande | Rôle | Exemple |
|---|---|---|
| `ss -tan` | Lister les connexions TCP et leur état | `ss -tan state established` |
| `ss -tan state time-wait` | Filtrer uniquement les connexions en TIME_WAIT | `ss -tan state time-wait \| wc -l` |
| `tcpdump -nn tcp` | Capturer et observer le trafic TCP | `sudo tcpdump -i eth0 -nn tcp port 443` |
| `sysctl net.ipv4.tcp_*` | Consulter/ajuster les paramètres TCP du noyau | `sysctl net.ipv4.tcp_fin_timeout` |
| Wireshark, filtre `tcp.flags.syn==1` | Isoler visuellement les paquets d'établissement de connexion | interface graphique |

**Erreur fréquente :** confondre un grand nombre de connexions en `TIME_WAIT` (généralement inoffensif en soi, comportement normal et attendu à fort trafic sortant) avec un signe de fuite de ressources ou de panne — c'est un état transitoire attendu, à ne considérer comme problématique que s'il conduit réellement à un épuisement de ports disponibles, vérifiable et quantifiable.

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** un service backend fait de nombreux appels sortants vers une API tierce, ouvrant et fermant une connexion TCP à chaque appel plutôt que de réutiliser une connexion persistante. Après une montée en charge, le service commence à échouer à établir de nouvelles connexions sortantes, avec des erreurs de type "impossible d'allouer un port local disponible".

**Analyse attendue :** ce symptôme est la signature classique d'un **épuisement de ports locaux** dû à une accumulation de connexions en `TIME_WAIT` — chaque connexion fermée par le service (qui en est l'initiateur) reste dans cet état pendant sa durée définie (2×MSL), immobilisant temporairement le couple (IP locale, port local) utilisé. À fort volume et avec des connexions courtes et non réutilisées, le nombre de connexions en `TIME_WAIT` peut dépasser la plage de ports éphémères disponibles avant qu'elles ne soient libérées. Diagnostic : `ss -tan state time-wait | wc -l` pour quantifier ; résolution de fond : réutiliser des connexions persistantes (connection pooling / keep-alive) plutôt que d'en ouvrir une nouvelle par appel — la vraie correction porte sur la conception applicative, pas sur un simple ajustement de paramètre noyau (bien que des paramètres comme `tcp_tw_reuse` existent et puissent être discutés comme mesure d'appoint, avec leurs propres compromis).

## 15. Sécurité

- **SYN flood** (rappel 7.2) : exploitation directe du mécanisme de handshake, en laissant volontairement de nombreuses connexions à moitié ouvertes — contre-mesures classiques : SYN cookies (le serveur ne réserve pas de ressource tant que l'ACK final n'est pas reçu, en encodant l'état nécessaire directement dans le numéro de séquence retourné)
- **Prédiction de numéro de séquence** : historiquement, des numéros de séquence initiaux prévisibles permettaient à un attaquant d'usurper une connexion TCP sans même observer le trafic légitime — les implémentations modernes utilisent des numéros de séquence initiaux générés de façon suffisamment aléatoire pour rendre cette attaque impraticable
- **RST injection** : un attaquant capable d'observer ou de deviner les paramètres d'une connexion (adresses, ports, numéro de séquence dans la bonne fenêtre) peut injecter un paquet RST pour couper la connexion de tiers — motivation supplémentaire pour le chiffrement (TLS) qui, bien qu'il n'empêche pas structurellement ce type d'attaque au niveau TCP, protège la confidentialité et l'intégrité des données elles-mêmes
- Une connaissance précise des états TCP (§7.8) est indispensable pour configurer correctement un pare-feu à états (stateful firewall), qui suit l'état de chaque connexion plutôt que de filtrer chaque paquet indépendamment (sujet approfondi en UE6/UE8)

## 16. Automatisation

Script Bash de surveillance du nombre de connexions par état, avec alerte sur un seuil anormal de `TIME_WAIT` ou de connexions à moitié ouvertes.

```bash
#!/usr/bin/env bash
# Surveille la répartition des connexions TCP par état et alerte sur des seuils anormaux
set -euo pipefail

SEUIL_TIME_WAIT=5000
SEUIL_SYN_RECV=100
LOGFILE="/var/log/tcp_states_watch.log"

NB_TIME_WAIT=$(ss -tan state time-wait | tail -n +2 | wc -l)
NB_SYN_RECV=$(ss -tan state syn-recv | tail -n +2 | wc -l)

echo "$(date '+%F %T') - TIME_WAIT=$NB_TIME_WAIT SYN_RECV=$NB_SYN_RECV" >> "$LOGFILE"

if (( NB_TIME_WAIT > SEUIL_TIME_WAIT )); then
    echo "$(date '+%F %T') - ALERTE: TIME_WAIT élevé ($NB_TIME_WAIT), risque d'épuisement de ports" \
        | tee -a "$LOGFILE"
fi

if (( NB_SYN_RECV > SEUIL_SYN_RECV )); then
    echo "$(date '+%F %T') - ALERTE: SYN_RECV élevé ($NB_SYN_RECV), possible SYN flood" \
        | tee -a "$LOGFILE"
fi
```

## 17. Troubleshooting

Scénario court : un client signale des connexions à une API qui se bloquent parfois plusieurs secondes avant de réussir, de façon intermittente, sans schéma horaire clair. Démarche : capturer le trafic (`tcpdump`) lors d'une reproduction du problème → chercher des **retransmissions TCP** dans la capture (segments dupliqués avec le même numéro de séquence, signe de perte détectée) → si des retransmissions sont visibles avant l'établissement effectif de la connexion (retransmission du SYN ou du SYN-ACK lui-même), le problème se situe avant même l'échange applicatif, dans l'établissement de connexion → orienter l'investigation vers la qualité du lien réseau ou la charge du serveur (files d'attente de connexions, `SYN_RECV` élevé) plutôt que vers le code applicatif.

## 18. Synthèse

- TCP construit un service fiable, ordonné et orienté connexion par-dessus IP (best-effort)
- Le three-way handshake (SYN, SYN-ACK, ACK) synchronise les numéros de séquence des deux extrémités avant tout échange de données
- Les numéros de séquence/acquittement permettent détection de perte, élimination de duplication, et réordonnancement
- La fenêtre glissante permet d'envoyer plusieurs segments sans attendre un ACK par segment ; le contrôle de flux (rwnd) protège le récepteur, le contrôle de congestion (cwnd) protège le réseau
- Une perte de segment déclenche une réduction significative de la fenêtre de congestion, dont la reconstruction est lente — expliquant l'impact disproportionné d'une perte même faible sur le débit observé
- La fermeture de connexion et l'état TIME_WAIT protègent contre la perte du dernier ACK et la confusion avec une connexion future réutilisant le même couple (IP, port)
- Un grand nombre de connexions en TIME_WAIT est normal à fort trafic sortant, problématique seulement s'il mène à un épuisement réel de ports

## 19. Questions d'évaluation

**QCM**
1. La troisième étape du three-way handshake sert à :
 a) transmettre les premières données applicatives b) confirmer au serveur la bonne réception de son SYN-ACK c) fermer la connexion d) renégocier la fenêtre de réception

2. Le contrôle de congestion protège :
 a) le récepteur contre un tampon saturé b) le réseau contre une saturation collective c) l'émetteur contre ses propres erreurs d) uniquement les connexions chiffrées

**Questions ouvertes**
- Expliquez pourquoi une perte de segment TCP entraîne une réduction de débit disproportionnée par rapport à la perte elle-même.
- Expliquez les deux raisons pour lesquelles l'état TIME_WAIT existe après la fermeture d'une connexion.

**Exercice technique**
- Donnez la commande permettant de compter le nombre de connexions TCP actuellement en état `TIME_WAIT` sur une machine Linux.

**Problème de diagnostic**
- Un service accumule un très grand nombre de connexions en TIME_WAIT au point de ne plus pouvoir ouvrir de nouvelles connexions sortantes. Proposez la cause la plus probable et une piste de correction de fond (pas seulement un ajustement de paramètre noyau).

## 20. Mini-projet

Développer un script (Bash ou Python) qui :
1. Surveille en continu la répartition des connexions TCP par état (`ss -tan`)
2. Détecte une croissance anormale et continue du nombre de connexions en TIME_WAIT sur une fenêtre glissante de mesures (signature d'un risque d'épuisement de ports à venir, avant que l'incident ne survienne réellement)
3. Détecte un nombre anormalement élevé de connexions en SYN_RECV (signature possible de SYN flood)
4. Journalise chaque détection avec horodatage et détails
5. Génère un rapport quotidien de synthèse, incluant la répartition complète par état

---

# PARTIE II — TD

### Niveau 1 — Compréhension

**Exercice 1.1**
Décrivez les trois étapes du three-way handshake TCP, avec le contenu (flags, numéros) de chaque message.

**Exercice 1.2**
Expliquez la différence entre contrôle de flux et contrôle de congestion : que protège chacun ?

**Exercice 1.3**
Citez les deux raisons pour lesquelles l'état TIME_WAIT existe après la fermeture d'une connexion TCP.

### Niveau 2 — Application

**Exercice 2.1**
Donnez la commande permettant de lister uniquement les connexions TCP actuellement à l'état `ESTABLISHED` sur une machine Linux.

**Exercice 2.2**
Donnez la commande permettant de consulter l'algorithme de contrôle de congestion actuellement actif sur le système.

**Exercice 2.3**
Dans un échange TCP, le récepteur envoie un ACK avec le numéro `5001`. Que signifie précisément ce numéro concernant les données déjà reçues ?

### Niveau 3 — Analyse

**Exercice 3.1**
Un transfert de fichier volumineux sur un lien avec un taux de perte de paquets très faible (0,1%) affiche pourtant un débit très inférieur à la bande passante théorique disponible. Expliquez ce phénomène en vous appuyant sur le comportement de la fenêtre de congestion décrit en cours.

**Exercice 3.2**
Un serveur web accumule plusieurs milliers de connexions en état `TIME_WAIT`. Un administrateur junior panique et redémarre le service en urgence. Était-ce nécessaire ? Justifiez en distinguant un état `TIME_WAIT` normal d'un signe réel de problème.

### Niveau 4 — Problème complexe

**Exercice 4.1**
Un serveur d'API sert un trafic normal en heures creuses, mais pendant les pics de charge, les clients rapportent des délais d'établissement de connexion très élevés (plusieurs secondes), alors que les connexions déjà établies fonctionnent normalement une fois ouvertes. En vous appuyant sur les états TCP et le mécanisme du three-way handshake vus en cours, formulez une hypothèse cohérente expliquant pourquoi seul l'**établissement** de nouvelles connexions est affecté (et non les connexions déjà actives), et proposez une commande permettant de confirmer cette hypothèse en observant un état TCP spécifique.

---

## Correction du TD

**1.1** — 1. Le client envoie un segment avec le flag SYN activé et un numéro de séquence initial x. 2. Le serveur répond avec les flags SYN et ACK activés simultanément, propose son propre numéro de séquence initial y, et acquitte le SYN du client (champ ACK = x+1). 3. Le client envoie un segment avec le flag ACK activé, acquittant le SYN du serveur (champ ACK = y+1) — la connexion est alors établie et l'échange de données peut commencer.

**1.2** — Le contrôle de flux protège le **récepteur** : il empêche l'émetteur d'envoyer plus de données que le récepteur ne peut en stocker dans son tampon de réception à un instant donné (géré via la fenêtre de réception, rwnd, annoncée par le récepteur). Le contrôle de congestion protège le **réseau** lui-même : il empêche l'émetteur d'injecter plus de trafic que ce que le chemin réseau (liens intermédiaires, routeurs) peut réellement absorber sans saturation collective, géré via la fenêtre de congestion (cwnd), ajustée dynamiquement par l'émetteur selon les signaux de perte observés.

**1.3** — (1) Garantir que le dernier ACK envoyé par l'initiateur de la fermeture a bien été reçu par l'autre extrémité — s'il se perdait, cette dernière retransmettrait son FIN, et l'extrémité en TIME_WAIT doit rester disponible pour répondre correctement à cette retransmission. (2) Empêcher que des segments retardés appartenant à l'ancienne connexion, encore en circulation sur le réseau, ne soient accidentellement acceptés comme appartenant à une nouvelle connexion qui réutiliserait par coïncidence le même couple (adresse IP, port).

**2.1** — `ss -tan state established`.

**2.2** — `sysctl net.ipv4.tcp_congestion_control`.

**2.3** — Ce numéro signifie que le récepteur a bien reçu, de façon continue et sans trou, tous les octets **jusqu'au numéro 5000 inclus**, et qu'il attend désormais l'octet numéro **5001** comme prochaine donnée à recevoir — le champ ACK indique toujours le prochain octet attendu, pas le dernier reçu.

**3.1** — Même un taux de perte apparemment très faible (0,1%) déclenche, à chaque perte détectée, une réduction significative de la fenêtre de congestion (division par deux ou réduction plus sévère selon le mécanisme de détection, vu en cours section 7.5). Sur un lien à forte latence ou à fort débit théorique, la phase de reconstruction de cette fenêtre (croissance lente en slow start, puis linéaire en congestion avoidance) prend un temps significatif comparé à la durée entre deux pertes successives : le système passe donc une part disproportionnée de son temps en régime de fenêtre réduite plutôt qu'à pleine capacité, expliquant un débit moyen observé très inférieur à la bande passante théorique, même pour un taux de perte objectivement faible.

**3.2** — Non, ce n'était probablement pas nécessaire, et l'administrateur a réagi de façon disproportionnée sans diagnostic préalable. Un grand nombre de connexions en `TIME_WAIT` est un **comportement normal et attendu** sur un serveur à fort trafic sortant ou traitant de nombreuses connexions courtes — c'est un état transitoire de sécurité (vu en cours), pas une fuite de ressources en soi. La distinction pertinente n'est pas "y a-t-il des connexions en TIME_WAIT" (il y en aura presque toujours sur un serveur actif) mais "ce nombre mène-t-il réellement à un épuisement mesurable de ports disponibles, empêchant l'établissement de nouvelles connexions" — un diagnostic quantifié (comparaison du nombre de connexions TIME_WAIT à la plage de ports éphémères disponibles) aurait dû précéder toute action radicale comme un redémarrage de service.

**4.1** — Hypothèse cohérente : lors des pics de charge, le serveur ne parvient plus à traiter les nouvelles demandes de connexion assez rapidement — la **file d'attente des connexions en cours d'établissement** (backlog de connexions en état `SYN_RECV`, en attente de l'ACK final du three-way handshake, ou même la file des connexions déjà complètement établies mais pas encore "acceptées" par l'application) se remplit, retardant fortement l'établissement de toute nouvelle connexion, sans affecter les connexions déjà établies et activement en cours d'échange de données (celles-ci ne dépendent plus de cette file d'attente une fois la connexion complète). Ce comportement est cohérent avec le fait que seul l'**établissement** soit affecté : la machine à états TCP distingue clairement la phase d'établissement (`SYN_RECV`, backlog) de la phase active (`ESTABLISHED`), et un goulot d'étranglement peut n'affecter que la première. Commande de confirmation : `ss -tan state syn-recv | wc -l` — un nombre anormalement élevé de connexions bloquées à cet état précis, corrélé temporellement avec les pics de charge rapportés, confirmerait l'hypothèse d'une saturation de la file d'attente d'établissement plutôt qu'un problème de traitement applicatif des connexions déjà actives.

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur réseau/système pour analyser le comportement TCP d'un service, diagnostiquer une accumulation suspecte de connexions, et développer un outil de surveillance des états TCP.

## Architecture

Une machine Linux avec accès réseau sortant, `tcpdump`, `ss`, et idéalement `curl`/`python3` pour générer du trafic de test.

## Objectifs
- Observer un three-way handshake complet en capture
- Observer et interpréter la répartition des connexions par état
- Provoquer intentionnellement une accumulation de connexions en TIME_WAIT
- Développer et tester un script de surveillance des états TCP

## Préparation

```bash
which tcpdump ss curl python3 2>/dev/null
```

## Étapes

### Étape 1 — Capturer et lire un handshake complet

```bash
sudo tcpdump -i any -nn 'tcp[tcpflags] & (tcp-syn|tcp-ack|tcp-fin) != 0' -c 12 &
sleep 1
curl -s -o /dev/null http://example.com
wait
```
*Question :* identifiez, dans la sortie, les trois segments du three-way handshake (indices SYN, SYN-ACK, ACK) et les segments de fermeture (FIN). Combien de FIN observez-vous au minimum, et pourquoi (rappel section 7.6) ?

### Étape 2 — Observer la répartition des connexions par état

```bash
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn
```
*Résultat attendu :* une répartition par état (ESTAB, TIME-WAIT, LISTEN...) avec leur nombre respectif.

### Étape 3 — Provoquer une accumulation de TIME_WAIT

```bash
cat << 'EOF' > /tmp/many_connections.py
import socket
import time

for i in range(50):
    try:
        s = socket.create_connection(("example.com", 80), timeout=2)
        s.send(b"GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n")
        s.recv(100)
        s.close()   # fermeture immédiate, pas de réutilisation -> favorise TIME_WAIT
    except Exception as e:
        print(f"Connexion {i}: erreur {e}")
    time.sleep(0.1)

print("Terminé")
EOF
python3 /tmp/many_connections.py

ss -tan state time-wait | wc -l
```
*Question :* le nombre de connexions en TIME_WAIT observé correspond-il approximativement au nombre de connexions ouvertes par le script ? Justifiez tout écart éventuel (délai de MSL non encore écoulé, connexions encore en cours...).

### Étape 4 — Script de surveillance des états TCP

Reprendre et tester le script de la section 16 du cours (`tcp_states_watch.sh`), en abaissant temporairement le seuil `SEUIL_TIME_WAIT` à une valeur basse (ex : 5) pour observer le déclenchement de l'alerte avec les connexions générées à l'étape 3.

```bash
bash tcp_states_watch.sh
cat /var/log/tcp_states_watch.log
```

## Commandes de référence
`tcpdump`, `ss -tan`, `sysctl net.ipv4.tcp_congestion_control`, `python3`

## Résultats attendus
- Capture du handshake complet (étape 1) avec annotation manuelle des 3+ segments identifiés
- Répartition des connexions par état (étape 2)
- Nombre de connexions TIME_WAIT observé après le script de génération (étape 3)
- Log d'alerte du script de surveillance (étape 4)

## Questions

1. Dans l'étape 3, si le script avait réutilisé une seule connexion persistante (`Connection: keep-alive` et plusieurs requêtes sur le même socket) au lieu d'en ouvrir 50 séparément, quel impact cela aurait-il eu sur le nombre de connexions en TIME_WAIT observées, et pourquoi cette pratique est-elle recommandée en conception d'API à fort volume ?
2. Pourquoi la capture de l'étape 1 filtre-t-elle spécifiquement sur les flags SYN, ACK et FIN plutôt que de capturer tout le trafic TCP indistinctement ? Quel est l'avantage pratique de ce filtrage ciblé pour ce cas d'usage précis ?
3. Si, à l'étape 4, vous observiez également un nombre élevé de connexions en `SYN_RECV` en même temps que les TIME_WAIT, ce second symptôme aurait-il la même cause probable que l'accumulation de TIME_WAIT, ou faudrait-il l'investiguer séparément ? Justifiez en vous appuyant sur le TD (exercice 4.1).

## Troubleshooting (panne à diagnostiquer)

**Situation fournie :** le script `tcp_states_watch.sh`, exécuté en continu sur un serveur de production réel, déclenche une alerte TIME_WAIT presque permanente, même en dehors de toute période de charge inhabituelle, sur un serveur dont l'équipe affirme qu'il n'a jamais eu de problème de connectivité rapporté par les utilisateurs.

**Démarche attendue :**
1. Symptôme : alerte quasi permanente sans incident utilisateur corrélé
2. Hypothèse : le **seuil** configuré dans le script (`SEUIL_TIME_WAIT`) a été fixé à une valeur arbitraire, sans être calibré par rapport à la charge normale réelle de ce serveur spécifique — ce serveur traite peut-être légitimement un volume de connexions courtes suffisamment élevé pour dépasser en permanence un seuil générique, sans que cela ne constitue un problème réel (rappel TD 3.2 : TIME_WAIT élevé n'est pas intrinsèquement un problème)
3. Vérification : comparer le nombre de connexions en TIME_WAIT observé à la taille réelle de la plage de ports éphémères disponibles sur ce système (`sysctl net.ipv4.ip_local_port_range`), pour établir un seuil réellement pertinent basé sur un risque quantifié d'épuisement, plutôt qu'une valeur arbitraire
4. Isolation : confirmer, sur plusieurs jours d'historique du log, qu'aucun échec réel de connexion ("impossible d'allouer un port") n'a jamais été observé malgré des dépassements répétés du seuil actuel
5. Correction : recalibrer le seuil du script en fonction de la plage de ports réellement disponible sur ce serveur (par exemple, alerter seulement au-delà de 80% de cette plage), plutôt que d'utiliser une valeur fixe générique identique pour tous les serveurs de l'infrastructure
6. Vérification : surveiller sur plusieurs jours après recalibrage, confirmer la disparition des alertes non pertinentes tout en conservant la capacité de détection d'un vrai risque
7. Documentation : noter ce cas comme rappel qu'un seuil d'alerte doit toujours être calibré par rapport à la ressource réellement limitante (ici, la plage de ports disponibles) et au comportement normal du système concerné, jamais fixé arbitrairement — leçon transversale applicable à tous les scripts de surveillance développés au fil de la formation

## Validation

Le TP est validé si l'étudiant fournit :
- Les 4 résultats/captures demandés
- Des réponses argumentées aux 3 questions
- Une identification correcte de la cause du scénario de troubleshooting (seuil mal calibré, pas un vrai problème réseau) et une méthode de recalibrage cohérente

# UE2 — Module 11 : UDP et ICMP

---

# PARTIE I — COURS

## 1. Titre du cours
UDP et ICMP : légèreté transactionnelle et signalisation de contrôle

## 2. Niveau
Élève ingénieur — cycle ingénieur, UE2 réseaux (S1)

## 3. Durée
CM : 3h30 (module plus court que M10 — +3h TD, +3h TP en Parties II et III)

## 4. Prérequis
- UE2-Module 10 : TCP en profondeur (ce module s'appuie constamment sur la comparaison)
- UE2-Module 9 : adressage IP

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Décrire la structure minimale de l'en-tête UDP et expliquer pourquoi elle est si réduite comparée à TCP
- Justifier le choix d'UDP plutôt que TCP pour une application donnée
- Décrire le rôle d'ICMP comme protocole de contrôle et de signalisation d'erreur, distinct des protocoles de transport applicatifs
- Citer et expliquer les types ICMP les plus courants (echo request/reply, destination unreachable, time exceeded)
- Expliquer comment `ping` et `traceroute` exploitent ICMP
- Diagnostiquer un problème réseau en s'appuyant sur les messages ICMP reçus (ou leur absence)

## 6. Motivation

Une application de streaming vidéo en direct, un jeu en ligne temps réel, ou une résolution DNS ont un point commun : ils préfèrent perdre occasionnellement un petit morceau de données plutôt que de subir le délai qu'imposerait TCP pour garantir une livraison parfaite et ordonnée (retransmissions, contrôle de congestion vus au module 10). C'est exactement le compromis qu'offre **UDP** : rapidité et simplicité, au prix de l'absence de toute garantie de livraison.

Par ailleurs, comment un routeur signale-t-il qu'un paquet ne peut pas atteindre sa destination ? Comment `ping` mesure-t-il un temps de réponse, et comment `traceroute` révèle-t-il le chemin complet emprunté par un paquet à travers plusieurs routeurs ? Ce sont des questions de **contrôle réseau**, distinctes du transport de données applicatives — c'est le rôle d'**ICMP**, omniprésent dans le diagnostic réseau quotidien de tout ingénieur système/réseau.

## 7. Concepts fondamentaux

### 7.1 UDP — structure et philosophie

L'en-tête UDP est remarquablement simple, comparé à celui de TCP vu au module 10 :

```
┌──────────────┬──────────────┐
│ Port source   │ Port dest.    │  2 + 2 octets
├──────────────┼──────────────┤
│ Longueur       │ Somme contrôle│  2 + 2 octets
└──────────────┴──────────────┘
```

Seulement **8 octets** d'en-tête (contre 20 octets minimum pour TCP), sans numéro de séquence, sans accusé de réception, sans fenêtre, sans mécanisme de connexion. UDP est un protocole **sans connexion (connectionless)** : aucun établissement préalable (pas de handshake), chaque datagramme est envoyé indépendamment, sans état conservé entre deux envois.

**Ce qu'UDP n'offre PAS, contrairement à TCP :**
- Aucune garantie de livraison (un datagramme perdu n'est jamais retransmis automatiquement)
- Aucune garantie d'ordre (deux datagrammes peuvent arriver dans le désordre)
- Aucun contrôle de flux ni de congestion natif (c'est à l'application, si nécessaire, de gérer ces aspects elle-même)
- Aucune détection de duplication

**Ce qu'UDP offre néanmoins :**
- Une somme de contrôle (checksum) optionnelle en IPv4 (obligatoire en IPv6), permettant de détecter (pas corriger) une corruption — un datagramme corrompu est simplement rejeté, sans retransmission automatique
- Un multiplexage par port, exactement comme TCP (permettant à plusieurs applications d'utiliser UDP simultanément sur une même machine)

### 7.2 Pourquoi choisir UDP plutôt que TCP ?

| Critère | TCP | UDP |
|---|---|---|
| Fiabilité | Garantie (retransmission) | Aucune garantie native |
| Ordre | Garanti | Non garanti |
| Latence d'établissement | Coût du handshake (module 10) | Aucun établissement, envoi immédiat |
| Overhead d'en-tête | 20+ octets | 8 octets |
| Adapté à | Transfert de fichiers, pages web, API, tout ce qui exige l'intégrité complète | Streaming temps réel, VoIP, jeux en ligne, DNS, découverte réseau |

**Principe de décision :** UDP convient quand la donnée perdue ou en retard **perd sa valeur** avant qu'une retransmission ne puisse arriver à temps (un paquet audio VoIP en retard de 500ms est inutile même retransmis avec succès), ou quand l'application préfère gérer elle-même sa propre logique de fiabilité, adaptée précisément à son besoin (ex : QUIC, protocole moderne bâti sur UDP, réimplémente une fiabilité optionnelle et plus flexible que TCP pour le web).

**Cas particulier DNS :** DNS utilise typiquement UDP pour ses requêtes/réponses (petites, gain de rapidité en évitant le handshake), avec bascule automatique vers TCP si la réponse dépasse une certaine taille (transferts de zone, réponses volumineuses) — un exemple concret de choix hybride selon le contexte, préfigurant le module DNS à venir.

### 7.3 ICMP — rôle et positionnement

**ICMP (Internet Control Message Protocol)** n'est ni TCP ni UDP : c'est un protocole de **contrôle**, directement encapsulé dans IP (pas dans un segment TCP/UDP), utilisé par les équipements réseau (routeurs, hôtes) pour signaler des conditions d'erreur ou fournir des informations de diagnostic — jamais pour transporter des données applicatives.

**Point clé pédagogique :** ICMP appartient à la couche 3 (réseau) au même titre qu'IP, contrairement à TCP/UDP qui appartiennent à la couche 4 (transport) — un ICMP n'a donc ni port source ni port destination, une différence fondamentale à ne pas confondre lors d'un diagnostic ou d'une configuration de pare-feu (filtrer ICMP par "port" n'a pas de sens).

### 7.4 Types ICMP courants

| Type | Nom | Usage typique |
|---|---|---|
| 0 | Echo Reply | Réponse à un ping |
| 3 | Destination Unreachable | La destination (ou un composant du chemin) ne peut être atteinte (avec plusieurs codes : réseau inaccessible, hôte inaccessible, port inaccessible...) |
| 5 | Redirect | Un routeur signale une route plus optimale à utiliser |
| 8 | Echo Request | Requête de ping |
| 11 | Time Exceeded | Le TTL (Time To Live) du paquet a atteint zéro avant d'arriver à destination |

**TTL et Time Exceeded — mécanisme fondamental pour `traceroute` :** chaque paquet IP porte un champ TTL, décrémenté de 1 à chaque passage par un routeur ; si le TTL atteint 0, le routeur qui le détecte **détruit le paquet** et renvoie un message ICMP "Time Exceeded" à l'émetteur d'origine. `traceroute` exploite ce mécanisme en envoyant des paquets avec un TTL volontairement croissant (1, puis 2, puis 3...), forçant chaque routeur intermédiaire successif à répondre par un "Time Exceeded" — révélant ainsi, saut par saut, l'intégralité du chemin emprunté jusqu'à la destination finale.

**Destination Unreachable — nuance importante :** ce message peut être émis par un routeur intermédiaire (réseau ou hôte inaccessible, un problème de routage) OU par la machine destination elle-même (port inaccessible — équivalent du "Connection refused" vu au module 9, mais pour UDP, qui n'a pas de mécanisme natif de refus explicite au niveau transport comme TCP).

### 7.5 `ping` et `traceroute` — application directe d'ICMP

- **`ping`** envoie une requête ICMP Echo Request (type 8) et attend une réponse Echo Reply (type 0), mesurant le temps aller-retour (RTT — Round Trip Time). Un `ping` sans réponse ne signifie pas nécessairement une panne — rappel direct du module 7 : ICMP peut être délibérément filtré par un pare-feu, sans que cela n'indique un problème de connectivité réel pour les autres protocoles.
- **`traceroute`** (ou `tracepath` sous Linux, `tracert` sous Windows) exploite le mécanisme TTL/Time Exceeded (7.4) pour révéler chaque saut du chemin réseau — un outil de diagnostic de couche 3 essentiel pour localiser précisément où un problème de routage se situe.

### 7.6 ICMP et sécurité — un équilibre délicat

ICMP est indispensable au bon fonctionnement et au diagnostic d'un réseau IP, mais représente aussi une surface d'attaque historique :
- **Ping flood / Smurf attack** : inondation d'une cible avec des requêtes ICMP Echo Request, potentiellement amplifiée en usurpant l'adresse source (les réponses affluent alors vers la victime usurpée plutôt que vers l'attaquant) — attaque historique par déni de service, aujourd'hui largement mitigée par les bonnes pratiques réseau modernes
- **Reconnaissance réseau** : un attaquant peut utiliser ICMP (et le comportement de `traceroute`) pour cartographier une infrastructure réseau avant une attaque plus ciblée

**D'où la pratique répandue de filtrer ICMP en bordure de réseau** — mais avec un compromis à bien comprendre : bloquer ICMP totalement casse aussi des mécanismes légitimes utiles (comme la découverte de MTU via ICMP "Fragmentation Needed", pouvant provoquer des pertes de paquets silencieuses et difficiles à diagnostiquer si mal géré) — un filtrage ICMP doit être réfléchi et sélectif (type par type), pas appliqué en bloc sans discernement.

## 8. Architecture

```
┌────────────────────────────────────────────────────────┐
│  Couche Application     (DNS, streaming, VoIP...)          │
├────────────────────────────────────────────────────────┤
│  Couche Transport        UDP (8 octets, sans connexion)      │
│                            TCP (20+ octets, avec connexion)    │
├────────────────────────────────────────────────────────┤
│  Couche Réseau            IP                                  │
│                             └── ICMP (encapsulé directement     │
│                                  dans IP, pas dans TCP/UDP)      │
└────────────────────────────────────────────────────────┘
```

## 9. Fonctionnement interne (cas Linux)

Le noyau Linux implémente ICMP directement au niveau de sa pile réseau IP, avec des paramètres ajustables via `sysctl` (ex : limitation du taux de réponses ICMP, `net.ipv4.icmp_ratelimit`, une protection intégrée contre certaines formes d'abus). Le comportement de `ping` en espace utilisateur repose sur un socket brut (raw socket) ou un socket ICMP dédié selon les permissions, contournant le modèle habituel de port applicatif propre à TCP/UDP — cohérent avec le fait qu'ICMP n'a pas de notion de port (vu en 7.3).

## 10. Démonstration

```bash
# Observer un échange UDP simple (requête DNS)
sudo tcpdump -i any -nn udp port 53 -c 2 &
dig example.com > /dev/null
wait

# Observer les messages ICMP échangés lors d'un traceroute
sudo tcpdump -i any -nn icmp -c 10 &
traceroute -n -m 5 8.8.8.8
wait
```

## 11. Commandes / outils

| Commande | Rôle | Exemple |
|---|---|---|
| `ping` | Tester la joignabilité couche 3 via ICMP Echo | `ping -c 4 8.8.8.8` |
| `traceroute` / `tracepath` | Révéler le chemin réseau saut par saut | `traceroute -n 8.8.8.8` |
| `tcpdump -nn icmp` | Capturer spécifiquement le trafic ICMP | `sudo tcpdump -i eth0 -nn icmp` |
| `tcpdump -nn udp` | Capturer spécifiquement le trafic UDP | `sudo tcpdump -i eth0 -nn udp` |
| `ss -uan` | Lister les sockets UDP actifs | `ss -uan` |
| `sysctl net.ipv4.icmp_ratelimit` | Consulter la limitation de débit ICMP du noyau | `sysctl net.ipv4.icmp_ratelimit` |

**Erreur fréquente :** utiliser `ss -tan` (filtre TCP) pour chercher un service UDP et conclure à tort qu'aucun service n'écoute — toujours vérifier avec `ss -uan` (option `u` pour UDP) pour les services qui utilisent ce protocole (DNS, DHCP, SNMP, streaming...).

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** après le déploiement d'un nouveau pare-feu périmétrique bloquant tout trafic ICMP entrant par défaut (mesure de sécurité jugée standard par l'équipe), les utilisateurs distants commencent à rapporter des transferts de fichiers volumineux qui se bloquent silencieusement ou se corrompent sur certaines connexions, alors que la navigation web classique fonctionne normalement.

**Analyse attendue :** ce symptôme, bien que non lié à un `ping` explicitement (les utilisateurs ne testent pas la connectivité par ICMP directement), est en réalité une conséquence indirecte du blocage ICMP total : le mécanisme de **découverte de MTU par chemin (Path MTU Discovery)** repose sur la réception de messages ICMP "Fragmentation Needed" (sous-type du message "Destination Unreachable", type 3 code 4) pour qu'un émetteur TCP réduise dynamiquement la taille de ses segments si un routeur intermédiaire ne peut transmettre un paquet trop volumineux sans fragmentation. Si ce message ICMP est bloqué avant d'atteindre l'émetteur, celui-ci ne reçoit jamais l'information et continue d'envoyer des paquets trop volumineux, silencieusement perdus en cours de route — un phénomène appelé "PMTU blackhole", connu pour affecter spécifiquement les transferts volumineux (dépassant la taille où ce mécanisme intervient) sans affecter le trafic web léger. Correction : autoriser spécifiquement les messages ICMP type 3 (Destination Unreachable, notamment le sous-type Fragmentation Needed) à travers le pare-feu, plutôt qu'un blocage ICMP total et indifférencié — illustration concrète de la mise en garde de la section 7.6 sur la nécessité d'un filtrage ICMP sélectif.

## 15. Sécurité

- Voir section 7.6 pour le compromis fondamental entre sécurité (limiter la reconnaissance réseau, atténuer certaines attaques par déni de service) et fonctionnalité (Path MTU Discovery, diagnostic légitime)
- UDP, sans connexion, est structurellement plus exposé aux attaques par **usurpation d'adresse source** (spoofing) que TCP — l'absence de handshake préalable signifie qu'aucune vérification bidirectionnelle n'est requise avant qu'un serveur ne traite (et potentiellement réponde à) un datagramme, base des attaques par **amplification DNS/NTP** (un attaquant envoie une petite requête UDP avec une adresse source usurpée vers un service ouvert, provoquant une réponse volumineuse dirigée vers la victime usurpée)
- Un service UDP mal sécurisé (sans authentification applicative) est intrinsèquement plus vulnérable à ce type d'abus qu'un service TCP équivalent, en raison de cette absence structurelle de vérification préalable de l'adresse source

## 16. Automatisation

Script Bash de diagnostic ICMP/UDP rapide, distinguant un blocage ICMP d'un vrai problème de couche 3, en croisant avec un test de couche 4.

```bash
#!/usr/bin/env bash
# Diagnostic croisé ICMP vs UDP/TCP pour éviter la conclusion hâtive "ICMP filtré = panne"
set -euo pipefail

CIBLE="${1:?Usage: $0 <cible>}"

echo "=== Test ICMP (ping) ==="
if ping -c 2 -W 2 "$CIBLE" > /dev/null 2>&1; then
    echo "OK: ICMP répond"
    ICMP_OK=1
else
    echo "ÉCHEC: ICMP ne répond pas (peut être filtré, pas nécessairement une panne)"
    ICMP_OK=0
fi

echo "=== Test DNS (UDP port 53, si applicable) ==="
if command -v dig > /dev/null && dig +time=2 +tries=1 @"$CIBLE" . > /dev/null 2>&1; then
    echo "OK: réponse UDP/DNS reçue"
elif timeout 2 bash -c "echo > /dev/udp/$CIBLE/53" 2>/dev/null; then
    echo "OK (partiel): envoi UDP possible, réponse non vérifiée (limite connue du test UDP sans application)"
else
    echo "Pas de test UDP concluant sur cette cible"
fi

if [[ $ICMP_OK -eq 0 ]]; then
    echo "=== CONCLUSION ==="
    echo "ICMP filtré détecté : ne pas conclure à une panne réseau sans un test de couche 4 concluant."
fi
```

*Note pédagogique :* tester UDP de façon fiable est structurellement plus difficile que TCP, précisément parce qu'UDP est sans connexion — l'absence de réponse à un envoi UDP brut ne prouve rien avec certitude (le datagramme a pu être reçu sans qu'aucune réponse ne soit générée, comportement normal pour de nombreux services UDP sans requête applicative valide) ; ce point est volontairement laissé comme limite explicite du script, à discuter en TD.

## 17. Troubleshooting

Scénario court : une application de VoIP interne fonctionne normalement en interne mais les appels depuis l'extérieur du réseau d'entreprise (via VPN) présentent une qualité audio dégradée, avec des coupures fréquentes. Démarche : capturer le trafic UDP (`tcpdump -nn udp port <port_voip>`) durant un appel dégradé → observer si des pertes de datagrammes sont visibles (analyse de numéros de séquence applicatifs si le protocole VoIP en fournit, ex : RTP) → tester la latence et la perte du chemin réseau via VPN avec `ping`/`mtr` (traceroute continu) → si le problème se situe sur le chemin réseau lui-même (pas dans la configuration applicative), orienter vers une investigation de qualité de lien (jitter, perte) plutôt que vers l'application VoIP elle-même, dont le protocole UDP sous-jacent ne peut structurellement pas compenser ces pertes.

## 18. Synthèse

- UDP offre un service minimal (8 octets d'en-tête), sans connexion ni garantie de livraison/ordre — adapté aux cas où la rapidité prime sur la fiabilité garantie
- Le choix TCP vs UDP dépend de la valeur de la donnée en cas de retard (une donnée périmée n'a pas d'intérêt à être retransmise)
- ICMP est un protocole de contrôle de couche 3, distinct de TCP/UDP, sans notion de port
- Le mécanisme TTL/Time Exceeded est directement exploité par `traceroute` pour révéler le chemin réseau saut par saut
- Un `ping` sans réponse ne prouve pas une panne réelle — ICMP peut être filtré délibérément, avec des conséquences parfois indirectes et contre-intuitives (Path MTU Discovery cassé)
- UDP, sans vérification bidirectionnelle préalable, est structurellement plus exposé aux attaques par usurpation et amplification que TCP

## 19. Questions d'évaluation

**QCM**
1. L'en-tête UDP fait, au minimum :
 a) 20 octets b) 8 octets c) 40 octets d) 4 octets

2. Le message ICMP "Time Exceeded" est émis quand :
 a) une connexion TCP expire b) le TTL d'un paquet atteint zéro c) un port est fermé d) une requête DNS échoue

**Questions ouvertes**
- Expliquez pourquoi UDP est préféré à TCP pour une application de streaming vidéo en direct, en identifiant précisément le compromis en jeu.
- Expliquez le mécanisme par lequel `traceroute` parvient à révéler chaque routeur intermédiaire d'un chemin réseau.

**Exercice technique**
- Donnez la commande permettant de lister les sockets UDP actuellement actifs sur une machine Linux.

**Problème de diagnostic**
- Un `ping` échoue vers un serveur, mais une requête DNS (UDP) vers ce même serveur aboutit normalement. Que peut-on en conclure sur l'état réel de la connectivité réseau vers ce serveur ?

## 20. Mini-projet

Développer un script (Bash ou Python) qui, pour une cible donnée :
1. Effectue un test ICMP (ping) et enregistre le résultat sans jamais l'interpréter isolément comme une preuve définitive
2. Effectue un test applicatif UDP pertinent si un service connu est identifiable (ex : requête DNS sur le port 53 si applicable)
3. Effectue, en complément, un `traceroute` limité (nombre de sauts configurable) pour visualiser le chemin réseau et repérer un saut où les réponses ICMP s'arrêtent (filtrage localisé plutôt que généralisé)
4. Produit un rapport clair distinguant "ICMP filtré, service UDP/TCP fonctionnel" de "problème réseau réel probable"
5. Journalise chaque exécution avec horodatage pour un historique de diagnostic

---

# PARTIE II — TD

### Niveau 1 — Compréhension

**Exercice 1.1**
Citez trois différences fondamentales entre UDP et TCP, vues en cours.

**Exercice 1.2**
Expliquez pourquoi ICMP n'a pas de numéro de port, contrairement à TCP et UDP.

**Exercice 1.3**
Citez trois exemples d'applications qui utilisent typiquement UDP plutôt que TCP, en justifiant brièvement chaque choix.

### Niveau 2 — Application

**Exercice 2.1**
Donnez la commande permettant de lister les sockets UDP actuellement en écoute sur une machine Linux.

**Exercice 2.2**
Donnez le nom du message ICMP émis par un routeur lorsque le TTL d'un paquet atteint zéro.

**Exercice 2.3**
Un service DNS écoute sur le port 53. Un étudiant utilise `ss -tan | grep :53` pour vérifier qu'il fonctionne, sans résultat. Le service est-il nécessairement en panne ? Justifiez et corrigez la commande utilisée.

### Niveau 3 — Analyse

**Exercice 3.1**
Un `ping` vers un serveur web échoue systématiquement, mais les utilisateurs rapportent que le site web fonctionne normalement dans leur navigateur. Expliquez ce paradoxe apparent en vous appuyant sur les mécanismes vus en cours (et au module 7).

**Exercice 3.2**
Après le déploiement d'un nouveau pare-feu bloquant tout ICMP entrant, des transferts de fichiers volumineux entre deux sites distants commencent à échouer silencieusement, alors que la navigation web légère continue de fonctionner normalement. Expliquez ce phénomène en identifiant précisément le mécanisme cassé par ce blocage total.

### Niveau 4 — Problème complexe

**Exercice 4.1**
Une entreprise héberge un service DNS public ouvert (répondant à toute requête entrante, sans restriction d'accès). Un jour, le service devient inaccessible, saturé par un volume de trafic sortant anormalement élevé vers des adresses IP extérieures que le service n'a jamais explicitement contactées de sa propre initiative. En vous appuyant sur les caractéristiques d'UDP vues en cours (absence de connexion, absence de vérification préalable de l'adresse source), expliquez le mécanisme d'attaque le plus probable à l'œuvre ici, en précisant pourquoi ce type d'attaque est structurellement plus facile à mener via UDP qu'via TCP.

---

## Correction du TD

**1.1** — (1) UDP est sans connexion (pas de handshake préalable), TCP est orienté connexion. (2) UDP n'offre aucune garantie de livraison ni de retransmission automatique, TCP garantit la livraison via accusés de réception et retransmissions. (3) UDP n'offre aucune garantie d'ordre de livraison, TCP réordonne les segments avant de les livrer à l'application.

**1.2** — ICMP appartient à la couche 3 (réseau), au même titre qu'IP lui-même, et non à la couche 4 (transport) comme TCP et UDP. Le numéro de port est un mécanisme de multiplexage propre à la couche transport, permettant d'identifier une application précise sur une machine ; ICMP, utilisé pour la signalisation de contrôle entre équipements réseau (pas pour transporter des données applicatives destinées à une application spécifique), n'a pas besoin de ce mécanisme de multiplexage par port.

**1.3** — Exemples possibles : streaming vidéo en direct (une donnée en retard perd sa valeur, la retransmission TCP serait inutile) ; VoIP/téléphonie sur IP (même raisonnement, la latence doit rester minimale, une perte ponctuelle est préférable à un délai) ; DNS (petites requêtes/réponses, la rapidité d'un envoi sans handshake préalable est privilégiée, avec bascule vers TCP uniquement si nécessaire pour les réponses volumineuses).

**2.1** — `ss -uan` (l'option `u` cible spécifiquement les sockets UDP, contrairement à `-t` qui cible TCP).

**2.2** — "Time Exceeded" (ICMP type 11).

**2.3** — Non, le service n'est pas nécessairement en panne — l'erreur vient de la commande de vérification elle-même : `ss -tan` filtre uniquement les sockets **TCP** (option `-t`), alors que DNS utilise typiquement **UDP**. La commande correcte serait `ss -uan | grep :53` (option `-u` pour UDP), ce qui pourrait révéler que le service fonctionne parfaitement, contrairement à la conclusion hâtive tirée d'un test utilisant le mauvais protocole.

**3.1** — Ce paradoxe s'explique par le fait qu'ICMP (utilisé par `ping`) et HTTP/HTTPS (utilisé par la navigation web, reposant sur TCP) sont des protocoles totalement distincts, pouvant être filtrés indépendamment l'un de l'autre par un pare-feu (rappel module 7 et section 7.5/7.6 de ce module) : un pare-feu peut parfaitement bloquer ICMP tout en laissant passer normalement le trafic TCP vers les ports web (80/443). L'échec du `ping` ne prouve donc absolument pas une panne de connectivité générale vers ce serveur — seul le protocole ICMP spécifiquement est concerné, sans rapport avec la disponibilité réelle du service web.

**3.2** — Le mécanisme cassé est la **découverte de MTU par chemin (Path MTU Discovery)**, vue en section 14 (étude de cas) : ce mécanisme repose sur la réception, par l'émetteur TCP, de messages ICMP "Destination Unreachable / Fragmentation Needed" (type 3, code 4) envoyés par un routeur intermédiaire incapable de transmettre un paquet trop volumineux sans le fragmenter. Si tout ICMP entrant est bloqué, l'émetteur ne reçoit jamais cette information et continue d'envoyer des paquets trop grands pour le chemin réel, qui sont silencieusement perdus en cours de route (un "PMTU blackhole") — un phénomène qui n'affecte que les transferts dont la taille de paquet dépasse la limite réelle du chemin (typiquement les gros transferts, pas la navigation web légère dont les paquets restent souvent sous ce seuil).

**4.1** — Le mécanisme le plus probable est une **attaque par amplification DNS** : un attaquant envoie de petites requêtes DNS au service ouvert de l'entreprise, en **usurpant l'adresse IP source** de sa victime réelle (spoofing) plutôt que sa propre adresse. Le service DNS, ne disposant d'aucun mécanisme de vérification de l'adresse source avant de répondre (caractéristique structurelle d'UDP, sans connexion ni handshake préalable, vue en cours), envoie sa réponse (potentiellement bien plus volumineuse que la requête initiale, d'où le terme "amplification") directement vers l'adresse usurpée de la victime, et non vers l'attaquant réel. Ce type d'attaque est structurellement **plus difficile à mener via TCP** car l'établissement d'une connexion TCP exige un three-way handshake complet (module 10) : un attaquant usurpant une adresse source ne recevrait jamais le SYN-ACK nécessaire pour compléter la connexion (celui-ci partant vers la victime usurpée), rendant impossible l'exploitation du service TCP usurpé de la même manière — UDP, sans cette vérification bidirectionnelle préalable, ne présente pas cette protection structurelle.

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur réseau junior pour diagnostiquer des symptômes réseau ambigus, en apprenant à distinguer un vrai problème de couche 3 d'un simple filtrage ICMP, et à tester correctement des services UDP.

## Architecture

Une machine Linux avec accès réseau sortant, `tcpdump`, `ping`, `traceroute`, `dig` (ou équivalent), `ss` disponibles.

## Objectifs
- Observer et capturer un échange UDP (requête DNS) et un échange ICMP (ping, traceroute)
- Comparer le comportement d'un test ICMP et d'un test applicatif sur la même cible
- Provoquer et observer l'utilisation du TTL par traceroute
- Développer et tester un script de diagnostic croisé ICMP/UDP

## Préparation

```bash
which tcpdump ping traceroute dig ss 2>/dev/null
```

## Étapes

### Étape 1 — Capturer une requête DNS (UDP)

```bash
sudo tcpdump -i any -nn udp port 53 -c 4 &
sleep 1
dig example.com > /dev/null
wait
```
*Question :* combien de paquets UDP observez-vous pour cette résolution (requête + réponse) ? Comparez avec le nombre de segments nécessaires pour un simple établissement de connexion TCP (module 10) — que constatez-vous sur la légèreté relative de l'échange ?

### Étape 2 — Observer ICMP via ping et traceroute

```bash
sudo tcpdump -i any -nn icmp -c 8 &
sleep 1
ping -c 2 8.8.8.8
wait

traceroute -n -m 6 8.8.8.8
```
*Question :* dans la sortie du `traceroute`, si un ou plusieurs sauts intermédiaires affichent des astérisques (`* * *`) au lieu d'un temps de réponse, cela signifie-t-il nécessairement que le paquet ne peut pas continuer son chemin au-delà de ce saut ? Justifiez en vous appuyant sur la section 7.6 du cours (filtrage ICMP sélectif).

### Étape 3 — Comparer test ICMP et test applicatif sur la même cible

```bash
CIBLE="8.8.8.8"   # ou une cible connue pour filtrer ICMP dans votre environnement, si disponible

echo "=== Test ping ==="
ping -c 2 -W 2 "$CIBLE" && echo "ICMP: OK" || echo "ICMP: ÉCHEC"

echo "=== Test DNS applicatif ==="
dig +time=2 +tries=1 @"$CIBLE" example.com > /dev/null && echo "DNS/UDP: OK" || echo "DNS/UDP: ÉCHEC"
```
*Question :* si vous trouvez une cible réelle où ICMP échoue mais où le service applicatif (DNS ou autre) répond normalement, que confirme cette observation par rapport au TD (exercice 3.1) ?

### Étape 4 — Script de diagnostic croisé

Reprendre et tester le script de la section 16 du cours (`diag_icmp_udp.sh`) sur deux cibles différentes si possible (une où ICMP répond, une où il pourrait être filtré).

```bash
bash diag_icmp_udp.sh 8.8.8.8
bash diag_icmp_udp.sh 1.1.1.1
```

## Commandes de référence
`tcpdump -nn udp`, `tcpdump -nn icmp`, `ping`, `traceroute -n`, `dig`, `ss -uan`

## Résultats attendus
- Capture de l'échange DNS/UDP (étape 1), avec commentaire sur le nombre de paquets échangés
- Sortie complète du `traceroute` (étape 2), avec identification des sauts éventuellement muets
- Résultat comparatif ping/DNS sur une cible (étape 3)
- Sortie du script de diagnostic croisé sur deux cibles (étape 4)

## Questions

1. Pourquoi la capture de l'étape 1 montre-t-elle généralement beaucoup moins de paquets échangés qu'un équivalent TCP simple (par exemple une requête HTTP basique, qui nécessite au minimum le three-way handshake avant même d'envoyer la première donnée) ? Reliez votre réponse à la structure de l'en-tête UDP.
2. Dans l'étape 2, si vous observiez qu'un saut intermédiaire du traceroute affiche des astérisques mais que tous les sauts suivants (plus proches de la destination finale) répondent normalement, quelle conclusion pourriez-vous en tirer sur l'état réel de la connectivité à travers ce saut muet ?
3. Pourquoi est-il structurellement plus difficile de tester "avec certitude" qu'un service UDP quelconque est bien actif, comparé à un test de port TCP (rappel module 9, test `/dev/tcp/`) ? Reliez votre réponse à l'absence de connexion propre à UDP.

## Troubleshooting (panne à diagnostiquer)

**Situation fournie :** un étudiant exécute le script de diagnostic croisé de l'étape 4 contre un service UDP applicatif interne (pas DNS) qu'il sait pourtant fonctionnel, et le script rapporte un résultat UDP ambigu ("Pas de test UDP concluant"), le laissant sans conclusion claire alors qu'il souhaitait confirmer rapidement la disponibilité du service.

**Démarche attendue :**
1. Symptôme : le script ne parvient pas à conclure clairement sur l'état du service UDP interne, contrairement à son comportement satisfaisant sur un service DNS standard
2. Hypothèse : le script (section 16 du cours) est volontairement limité pour UDP — il ne dispose d'un test applicatif fiable que pour DNS (via `dig`), et se rabat sur un simple envoi UDP brut pour tout autre cas, qui ne prouve rien avec certitude (limite explicitement documentée dans la note pédagogique du cours) : un service UDP interne non-DNS ne peut pas être testé de façon fiable par ce script générique
3. Vérification : consulter la documentation/les commentaires du script pour confirmer cette limite assumée, plutôt que de supposer un bug
4. Isolation : déterminer si un test applicatif spécifique existe pour ce service précis (une requête minimale connue du protocole applicatif concerné, si le service en propose une, à la manière de ce que `dig` fait pour DNS)
5. Correction : soit écrire un test applicatif dédié à ce service précis (approche recommandée, la plus fiable), soit accepter la limite du script générique et se contenter d'un test côté serveur directement (`ss -uan | grep <port>` pour confirmer que le service écoute bien, sans garantir qu'il répond correctement à une requête réelle)
6. Vérification : comparer les résultats du test dédié avec l'état réel connu du service (fonctionnel), confirmer la cohérence
7. Documentation : noter cette limite structurelle des tests UDP génériques comme un point de vigilance méthodologique transversal — contrairement à TCP où l'établissement de connexion lui-même constitue une preuve minimale de disponibilité (module 9), UDP ne peut être validé avec certitude qu'à travers un test réellement applicatif spécifique au protocole concerné

## Validation

Le TP est validé si l'étudiant fournit :
- Les 4 résultats/captures demandés
- Des réponses argumentées aux 3 questions, notamment une explication correcte de la limite structurelle des tests UDP génériques
- Une identification correcte de la cause du scénario de troubleshooting, distinguant limite assumée du script et véritable panne du service

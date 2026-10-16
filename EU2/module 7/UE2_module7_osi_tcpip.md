# UE2 — Module 1 : Modèle OSI et TCP/IP

*Premier module de l'UE2 (Fondamentaux réseaux et TCP/IP) — numéroté Module 7 dans la continuité du programme complet.*

---

# PARTIE I — COURS

## 1. Titre du cours
Modèles en couches : OSI, TCP/IP et le principe de l'encapsulation

## 2. Niveau
Élève ingénieur — cycle ingénieur, entrée en UE2 réseaux (S1)

## 3. Durée
CM : 4h (module complet : +3h TD, +4h TP en Parties II et III)

## 4. Prérequis
- UE1 complète : notions de system calls, sockets en tant que mécanisme d'IPC (M5), "tout est fichier" (M4)
- Aucun prérequis réseau spécifique — ce module est le point d'entrée de l'UE2

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Citer et expliquer le rôle de chacune des 7 couches du modèle OSI
- Expliquer la correspondance (et les différences) entre le modèle OSI et le modèle TCP/IP
- Expliquer le principe d'encapsulation/décapsulation et le vocabulaire associé (PDU par couche)
- Identifier, pour un protocole donné (HTTP, TCP, IP, Ethernet), à quelle couche il appartient et quel problème il résout
- Lire une capture Wireshark simple et identifier les couches représentées dans chaque trame

## 6. Motivation

Quand vous chargez une page web, des dizaines de mécanismes s'enchaînent en quelques millisecondes : résolution de nom, établissement de connexion, chiffrement, découpage en paquets, adressage physique, transmission électrique sur le câble. Aucun développeur web n'a besoin de gérer manuellement le câblage Ethernet ; aucun ingénieur réseau bas niveau n'a besoin de comprendre le protocole HTTP pour router un paquet. Cette indépendance est rendue possible par un principe fondamental de l'ingénierie réseau : la **conception en couches**.

Comprendre ces couches n'est pas un exercice de mémorisation scolaire — c'est ce qui permet à un ingénieur de savoir *où chercher* quand un problème survient : un problème de résolution de nom n'est pas un problème de câblage, un problème de connexion TCP refusée n'est pas un problème applicatif. Toute la méthodologie de troubleshooting réseau enseignée dans les modules suivants repose sur cette capacité à localiser un problème dans la bonne couche.

## 7. Concepts fondamentaux

### 7.1 Pourquoi un modèle en couches ?

Sans découpage en couches, chaque application devrait réimplémenter l'intégralité de la pile de communication : gestion du câblage, adressage physique, routage, fiabilité de transmission, jusqu'au protocole applicatif lui-même. Le modèle en couches répond à trois besoins :

1. **Modularité** : chaque couche résout un problème précis et expose une interface simple aux couches voisines, sans exposer sa complexité interne
2. **Interopérabilité** : deux systèmes très différents (un serveur Linux et un smartphone) peuvent communiquer tant qu'ils respectent les mêmes protocoles à chaque couche, indépendamment de leur implémentation interne
3. **Évolutivité** : une couche peut évoluer (ex : passage d'Ethernet filaire à Wi-Fi) sans modifier les couches supérieures, tant que l'interface entre les couches reste stable

### 7.2 Le modèle OSI (Open Systems Interconnection) — 7 couches

| # | Couche | Rôle | Exemples |
|---|---|---|---|
| 7 | Application | Interface avec l'utilisateur/le programme | HTTP, DNS, SSH, SMTP |
| 6 | Présentation | Format des données, chiffrement, compression | TLS (partiellement), encodage de caractères |
| 5 | Session | Établissement, gestion, fin d'une session de communication | Notion de session applicative |
| 4 | Transport | Livraison de bout en bout, fiabilité, découpage en segments | TCP, UDP |
| 3 | Réseau | Adressage logique, routage entre réseaux | IP (IPv4/IPv6), ICMP |
| 2 | Liaison de données | Adressage physique local, détection d'erreurs sur un lien | Ethernet, ARP, commutation (switching) |
| 1 | Physique | Transmission des bits sur le support physique | Câblage cuivre, fibre optique, ondes radio |

**Moyen mnémotechnique classique (à adapter selon la langue d'enseignement) :** de haut en bas, "All People Seem To Need Data Processing" (Application, Presentation, Session, Transport, Network, Data Link, Physical).

### 7.3 Le modèle TCP/IP — 4 couches (le modèle réellement implémenté)

Le modèle OSI est un modèle **théorique et pédagogique**, rarement implémenté strictement tel quel. Le modèle **TCP/IP**, plus ancien historiquement et directement issu de la conception réelle d'Internet, est celui effectivement utilisé en pratique :

| Couche TCP/IP | Correspondance OSI approximative | Rôle | Exemples |
|---|---|---|---|
| Application | Application + Présentation + Session (7-6-5) | Protocoles applicatifs | HTTP, DNS, SSH |
| Transport | Transport (4) | Livraison de bout en bout | TCP, UDP |
| Internet | Réseau (3) | Adressage logique, routage | IP, ICMP |
| Accès réseau (Liaison + Physique) | Liaison de données + Physique (2-1) | Adressage physique local, transmission | Ethernet, Wi-Fi |

**Point clé pédagogique :** les couches 5, 6 et 7 d'OSI n'ont **pas d'équivalent séparé** dans la pratique de la plupart des protocoles applicatifs modernes — HTTP, par exemple, gère lui-même ce que OSI aurait réparti sur trois couches distinctes. C'est pourquoi le modèle OSI est surtout utilisé aujourd'hui comme **outil pédagogique et vocabulaire commun de diagnostic** ("un problème de couche 2", "un problème de couche 7"), plutôt que comme description fidèle d'une pile réseau réelle.

### 7.4 Encapsulation et décapsulation

Quand une application envoie des données, chaque couche traversée **ajoute son propre en-tête** (et parfois une queue/trailer) autour des données reçues de la couche supérieure, sans jamais modifier leur contenu — c'est l'**encapsulation**.

```
┌─────────────────────────────────────────────────────┐
│ Application : données (ex: requête HTTP)               │
├─────────────────────────────────────────────────────┤
│ En-tête TCP │ Application : données                     │  ← Segment
├─────────────────────────────────────────────────────┤
│ En-tête IP │ En-tête TCP │ Application : données         │  ← Paquet
├─────────────────────────────────────────────────────┤
│ En-tête Ethernet │ ... │ Queue Ethernet (FCS)            │  ← Trame
└─────────────────────────────────────────────────────┘
```

À la réception, le processus inverse — la **décapsulation** — retire successivement chaque en-tête, couche par couche, jusqu'à ne laisser que les données applicatives d'origine, chaque couche du récepteur ne lisant *que* son propre en-tête pour décider du traitement à appliquer (à qui transmettre, quelle erreur signaler...).

### 7.5 Vocabulaire des unités de données (PDU — Protocol Data Unit)

Chaque couche donne un nom spécifique à l'unité de données qu'elle manipule :

| Couche | Nom de la PDU |
|---|---|
| Application | Données (data) |
| Transport | Segment (TCP) ou Datagramme (UDP) |
| Réseau/Internet | Paquet |
| Liaison de données | Trame |
| Physique | Bits |

**Erreur fréquente chez les débutants :** utiliser "paquet" pour désigner indifféremment toute unité de données à toute couche — en contexte professionnel rigoureux, un ingénieur réseau précis dira "trame" pour la couche 2 et "segment" pour un flux TCP à la couche 4, ce qui évite les ambiguïtés en diagnostic (ex : "perte de trames" pointe vers la couche 2, "retransmission de segments" pointe vers la couche 4 — deux causes racines très différentes).

### 7.6 Adressage par couche

Chaque couche a son propre système d'adressage, avec une portée différente :

- **Couche 2 (liaison)** : adresse **MAC**, identifiant physique de la carte réseau, valide uniquement sur le **segment local** (le même réseau physique/logique direct)
- **Couche 3 (réseau)** : adresse **IP**, identifiant logique valide à l'échelle du réseau global (Internet ou un réseau d'entreprise routé), indépendant du matériel physique
- **Couche 4 (transport)** : **numéro de port**, permettant d'identifier une application/un service précis sur une machine donnée (un même hôte IP peut exécuter plusieurs services simultanément, différenciés par leur port)

Cette hiérarchie d'adressage (MAC local → IP globale → port applicatif) est ce qui permet à un paquet de traverser plusieurs réseaux physiques différents (changement d'adresse MAC à chaque saut, via le routage) tout en conservant la même adresse IP source/destination de bout en bout, et d'être finalement délivré à la bonne application via le port.

## 8. Architecture

```
┌──────────────────────────────────────────────────────────┐
│  Machine A (émetteur)              Machine B (récepteur)    │
│  ┌────────────┐                    ┌────────────┐          │
│  │ Application │                    │ Application │  ▲       │
│  ├────────────┤                    ├────────────┤   │       │
│  │ Transport   │                    │ Transport   │  │       │
│  ├────────────┤   Encapsulation    ├────────────┤   │ Décap. │
│  │ Internet    │   ──────────▶     │ Internet    │  │       │
│  ├────────────┤                    ├────────────┤   │       │
│  │ Accès réseau│                    │ Accès réseau│  │       │
│  └──────┬─────┘                    └──────┬─────┘  ▼       │
│         │                                   │              │
│         └──────── Support physique ─────────┘              │
│                (câble, fibre, radio)                          │
└──────────────────────────────────────────────────────────┘
```

## 9. Fonctionnement interne

Sous Linux, cette architecture en couches se retrouve directement dans l'implémentation du noyau : la couche socket (interface `AF_INET`/`SOCK_STREAM` utilisée par les applications, cohérente avec l'IPC vu en UE1-M5) s'appuie sur une pile réseau interne organisée en modules correspondant approximativement aux couches TCP/IP — un module de gestion TCP/UDP, un module IP, un module de gestion des interfaces réseau (correspondant à la couche accès réseau). Chaque module ajoute ou retire son propre en-tête au buffer de données (structure noyau `sk_buff`), sans jamais avoir besoin de connaître le contenu ou la structure interne des couches voisines — une implémentation directe du principe d'encapsulation vu en cours, y compris au niveau du code source du noyau.

## 10. Démonstration

```bash
# Observer les interfaces réseau (couche accès réseau) et leur adresse MAC/IP
ip addr

# Observer la table ARP (correspondance IP couche 3 / MAC couche 2)
ip neigh

# Capturer et observer les couches d'une trame réelle
sudo tcpdump -i any -c 5 -nn -X
```

## 11. Commandes / outils

| Commande | Rôle | Exemple |
|---|---|---|
| `ip addr` | Afficher les adresses IP et MAC des interfaces | `ip addr show` |
| `ip neigh` | Afficher la table de correspondance IP↔MAC (cache ARP) | `ip neigh show` |
| `tcpdump` | Capturer le trafic réseau en ligne de commande | `sudo tcpdump -i eth0 -nn` |
| Wireshark | Capturer et analyser visuellement le trafic, couche par couche | interface graphique |
| `ss -tuln` | Lister les ports en écoute (couche transport) | `ss -tuln` |

**Erreur fréquente :** confondre l'adresse MAC (couche 2, change à chaque saut de routeur) avec l'adresse IP (couche 3, reste identique de bout en bout sauf NAT) — un étudiant qui pense qu'un paquet garde la même adresse MAC source/destination sur tout son trajet à travers plusieurs réseaux fait une erreur fondamentale de compréhension du modèle en couches.

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** un utilisateur signale qu'il ne peut pas accéder à une application web interne, avec le message d'erreur "Connection refused" dans son navigateur. Un collègue junior commence immédiatement à vérifier le câblage réseau (couche 1/2) du poste de l'utilisateur.

**Analyse attendue :** le message "Connection refused" est une réponse explicite reçue **au niveau de la couche transport (TCP)** — cela signifie que les paquets ont déjà traversé avec succès les couches 1, 2 et 3 (le poste a bien atteint le serveur au niveau réseau), mais qu'aucun processus applicatif n'écoute sur le port demandé côté serveur (ou qu'un pare-feu répond explicitement par un refus plutôt qu'un silence). Vérifier le câblage physique est donc une perte de temps ici : le diagnostic doit se concentrer directement sur la couche 4 (le service est-il démarré ? `ss -tuln` côté serveur) puis la couche 7 (configuration applicative), pas sur les couches basses déjà validées implicitement par la nature même du message d'erreur reçu.

## 15. Sécurité

- Chaque couche présente sa propre surface d'attaque : usurpation d'adresse MAC (couche 2, ARP spoofing — approfondi en UE2 module ultérieur), usurpation d'adresse IP (couche 3, IP spoofing), attaques sur les sessions TCP (couche 4), attaques applicatives (couche 7, ex : injection SQL, XSS — hors périmètre réseau strict mais bâties sur cette base)
- Un pare-feu peut filtrer à différentes couches : un pare-feu simple filtre sur IP/port (couches 3-4), un pare-feu applicatif (WAF — Web Application Firewall) inspecte le contenu au niveau couche 7
- Comprendre à quelle couche appartient une menace est le prérequis pour choisir la bonne contre-mesure : un problème de couche 2 (ARP spoofing sur un réseau local) ne se résout pas avec un pare-feu configuré au niveau IP

## 16. Automatisation

Script Bash de vérification rapide multi-couches (diagnostic express avant intervention manuelle).

```bash
#!/usr/bin/env bash
# Diagnostic express multi-couches vers une cible donnée
set -euo pipefail

CIBLE="${1:?Usage: $0 <adresse_ou_nom_cible> <port>}"
PORT="${2:?Usage: $0 <adresse_ou_nom_cible> <port>}"

echo "=== Couche 3 (Réseau) : test de joignabilité IP ==="
if ping -c 2 -W 2 "$CIBLE" > /dev/null 2>&1; then
    echo "OK: $CIBLE répond au ping"
else
    echo "ÉCHEC: $CIBLE ne répond pas au ping (peut être normal si ICMP filtré)"
fi

echo "=== Couche 4 (Transport) : test du port applicatif ==="
if timeout 3 bash -c "echo > /dev/tcp/$CIBLE/$PORT" 2>/dev/null; then
    echo "OK: port $PORT ouvert sur $CIBLE"
else
    echo "ÉCHEC: port $PORT injoignable sur $CIBLE (service arrêté, pare-feu, ou couche réseau en cause)"
fi
```

## 17. Troubleshooting

Scénario court : deux serveurs sur le même réseau local ne parviennent plus à communiquer, alors qu'ils communiquaient normalement la veille, sans changement de configuration IP connu. Démarche méthodique multi-couches : `ping` (couche 3) échoue → vérifier la table ARP (`ip neigh`, couche 2/3) pour une entrée incohérente ou absente → vérifier l'état de l'interface physique (`ip link`, couche 1/2, câble débranché ou interface down) → isoler progressivement, couche par couche, du bas vers le haut, jusqu'à identifier la couche exacte en cause — méthodologie qui préfigure directement les modules réseau avancés à venir.

## 18. Synthèse

- Le modèle en couches (OSI théorique à 7 couches, TCP/IP pratique à 4 couches) permet modularité, interopérabilité et évolutivité
- OSI reste surtout un vocabulaire de diagnostic partagé ("problème couche 2/3/4/7") plutôt qu'une implémentation stricte
- L'encapsulation ajoute un en-tête par couche traversée (Application → Segment → Paquet → Trame → Bits) ; la décapsulation fait l'inverse à la réception
- Chaque couche a son propre adressage : MAC (local, couche 2), IP (global, couche 3), port (applicatif, couche 4)
- Un diagnostic réseau rigoureux commence toujours par identifier *à quelle couche* se situe le symptôme observé, avant d'investiguer plus profondément

## 19. Questions d'évaluation

**QCM**
1. Dans le modèle OSI, la couche responsable de l'adressage physique local (MAC) est :
 a) couche 3 (Réseau) b) couche 2 (Liaison de données) c) couche 4 (Transport) d) couche 7 (Application)

2. Un message "Connection refused" reçu par un client indique généralement que le problème se situe :
 a) au niveau du câblage physique b) au niveau de la couche transport (le port cible n'est pas ouvert) c) toujours dans le DNS d) dans le chiffrement TLS

**Questions ouvertes**
- Expliquez pourquoi le modèle OSI à 7 couches, bien qu'enseigné en détail, ne correspond pas exactement à l'implémentation réelle des piles réseau modernes.
- Un paquet traverse trois routeurs entre l'émetteur et le récepteur. Que devient l'adresse IP source à chaque saut ? Et l'adresse MAC source ? Justifiez la différence.

**Exercice technique**
- Nommez précisément la PDU (unité de données) manipulée à chacune des couches suivantes : Transport (TCP), Réseau, Liaison de données.

**Problème de diagnostic**
- Un `ping` échoue entre deux machines du même réseau local. Proposez une démarche de diagnostic en au moins 3 étapes, en partant de la couche la plus basse.

## 20. Mini-projet

Développer un script (Bash ou Python) qui, à partir d'une cible (IP ou nom d'hôte) et d'un port fournis en argument :
1. Teste la joignabilité couche 3 (`ping`)
2. Teste l'ouverture du port couche 4 (connexion TCP simple)
3. Si les deux réussissent, effectue une requête applicative minimale couche 7 si le port correspond à un protocole connu (ex : requête HTTP basique sur le port 80/443)
4. Produit un rapport clair indiquant à quelle couche, le cas échéant, le diagnostic s'est arrêté en échec
5. Journalise chaque exécution avec horodatage pour constituer un historique de disponibilité multi-couches

---

# PARTIE II — TD

### Niveau 1 — Compréhension

**Exercice 1.1**
Listez les 7 couches du modèle OSI dans l'ordre, de la couche 1 à la couche 7, avec le rôle de chacune en une phrase.

**Exercice 1.2**
Expliquez avec vos propres mots ce que signifie "encapsulation" dans le contexte des réseaux, en citant un exemple concret d'en-tête ajouté à une couche donnée.

**Exercice 1.3**
Pourquoi dit-on que le modèle OSI est surtout un outil pédagogique et de vocabulaire, plutôt qu'une description fidèle des piles réseau réellement implémentées ?

### Niveau 2 — Application

**Exercice 2.1**
Associez chaque protocole à sa couche OSI principale : HTTP, TCP, IP, Ethernet, DNS.

**Exercice 2.2**
Donnez le nom exact de la PDU (unité de données) à la couche Transport pour TCP, et à la couche Liaison de données.

**Exercice 2.3**
Donnez la commande Linux permettant d'observer la table de correspondance entre adresses IP et adresses MAC connues localement.

### Niveau 3 — Analyse

**Exercice 3.1**
Une application cliente reçoit une erreur "Connection timed out" (pas de réponse du tout, contrairement à "Connection refused"). En vous appuyant sur la distinction faite en cours entre ces deux types d'erreurs, à quelle(s) couche(s) ce symptôme oriente-t-il le diagnostic, et pourquoi cette absence totale de réponse est-elle différente, en termes de cause probable, d'un refus explicite ?

**Exercice 3.2**
Un paquet part d'une machine A, traverse deux routeurs (R1 et R2), et arrive à une machine B. Décrivez précisément comment évoluent l'adresse IP source et l'adresse MAC source du paquet à chaque saut (A→R1, R1→R2, R2→B), en justifiant pourquoi l'une change et l'autre non.

### Niveau 4 — Problème complexe

**Exercice 4.1**
Un ingénieur junior diagnostique un incident où une application web est inaccessible. Il constate que `ping` vers le serveur fonctionne, mais que la connexion au port 443 (HTTPS) échoue avec un timeout (aucune réponse), tandis qu'un test sur le port 80 (HTTP) reçoit bien une réponse "Connection refused" explicite. Expliquez ce que cette différence de comportement entre les deux ports révèle sur l'état probable de chaque couche (réseau, transport, voire pare-feu), en construisant un raisonnement qui exclut certaines causes et en oriente d'autres, cohérent avec la méthodologie multi-couches vue en cours.

---

## Correction du TD

**1.1** — 1. Physique : transmission des bits sur le support (câble, fibre, radio). 2. Liaison de données : adressage physique local (MAC), détection d'erreurs sur un lien direct. 3. Réseau : adressage logique (IP), routage entre réseaux différents. 4. Transport : livraison de bout en bout, fiabilité, découpage en segments. 5. Session : établissement/gestion/fin d'une session de communication. 6. Présentation : format des données, chiffrement, compression. 7. Application : interface avec l'utilisateur/le programme (protocoles applicatifs).

**1.2** — L'encapsulation consiste, pour chaque couche traversée lors de l'émission, à ajouter son propre en-tête autour des données reçues de la couche supérieure, sans modifier leur contenu. Exemple concret : quand une application envoie des données via TCP, la couche transport ajoute un en-tête TCP (contenant notamment les ports source/destination et les numéros de séquence) devant les données applicatives, formant un segment ; ce segment est ensuite lui-même encapsulé dans un en-tête IP à la couche réseau.

**1.3** — Parce que la plupart des protocoles applicatifs modernes (HTTP, par exemple) ne séparent pas explicitement les fonctions attribuées aux couches 5 (Session), 6 (Présentation) et 7 (Application) par OSI — un seul protocole applicatif gère souvent, à lui seul, ce que OSI répartirait sur trois couches distinctes. Le modèle réellement implémenté est le modèle TCP/IP à 4 couches ; OSI reste néanmoins utile comme vocabulaire commun et cadre de diagnostic partagé entre ingénieurs ("un problème de couche 2" est compris universellement, même si aucune pile réseau réelle n'implémente OSI à la lettre).

**2.1** — HTTP : couche 7 (Application). TCP : couche 4 (Transport). IP : couche 3 (Réseau). Ethernet : couche 2 (Liaison de données). DNS : couche 7 (Application) — bien qu'il utilise typiquement UDP (couche 4) comme transport sous-jacent, le protocole DNS lui-même est un protocole applicatif.

**2.2** — Couche Transport (TCP) : le **segment**. Couche Liaison de données : la **trame**.

**2.3** — `ip neigh` (ou `ip neigh show`) — affiche le cache ARP, associant adresses IP et adresses MAC pour les machines du réseau local récemment contactées.

**3.1** — "Connection refused" (refus explicite) indique que les paquets ont atteint la machine cible avec succès (couches 1 à 3 fonctionnelles) et qu'une réponse explicite a été générée au niveau transport, indiquant qu'aucun service n'écoute sur le port demandé — le diagnostic doit se concentrer sur la couche 4/7 côté serveur (service arrêté). "Connection timed out" (absence totale de réponse) est différent : cela peut signifier que les paquets n'atteignent jamais la cible (problème de couche 3, routage, ou couche 1/2, câblage/interface), ou qu'un pare-feu **bloque silencieusement** (drop) plutôt que de refuser explicitement (reject) — dans ce second cas, le diagnostic doit remonter vers les couches basses (3 et en-dessous) ou vers une politique de pare-feu spécifique, avant même de supposer un problème purement applicatif.

**3.2** — L'adresse **IP source reste identique** (celle de la machine A) tout au long du trajet A→R1→R2→B, car c'est une adresse de couche 3, valide de bout en bout indépendamment des sauts intermédiaires (le routage se base justement sur cette persistance pour acheminer le paquet vers la bonne destination finale). L'adresse **MAC source change à chaque saut** : sur le segment A→R1, l'adresse MAC source est celle de A ; sur le segment R1→R2, le paquet a été réencapsulé par R1 avec sa propre adresse MAC comme source (destinée à R2) ; de même pour R2→B. C'est parce que l'adresse MAC (couche 2) n'a de sens que sur un segment physique local direct — chaque routeur, en retransmettant le paquet vers le saut suivant, doit reconstruire un nouvel en-tête de couche 2 adapté au segment local suivant, tandis que l'en-tête IP (couche 3) contenant les adresses source/destination globales reste inchangé de bout en bout (hors cas de NAT, hors périmètre de ce module).

**4.1** — Le port 80 renvoyant un "Connection refused" explicite prouve que les couches 1, 2 et 3 fonctionnent parfaitement (le paquet atteint bien le serveur et une réponse en revient), et qu'aucun pare-feu ne bloque silencieusement le trafic vers ce serveur en général — le serveur "existe" bien et répond activement au niveau réseau/transport. Le refus explicite sur le port 80 signifie simplement qu'aucun service HTTP n'écoute sur ce port (ce qui est cohérent si le service est censé être en HTTPS uniquement). En revanche, le port 443 recevant un **timeout** (absence totale de réponse) alors même que la couche réseau est manifestement fonctionnelle (prouvé par le comportement du port 80) oriente fortement vers une cause **spécifique au port 443** plutôt qu'un problème réseau général : très probablement un pare-feu configuré pour **bloquer silencieusement (drop)** spécifiquement le trafic vers le port 443, ou un service HTTPS effectivement démarré mais figé/en attente sans jamais répondre (moins probable si le port 80 fonctionne sur la même machine). Le raisonnement exclut donc une cause de couche 1-2-3 générale, et oriente l'investigation vers la configuration du pare-feu ou l'état spécifique du service HTTPS.

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur réseau junior pour former une nouvelle recrue à la lecture de captures réseau. Votre mission : capturer du trafic réel, identifier précisément les couches et protocoles en jeu, et construire une intuition solide de l'encapsulation avant d'aborder les modules réseau plus avancés.

## Architecture

Une machine Linux (VM ou conteneur avec accès réseau sortant), avec `tcpdump` installé (Wireshark en interface graphique si disponible sur poste étudiant, sinon lecture des fichiers `.pcap` générés par `tcpdump` avec Wireshark en local après export).

## Objectifs
- Capturer du trafic réseau réel et identifier les couches représentées
- Observer concrètement l'encapsulation (en-têtes Ethernet, IP, TCP visibles dans une même trame)
- Différencier un comportement de couche 3/4 (ping, port fermé) d'un comportement applicatif
- Construire un script de diagnostic multi-couches basique

## Préparation

```bash
which tcpdump ping ip ss 2>/dev/null
ip addr show
```

## Étapes

### Étape 1 — Observer les couches basses localement

```bash
ip addr show      # adresses IP et MAC de vos interfaces (couches 2 et 3)
ip neigh show      # cache ARP (correspondance couche 2 / couche 3)
ip route show       # table de routage (décisions de couche 3)
```
*Question :* identifiez, dans la sortie de `ip addr show`, l'adresse qui correspond à la couche 2 et celle qui correspond à la couche 3 pour une même interface.

### Étape 2 — Capturer et décortiquer une trame complète

```bash
sudo tcpdump -i any -c 10 -nn -X icmp &
sleep 1
ping -c 3 8.8.8.8
wait
```
*Résultat attendu :* la sortie `-X` de `tcpdump` affiche le contenu brut de chaque trame en hexadécimal — observez comment les premiers octets correspondent à l'en-tête Ethernet (couche 2), suivis de l'en-tête IP (couche 3), puis du contenu ICMP.

*Question :* combien d'octets fait typiquement un en-tête IPv4 minimal (sans options) ? Vérifiez en comptant dans la sortie hexadécimale, ou en consultant `man tcpdump`/une ressource de référence.

### Étape 3 — Comparer un port ouvert et un port fermé (couche 4)

```bash
# Port probablement fermé sur une machine distante accessible (adapter selon environnement)
timeout 3 bash -c "echo > /dev/tcp/8.8.8.8/12345" 2>&1; echo "Code retour: $?"

# Port probablement ouvert (DNS)
timeout 3 bash -c "echo > /dev/tcp/8.8.8.8/53" 2>&1; echo "Code retour: $?"
```
*Question :* les deux tests échouent-ils de la même façon ? Reliez votre observation à la distinction "Connection refused" vs "timeout" vue en cours et en TD.

### Étape 4 — Script de diagnostic multi-couches

Reprendre et tester le script de la section 16 du cours (`diag_multicouches.sh`), contre une cible connue (ex : `8.8.8.8` port `53`, puis une cible dont le port est fermé pour comparaison).

```bash
bash diag_multicouches.sh 8.8.8.8 53
bash diag_multicouches.sh 8.8.8.8 12345
```

## Commandes de référence
`ip addr`, `ip neigh`, `ip route`, `tcpdump -X`, `ping`, `/dev/tcp/` (redirection Bash)

## Résultats attendus
- Capture montrant les adresses MAC et IP identifiées séparément à l'étape 1
- Capture hexadécimale d'une trame ICMP avec identification manuelle des limites entre en-têtes Ethernet/IP (étape 2)
- Comparaison des deux comportements de port (étape 3), avec code retour et interprétation
- Sortie du script de diagnostic sur les deux cas de l'étape 4

## Questions

1. Dans la capture de l'étape 2, si vous aviez capturé le trafic sur une machine intermédiaire (un routeur) plutôt qu'à la source, quelle partie de la trame (Ethernet ou IP) vous attendriez-vous à voir différente par rapport à une capture à la source, conformément au TD exercice 3.2 ?
2. Pourquoi le test de l'étape 3 utilise-t-il une redirection Bash (`/dev/tcp/`) plutôt qu'un simple `ping`, pour tester un port applicatif ? Quelle couche chaque outil teste-t-il réellement ?
3. Si le script de l'étape 4 échouait dès le test `ping` (couche 3) pour une cible que vous savez pourtant en ligne, quelle hypothèse alternative (non liée à une vraie panne réseau) faudrait-il envisager avant de conclure à un problème de couche 3 ?

## Troubleshooting (panne à diagnostiquer)

**Situation fournie :** un étudiant exécute le script de diagnostic de l'étape 4 contre un serveur web interne qu'il sait pourtant fonctionnel (accessible normalement depuis son navigateur quelques minutes plus tôt). Le script échoue dès le test `ping` (couche 3), ce qui le conduit à conclure hâtivement à une panne réseau généralisée.

**Démarche attendue :**
1. Symptôme : échec du `ping` alors que le service est réellement accessible via navigateur
2. Hypothèse : le serveur (ou un pare-feu intermédiaire) **bloque spécifiquement le protocole ICMP** (utilisé par `ping`) tout en laissant passer normalement le trafic HTTP/HTTPS — une configuration de sécurité courante, pas une vraie panne
3. Vérification : tester directement le port applicatif (80/443) avec le test de l'étape 3, indépendamment du résultat du `ping`
4. Isolation : confirmer que le port applicatif répond correctement malgré l'échec du `ping`, ce qui prouve que les couches 1 à 3 fonctionnent bien en réalité — seul ICMP spécifiquement est filtré, un choix de configuration au niveau pare-feu, pas un dysfonctionnement de la couche réseau elle-même
5. Correction de la méthodologie (pas du réseau, qui n'a pas de panne réelle) : ne jamais interpréter un échec de `ping` isolé comme une preuve définitive d'indisponibilité réseau — toujours croiser avec un test au niveau de la couche réellement utile pour le service concerné (couche 4/7)
6. Documentation : consigner ce piège classique de méthodologie ("ICMP filtré ≠ service indisponible") comme point de vigilance pour tout futur diagnostic multi-couches

## Validation

Le TP est validé si l'étudiant fournit :
- Les 4 résultats/captures demandés
- Des réponses argumentées aux 3 questions, notamment une distinction correcte entre test de couche 3 (`ping`) et test de couche 4 (`/dev/tcp/`)
- Une identification correcte du piège méthodologique (ICMP filtré vs vraie panne) dans le scénario de troubleshooting

# UE2 — Module 8 : Ethernet, ARP et Commutation

---

# PARTIE I — COURS

## 1. Titre du cours
Ethernet, ARP et commutation : la couche 2 en pratique

## 2. Niveau
Élève ingénieur — cycle ingénieur, UE2 réseaux (S1)

## 3. Durée
CM : 4h (module complet : +3h TD, +4h TP en Parties II et III)

## 4. Prérequis
- UE2-Module 7 : modèle OSI/TCP-IP, encapsulation, adressage MAC vs IP

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Décrire la structure d'une trame Ethernet et le rôle de chacun de ses champs
- Expliquer le fonctionnement d'un commutateur (switch) : apprentissage, inondation, commutation
- Expliquer le protocole ARP et son rôle dans la traduction IP → MAC
- Différencier domaine de collision et domaine de diffusion (broadcast domain)
- Expliquer le rôle des VLAN et du trunking pour la segmentation logique d'un réseau physique
- Diagnostiquer un problème de couche 2 (table MAC, ARP, VLAN mal configuré) avec les outils Linux

## 6. Motivation

Avant qu'un paquet IP ne puisse voyager entre réseaux (couche 3, module suivant), il doit d'abord être physiquement transmis sur le réseau local — et c'est là qu'intervient la couche 2. Comment un commutateur sait-il vers quel port envoyer une trame sans jamais avoir été configuré manuellement avec une table d'adresses ? Comment une machine connaît-elle l'adresse MAC d'une autre machine dont elle ne connaît que l'adresse IP ? Pourquoi brancher deux câbles entre les mêmes deux switchs peut-il provoquer une tempête de diffusion (broadcast storm) qui paralyse tout un réseau ?

Ce module répond à ces questions très concrètes, à la base de tout diagnostic réseau local — la majorité des incidents "le réseau ne marche pas" en environnement d'entreprise se résolvent en réalité au niveau de cette couche, bien avant d'atteindre un problème de routage ou applicatif.

## 7. Concepts fondamentaux

### 7.1 La trame Ethernet

```
┌──────────┬──────────┬──────┬──────────────────┬─────┐
│ MAC dest.│ MAC src. │ Type │ Données (payload) │ FCS │
│ 6 octets │ 6 octets │2 oct.│ 46 à 1500 octets  │4 oct│
└──────────┴──────────┴──────┴──────────────────┴─────┘
```

- **Adresse MAC destination/source** : identifiants physiques sur 48 bits (6 octets), généralement affichés en hexadécimal (`aa:bb:cc:dd:ee:ff`). Les 3 premiers octets identifient le fabricant (OUI — Organizationally Unique Identifier).
- **Type/Longueur (EtherType)** : indique le protocole encapsulé (ex : `0x0800` pour IPv4, `0x0806` pour ARP, `0x86DD` pour IPv6) — c'est ce champ qui permet à la couche 2 de savoir à quelle couche 3 délivrer le contenu, exactement le même principe de démultiplexage qui sera revu au niveau des ports TCP/UDP.
- **FCS (Frame Check Sequence)** : somme de contrôle permettant de détecter (pas de corriger) une corruption de la trame durant la transmission — une trame corrompue est simplement rejetée par le récepteur, sans mécanisme de retransmission à ce niveau (cette responsabilité incombe aux couches supérieures si nécessaire, ex : TCP).

**Adresses MAC spéciales :** l'adresse de diffusion (broadcast) `ff:ff:ff:ff:ff:ff` est reçue et traitée par toutes les machines du domaine de diffusion ; des adresses multicast permettent de cibler un sous-ensemble de machines abonnées à un groupe donné.

### 7.2 Domaine de collision vs domaine de diffusion

- **Domaine de collision** : ensemble de machines qui pourraient potentiellement provoquer une collision électrique si elles émettent simultanément sur le même support physique partagé. Historiquement pertinent avec les concentrateurs (hubs, aujourd'hui obsolètes), qui répètent chaque signal électrique sur tous les ports sans discernement. Un **commutateur (switch)** moderne, en full-duplex, élimine pratiquement les collisions : chaque port constitue son propre domaine de collision.
- **Domaine de diffusion (broadcast domain)** : ensemble de machines qui reçoivent une même trame de diffusion (broadcast). Contrairement aux collisions, les switchs ne segmentent **pas** nativement les domaines de diffusion (une trame broadcast est transmise sur tous les ports par défaut) — seuls les **routeurs** (couche 3) ou les **VLAN** (voir 7.5) segmentent réellement les domaines de diffusion.

**Point clé pédagogique :** un réseau avec un domaine de diffusion trop large (trop de machines sur un même segment logique, sans VLAN) subit une dégradation de performance à mesure que le trafic broadcast (ARP, découverte de services...) croît proportionnellement au nombre de machines — c'est une des raisons principales de la segmentation en VLAN dans les réseaux d'entreprise de taille significative.

### 7.3 Fonctionnement d'un commutateur (switch)

Un commutateur maintient une **table d'adresses MAC** (CAM table — Content Addressable Memory), associant chaque adresse MAC connue au port physique par lequel elle a été vue. Le fonctionnement repose sur trois mécanismes :

1. **Apprentissage (learning)** : à chaque trame reçue, le switch enregistre l'association (adresse MAC source, port d'entrée) dans sa table, même si cette trame n'est pas destinée à ce switch en particulier — un apprentissage passif et continu.
2. **Inondation (flooding)** : si l'adresse MAC destination d'une trame n'est **pas encore connue** dans la table (ou s'il s'agit d'une adresse de diffusion), le switch transmet la trame sur **tous les ports** sauf celui d'origine, en attendant qu'une réponse permette d'apprendre la bonne association.
3. **Commutation (forwarding)** : une fois l'adresse MAC destination connue dans la table, le switch transmet la trame **uniquement** sur le port correspondant, sans inonder les autres ports — c'est ce comportement ciblé qui différencie fondamentalement un switch d'un simple hub, et qui limite la congestion inutile du réseau.

**Vieillissement (aging)** : les entrées de la table MAC ont une durée de vie limitée (typiquement 300 secondes par défaut) ; une entrée non rafraîchie par un nouveau trafic est supprimée, ce qui garantit que la table reste à jour si une machine change de port ou de switch.

### 7.4 ARP (Address Resolution Protocol)

Une machine qui doit envoyer un paquet IP à une autre machine du même réseau local a besoin de connaître son adresse MAC pour construire la trame Ethernet — mais elle ne connaît généralement que son adresse IP. **ARP** résout ce problème :

1. La machine émettrice diffuse une **requête ARP** ("ARP Request") en broadcast : *"Qui a l'adresse IP X.X.X.X ? Merci de répondre à mon adresse MAC."*
2. Seule la machine possédant réellement cette adresse IP répond, en **unicast**, avec une **réponse ARP** ("ARP Reply") contenant son adresse MAC
3. L'émetteur original enregistre cette association (IP ↔ MAC) dans son **cache ARP** (visible via `ip neigh`, déjà vu en module 7), évitant de refaire une requête ARP à chaque paquet

**Lien direct avec la sécurité (approfondissement du module 7) :** ARP, dans sa forme classique, ne comporte **aucune authentification** — n'importe quelle machine du réseau local peut répondre à une requête ARP en prétendant posséder une adresse IP qui n'est pas la sienne. C'est le principe de l'**ARP spoofing** (ou ARP poisoning), une attaque classique de réseau local permettant une interception de trafic (attaque de l'homme du milieu), traitée plus en détail dans le module sécurité (UE8) mais dont la compréhension du mécanisme légitime, vue ici, est le prérequis indispensable.

### 7.5 VLAN et Trunking

Un **VLAN** (Virtual LAN) permet de segmenter logiquement un réseau physique unique en plusieurs domaines de diffusion distincts, sans nécessiter de câblage physique séparé. Chaque port d'un switch peut être assigné à un VLAN spécifique ; les machines de VLAN différents ne peuvent pas communiquer directement au niveau 2, même si elles sont connectées au même switch physique — une communication inter-VLAN nécessite un routage (couche 3), abordé dans les modules suivants.

Le **trunking** (généralement via le standard **802.1Q**) permet de faire transiter le trafic de **plusieurs VLAN simultanément** sur un seul lien physique entre deux switchs, en ajoutant un **tag VLAN** (identifiant à 12 bits) dans l'en-tête de chaque trame concernée — un mécanisme indispensable dès qu'un réseau d'entreprise dépasse un unique switch avec ses VLAN.

**Pourquoi les VLAN existent-ils ?** Au-delà de la réduction du domaine de diffusion (performance), les VLAN permettent une **isolation logique de sécurité** (séparer le trafic de différents services ou clients sur la même infrastructure physique) et une **flexibilité d'administration** (réorganiser logiquement un réseau sans reconfiguration physique du câblage).

### 7.6 Notion de spanning tree (introduction)

Lorsque des switchs sont interconnectés avec des liens redondants (pour la tolérance de panne), une boucle physique peut apparaître dans la topologie — problématique car une trame de diffusion (broadcast) pourrait alors circuler indéfiniment en boucle, provoquant une **tempête de diffusion (broadcast storm)** paralysant rapidement le réseau. Le protocole **STP (Spanning Tree Protocol)** détecte ces boucles et désactive logiquement certains liens redondants (en les mettant en état de blocage), tout en les gardant disponibles comme secours automatique en cas de panne d'un lien actif — approfondi dans un module réseau ultérieur, mais son existence doit être connue dès ce module comme explication d'un phénomène observable en pratique.

## 8. Architecture

```
┌────────────────────────────────────────────────────────┐
│                  Commutateur (Switch)                     │
│                                                             │
│   Port 1 ── MAC-A                Table MAC (CAM)            │
│   Port 2 ── MAC-B         ┌───────────────────┐            │
│   Port 3 ── MAC-C         │ MAC-A → Port 1     │            │
│   Port 4 ── (VLAN 20)     │ MAC-B → Port 2     │            │
│                             │ MAC-C → Port 3     │            │
│                             └───────────────────┘            │
│                                                             │
│   Trame de A vers B :                                        │
│   1. Apprentissage : MAC-A associée au Port 1                │
│   2. Consultation table : MAC-B connue → Port 2               │
│   3. Commutation ciblée : trame envoyée uniquement Port 2     │
└────────────────────────────────────────────────────────┘
```

## 9. Fonctionnement interne (cas Linux)

Sous Linux, le noyau maintient sa propre implémentation de cache ARP (consultable via `ip neigh`, déjà vu au module 7), avec des états explicites pour chaque entrée : `REACHABLE` (récemment confirmée), `STALE` (potentiellement obsolète, sera revérifiée au prochain usage), `FAILED` (la résolution a échoué). Un pont logiciel Linux (`bridge`, utilisé notamment par Docker pour la mise en réseau des conteneurs) implémente lui-même une table d'adresses MAC et un comportement d'apprentissage/inondation/commutation directement analogue à celui d'un switch matériel — ce qui permet d'observer concrètement le comportement de couche 2 sans matériel physique dédié, en laboratoire.

## 10. Démonstration

```bash
# Observer la trame ARP en cours de résolution
sudo tcpdump -i any -nn arp &
ping -c 2 <IP_voisine_sur_le_meme_reseau>
wait

# Observer le cache ARP après résolution
ip neigh show

# Observer une interface bridge Linux (analogue logiciel d'un switch)
ip link show type bridge
bridge fdb show   # table d'adresses MAC du bridge, équivalent de la table CAM
```

## 11. Commandes / outils

| Commande | Rôle | Exemple |
|---|---|---|
| `ip neigh` | Afficher/gérer le cache ARP | `ip neigh show` |
| `ip neigh flush` | Vider le cache ARP (forcer une nouvelle résolution) | `ip neigh flush all` |
| `arping` | Envoyer des requêtes ARP manuelles (diagnostic) | `arping -c 3 192.168.1.1` |
| `bridge fdb show` | Afficher la table d'adresses MAC d'un bridge Linux | `bridge fdb show` |
| `ip link` | Afficher/gérer les interfaces réseau (état up/down) | `ip link show` |
| `tcpdump -nn arp` | Capturer spécifiquement le trafic ARP | `sudo tcpdump -i eth0 -nn arp` |

**Erreur fréquente :** confondre une entrée ARP en état `FAILED` (la résolution a réellement échoué, la machine cible ne répond pas) avec un simple délai réseau temporaire — toujours vérifier `ip neigh` après un court délai avant de conclure à une panne réelle de couche 2/3.

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** après le branchement d'un nouveau switch en tant que redondance sur un réseau d'entreprise (deux câbles reliant le nouveau switch à deux switchs existants différents, sans activation préalable de STP), l'ensemble du réseau devient inutilisable en quelques secondes : latence extrême, machines qui perdent leur connectivité, charge CPU anormalement élevée sur tous les switchs.

**Analyse attendue :** ce scénario est la signature classique d'une **tempête de diffusion (broadcast storm)** causée par une boucle de couche 2 non gérée (rappel section 7.6) : les deux câbles créent une boucle physique, et toute trame de diffusion (notamment le trafic ARP, très fréquent) se met à circuler indéfiniment en boucle entre les switchs, se multipliant à chaque passage. Résolution immédiate : débrancher physiquement l'un des deux câbles redondants pour casser la boucle et restaurer le service ; résolution de fond : activer STP (ou une variante comme RSTP) sur l'ensemble de l'infrastructure switching **avant** tout ajout de lien redondant, pour que la protection soit active dès le branchement.

## 15. Sécurité

- **ARP spoofing** (rappel section 7.4) : l'absence d'authentification du protocole ARP permet une usurpation facilitant l'interception de trafic — des mécanismes de protection existent (Dynamic ARP Inspection sur des switchs manageables, ports statiques)
- **VLAN hopping** : une mauvaise configuration de trunking peut permettre à un attaquant de faire croire à un switch qu'il est lui-même un port trunk légitime, contournant ainsi la segmentation VLAN prévue — d'où l'importance de désactiver explicitement le trunking automatique sur les ports destinés aux machines finales
- **MAC flooding** : un attaquant peut délibérément saturer la table CAM d'un switch avec un très grand nombre d'adresses MAC falsifiées ; certains switchs, une fois leur table pleine, basculent en mode "fail-open" et se comportent temporairement comme un hub (inondant tout le trafic sur tous les ports), permettant l'interception de trafic normalement isolé par port
- **Port security** : fonctionnalité des switchs manageables permettant de limiter le nombre d'adresses MAC autorisées par port, contre-mesure directe au MAC flooding

## 16. Automatisation

Script Bash de détection d'anomalie ARP (adresse IP associée à plusieurs adresses MAC différentes dans un court intervalle — signature possible d'ARP spoofing).

```bash
#!/usr/bin/env bash
# Surveille le cache ARP et détecte les changements suspects d'association IP/MAC
set -euo pipefail

LOGFILE="/var/log/arp_watch.log"
ETAT_PRECEDENT="/tmp/arp_etat_precedent.txt"

ETAT_ACTUEL=$(ip neigh show | awk '{print $1, $5}' | sort)

if [[ -f "$ETAT_PRECEDENT" ]]; then
    while read -r ip mac; do
        ancien_mac=$(grep "^$ip " "$ETAT_PRECEDENT" | awk '{print $2}' || true)
        if [[ -n "$ancien_mac" && "$ancien_mac" != "$mac" ]]; then
            echo "$(date '+%F %T') - ALERTE: IP $ip a changé de MAC ($ancien_mac -> $mac)" \
                | tee -a "$LOGFILE"
        fi
    done <<< "$ETAT_ACTUEL"
fi

echo "$ETAT_ACTUEL" > "$ETAT_PRECEDENT"
```

*Note pédagogique :* ce script est un détecteur simple à but éducatif — un changement légitime de carte réseau (remplacement matériel) déclenchera aussi une alerte (faux positif), à discuter en TD comme limite connue face à un outil de détection ARP spoofing professionnel (qui croiserait d'autres signaux).

## 17. Troubleshooting

Scénario court : une machine nouvellement connectée au réseau ne parvient à joindre aucune autre machine, y compris celles du même sous-réseau physique. Démarche : `ip link show` pour confirmer que l'interface est bien `UP` (couche 1/2 physique) → `ip addr show` pour confirmer la présence d'une adresse IP cohérente → tenter un `ping` vers une machine voisine connue → si échec, examiner `ip neigh` : l'entrée reste-t-elle en `INCOMPLETE`/`FAILED` ? → capturer le trafic ARP (`tcpdump -nn arp`) pour vérifier si la requête ARP part bien et si une réponse revient → si aucune requête ARP ne part, suspecter un problème de VLAN (le port du switch n'est peut-être pas dans le bon VLAN) plutôt qu'un problème purement applicatif ou IP.

## 18. Synthèse

- La trame Ethernet encapsule les données avec adresses MAC source/destination, un EtherType, et une somme de contrôle (FCS, détection uniquement, pas de correction)
- Un switch apprend dynamiquement les associations MAC↔port (apprentissage), inonde tant qu'une destination est inconnue (flooding), puis commute de façon ciblée (forwarding)
- ARP traduit une adresse IP en adresse MAC via une requête broadcast et une réponse unicast, sans aucune authentification native — base de l'ARP spoofing
- Les VLAN segmentent logiquement les domaines de diffusion sur une infrastructure physique unique ; le trunking (802.1Q) permet de faire transiter plusieurs VLAN sur un même lien
- Une boucle de couche 2 non protégée par STP provoque une tempête de diffusion capable de paralyser un réseau entier en quelques secondes
- Un diagnostic de couche 2 commence par l'état de l'interface, puis le cache ARP, puis une capture de trafic ARP ciblée

## 19. Questions d'évaluation

**QCM**
1. Un switch qui reçoit une trame dont l'adresse MAC destination est inconnue de sa table procède à :
 a) un rejet de la trame b) une inondation (flooding) sur tous les ports sauf celui d'origine c) une requête ARP automatique d) un blocage définitif du port

2. Le protocole ARP traduit :
 a) un nom de domaine en adresse IP b) une adresse IP en adresse MAC c) un port en service applicatif d) une adresse MAC en nom de domaine

**Questions ouvertes**
- Expliquez pourquoi une boucle physique entre deux switchs, sans protection STP, peut rendre un réseau entier inutilisable en quelques secondes.
- Expliquez pourquoi ARP est vulnérable à l'usurpation (spoofing), en identifiant précisément la propriété du protocole qui rend cela possible.

**Exercice technique**
- Donnez la commande permettant d'observer, sous Linux, l'ensemble des associations IP/MAC actuellement connues (cache ARP).

**Problème de diagnostic**
- Une machine ne parvient à joindre aucune autre machine du même réseau physique, bien que son interface soit `UP` avec une adresse IP valide. Proposez une démarche de diagnostic en au moins 3 étapes centrées sur la couche 2.

## 20. Mini-projet

Développer un script (Bash ou Python) qui :
1. Surveille en continu le cache ARP local et détecte tout changement d'association IP↔MAC dans un intervalle de temps court (signature possible d'ARP spoofing, avec les limites de faux positifs identifiées en cours)
2. Journalise chaque changement détecté avec horodatage, ancienne et nouvelle adresse MAC
3. Maintient une liste blanche configurable d'associations IP/MAC connues et légitimes (ex : passerelle par défaut), pour laquelle toute déviation déclenche une alerte de priorité plus élevée
4. Produit un rapport quotidien de synthèse
5. Documente explicitement les limites de la détection (faux positifs possibles lors d'un remplacement matériel légitime, absence de détection si l'attaquant cible une machine autre que celle exécutant le script)

---

# PARTIE II — TD

### Niveau 1 — Compréhension

**Exercice 1.1**
Décrivez les champs principaux d'une trame Ethernet et leur rôle respectif.

**Exercice 1.2**
Expliquez les trois mécanismes de fonctionnement d'un switch (apprentissage, inondation, commutation) avec vos propres mots.

**Exercice 1.3**
Expliquez le déroulement d'une résolution ARP, de la requête à la mise à jour du cache local.

### Niveau 2 — Application

**Exercice 2.1**
Donnez la commande permettant de vider le cache ARP local, forçant une nouvelle résolution pour toute communication future.

**Exercice 2.2**
Donnez la commande permettant d'observer la table d'adresses MAC apprise par un bridge Linux.

**Exercice 2.3**
Un switch reçoit une trame de diffusion (broadcast, MAC destination `ff:ff:ff:ff:ff:ff`). Sur combien de ports cette trame est-elle transmise (hors port d'origine) ?

### Niveau 3 — Analyse

**Exercice 3.1**
Un administrateur constate qu'une entrée de son cache ARP pour la passerelle par défaut change fréquemment d'adresse MAC associée, plusieurs fois par minute, sans qu'aucun changement matériel légitime n'ait été effectué. Formulez l'hypothèse la plus probable et expliquez pourquoi ce comportement est anormal au regard du fonctionnement légitime d'ARP décrit en cours.

**Exercice 3.2**
Deux VLAN distincts (VLAN 10 et VLAN 20) sont configurés sur un même switch. Une machine du VLAN 10 tente de joindre en `ping` une machine du VLAN 20 branchée sur le même switch physique. Le ping échoue. Est-ce un dysfonctionnement du switch ? Justifiez en vous appuyant sur la définition d'un VLAN vue en cours, et précisez ce qui serait nécessaire pour permettre cette communication.

### Niveau 4 — Problème complexe

**Exercice 4.1**
Une entreprise ajoute un nouveau switch à son infrastructure existante, avec deux liens redondants vers deux switchs différents pour la tolérance de panne, mais sans avoir vérifié au préalable que STP était actif sur l'ensemble de l'infrastructure. Quelques secondes après le branchement du second câble, tout le réseau devient inutilisable. En vous appuyant sur les mécanismes de commutation (apprentissage, inondation) et sur la notion de domaine de diffusion vus en cours, expliquez précisément pourquoi une simple trame de diffusion (par exemple une requête ARP) peut suffire à saturer entièrement le réseau dans ce scénario, en décrivant le phénomène de multiplication qui se produit à chaque passage de la trame dans la boucle.

---

## Correction du TD

**1.1** — Adresse MAC destination (6 octets, identifie le destinataire physique) ; adresse MAC source (6 octets, identifie l'émetteur) ; champ Type/EtherType (2 octets, indique le protocole encapsulé, ex: IPv4, ARP) ; données/payload (46 à 1500 octets, contenu encapsulé de la couche supérieure) ; FCS (4 octets, somme de contrôle permettant de détecter, mais pas corriger, une corruption de la trame).

**1.2** — Apprentissage : le switch enregistre, pour chaque trame reçue, l'association entre l'adresse MAC source de la trame et le port physique par lequel elle est arrivée, construisant ainsi progressivement sa table sans configuration manuelle. Inondation : quand l'adresse MAC destination d'une trame n'est pas encore connue dans la table (ou qu'il s'agit d'une diffusion), le switch transmet la trame sur tous les ports sauf celui d'origine, par prudence, pour être sûr qu'elle atteigne son destinataire même inconnu. Commutation : une fois l'adresse MAC destination connue dans la table, le switch transmet la trame uniquement sur le port correspondant, évitant de charger inutilement les autres ports.

**1.3** — La machine émettrice diffuse une requête ARP en broadcast, demandant "qui possède telle adresse IP ?". Seule la machine réellement propriétaire de cette adresse IP répond, en unicast, avec sa propre adresse MAC. L'émetteur original enregistre alors cette association IP↔MAC dans son cache ARP local, ce qui lui évite de refaire une requête ARP pour chaque paquet ultérieur destiné à la même machine, tant que l'entrée reste valide (non expirée).

**2.1** — `ip neigh flush all` (vide l'intégralité du cache ARP, forçant une nouvelle résolution à la prochaine communication).

**2.2** — `bridge fdb show` (affiche la table d'adresses MAC — Forwarding DataBase — apprise par le bridge Linux, analogue logicielle de la table CAM d'un switch matériel).

**2.3** — Sur **tous les ports** actifs du switch, à l'exception du port d'origine — c'est la définition même d'une trame de diffusion (broadcast) : elle doit atteindre toutes les machines du domaine de diffusion, indépendamment du contenu de la table MAC (le switch ne cherche même pas à consulter sa table pour ce type d'adresse destination spéciale).

**3.1** — Hypothèse la plus probable : une attaque **ARP spoofing** est en cours — un attaquant envoie régulièrement de fausses réponses ARP prétendant posséder l'adresse IP de la passerelle, dans le but de rediriger le trafic vers sa propre machine (interception de type homme du milieu). Ce comportement est anormal car, dans un fonctionnement légitime d'ARP décrit en cours, l'association IP↔MAC d'une machine physique donnée (ici la passerelle) ne devrait changer que lors d'un événement matériel réel et rare (remplacement de la carte réseau du routeur, par exemple) — des changements répétés en quelques minutes, sans aucun événement matériel légitime correspondant, ne peuvent raisonnablement s'expliquer par le fonctionnement normal du protocole.

**3.2** — Non, ce n'est pas un dysfonctionnement — c'est le comportement **attendu et voulu** d'un VLAN. Par définition (section 7.5 du cours), deux machines de VLAN différents constituent deux domaines de diffusion logiquement séparés, même si elles partagent le même switch physique : aucune communication directe au niveau 2 n'est possible entre elles, quel que soit le trafic (y compris ARP, lui-même limité à son propre domaine de diffusion). Pour permettre cette communication, il faudrait un **routage inter-VLAN** (fonction de couche 3, réalisée par un routeur ou un switch de niveau 3 — "router-on-a-stick" ou switch multicouche), sujet abordé dans les modules réseau suivants, pas par une simple reconfiguration de couche 2.

**4.1** — Une boucle physique entre switchs, sans STP actif, ne pose problème que pour le trafic de diffusion (broadcast) — et ARP en génère fréquemment. Lorsqu'une trame de diffusion (ex: une requête ARP) entre dans la boucle, chaque switch impliqué, ne connaissant pas encore l'adresse MAC destination (ici, l'adresse broadcast elle-même, jamais "apprise" au sens propre), applique le mécanisme d'inondation vu en cours : il retransmet la trame sur tous ses ports sauf celui d'origine — y compris vers l'autre switch de la boucle. Ce second switch, recevant à son tour cette copie de la trame par un port différent, applique lui aussi l'inondation, la retransmettant notamment vers le premier switch par l'autre lien de la boucle. La trame originale se retrouve ainsi dupliquée et réinjectée continuellement dans la boucle, chaque passage générant de nouvelles copies sur tous les ports concernés — une **multiplication exponentielle** du volume de trafic en quelques instants, saturant rapidement la bande passante disponible et la capacité de traitement (CPU) de chaque switch, jusqu'à rendre le réseau entier inutilisable. STP existe précisément pour détecter une telle boucle topologique et désactiver logiquement l'un des liens redondants avant qu'un tel phénomène ne puisse se produire.

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur réseau junior pour diagnostiquer des problèmes de connectivité locale sur un petit réseau de laboratoire, et pour construire un outil de surveillance basique du cache ARP.

## Architecture

Une machine Linux (VM ou conteneur avec capacités réseau étendues — privilèges root nécessaires pour la manipulation de bridges et de captures ARP), avec `ip`, `bridge`, `tcpdump`, `arping` disponibles. Pour observer un comportement de switch réel, l'usage d'un bridge Linux logiciel est recommandé (accessible sans matériel physique dédié).

## Objectifs
- Observer concrètement une résolution ARP en temps réel
- Observer le cache ARP et provoquer son expiration/rafraîchissement
- Créer et observer un bridge Linux (switch logiciel) avec sa table d'adresses MAC
- Développer un script de détection de changement suspect d'association ARP

## Préparation

```bash
which ip bridge tcpdump arping 2>/dev/null
```

## Étapes

### Étape 1 — Observer une résolution ARP en temps réel

```bash
# Choisir une IP voisine sur le même réseau (adapter selon l'environnement, ex: la passerelle)
CIBLE=$(ip route show default | awk '{print $3}')
echo "Cible: $CIBLE"

ip neigh flush all
sudo tcpdump -i any -nn arp -c 4 &
sleep 1
ping -c 1 "$CIBLE"
wait

ip neigh show
```
*Question :* combien de trames ARP (requête + réponse) observez-vous pour cette unique résolution ? Le nombre correspond-il à ce qui est décrit en cours ?

### Étape 2 — Créer un bridge Linux et observer sa table MAC

```bash
sudo ip link add name lab-bridge type bridge
sudo ip link set lab-bridge up
bridge link show
bridge fdb show br lab-bridge
```
*Note :* dans un environnement de laboratoire simplifié sans interfaces physiques supplémentaires à y attacher, cette étape vise surtout à observer la syntaxe et la structure de la table `fdb`, qui restera vide ou minimale — comparez avec la sortie `bridge fdb show` sur l'interface bridge par défaut de Docker si disponible (`docker0`), qui contient généralement des entrées réelles si des conteneurs sont actifs.

```bash
# Si Docker est disponible :
bridge fdb show br docker0 2>/dev/null || echo "docker0 non disponible dans cet environnement"
```

### Étape 3 — Provoquer et observer une entrée ARP obsolète

```bash
# Résoudre une IP voisine
ping -c 1 "$CIBLE"
ip neigh show | grep "$CIBLE"

# Attendre au-delà du délai de vieillissement configuré, ou forcer l'état STALE
sudo ip neigh change "$CIBLE" dev $(ip route show default | awk '{print $5}') nud stale
ip neigh show | grep "$CIBLE"
```
*Question :* quelle différence de comportement attendez-vous, côté noyau, entre une entrée `REACHABLE` et une entrée `STALE` lors du prochain paquet envoyé vers cette IP ?

### Étape 4 — Script de détection de changement ARP

Reprendre et tester le script de la section 16 du cours (`arp_watch.sh`), exécuté à quelques secondes d'intervalle, en simulant un changement d'association :

```bash
bash arp_watch.sh   # première exécution, établit l'état de référence
# Simuler manuellement un changement suspect (à but pédagogique uniquement) :
sudo ip neigh replace "$CIBLE" lladdr 00:11:22:33:44:55 dev $(ip route show default | awk '{print $5}') nud permanent
bash arp_watch.sh   # seconde exécution, doit détecter le changement
cat /var/log/arp_watch.log
```

## Commandes de référence
`ip neigh`, `ip link`, `bridge fdb`, `bridge link`, `tcpdump -nn arp`, `arping`

## Résultats attendus
- Capture des trames ARP observées à l'étape 1 (requête et réponse)
- Sortie de `bridge fdb show` à l'étape 2, avec commentaire sur son contenu (vide ou peuplé)
- Capture montrant le changement d'état `REACHABLE` → `STALE` à l'étape 3
- Log d'alerte généré par le script de détection à l'étape 4, montrant la détection du changement simulé

## Questions

1. Dans l'étape 1, si la machine cible avait déjà une entrée `REACHABLE` valide dans le cache ARP avant le `ping`, vous attendiez-vous à observer une nouvelle requête ARP sur le réseau ? Pourquoi ?
2. Pourquoi la simulation de l'étape 4 (changement manuel via `ip neigh replace`) est-elle un bon substitut pédagogique à une vraie attaque ARP spoofing, sans nécessiter d'outils offensifs réels en salle de TP ?
3. Le script `arp_watch.sh` détecterait-il une attaque ARP spoofing menée contre une **autre** machine du réseau (pas celle qui exécute le script) ? Justifiez, et reliez votre réponse à la limite documentée dans le mini-projet du cours.

## Troubleshooting (panne à diagnostiquer)

**Situation fournie :** après avoir déployé `arp_watch.sh` en surveillance continue sur plusieurs postes d'un même laboratoire, l'équipe reçoit une alerte quasi quotidienne à heure fixe, chaque matin au démarrage des machines, pour la même IP (la passerelle par défaut), sans qu'aucune attaque réelle ne soit plausible dans ce contexte.

**Démarche attendue :**
1. Symptôme : alerte récurrente et prévisible, corrélée au démarrage matinal des postes
2. Hypothèse : au démarrage, le cache ARP de chaque machine est vide (flush implicite au redémarrage) ; la première résolution de la passerelle après démarrage crée une entrée qui n'existait pas dans l'état de référence sauvegardé par le script la veille (fichier `/tmp/arp_etat_precedent.txt`, qui a pu être perdu si `/tmp` est nettoyé au redémarrage, ou contenir un état différent) — ce n'est pas un changement d'association MAC suspecte, mais une simple première apparition
3. Vérification : examiner le contenu exact de l'alerte journalisée — s'agit-il réellement d'un changement d'un MAC connu vers un autre MAC différent, ou le script a-t-il mal interprété une entrée absente comme un "changement" ?
4. Isolation : relire la logique du script (section 16 du cours) — la condition `[[ -n "$ancien_mac" && "$ancien_mac" != "$mac" ]]` ne devrait déclencher une alerte que si un ancien MAC existait réellement et diffère du nouveau ; si le fichier d'état précédent est vide ou absent au redémarrage, la condition `-n "$ancien_mac"` devrait être fausse et n'aurait pas dû déclencher d'alerte — vérifier si `/tmp` est bien persistant sur ces machines ou nettoyé au reboot
5. Correction : stocker le fichier d'état de référence dans un emplacement persistant (`/var/lib/arp_watch/` plutôt que `/tmp`), pour éviter la perte d'état au redémarrage qui provoque cette fausse alerte quotidienne
6. Vérification : surveiller sur plusieurs redémarrages consécutifs après correction, confirmer l'absence de fausse alerte
7. Documentation : noter le choix de `/tmp` comme erreur de conception initiale à corriger systématiquement dans tout script de surveillance destiné à survivre aux redémarrages — leçon transversale applicable à d'autres scripts développés au fil de la formation

## Validation

Le TP est validé si l'étudiant fournit :
- Les 4 résultats/captures demandés
- Des réponses argumentées aux 3 questions, notamment une identification correcte de la limite de portée du script (question 3)
- Une identification correcte de la cause du scénario de troubleshooting (fichier d'état non persistant dans `/tmp`)

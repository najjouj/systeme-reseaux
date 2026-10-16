# UE2 — Module 9 : IPv4/IPv6, Subnetting, CIDR et Routage Statique

---

# PARTIE I — COURS

## 1. Titre du cours
Adressage IP et routage statique : IPv4, IPv6, subnetting et CIDR

## 2. Niveau
Élève ingénieur — cycle ingénieur, UE2 réseaux (S1)

## 3. Durée
CM : 5h (module dense, calculatoire — +4h TD, +4h TP en Parties II et III)

## 4. Prérequis
- UE2-Module 7 : modèle OSI/TCP-IP, encapsulation
- UE2-Module 8 : Ethernet, ARP (le routage s'appuie sur la résolution de couche 2 pour le dernier saut)
- Représentation binaire des nombres (rappel UE1-M1)

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Convertir une adresse IPv4 entre notation décimale pointée et binaire
- Calculer un plan d'adressage complet (réseau, diffusion, plage d'hôtes utilisables, masque) à partir d'une adresse et d'un préfixe CIDR
- Découper un réseau en sous-réseaux de tailles adaptées à des besoins différents (subnetting à taille variable, VLSM)
- Décrire la structure d'une adresse IPv6 et ses principaux types (unicast global, link-local, multicast)
- Lire et interpréter une table de routage, et configurer une route statique simple
- Diagnostiquer un problème d'adressage ou de routage avec les outils Linux

## 6. Motivation

Une entreprise dispose d'un bloc d'adresses `192.168.10.0/24` et doit y héberger trois services : un réseau de serveurs (30 machines), un réseau de postes utilisateurs (100 machines), et un réseau d'administration isolé (10 machines). Comment découper ce bloc unique de façon à satisfaire ces trois besoins sans gaspiller d'adresses, tout en gardant une frontière claire entre chaque segment ? Et une fois ce découpage fait, comment les paquets savent-ils, à chaque routeur, vers quelle direction continuer leur chemin jusqu'à leur destination finale ?

Ce module donne les compétences de calcul (subnetting) et de configuration (routage) qui sont, avec le TCP vu au module suivant, le cœur du métier d'ingénieur réseau au quotidien — une compétence qui ne s'acquiert que par la pratique répétée du calcul, d'où l'importance particulière du TD dans ce module.

## 7. Concepts fondamentaux

### 7.1 Structure d'une adresse IPv4

Une adresse IPv4 est codée sur **32 bits**, généralement représentée en **notation décimale pointée** : 4 groupes de 8 bits (octets), chacun exprimé en décimal de 0 à 255, séparés par des points (ex : `192.168.10.25`).

Chaque adresse est logiquement divisée en deux parties : une **partie réseau** (identifie le sous-réseau) et une **partie hôte** (identifie une machine précise au sein de ce sous-réseau). La frontière entre ces deux parties est définie par le **masque de sous-réseau**.

### 7.2 Masque de sous-réseau et notation CIDR

Un masque de sous-réseau (ex : `255.255.255.0`) est lui aussi codé sur 32 bits : les bits à `1` désignent la partie réseau, les bits à `0` désignent la partie hôte.

La **notation CIDR** (Classless Inter-Domain Routing, ex : `/24`) exprime directement le nombre de bits réservés à la partie réseau, remplaçant avantageusement l'ancien système historique de classes A/B/C (rigide, aujourd'hui obsolète en pratique mais toujours utile à connaître pour comprendre certains defaults historiques) :

| Notation CIDR | Masque décimal | Nombre d'adresses totales | Nombre d'hôtes utilisables |
|---|---|---|---|
| /24 | 255.255.255.0 | 256 | 254 |
| /25 | 255.255.255.128 | 128 | 126 |
| /26 | 255.255.255.192 | 64 | 62 |
| /27 | 255.255.255.224 | 32 | 30 |
| /28 | 255.255.255.240 | 16 | 14 |
| /30 | 255.255.255.252 | 4 | 2 |

**Pourquoi toujours "-2" pour les hôtes utilisables ?** La première adresse d'un sous-réseau identifie le **réseau lui-même** (tous les bits hôte à 0), et la dernière est l'**adresse de diffusion** (broadcast, tous les bits hôte à 1, rappel module 8) — ni l'une ni l'autre ne peut être attribuée à une machine.

### 7.3 Calcul de sous-réseau — méthode

Pour une adresse `192.168.10.77/27` :

1. **Convertir en binaire** : `192.168.10.77` = `11000000.10101000.00001010.01001101`
2. **Identifier la frontière** : `/27` signifie 27 bits de réseau, donc 5 bits d'hôte dans le dernier octet
3. **Adresse réseau** : mettre tous les bits hôte à 0 → `11000000.10101000.00001010.01000000` = `192.168.10.64`
4. **Adresse de diffusion** : mettre tous les bits hôte à 1 → `11000000.10101000.00001010.01011111` = `192.168.10.95`
5. **Plage d'hôtes utilisables** : de `192.168.10.65` à `192.168.10.94` (30 adresses)

**Raccourci pratique (sans conversion binaire complète) :** avec un masque `/27`, le "saut" entre sous-réseaux consécutifs dans le dernier octet concerné est de `256 - 224 = 32`. Les sous-réseaux `/27` successifs commencent donc à `.0, .32, .64, .96, .128...` — l'adresse `192.168.10.77` tombe dans l'intervalle `.64` à `.95`, confirmant le calcul binaire.

### 7.4 VLSM (Variable Length Subnet Masking)

Le **VLSM** permet de découper un même bloc d'adresses en sous-réseaux de **tailles différentes**, adaptées aux besoins réels de chaque segment, plutôt que d'imposer un découpage uniforme gaspillant des adresses. Méthode : allouer d'abord les sous-réseaux les plus grands (besoins les plus importants), puis subdiviser l'espace restant pour les besoins plus petits — dans l'ordre décroissant de taille, pour éviter les conflits de chevauchement.

**Exemple appliqué à la motivation du cours** (`192.168.10.0/24`, besoins de 100, 30 et 10 machines) :
- Réseau utilisateurs (100 machines → besoin de 126 adresses utilisables minimum) : `192.168.10.0/25` (126 hôtes utilisables)
- Réseau serveurs (30 machines → besoin de 30 adresses utilisables minimum) : `192.168.10.128/27` (30 hôtes utilisables)
- Réseau administration (10 machines → besoin de 14 adresses utilisables minimum, prochaine puissance de 2 disponible) : `192.168.10.160/28` (14 hôtes utilisables)

### 7.5 Adresses IPv4 privées et NAT (introduction)

Certaines plages d'adresses IPv4 sont réservées à un usage **privé** (non routable directement sur Internet), définies par la RFC 1918 :
- `10.0.0.0/8`
- `172.16.0.0/12`
- `192.168.0.0/16`

Ces plages permettent à des réseaux internes de fonctionner sans consommer d'adresses IPv4 publiques (ressource limitée) ; la traduction d'adresse (**NAT — Network Address Translation**, approfondie dans un module ultérieur) permet à des machines en adressage privé de communiquer avec Internet via une adresse publique partagée.

### 7.6 Structure d'une adresse IPv6

Face à l'épuisement progressif des adresses IPv4 (32 bits, ~4,3 milliards d'adresses), **IPv6** utilise des adresses codées sur **128 bits**, offrant un espace d'adressage considérablement plus vaste. Notation : 8 groupes de 16 bits en hexadécimal, séparés par `:` (ex : `2001:0db8:0000:0000:0000:ff00:0042:8329`), avec deux simplifications d'écriture autorisées :
- Suppression des zéros non significatifs en tête de chaque groupe (`0db8` → `db8`)
- Remplacement d'**une seule** séquence continue de groupes entièrement nuls par `::` (ex : l'adresse ci-dessus devient `2001:db8::ff00:42:8329`)

**Principaux types d'adresses IPv6 :**
- **Unicast global** : équivalent d'une adresse IPv4 publique routable, identifie une interface unique
- **Link-local** (préfixe `fe80::/10`) : valide uniquement sur le lien local (segment physique direct), non routée, automatiquement assignée à chaque interface — utilisée notamment pour la découverte de voisins (NDP, successeur d'ARP en IPv6)
- **Multicast** (préfixe `ff00::/8`) : remplace entièrement le broadcast, absent en IPv6 — toute communication de groupe passe par des adresses multicast dédiées, plus économe en traitement que la diffusion générale

**Point clé pédagogique :** IPv6 **supprime le broadcast** au profit exclusif du multicast — un choix de conception directement motivé par les problèmes de domaine de diffusion et de tempêtes vus au module 8 : le multicast cible uniquement les machines réellement intéressées par un type de message donné, réduisant structurellement la charge inutile sur le réseau.

### 7.7 Routage statique

Le **routage** est la fonction qui permet à un paquet de traverser plusieurs réseaux distincts pour atteindre sa destination — contrairement à la commutation (couche 2, un seul réseau physique/logique), le routage opère à la **couche 3** et repose sur les adresses IP.

Une **table de routage** (consultable via `ip route`) associe des préfixes réseau à des interfaces de sortie et/ou des adresses de "prochain saut" (next hop). Pour chaque paquet, le routeur choisit la route la **plus spécifique** correspondant à l'adresse destination (principe du "longest prefix match" — la route avec le masque le plus long/précis qui correspond est toujours préférée à une route plus générale).

La **route par défaut** (`0.0.0.0/0` en IPv4, `::/0` en IPv6) est utilisée quand aucune route plus spécifique ne correspond à la destination — c'est elle qui permet à une machine d'atteindre Internet via sa passerelle, sans connaître individuellement toutes les routes possibles.

Le **routage statique** consiste à configurer manuellement chaque route nécessaire, adapté aux petites topologies stables ; le routage dynamique (protocoles comme OSPF, BGP — hors périmètre de ce module, abordé en UE6) automatise cette configuration pour des réseaux de grande taille ou changeants.

## 8. Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Machine A (192.168.10.77/27)                              │
│  Table de routage locale :                                    │
│    192.168.10.64/27 → directement connecté (eth0)             │
│    0.0.0.0/0 → via 192.168.10.65 (passerelle, eth0)            │
└──────────────────────┬──────────────────────────────────┘
                        │ paquet vers 8.8.8.8 (hors réseau local)
                        ▼
┌─────────────────────────────────────────────────────────┐
│  Routeur (passerelle 192.168.10.65)                          │
│  Table de routage :                                            │
│    192.168.10.64/27 → directement connecté                      │
│    0.0.0.0/0 → via <FAI, prochain saut>                          │
└──────────────────────┬──────────────────────────────────┘
                        ▼
                  ... vers Internet ...
```

## 9. Fonctionnement interne (cas Linux)

Sous Linux, la table de routage est maintenue par le noyau et consultée pour chaque paquet sortant. La recherche applique le principe du **longest prefix match** : parmi toutes les routes dont le préfixe correspond à l'adresse destination, celle avec le masque le plus spécifique (le plus long, ex : `/32` avant `/24` avant `/0`) est toujours choisie en priorité — c'est ce mécanisme qui permet de définir une route par défaut générale (`/0`) tout en la court-circuitant automatiquement pour des destinations spécifiques disposant d'une route plus précise, sans configuration explicite d'exception à gérer manuellement.

Chaque interface réseau peut porter plusieurs adresses IP simultanément (visible avec `ip addr`) ; la table de routage détermine, pour chaque destination, quelle interface et quelle adresse source seront utilisées, indépendamment du nombre d'adresses configurées sur la machine.

## 10. Démonstration

```bash
# Observer la table de routage locale
ip route show

# Calculer un sous-réseau (vérification avec un outil dédié)
ipcalc 192.168.10.77/27   # si disponible ; sinon calcul manuel comme en section 7.3

# Ajouter une route statique temporaire (exemple de laboratoire)
sudo ip route add 10.20.0.0/24 via 192.168.10.1

# Observer une adresse IPv6 locale
ip -6 addr show
```

## 11. Commandes / outils

| Commande | Rôle | Exemple |
|---|---|---|
| `ip addr` | Afficher les adresses IP (v4/v6) des interfaces | `ip addr show` |
| `ip route show` | Afficher la table de routage | `ip route show` |
| `ip route add` | Ajouter une route statique | `ip route add 10.0.0.0/8 via 192.168.1.1` |
| `ip route del` | Supprimer une route | `ip route del 10.0.0.0/8` |
| `ip -6 addr` / `ip -6 route` | Équivalents IPv6 | `ip -6 route show` |
| `ipcalc` | Calculateur de sous-réseau (si installé) | `ipcalc 192.168.10.0/27` |
| `traceroute` / `tracepath` | Visualiser le chemin (les sauts) vers une destination | `traceroute 8.8.8.8` |

**Erreur fréquente :** oublier qu'une route ajoutée avec `ip route add` (sans persistance via un fichier de configuration réseau) **disparaît au redémarrage** — piège classique pour un étudiant qui teste une route en laboratoire puis s'étonne de sa disparition après un reboot ; en production, une route statique permanente doit être déclarée dans la configuration réseau persistante appropriée à la distribution utilisée.

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** après l'ajout d'un nouveau segment réseau `192.168.20.0/24` relié par un routeur dédié, les machines de ce nouveau segment ne parviennent pas à joindre Internet, bien qu'elles puissent communiquer entre elles normalement et joindre le routeur local.

**Analyse attendue :** la communication interne au segment fonctionnant (couches 1-2-3 locales opérationnelles, cohérent avec le module 8), le problème se situe probablement dans le **routage au-delà du segment local** : soit la route par défaut n'est pas configurée sur les machines du nouveau segment, soit le routeur du segment lui-même ne dispose pas d'une route (statique ou par défaut) vers Internet, soit un routeur intermédiaire ne connaît pas de route de retour vers ce nouveau segment `192.168.20.0/24` (un problème de routage asymétrique où l'aller fonctionnerait mais pas le retour). Démarche : `ip route show` sur une machine du segment pour confirmer la présence d'une route par défaut cohérente ; `traceroute 8.8.8.8` pour identifier précisément à quel saut la communication s'arrête ; vérifier la configuration de routage du routeur du segment lui-même.

## 15. Sécurité

- Un plan d'adressage mal segmenté (pas de séparation VLSM entre réseau serveurs sensibles et réseau utilisateurs) facilite la propagation latérale d'une attaque — la segmentation réseau (couplée aux VLAN du module 8) est une mesure de défense en profondeur de base
- Les routes statiques mal configurées peuvent créer des chemins de routage non désirés (ex : exposer accidentellement un réseau d'administration censé être isolé) — chaque route ajoutée doit être justifiée et documentée
- IPv6 supprime le broadcast (7.6) mais introduit ses propres enjeux de sécurité liés au multicast et à la découverte de voisins (NDP), avec des attaques analogues en esprit à l'ARP spoofing (rappel module 8), abordées plus en détail en UE8
- L'usage d'adresses privées (RFC 1918) associées au NAT constitue une forme basique d'isolation vis-à-vis d'Internet, mais ne doit jamais être confondu avec une véritable mesure de sécurité (un pare-feu reste nécessaire, sujet d'un module ultérieur)

## 16. Automatisation

Script Bash de calcul rapide de plage d'un sous-réseau IPv4 à partir d'une adresse et d'un préfixe CIDR, sans dépendance externe (`ipcalc` non toujours disponible).

```bash
#!/usr/bin/env bash
# Calcule adresse réseau, diffusion et plage utilisable pour une entrée CIDR IPv4
set -euo pipefail

ENTREE="${1:?Usage: $0 <adresse/prefixe>, ex: 192.168.10.77/27}"
IP="${ENTREE%/*}"
PREFIXE="${ENTREE#*/}"

IFS='.' read -r o1 o2 o3 o4 <<< "$IP"
IP_DEC=$(( (o1 << 24) + (o2 << 16) + (o3 << 8) + o4 ))

MASQUE_DEC=$(( 0xFFFFFFFF << (32 - PREFIXE) & 0xFFFFFFFF ))
RESEAU_DEC=$(( IP_DEC & MASQUE_DEC ))
DIFFUSION_DEC=$(( RESEAU_DEC | (~MASQUE_DEC & 0xFFFFFFFF) ))

vers_decimal_pointe() {
    local d=$1
    echo "$(( (d >> 24) & 255 )).$(( (d >> 16) & 255 )).$(( (d >> 8) & 255 )).$(( d & 255 ))"
}

echo "Adresse réseau   : $(vers_decimal_pointe $RESEAU_DEC)/$PREFIXE"
echo "Adresse diffusion: $(vers_decimal_pointe $DIFFUSION_DEC)"
echo "Premier hôte     : $(vers_decimal_pointe $((RESEAU_DEC + 1)))"
echo "Dernier hôte     : $(vers_decimal_pointe $((DIFFUSION_DEC - 1)))"
echo "Nombre d'hôtes   : $(( DIFFUSION_DEC - RESEAU_DEC - 1 ))"
```

## 17. Troubleshooting

Scénario court : une machine nouvellement configurée avec une adresse IP statique ne parvient à joindre ni son réseau local ni Internet, alors que le câble est bien connecté (interface `UP`, confirmé en module 8). Démarche : `ip addr show` pour vérifier que l'adresse IP et le masque configurés sont cohérents avec le reste du réseau (masque incorrect classique : une adresse `/24` configurée par erreur en `/32`, isolant la machine dans son propre "sous-réseau" d'une seule adresse) → `ip route show` pour vérifier la présence et la cohérence de la route par défaut → si le masque est erroné, corriger la configuration et revérifier — un des diagnostics les plus fréquents en environnement réel, directement lié à une maîtrise insuffisante du calcul de sous-réseau vu dans ce module.

## 18. Synthèse

- Une adresse IPv4 (32 bits) se divise en partie réseau et partie hôte, la frontière étant fixée par le masque/préfixe CIDR
- La première adresse d'un sous-réseau identifie le réseau, la dernière est l'adresse de diffusion — ni l'une ni l'autre n'est attribuable à un hôte
- Le VLSM permet un découpage à tailles variables, adapté aux besoins réels de chaque segment, sans gaspillage d'adresses
- IPv6 (128 bits) élimine l'épuisement d'adressage et supprime le broadcast au profit du multicast, avec une adresse link-local automatique par interface
- La table de routage applique le principe du "longest prefix match" ; la route par défaut (`/0`) sert de filet de sécurité pour toute destination non spécifiquement connue
- Un masque de sous-réseau incorrect est une des causes les plus fréquentes et les plus faciles à diagnostiquer d'isolement réseau, à condition de savoir calculer correctement une plage d'adresses

## 19. Questions d'évaluation

**QCM**
1. Pour un réseau `/28`, le nombre d'hôtes utilisables est :
 a) 16 b) 14 c) 30 d) 8

2. Le principe appliqué par une table de routage pour choisir entre plusieurs routes correspondant à une même destination est :
 a) le premier ajout chronologique b) le préfixe le plus long (longest prefix match) c) l'ordre alphabétique d) le nombre de sauts minimal uniquement

**Questions ouvertes**
- Calculez l'adresse réseau, l'adresse de diffusion et la plage d'hôtes utilisables pour `172.16.5.130/26`. Montrez votre calcul.
- Expliquez pourquoi IPv6 n'a pas besoin d'adresse de diffusion (broadcast), contrairement à IPv4.

**Exercice technique**
- Donnez la commande permettant d'ajouter une route statique vers le réseau `10.50.0.0/16` via la passerelle `192.168.1.254`.

**Problème de diagnostic**
- Une machine configurée avec l'adresse `192.168.1.50/32` (au lieu de `/24` attendu) ne parvient à joindre aucune autre machine de son réseau local. Expliquez précisément pourquoi ce masque erroné cause ce symptôme.

## 20. Mini-projet

Développer un script (Bash ou Python) qui, à partir d'un bloc d'adresses de départ (ex : `192.168.0.0/24`) et d'une liste de besoins en nombre d'hôtes par segment (ex : 100, 50, 20, 10) :
1. Calcule automatiquement un découpage VLSM valide, en respectant l'ordre décroissant de taille (méthode vue en cours)
2. Vérifie qu'aucun chevauchement n'existe entre les sous-réseaux calculés
3. Vérifie que le bloc de départ est suffisant pour satisfaire tous les besoins ; sinon, signale clairement l'insuffisance plutôt que de produire un résultat incorrect
4. Produit un rapport listant, pour chaque segment : adresse réseau, préfixe CIDR, adresse de diffusion, plage d'hôtes utilisables, nombre d'hôtes réellement disponibles
5. Exporte ce rapport en format texte structuré, réutilisable pour une documentation d'infrastructure

---

# PARTIE II — TD

*Ce TD est volontairement calculatoire — la maîtrise du calcul de sous-réseau ne s'acquiert que par la pratique répétée.*

### Niveau 1 — Compréhension

**Exercice 1.1**
Convertissez l'adresse `192.168.1.200` en binaire, octet par octet.

**Exercice 1.2**
Pour un masque `/26`, combien de bits sont réservés à la partie réseau, et combien à la partie hôte ? Combien d'adresses totales cela représente-t-il ?

**Exercice 1.3**
Expliquez pourquoi la première et la dernière adresse d'un sous-réseau ne peuvent jamais être attribuées à une machine.

### Niveau 2 — Application

**Exercice 2.1**
Pour l'adresse `10.0.0.50/28`, calculez : adresse réseau, adresse de diffusion, plage d'hôtes utilisables.

**Exercice 2.2**
Donnez le masque décimal pointé correspondant à `/29`.

**Exercice 2.3**
Donnez la commande permettant d'afficher la table de routage IPv6 d'une machine Linux.

### Niveau 3 — Analyse

**Exercice 3.1**
Une entreprise dispose du bloc `172.20.0.0/22` et doit créer 4 sous-réseaux de taille égale. Calculez le préfixe CIDR résultant pour chaque sous-réseau, et donnez l'adresse réseau de chacun des 4 sous-réseaux.

**Exercice 3.2**
Une machine affiche la table de routage suivante :
```
10.0.0.0/8 via 192.168.1.1
10.0.5.0/24 via 192.168.1.2
0.0.0.0/0 via 192.168.1.1
```
Un paquet est destiné à `10.0.5.30`. Par quelle passerelle sera-t-il envoyé ? Justifiez précisément en citant le principe vu en cours qui gouverne ce choix.

### Niveau 4 — Problème complexe

**Exercice 4.1**
Une entreprise dispose du bloc `192.168.100.0/24` et doit héberger, sur ce bloc unique, quatre segments aux besoins suivants : un réseau serveurs de 50 machines, un réseau utilisateurs de 90 machines, un réseau invités (Wi-Fi) de 20 machines, et une liaison point-à-point entre deux routeurs (qui ne nécessite que 2 adresses utilisables). En appliquant la méthode VLSM vue en cours (allocation par ordre décroissant de taille), proposez un découpage complet : pour chaque segment, donnez l'adresse réseau, le préfixe CIDR, et la plage d'hôtes utilisables, en vous assurant qu'aucun chevauchement n'existe entre les segments et que le bloc de départ suffit.

---

## Correction du TD

**1.1** — `192` = `11000000` ; `168` = `10101000` ; `1` = `00000001` ; `200` = `11001000`. Soit : `11000000.10101000.00000001.11001000`.

**1.2** — Un masque `/26` réserve 26 bits à la partie réseau et 6 bits à la partie hôte (32 - 26 = 6). Nombre d'adresses totales : 2⁶ = 64.

**1.3** — La première adresse (tous les bits hôte à 0) désigne conventionnellement le **réseau lui-même**, utilisée pour identifier le sous-réseau dans les tables de routage, pas une machine individuelle. La dernière adresse (tous les bits hôte à 1) est l'**adresse de diffusion**, utilisée pour envoyer une trame/un paquet à toutes les machines du sous-réseau simultanément (rappel module 8) — l'attribuer à une seule machine créerait une ambiguïté avec cette fonction de diffusion collective.

**2.1** — `/28` réserve 4 bits à la partie hôte, soit un saut de 16 (256-240=16) dans le dernier octet. Les sous-réseaux `/28` commencent à `.0, .16, .32, .48...`. L'adresse `10.0.0.50` tombe dans l'intervalle `.48` à `.63`. Adresse réseau : `10.0.0.48/28`. Adresse de diffusion : `10.0.0.63`. Plage d'hôtes utilisables : `10.0.0.49` à `10.0.0.62`.

**2.2** — `/29` réserve 3 bits à la partie hôte (32-29=3), soit un masque dont le dernier octet est `11111000` = `248`. Masque complet : `255.255.255.248`.

**2.3** — `ip -6 route show`.

**3.1** — Bloc de départ `172.20.0.0/22` : 22 bits de réseau, 10 bits d'hôte (2¹⁰ = 1024 adresses totales dans le bloc). Pour créer 4 sous-réseaux de taille égale, il faut emprunter 2 bits supplémentaires à la partie hôte (2² = 4 sous-réseaux), donnant un préfixe `/24` pour chacun (22+2=24). Les 4 sous-réseaux résultants : `172.20.0.0/24`, `172.20.1.0/24`, `172.20.2.0/24`, `172.20.3.0/24` (le "saut" au niveau du 3ᵉ octet étant de 1, puisque `/24` correspond exactement à une frontière d'octet).

**3.2** — Le paquet sera envoyé via `192.168.1.2`. Justification : bien que `10.0.0.0/8` corresponde également à la destination `10.0.5.30` (elle appartient à ce bloc plus large), la route `10.0.5.0/24` est **plus spécifique** (préfixe plus long, 24 bits contre 8) et correspond également à cette destination précise. Le principe du **longest prefix match** vu en cours impose de toujours préférer la route la plus spécifique disponible parmi celles qui correspondent — la route générale `/8` ne serait utilisée que pour des destinations `10.x.x.x` ne correspondant à aucune route plus précise déclarée.

**4.1** — Allocation par ordre décroissant de taille, méthode VLSM :
- **Réseau utilisateurs (90 machines)** : nécessite au moins 90 adresses utilisables → prochaine puissance de 2 suffisante : 2⁷-2=126 → préfixe `/25`. Alloué en premier (plus grand besoin) : `192.168.100.0/25` (hôtes utilisables : `192.168.100.1` à `192.168.100.126`)
- **Réseau serveurs (50 machines)** : nécessite au moins 50 adresses utilisables → 2⁶-2=62 → préfixe `/26`. Espace restant après le premier bloc : à partir de `192.168.100.128`. Alloué : `192.168.100.128/26` (hôtes utilisables : `192.168.100.129` à `192.168.100.190`)
- **Réseau invités (20 machines)** : nécessite au moins 20 adresses utilisables → 2⁵-2=30 → préfixe `/27`. Espace restant à partir de `192.168.100.192`. Alloué : `192.168.100.192/27` (hôtes utilisables : `192.168.100.193` à `192.168.100.222`)
- **Liaison point-à-point (2 adresses utilisables)** : 2²-2=2 → préfixe `/30`. Espace restant à partir de `192.168.100.224`. Alloué : `192.168.100.224/30` (hôtes utilisables : `192.168.100.225` et `192.168.100.226`)

Vérification de non-chevauchement : `.0-.127` (utilisateurs), `.128-.191` (serveurs), `.192-.223` (invités), `.224-.227` (liaison) — aucun chevauchement, et l'ensemble reste bien dans le bloc de départ `192.168.100.0/24` (`.0` à `.255`), avec de la marge restante (`.228` à `.255`) pour une extension future.

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur réseau junior pour configurer l'adressage et le routage statique d'un petit laboratoire multi-segments, et pour valider vos calculs de sous-réseau avec des outils réels.

## Architecture

Une VM ou un ensemble de conteneurs disposant d'au moins deux interfaces réseau (ou de la possibilité d'en créer virtuellement, ex : interfaces `veth` sous Linux), permettant de simuler un routage entre deux segments distincts. À défaut d'environnement multi-interfaces, les étapes 1 et 2 (calcul et vérification) restent réalisables sur une machine unique.

## Objectifs
- Vérifier des calculs de sous-réseau manuels avec un outil ou un script dédié
- Observer et interpréter une table de routage réelle
- Configurer et tester une route statique
- Diagnostiquer un cas classique de masque de sous-réseau incorrect

## Préparation

```bash
which ip ipcalc traceroute 2>/dev/null
ip addr show
ip route show
```

## Étapes

### Étape 1 — Vérifier des calculs de sous-réseau

```bash
# Reprendre le script de la section 16 du cours (subnet_calc.sh)
bash subnet_calc.sh 192.168.10.77/27
bash subnet_calc.sh 172.16.5.130/26
bash subnet_calc.sh 10.0.0.50/28
```
*Question :* comparez les résultats du script avec vos calculs manuels du TD (exercices 2.1 et l'exemple de la section 7.3 du cours). Concordent-ils ?

### Étape 2 — Observer et interpréter la table de routage locale

```bash
ip route show
ip route get 8.8.8.8   # affiche la route qui sera réellement utilisée pour cette destination précise
```
*Question :* la route retournée par `ip route get` correspond-elle à la route par défaut (`0.0.0.0/0`), ou à une route plus spécifique ? Pourquoi, dans le cas général d'une machine standard reliée à Internet via une seule passerelle ?

### Étape 3 — Configurer une route statique de laboratoire

```bash
# Créer une route statique factice vers un réseau fictif (adapter la passerelle à votre environnement réel)
PASSERELLE=$(ip route show default | awk '{print $3}')
sudo ip route add 172.31.0.0/24 via "$PASSERELLE"

ip route show | grep 172.31
ip route get 172.31.0.5   # doit maintenant emprunter la nouvelle route statique

# Nettoyage
sudo ip route del 172.31.0.0/24
```
*Question :* si vous redémarriez la machine à cet instant (avant le nettoyage), cette route statique serait-elle encore présente après le redémarrage ? Justifiez en vous appuyant sur la note du cours (section 11).

### Étape 4 — Diagnostiquer un masque de sous-réseau incorrect

```bash
# Simuler l'erreur classique : configurer une adresse avec un masque /32 au lieu du masque réel du réseau
IP_ACTUELLE=$(ip -4 addr show scope global | grep -oP '(?<=inet\s)\d+(\.\d+){3}' | head -1)
echo "Adresse actuelle: $IP_ACTUELLE"

# NE PAS exécuter sur une interface de production sans précaution — utiliser une interface de laboratoire dédiée
# ip addr add ${IP_ACTUELLE}/32 dev <interface_lab>   # exemple pédagogique, à adapter/simuler selon l'environnement

echo "Avec un masque /32, cette machine se considérerait seule dans son propre sous-réseau."
bash subnet_calc.sh "${IP_ACTUELLE}/32"
```
*Question :* d'après le résultat du script, combien d'hôtes ce sous-réseau `/32` contient-il ? Expliquez pourquoi une machine ainsi configurée ne pourrait joindre aucune autre machine sans passer systématiquement par une route explicite (ce qui n'est normalement pas souhaité pour un réseau local).

## Commandes de référence
`ip addr`, `ip route show`, `ip route get`, `ip route add/del`, `traceroute`

## Résultats attendus
- Sortie du script `subnet_calc.sh` pour les 3 cas de l'étape 1, avec comparaison aux calculs manuels du TD
- Sortie de `ip route get 8.8.8.8` avec interprétation écrite (étape 2)
- Preuve de l'ajout et de la suppression de la route statique de l'étape 3
- Analyse écrite du cas `/32` de l'étape 4

## Questions

1. Pourquoi le calcul manuel de sous-réseau (méthode binaire vue en cours) reste-t-il une compétence indispensable pour un ingénieur réseau, alors que des outils comme `ipcalc` ou le script de la section 16 existent et automatisent ce calcul ?
2. Dans l'étape 3, si la route statique ajoutée avait pointé vers une passerelle inexistante ou injoignable (erreur de frappe), quel serait le comportement observé lors d'un test vers une destination empruntant cette route — un rejet immédiat, ou un timeout ? Justifiez en reliant votre réponse au module 7 (distinction refus explicite / silence).
3. En production, quelle méthode utiliseriez-vous pour rendre une route statique persistante après redémarrage, plutôt que de la reconfigurer manuellement à chaque reboot (réponse générale, sans nécessairement citer une syntaxe de fichier précise si elle dépend de la distribution) ?

## Troubleshooting (panne à diagnostiquer)

**Situation fournie :** un étudiant configure manuellement l'adresse IP d'une nouvelle machine de laboratoire en copiant-collant une configuration d'une autre machine du même réseau, mais oublie d'ajuster le dernier octet de l'adresse IP tout en gardant le même masque — deux machines se retrouvent avec exactement la **même adresse IP** sur le même réseau.

**Démarche attendue :**
1. Symptôme : comportement réseau erratique et intermittent pour les deux machines concernées (parfois l'une répond, parfois l'autre, ou aucune ne répond de façon fiable) — un symptôme différent d'un simple masque incorrect (vu à l'étape 4), à ne pas confondre
2. Hypothèse : conflit d'adresse IP (deux machines avec la même adresse sur le même segment)
3. Vérification : `arping -D -c 3 <IP_suspectée>` (mode détection de duplication) sur une troisième machine du même réseau, ou observation du cache ARP (`ip neigh`) montrant des changements erratiques d'association MAC pour cette IP unique — un symptôme qui peut *ressembler* superficiellement à de l'ARP spoofing (module 8) mais dont la cause est ici une simple erreur de configuration, pas une attaque
4. Isolation : identifier les deux machines physiques réellement configurées avec cette même adresse (par inspection directe de leur configuration réseau, `ip addr show` sur chacune)
5. Correction : réattribuer une adresse IP unique et cohérente à l'une des deux machines, dans la plage prévue par le plan d'adressage du sous-réseau
6. Vérification : confirmer la disparition du conflit via `arping -D` et un comportement réseau stable pour les deux machines
7. Documentation : noter ce cas comme rappel qu'un changement d'association ARP répété (déjà vu comme signal possible d'ARP spoofing au module 8) doit aussi être interprété, en environnement de laboratoire ou lors d'une reconfiguration manuelle, à la lumière d'une cause bien plus banale et fréquente : le conflit d'adresse IP

## Validation

Le TP est validé si l'étudiant fournit :
- Les résultats des 4 étapes
- Des réponses argumentées aux 3 questions
- Une identification correcte de la cause du scénario de troubleshooting (conflit d'IP, distingué explicitement d'une attaque ARP spoofing malgré la ressemblance des symptômes)

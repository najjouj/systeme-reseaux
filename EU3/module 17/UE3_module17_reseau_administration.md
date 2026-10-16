# UE3 — Module 17 : Réseau Côté Administration

---

# PARTIE I — COURS

## 1. Titre du cours
Administration réseau Linux : interfaces, routage et pare-feu (iptables/nftables)

## 2. Niveau
Élève ingénieur — cycle ingénieur, UE3 administration Linux (S2)

## 3. Durée
CM : 4h30 (module dense — +3h TD, +5h TP en Parties II et III)

## 4. Prérequis
- UE2-Module 9 : adressage IP, routage statique (théorie déjà posée)
- UE2-Module 8 : Ethernet, ARP (pour la configuration d'interface)
- UE3-Module 14 : permissions (opérations réseau généralement privilégiées)

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Configurer une interface réseau de façon persistante (au-delà de la configuration temporaire déjà vue en UE2)
- Appliquer et rendre persistante une route statique en environnement de production
- Expliquer l'architecture de `netfilter` (tables, chaînes) sous-jacente à `iptables` et `nftables`
- Écrire des règles de pare-feu de base : filtrage par adresse/port/protocole, politique par défaut
- Configurer une règle NAT simple (masquerade) pour un réseau interne
- Diagnostiquer un problème de connectivité lié à une règle de pare-feu mal configurée

## 6. Motivation

L'UE2 a donné toute la théorie du routage et des protocoles réseau ; ce module transforme cette théorie en compétence d'administration quotidienne sur un serveur Linux réel. Configurer une interface manuellement avec `ip addr add` (vu en UE2) fonctionne, mais disparaît au redémarrage — un serveur de production a besoin d'une configuration **persistante**. De même, un serveur exposé sans aucun pare-feu est une porte ouverte sur tous ses services, quelle que soit la rigueur de leur propre sécurité applicative — le pare-feu constitue la première ligne de défense au niveau système, complémentaire (jamais substitut) à la sécurité applicative.

Ce module est directement opérationnel : à l'issue, l'étudiant doit être capable de configurer et sécuriser l'accès réseau d'un serveur Linux réel de bout en bout.

## 7. Concepts fondamentaux

### 7.1 Configuration persistante d'interface réseau

La commande `ip addr add` (vue en UE2-M9) configure une interface de façon **temporaire**, perdue au redémarrage — exactement comme une route ajoutée avec `ip route add` sans persistance (rappel direct du même module). La configuration persistante dépend de l'outil de gestion réseau de la distribution :

- **Netplan** (Ubuntu récent) : fichiers YAML sous `/etc/netplan/`, généralement backés par `systemd-networkd` ou `NetworkManager`
- **NetworkManager** : outil courant sur de nombreuses distributions, gérable en ligne de commande via `nmcli`
- **systemd-networkd** : gestionnaire réseau natif de systemd, configuration par fichiers `.network`

**Ce cours ne privilégie pas un outil spécifique** (le choix dépend de la distribution utilisée en environnement de laboratoire) mais insiste sur le principe commun : toute configuration réseau de production doit survivre à un redémarrage, jamais reposer uniquement sur des commandes `ip` exécutées manuellement.

Exemple avec `systemd-networkd` :
```ini
# /etc/systemd/network/10-eth0.network
[Match]
Name=eth0

[Network]
Address=192.168.10.50/24
Gateway=192.168.10.1
DNS=192.168.10.1
```

### 7.2 Route statique persistante

Rappel direct d'UE2-M9 : `ip route add` ne survit pas à un redémarrage. La persistance dépend également de l'outil de gestion réseau utilisé — avec `systemd-networkd`, une route statique additionnelle se déclare dans le même fichier `.network` ou un fichier `.network` dédié :

```ini
[Route]
Destination=10.20.0.0/24
Gateway=192.168.10.254
```

### 7.3 Architecture netfilter — tables et chaînes

**`netfilter`** est le sous-système du noyau Linux responsable du filtrage de paquets, sur lequel s'appuient à la fois l'historique `iptables` et le plus récent `nftables` (destiné à terme à le remplacer, avec une syntaxe unifiée et de meilleures performances). Le modèle conceptuel repose sur des **tables**, elles-mêmes composées de **chaînes**, elles-mêmes composées de **règles** :

**Tables principales :**
- `filter` : filtrage de paquets (autoriser/bloquer) — la table par défaut et la plus utilisée
- `nat` : traduction d'adresses (rappel UE2-M9 sur le NAT, ici appliqué concrètement)
- `mangle` : modification de paquets (moins courant, hors périmètre de ce module)

**Chaînes principales de la table `filter` :**
- `INPUT` : paquets destinés à la machine locale elle-même
- `OUTPUT` : paquets émis par la machine locale elle-même
- `FORWARD` : paquets traversant la machine (cas d'un routeur/pare-feu faisant transiter du trafic entre deux réseaux, sans être lui-même la source ou la destination finale)

**Point clé pédagogique :** cette distinction INPUT/OUTPUT/FORWARD reflète directement le rôle joué par la machine dans un échange donné — un serveur web filtre son trafic entrant via `INPUT`, tandis qu'un routeur Linux faisant transiter du trafic entre deux réseaux internes utilise `FORWARD`, une distinction fréquemment source de confusion chez les débutants qui tentent d'appliquer une règle `INPUT` à du trafic qui, en réalité, ne fait que traverser la machine.

### 7.4 Politique par défaut et ordre d'évaluation des règles

Chaque chaîne a une **politique par défaut** (`ACCEPT` ou `DROP`), appliquée si aucune règle explicite ne correspond au paquet. Les règles d'une chaîne sont évaluées **dans l'ordre**, et la première règle qui correspond détermine le sort du paquet (généralement, sauf mécanismes avancés hors périmètre) — l'ordre des règles est donc fondamental, une règle générale placée avant une règle spécifique peut rendre cette dernière totalement inopérante.

**Principe de sécurité recommandé :** politique par défaut `DROP` sur `INPUT` (tout refuser sauf ce qui est explicitement autorisé), plutôt que `ACCEPT` par défaut avec des règles de blocage ponctuelles — une approche "liste blanche" nettement plus sûre qu'une approche "liste noire", cohérente avec le principe du moindre privilège déjà rencontré à plusieurs reprises dans le programme.

### 7.5 `iptables` — syntaxe de base

```bash
# Politique par défaut restrictive
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP

# Autoriser le trafic déjà établi (connexions initiées légitimement, rappel UE2-M10 sur les états TCP)
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Autoriser le loopback (communication interne à la machine elle-même)
sudo iptables -A INPUT -i lo -j ACCEPT

# Autoriser SSH (port 22) et HTTP (port 80) entrants
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT

# Visualiser les règles actives
sudo iptables -L -v -n
```

**Le filtrage par état (`-m state --state ESTABLISHED,RELATED`)** est une application directe de la compréhension des états TCP vue en UE2-M10 : un pare-feu à états ("stateful") suit le cycle de vie de chaque connexion et autorise automatiquement le trafic de retour d'une connexion légitimement établie, sans nécessiter une règle explicite symétrique pour chaque direction.

### 7.6 `nftables` — syntaxe moderne équivalente

```bash
sudo nft add table inet filter
sudo nft add chain inet filter input { type filter hook input priority 0 \; policy drop \; }
sudo nft add rule inet filter input ct state established,related accept
sudo nft add rule inet filter input iif lo accept
sudo nft add rule inet filter input tcp dport 22 accept
sudo nft add rule inet filter input tcp dport 80 accept

sudo nft list ruleset
```

**Différences notables avec `iptables` :** syntaxe unifiée pour IPv4/IPv6 (`inet`, contrairement à `iptables`/`ip6tables` historiquement séparés), meilleure performance sur de grands ensembles de règles, et une structure de règles plus proche d'un langage déclaratif lisible.

### 7.7 NAT — masquerade pour un réseau interne

Rappel direct d'UE2-M9 (adresses privées RFC 1918) : une machine Linux peut agir comme passerelle NAT pour un réseau interne, traduisant les adresses privées internes en une adresse publique unique pour sortir vers Internet :

```bash
# Activer le transfert de paquets au niveau noyau (indispensable pour tout rôle de routeur)
sudo sysctl -w net.ipv4.ip_forward=1

# Règle de masquerade (NAT dynamique) sur l'interface de sortie
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

**`ip_forward` — un prérequis souvent oublié :** sans cette activation explicite au niveau noyau, une machine Linux ne transmettra **jamais** de paquets entre ses interfaces, quelle que soit la justesse des règles de pare-feu et de routage configurées par ailleurs — un oubli fréquent lors de la mise en place d'un premier routeur/passerelle Linux.

### 7.8 Persistance des règles de pare-feu

Comme pour les interfaces et les routes, les règles `iptables`/`nftables` appliquées en ligne de commande sont **volatiles**, perdues au redémarrage sauf sauvegarde et restauration explicites (`iptables-save`/`iptables-restore`, ou un fichier de règles chargé automatiquement au démarrage via un service dédié selon la distribution) — un rappel systématique de la même leçon rencontrée pour chaque brique de configuration réseau de ce module.

## 8. Architecture

```
┌────────────────────────────────────────────────────────────┐
│                        Machine Linux                             │
│                                                                   │
│   Paquet entrant ──▶ [PREROUTING] ──▶ destiné à cette machine ?     │
│                                          │                          │
│                              OUI ────────┼──── NON                  │
│                               ▼                    ▼                 │
│                          [INPUT]              [FORWARD]                │
│                               │                    │                 │
│                        Processus local      [POSTROUTING] (NAT)        │
│                               │                    │                 │
│                          [OUTPUT] ──────────────────┴──▶ Paquet sortant │
└────────────────────────────────────────────────────────────┘
```

## 9. Fonctionnement interne

`netfilter` s'intègre directement dans la pile réseau du noyau Linux via des points d'ancrage (hooks) à des étapes précises du traitement d'un paquet (avant routage, après routage, destiné localement...) — c'est cette intégration bas niveau qui permet un filtrage extrêmement performant, bien avant que le paquet n'atteigne l'espace utilisateur ou même la couche socket vue en UE1-M5. `nftables` utilise une machine virtuelle interne au noyau (BPF-like) pour évaluer les règles de façon plus efficace que le modèle séquentiel plus rigide d'`iptables`, expliquant en partie l'écart de performance sur de grands ensembles de règles.

## 10. Démonstration

```bash
# Observer les interfaces et leur configuration actuelle
ip addr show
ip route show

# Observer les règles de pare-feu actives
sudo iptables -L -v -n --line-numbers

# Observer si le transfert de paquets est activé
sysctl net.ipv4.ip_forward
```

## 11. Commandes / outils

| Commande | Rôle | Exemple |
|---|---|---|
| `nmcli` | Gérer la configuration réseau via NetworkManager | `nmcli con show` |
| `iptables -L -v -n` | Lister les règles actives avec compteurs | `sudo iptables -L -v -n` |
| `nft list ruleset` | Lister les règles nftables actives | `sudo nft list ruleset` |
| `iptables-save` / `iptables-restore` | Sauvegarder/restaurer un jeu de règles | `sudo iptables-save > /etc/iptables/rules.v4` |
| `sysctl net.ipv4.ip_forward` | Consulter/activer le transfert de paquets | `sudo sysctl -w net.ipv4.ip_forward=1` |
| `ss -tuln` | Vérifier quels services écoutent réellement (rappel UE2-M11) | `ss -tuln` |

**Erreur fréquente :** appliquer une politique par défaut `DROP` sur `INPUT` **avant** d'avoir ajouté la règle autorisant sa propre connexion SSH en cours — une erreur classique et redoutée qui coupe immédiatement l'accès administrateur à distance à la machine, nécessitant un accès console physique ou une console de secours (rappel indirect du mode de secours vu en UE1-M6) pour corriger la situation.

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** après le déploiement d'un nouveau pare-feu sur un serveur applicatif, une équipe constate que les utilisateurs peuvent initier des connexions vers le service (le port applicatif est bien accessible), mais que certaines fonctionnalités de l'application, qui reposent sur des connexions sortantes initiées par le serveur lui-même vers une API tierce, échouent systématiquement.

**Analyse attendue :** ce symptôme oriente vers une chaîne `OUTPUT` trop restrictive — la politique par défaut a probablement été positionnée à `DROP` sur `OUTPUT` (en plus d'`INPUT`, une pratique de durcissement plus poussée mais qui nécessite une attention particulière), sans règle explicite autorisant le trafic sortant nécessaire à cette intégration avec l'API tierce. Contrairement à `INPUT` (dont l'oubli bloque immédiatement et visiblement tout accès entrant, vite remarqué), une chaîne `OUTPUT` trop restrictive peut passer inaperçue plus longtemps, seules certaines fonctionnalités précises étant affectées plutôt que l'ensemble du service. Diagnostic : `iptables -L OUTPUT -v -n` pour examiner la politique et les règles existantes, en cherchant l'absence d'une règle autorisant le trafic sortant vers le port/l'adresse de l'API tierce concernée ; correction : ajouter la règle manquante, en s'assurant par ailleurs (rappel section 7.5) que le trafic de retour associé aux connexions sortantes légitimes reste également autorisé sur `INPUT` via le filtrage par état.

## 15. Sécurité

- Le pare-feu système est une **couche de défense complémentaire**, jamais un substitut à la sécurisation applicative elle-même (rappel du principe de défense en profondeur, déjà rencontré en UE1)
- Une politique par défaut `DROP` (liste blanche) est structurellement plus sûre qu'une politique `ACCEPT` avec blocages ponctuels (liste noire) — un nouveau service accidentellement exposé par erreur de configuration applicative reste protégé par défaut dans le premier modèle, exposé sans protection dans le second
- Le filtrage par état (`ESTABLISHED,RELATED`) est indispensable à la fois pour la sécurité (ne pas avoir à ouvrir de larges plages de ports en retour) et pour l'usabilité (le trafic légitime de retour ne doit pas être bloqué par erreur)
- Toute règle de pare-feu ajoutée en urgence pour résoudre un incident doit être documentée et réévaluée — une règle temporaire oubliée devient une faille de sécurité permanente non intentionnelle, un schéma de dérive déjà illustré à plusieurs reprises dans le programme (ex : le `chmod 777` "temporaire" du module 14)

## 16. Automatisation

Script Bash de vérification de cohérence entre les règles de pare-feu actives et une configuration de référence attendue, avant et après tout redémarrage.

```bash
#!/usr/bin/env bash
# Compare les règles de pare-feu actives à une référence sauvegardée, alerte sur tout écart
set -euo pipefail

REFERENCE="/etc/security/firewall_reference.rules"
LOGFILE="/var/log/firewall_audit.log"

if [[ ! -f "$REFERENCE" ]]; then
    echo "Génération de la référence initiale (à valider manuellement)"
    sudo iptables-save > "$REFERENCE"
    exit 0
fi

ACTUEL=$(sudo iptables-save)
REFERENCE_CONTENU=$(cat "$REFERENCE")

if [[ "$ACTUEL" != "$REFERENCE_CONTENU" ]]; then
    echo "$(date '+%F %T') - ALERTE: divergence détectée entre les règles actives et la référence" \
        | tee -a "$LOGFILE"
    diff <(echo "$REFERENCE_CONTENU") <(echo "$ACTUEL") | tee -a "$LOGFILE"
    exit 1
fi

echo "$(date '+%F %T') - OK: règles conformes à la référence" >> "$LOGFILE"
exit 0
```

## 17. Troubleshooting

Scénario court : après application d'une nouvelle politique de pare-feu, un administrateur perd immédiatement sa connexion SSH à un serveur distant, sans accès console physique disponible dans l'immédiat. Démarche préventive (à appliquer **avant** tout changement de politique par défaut en production) : toujours tester une nouvelle politique de pare-feu via un mécanisme de rollback automatique programmé (ex : une commande `at` planifiée quelques minutes dans le futur pour restaurer une configuration de secours si aucune confirmation manuelle n'intervient entretemps), ou dans un environnement de test identique avant application en production — une règle d'or de l'administration réseau à distance, qui aurait évité l'incident plutôt que d'avoir à le résoudre après coup sans accès.

## 18. Synthèse

- La configuration réseau (interfaces, routes, règles de pare-feu) appliquée en ligne de commande est systématiquement volatile — la persistance dépend de l'outil de gestion réseau de la distribution et doit toujours être vérifiée
- `netfilter` structure le filtrage en tables (filter, nat...) et chaînes (INPUT, OUTPUT, FORWARD), reflétant le rôle joué par la machine dans chaque échange
- Une politique par défaut `DROP` (liste blanche) est structurellement plus sûre qu'`ACCEPT` avec blocages ponctuels
- Le filtrage par état (`ESTABLISHED,RELATED`) applique directement la compréhension des connexions TCP de l'UE2 à la configuration pratique du pare-feu
- `ip_forward` doit être explicitement activé au niveau noyau pour tout rôle de routeur/passerelle NAT, indépendamment des règles de pare-feu elles-mêmes
- Toute modification de politique de pare-feu à distance comporte un risque de coupure d'accès — un mécanisme de rollback automatique est une précaution professionnelle de base

## 19. Questions d'évaluation

**QCM**
1. La chaîne `netfilter` appropriée pour filtrer du trafic destiné à la machine locale elle-même est :
 a) FORWARD b) INPUT c) OUTPUT d) POSTROUTING

2. Une politique par défaut `DROP` sur `INPUT`, plutôt qu'`ACCEPT`, correspond à une approche :
 a) liste noire b) liste blanche, structurellement plus sûre c) sans aucun effet réel d) réservée à IPv6 uniquement

**Questions ouvertes**
- Expliquez pourquoi `ip_forward` doit être activé explicitement pour qu'une machine Linux joue le rôle de routeur, indépendamment des règles de pare-feu.
- Expliquez le risque associé à l'application d'une politique par défaut `DROP` sur `INPUT` sans avoir au préalable autorisé sa propre connexion d'administration distante.

**Exercice technique**
- Donnez la commande `iptables` permettant d'autoriser le trafic entrant déjà établi ou lié à une connexion existante (filtrage par état).

**Problème de diagnostic**
- Un service accepte les connexions entrantes normalement, mais certaines de ses fonctionnalités reposant sur des connexions sortantes échouent. Proposez la cause la plus probable et la commande de vérification.

## 20. Mini-projet

Développer un script (Bash ou Python) qui :
1. Sauvegarde automatiquement l'état actuel des règles de pare-feu avant toute modification planifiée (horodaté)
2. Applique une nouvelle politique de pare-feu à partir d'un fichier de règles fourni
3. Programme un mécanisme de rollback automatique (via `at` ou une minuterie applicative) qui restaure la sauvegarde précédente si aucune confirmation explicite n'est reçue dans un délai configurable (ex : 5 minutes)
4. Journalise chaque application de règles et chaque rollback (déclenché ou annulé) avec horodatage
5. Fournit une commande de confirmation simple que l'administrateur doit exécuter pour annuler le rollback programmé une fois la nouvelle configuration validée comme fonctionnelle

---

# PARTIE II — TD

### Niveau 1 — Compréhension

**Exercice 1.1**
Expliquez pourquoi une configuration réseau appliquée avec `ip addr add` ou `ip route add` ne survit pas à un redémarrage, et ce qui est nécessaire pour la rendre persistante.

**Exercice 1.2**
Décrivez les trois chaînes principales de la table `filter` (INPUT, OUTPUT, FORWARD) et le type de trafic concerné par chacune.

**Exercice 1.3**
Expliquez la différence entre une politique de pare-feu de type liste blanche et liste noire, et laquelle est recommandée en cours.

### Niveau 2 — Application

**Exercice 2.1**
Donnez la commande `iptables` permettant d'autoriser le trafic entrant sur le port TCP 443.

**Exercice 2.2**
Donnez la commande permettant d'afficher les règles de pare-feu actives avec leurs compteurs de paquets.

**Exercice 2.3**
Donnez la commande permettant d'activer le transfert de paquets IPv4 au niveau noyau.

### Niveau 3 — Analyse

**Exercice 3.1**
Un administrateur configure une politique `DROP` par défaut sur `INPUT`, ajoute une règle autorisant le port SSH (22), puis applique la configuration à distance via une connexion SSH déjà établie. Sa session SSH reste-t-elle active après l'application ? Justifiez en vous appuyant sur le filtrage par état vu en cours.

**Exercice 3.2**
Un serveur configuré comme passerelle NAT pour un réseau interne ne transmet aucun trafic entre les deux réseaux, malgré des règles de routage et de pare-feu apparemment correctes. Quelle vérification simple, souvent oubliée, faudrait-il effectuer en priorité ?

### Niveau 4 — Problème complexe

**Exercice 4.1**
Une équipe déploie un nouveau pare-feu sur un serveur applicatif avec une politique `DROP` par défaut sur `OUTPUT`, en plus d'`INPUT`. Les connexions entrantes fonctionnent normalement. Cependant, quelques jours plus tard, l'équipe reçoit des alertes indiquant qu'un composant de supervision installé sur le serveur ne parvient plus à envoyer ses métriques vers un serveur de collecte externe, alors que ce composant fonctionnait normalement avant le déploiement du pare-feu. En vous appuyant sur les mécanismes de chaînes vus en cours, expliquez pourquoi ce type de problème peut rester inaperçu plus longtemps qu'un problème équivalent sur `INPUT`, et proposez une démarche de diagnostic et de correction.

---

## Correction du TD

**1.1** — Ces commandes modifient directement l'état du noyau en mémoire (table de routage, configuration d'interface), sans jamais écrire cette configuration dans un fichier persistant lu au démarrage. Au redémarrage, le noyau reconstruit son état réseau uniquement à partir des fichiers de configuration gérés par l'outil de gestion réseau de la distribution (Netplan, NetworkManager, systemd-networkd...) — toute modification faite uniquement en ligne de commande, sans être répercutée dans ces fichiers de configuration persistants, est donc perdue. Pour la rendre persistante, il faut déclarer la configuration dans le fichier approprié à l'outil utilisé par la distribution.

**1.2** — `INPUT` concerne le trafic destiné à la machine locale elle-même (ex : une connexion SSH vers ce serveur). `OUTPUT` concerne le trafic émis par la machine locale elle-même (ex : une requête sortante initiée par une application locale). `FORWARD` concerne le trafic qui ne fait que traverser la machine, sans en être ni la source ni la destination finale (cas d'un routeur faisant transiter du trafic entre deux réseaux).

**1.3** — Une politique liste noire part d'une politique par défaut `ACCEPT` et bloque explicitement ce qui est jugé indésirable ; une politique liste blanche part d'une politique par défaut `DROP` et autorise explicitement uniquement ce qui est nécessaire. Le cours recommande la liste blanche, car elle protège automatiquement contre tout service exposé par erreur ou par une future modification de configuration non anticipée — un nouveau risque non prévu reste bloqué par défaut, contrairement à une liste noire où un nouveau risque non explicitement bloqué serait automatiquement autorisé.

**2.1** — `sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT`.

**2.2** — `sudo iptables -L -v -n`.

**2.3** — `sudo sysctl -w net.ipv4.ip_forward=1`.

**3.1** — Oui, sa session SSH reste active. Bien que la politique par défaut passe à `DROP` et qu'une règle explicite autorisant le port 22 en entrée ait été ajoutée, ce n'est pas cette règle spécifique au port 22 qui maintient la session existante active en premier lieu, mais le mécanisme de filtrage par état (`ESTABLISHED,RELATED`, généralement configuré en priorité dans une politique de pare-feu bien conçue, rappel section 7.5) : la connexion SSH déjà établie est reconnue comme faisant partie d'un état de connexion existant et légitime, et continue donc d'être acceptée par cette règle, indépendamment de la nouvelle politique par défaut appliquée aux **nouvelles** connexions. Sans une règle de filtrage par état correctement positionnée avant l'application de la nouvelle politique, la coupure aurait effectivement pu se produire.

**3.2** — La vérification à effectuer en priorité est celle du paramètre noyau `net.ipv4.ip_forward` (rappel section 7.7) : sans son activation explicite (`sysctl net.ipv4.ip_forward`, qui doit retourner `1`), une machine Linux ne transmettra jamais de paquets entre ses interfaces, quelle que soit la justesse par ailleurs des règles de routage et de pare-feu configurées — un oubli très fréquent et facile à vérifier avant d'investiguer des causes plus complexes.

**4.1** — Ce type de problème peut rester inaperçu plus longtemps car une chaîne `OUTPUT` trop restrictive n'affecte que les fonctionnalités **spécifiques** qui dépendent de connexions sortantes initiées par le serveur lui-même — contrairement à une chaîne `INPUT` trop restrictive, qui bloque immédiatement et de façon très visible **tout** accès entrant au service principal (remarqué en quelques minutes par les utilisateurs ou les systèmes de supervision externes), une restriction sur `OUTPUT` peut n'affecter qu'un composant secondaire (ici, l'envoi de métriques de supervision) dont l'absence de fonctionnement n'est détectée qu'indirectement, après un délai, via l'absence de données reçues côté collecteur externe — un symptôme moins immédiat et moins directement associé par les équipes au déploiement du pare-feu survenu quelques jours plus tôt. Démarche de diagnostic : `iptables -L OUTPUT -v -n` pour examiner la politique et les règles existantes sur cette chaîne, en cherchant l'absence d'une règle autorisant explicitement le trafic sortant vers l'adresse/le port du serveur de collecte de métriques. Correction : ajouter la règle manquante autorisant ce trafic sortant spécifique, en veillant également (rappel section 7.5) à ce que le trafic de retour associé reste correctement autorisé côté `INPUT` via le filtrage par état déjà en place.

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur système junior pour sécuriser l'accès réseau d'un serveur applicatif avec une politique de pare-feu restrictive, en évitant le piège classique de coupure d'accès SSH, puis pour configurer un rôle de passerelle NAT basique.

## Architecture

Une VM Linux avec accès SSH (pour observer concrètement le risque de coupure), `iptables` (ou `nftables`) disponible, privilèges root.

**Avertissement de sécurité pour ce TP :** certaines étapes manipulent directement les règles de pare-feu d'accès SSH. Travaillez impérativement sur une VM de laboratoire jetable/réinitialisable, jamais sur un système de production ou un accès SSH que vous ne pouvez pas vous permettre de perdre.

## Objectifs
- Appliquer une politique de pare-feu restrictive sans perdre l'accès SSH
- Observer concrètement le filtrage par état
- Configurer et vérifier un mécanisme de rollback automatique de sécurité
- Configurer une règle NAT basique et vérifier l'activation d'`ip_forward`

## Préparation

```bash
which iptables nft sysctl at 2>/dev/null
sudo iptables -L -v -n   # état initial, probablement vide ou politique ACCEPT par défaut
```

## Étapes

### Étape 1 — Politique restrictive sécurisée (ordre correct)

```bash
# ORDRE IMPORTANT : autoriser SSH et l'état AVANT de changer la politique par défaut
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Seulement maintenant, changer la politique par défaut
sudo iptables -P INPUT DROP

sudo iptables -L -v -n --line-numbers
```
*Question :* que se serait-il passé si vous aviez inversé l'ordre (politique `DROP` appliquée avant la règle SSH) et que vous étiez connecté à distance ? Reliez votre réponse à l'exercice 3.1 du TD.

### Étape 2 — Mécanisme de rollback automatique (sécurité professionnelle)

```bash
sudo iptables-save > /tmp/firewall_avant_test.rules

# Programmer un rollback automatique dans 3 minutes (accéléré pour le TP,
# une valeur réaliste en production serait plutôt de 5 à 10 minutes)
echo "iptables-restore < /tmp/firewall_avant_test.rules" | sudo at now + 3 minutes

atq   # vérifier que la tâche de rollback est bien programmée

# Simuler une modification risquée
sudo iptables -A INPUT -p tcp --dport 22 -j DROP   # règle qui bloquerait SSH en priorité
sudo iptables -L -v -n --line-numbers
```
*Question :* si vous ne faites rien, que va-t-il se passer dans 3 minutes ? Quelle commande auriez-vous dû exécuter immédiatement si cette configuration bloquante avait été volontaire et validée, pour annuler le rollback programmé ?

```bash
# Annuler le rollback programmé (à exécuter avant les 3 minutes pour ce TP,
# afin de poursuivre sereinement vers l'étape 3)
atrm $(atq | awk '{print $1}')
sudo iptables-restore < /tmp/firewall_avant_test.rules
sudo iptables -L -v -n --line-numbers
```

### Étape 3 — NAT et `ip_forward`

```bash
sysctl net.ipv4.ip_forward   # observer l'état initial, probablement 0

sudo sysctl -w net.ipv4.ip_forward=1
sysctl net.ipv4.ip_forward

# Règle de masquerade (adapter le nom d'interface selon votre environnement)
INTERFACE_SORTIE=$(ip route show default | awk '{print $5}')
sudo iptables -t nat -A POSTROUTING -o "$INTERFACE_SORTIE" -j MASQUERADE

sudo iptables -t nat -L -v -n
```
*Question :* pourquoi cette configuration seule (NAT + `ip_forward`) ne suffit-elle pas nécessairement à faire fonctionner un vrai routage inter-réseaux en conditions réelles ? Quel élément supplémentaire, vu en cours (section 7.3 sur les chaînes), faudrait-il vérifier ?

### Étape 4 — Script d'audit de cohérence

Reprendre et tester le script de la section 16 du cours (`firewall_audit.sh`) :

```bash
bash firewall_audit.sh   # première exécution, établit la référence
sudo iptables -A INPUT -p tcp --dport 8080 -j ACCEPT   # modification non documentée
bash firewall_audit.sh   # seconde exécution, doit détecter l'écart
```

## Commandes de référence
`iptables -A`, `iptables -P`, `iptables -L -v -n`, `iptables-save`/`iptables-restore`, `sysctl net.ipv4.ip_forward`, `at`, `atq`, `atrm`

## Résultats attendus
- Règles finales de l'étape 1 avec politique `DROP` et accès SSH préservé
- Preuve du mécanisme de rollback programmé (`atq`) et de son annulation contrôlée (étape 2)
- Confirmation d'activation d'`ip_forward` et règle NAT active (étape 3)
- Détection de la divergence par le script d'audit (étape 4)

## Questions

1. Dans l'étape 1, pourquoi la règle autorisant le loopback (`-i lo`) est-elle nécessaire même sur un serveur qui n'est théoriquement accédé que depuis l'extérieur ? Quels types de communications internes à la machine elle-même pourraient être affectés sans cette règle ?
2. Le mécanisme de rollback de l'étape 2 repose sur la commande `at`. Quel serait, selon vous, le risque si cette tâche planifiée elle-même échouait silencieusement (ex : le service `atd` non démarré) au moment critique ? Comment un ingénieur prudent pourrait-il se prémunir contre ce risque secondaire ?
3. Pourquoi la vérification de l'étape 3 (routage inter-réseaux réel) est-elle difficile à tester complètement dans un environnement de laboratoire à une seule VM, et quel type d'environnement serait nécessaire pour une validation complète ?

## Troubleshooting (panne à diagnostiquer)

**Situation fournie :** un étudiant applique une politique `DROP` par défaut sur `OUTPUT` (en plus d'`INPUT`) sur sa VM de laboratoire, dans le cadre d'un exercice de durcissement plus poussé. Immédiatement après, ses commandes `apt update` (ou équivalent gestionnaire de paquets) échouent systématiquement, alors que sa connexion SSH entrante reste parfaitement fonctionnelle.

**Démarche attendue :**
1. Symptôme : connexions sortantes bloquées (gestionnaire de paquets), connexions entrantes fonctionnelles (SSH)
2. Hypothèse : cohérente avec le TD (exercice 4.1) — une politique `DROP` sur `OUTPUT` sans règles explicites d'autorisation bloque toute nouvelle connexion sortante initiée par la machine elle-même, y compris celles nécessaires au gestionnaire de paquets pour contacter les dépôts distants, alors que la connexion SSH entrante continue de fonctionner car elle a été établie **avant** le changement de politique et bénéficie du filtrage par état (si une règle `ESTABLISHED,RELATED` a bien été positionnée sur `OUTPUT` également, ce qui doit être vérifié)
3. Vérification : `iptables -L OUTPUT -v -n` pour confirmer la politique `DROP` et l'absence de règle autorisant le trafic sortant nécessaire (DNS, HTTP/HTTPS vers les dépôts)
4. Isolation : confirmer que le blocage concerne bien spécifiquement les nouvelles connexions sortantes (test avec une commande générant une connexion sortante simple, comme `curl` vers une adresse externe, qui devrait échouer de la même façon)
5. Correction : ajouter les règles `OUTPUT` nécessaires (a minima : filtrage par état pour le retour des connexions entrantes déjà établies, autorisation explicite du trafic sortant DNS/HTTP/HTTPS nécessaire aux opérations courantes du système), en gardant à l'esprit qu'une politique `OUTPUT` restrictive complète nécessite d'anticiper **tous** les besoins sortants légitimes de la machine, une démarche plus exigeante et plus risquée d'omissions qu'une politique `INPUT` seule
6. Vérification : relancer `apt update` (ou équivalent) après correction, confirmer le rétablissement du fonctionnement normal
7. Documentation : noter cette expérience comme illustration directe et personnellement vécue de l'exercice 4.1 du TD — un problème sur `OUTPUT` est structurellement plus facile à mal anticiper qu'un problème sur `INPUT`, car il exige de lister exhaustivement tous les besoins de communication sortante légitimes de la machine, une liste souvent incomplète à la première tentative

## Validation

Le TP est validé si l'étudiant fournit :
- Les 4 résultats/captures demandés
- Des réponses argumentées aux 3 questions
- Une identification correcte de la cause du scénario de troubleshooting (politique OUTPUT trop restrictive), avec une correction cohérente et une réflexion sur la difficulté d'anticiver exhaustivement les besoins sortants

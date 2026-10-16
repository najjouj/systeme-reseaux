# UE2 — Module 12 : DNS et DHCP

---

# PARTIE I — COURS

## 1. Titre du cours
DNS et DHCP : résolution de noms et attribution dynamique d'adresses

## 2. Niveau
Élève ingénieur — cycle ingénieur, UE2 réseaux (S1)

## 3. Durée
CM : 4h30 (module double thématique — +3h TD, +4h TP en Parties II et III)

## 4. Prérequis
- UE2-Module 9 : adressage IPv4, notion de passerelle
- UE2-Module 11 : UDP (DNS et DHCP reposent tous deux principalement sur UDP)

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Décrire la hiérarchie DNS (racine, TLD, domaine, sous-domaine) et le rôle de chaque niveau
- Différencier résolution récursive et itérative, et expliquer le rôle d'un résolveur récursif
- Citer les types d'enregistrements DNS courants et leur usage (A, AAAA, CNAME, MX, NS, PTR, TXT)
- Expliquer le rôle du TTL et du cache DNS, et leurs implications en cas de changement de configuration
- Décrire le processus DHCP en 4 étapes (DORA) et le rôle d'un bail (lease)
- Diagnostiquer un problème de résolution DNS ou d'attribution DHCP avec les outils Linux

## 6. Motivation

Personne ne mémorise une adresse IP pour accéder à un service — on tape un nom de domaine, et DNS se charge de la traduction. De même, personne ne configure manuellement l'adresse IP de chaque nouvel appareil qui rejoint un réseau — DHCP l'attribue automatiquement. Ces deux services, souvent invisibles tant qu'ils fonctionnent, sont pourtant des points de défaillance critiques : un DNS en panne rend un site "inaccessible" même si le serveur applicatif fonctionne parfaitement ; un serveur DHCP en panne empêche tout nouvel appareil de rejoindre le réseau, même si l'infrastructure réseau elle-même est saine.

Ce module donne les bases indispensables pour diagnostiquer ces deux services au quotidien — une part très importante des tickets d'incident réseau en environnement professionnel commence par "je n'arrive pas à me connecter" ou "le site ne se charge pas", et se résout in fine par un problème DNS ou DHCP mal diagnostiqué au départ.

## 7. Concepts fondamentaux

### 7.1 Hiérarchie DNS

DNS (Domain Name System) est organisé en une hiérarchie arborescente inversée :

```
                    . (racine)
                    │
      ┌─────────────┼─────────────┐
     .com           .org          .fr
      │
  example.com
      │
  www.example.com  (sous-domaine)
```

- **Racine** (`.`, implicite) : gérée par un ensemble de serveurs racine faisant autorité sur l'existence des TLD
- **TLD (Top-Level Domain)** : `.com`, `.org`, `.fr`... gérés par des registres délégués
- **Domaine** : `example.com`, enregistré par une organisation, avec ses propres serveurs faisant autorité
- **Sous-domaine** : `www.example.com`, `mail.example.com`, définis librement par le propriétaire du domaine

Chaque niveau **délègue** l'autorité du niveau suivant, un mécanisme distribué qui permet à DNS de fonctionner à l'échelle mondiale sans autorité centrale unique gérant chaque nom individuellement.

### 7.2 Résolution récursive vs itérative

Quand une application demande la résolution d'un nom, elle s'adresse généralement à un **résolveur récursif** (souvent fourni par le FAI, l'entreprise, ou un service public comme 8.8.8.8/1.1.1.1) :

```
Client → Résolveur récursif → Serveur racine → Serveur TLD (.com) → Serveur faisant autorité (example.com) → Réponse
```

- **Résolution récursive** (vue par le client) : le client pose **une seule question** au résolveur, qui se charge de faire tout le travail nécessaire (potentiellement plusieurs requêtes en chaîne) et retourne la réponse finale complète — le client n'a jamais besoin de contacter directement la racine, un TLD, ou le serveur faisant autorité.
- **Résolution itérative** (vue par le résolveur) : le résolveur interroge successivement la racine (qui indique quel serveur TLD contacter), puis le TLD (qui indique quel serveur fait autorité pour le domaine), puis ce serveur faisant autorité (qui fournit enfin la réponse) — chaque serveur interrogé renvoie soit la réponse finale, soit une **référence** vers le serveur suivant à contacter, sans jamais faire ce travail à la place du résolveur.

**Point clé pédagogique :** le client fait toujours une requête récursive à son résolveur configuré ; c'est le résolveur qui effectue ensuite, de son côté, une série de requêtes itératives vers la hiérarchie DNS — les deux modes coexistent à des points différents de la chaîne, une confusion fréquente chez les débutants.

### 7.3 Types d'enregistrements DNS

| Type | Rôle | Exemple |
|---|---|---|
| A | Associe un nom à une adresse IPv4 | `example.com → 93.184.216.34` |
| AAAA | Associe un nom à une adresse IPv6 | `example.com → 2606:2800:220:1::` |
| CNAME | Alias : un nom pointe vers un autre nom (pas directement une IP) | `www.example.com → example.com` |
| MX | Indique le(s) serveur(s) de messagerie d'un domaine, avec une priorité | `example.com → 10 mail.example.com` |
| NS | Indique quel(s) serveur(s) font autorité pour un domaine | `example.com → ns1.example.com` |
| PTR | Résolution inverse : adresse IP vers nom (utilisée notamment pour la vérification anti-spam) | `34.216.184.93.in-addr.arpa → example.com` |
| TXT | Enregistrement texte libre, souvent utilisé pour la validation de domaine ou des politiques (SPF, DKIM) | `example.com → "v=spf1 ..."` |

**Erreur fréquente :** confondre un enregistrement A (résolution directe nom→IP) avec un CNAME (alias nom→nom) — un CNAME ne peut jamais coexister avec un autre enregistrement pour le même nom exact (une règle stricte du protocole DNS, source d'erreurs de configuration classiques).

### 7.4 Cache et TTL

Chaque enregistrement DNS porte un **TTL (Time To Live)**, indiquant combien de temps un résolveur est autorisé à conserver la réponse en cache avant de devoir la revérifier auprès du serveur faisant autorité. Ce cache réduit considérablement la charge sur les serveurs DNS et accélère les résolutions répétées — mais introduit un **délai de propagation** lors de tout changement : un enregistrement modifié ne sera visible par tous les résolveurs qu'après expiration du TTL précédemment mis en cache par chacun d'eux, potentiellement des heures selon la valeur configurée.

**Implication pratique en production :** avant une migration planifiée (changement d'adresse IP d'un service), une bonne pratique consiste à **réduire le TTL à l'avance** (ex : de 24h à 5 minutes, plusieurs heures avant la migration), pour que le changement effectif se propage rapidement une fois appliqué, plutôt que de découvrir après coup que certains utilisateurs continuent à joindre l'ancienne adresse pendant des heures.

### 7.5 DHCP — le processus DORA

**DHCP (Dynamic Host Configuration Protocol)** attribue automatiquement une configuration réseau (adresse IP, masque, passerelle, serveurs DNS...) à une machine qui rejoint un réseau, via un échange en quatre étapes connu sous l'acronyme **DORA** :

```
Client                                    Serveur DHCP
  │                                            │
  │──────── DHCPDISCOVER (broadcast) ────────▶│   1. "Y a-t-il un serveur DHCP disponible ?"
  │                                            │
  │◀─────── DHCPOFFER ────────────────────────│   2. "Voici une proposition d'adresse et de config"
  │                                            │
  │──────── DHCPREQUEST (broadcast) ─────────▶│   3. "J'accepte cette proposition" (confirmée en broadcast,
  │                                            │      pour informer les autres serveurs DHCP éventuels)
  │◀─────── DHCPACK ───────────────────────────│   4. "Confirmé, voici le bail complet"
```

**Pourquoi la requête initiale (DISCOVER) est-elle en broadcast ?** À ce stade, le client n'a **aucune configuration IP** (c'est précisément ce qu'il cherche à obtenir) — il ne peut donc pas envoyer de requête unicast ciblée vers un serveur DHCP dont il ne connaît pas encore l'adresse. Le broadcast permet d'atteindre tout serveur DHCP présent sur le même domaine de diffusion (rappel module 8), sans configuration préalable.

**Pourquoi le REQUEST est-il aussi en broadcast, alors que le client connaît déjà l'adresse proposée par l'OFFER ?** Pour informer, par la même diffusion, tous les autres serveurs DHCP éventuellement présents sur le réseau (dans un scénario avec plusieurs serveurs DHCP, ce qui peut survenir intentionnellement en redondance ou accidentellement par erreur de configuration) que leur propre offre, si elle en avait émis une, n'a pas été retenue — leur permettant de libérer l'adresse qu'ils avaient provisoirement réservée pour ce client.

### 7.6 Bail DHCP (lease)

L'attribution DHCP n'est pas permanente : elle est accordée pour une durée définie, le **bail (lease)**. Avant l'expiration de ce bail, le client tente de le **renouveler** (renewal, généralement à la moitié de la durée du bail, via un échange direct unicast REQUEST/ACK avec le serveur, sans repasser par un DISCOVER complet) ; si le renouvellement échoue et que le bail expire complètement, le client doit reprendre le processus DORA complet depuis le début.

**Relais DHCP (DHCP relay) :** dans un réseau segmenté en plusieurs VLAN/sous-réseaux (rappel module 8), un serveur DHCP unique peut malgré tout servir plusieurs segments grâce à un **agent relais**, configuré sur le routeur de chaque segment, qui transmet en unicast les requêtes DHCP broadcast reçues localement vers le serveur DHCP central (potentiellement situé sur un tout autre segment) — évitant de déployer un serveur DHCP physique sur chaque sous-réseau.

## 8. Architecture

```
┌───────────────────────────────────────────────────────┐
│  Client                                                    │
│    1. DHCP DORA → obtient IP, masque, passerelle, DNS       │
│    2. Requête DNS récursive → résolveur configuré            │
└──────────────────────┬──────────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────────┐
│  Résolveur DNS récursif                                       │
│    Requêtes itératives : Racine → TLD → Autorité               │
└──────────────────────┬──────────────────────────────────┘
                        ▼
              Réponse DNS finale (enregistrement)
```

## 9. Fonctionnement interne (cas Linux)

Sous Linux, la configuration DNS d'une machine cliente est typiquement stockée dans `/etc/resolv.conf` (souvent généré automatiquement par le client DHCP, `dhclient` ou `systemd-networkd`/`NetworkManager`, à partir des options reçues lors du bail DORA). La résolution de nom côté application passe généralement par la bibliothèque C standard (résolveur "stub"), qui interroge simplement le ou les résolveurs récursifs configurés — l'application elle-même n'implémente jamais la logique itérative complète, entièrement déléguée au résolveur récursif configuré. Un cache DNS local (ex : `systemd-resolved`) peut s'intercaler entre les applications et le résolveur récursif externe pour réduire encore la latence des résolutions répétées, avec sa propre gestion de TTL.

## 10. Démonstration

```bash
# Résolution DNS avec détail de la requête
dig example.com

# Observer le processus DHCP en capture (nécessite de renouveler ou d'obtenir un bail)
sudo tcpdump -i any -nn port 67 or port 68 &
sudo dhclient -r && sudo dhclient   # relâche puis redemande un bail (attention : coupe temporairement le réseau)
wait

# Consulter le bail DHCP actuel
cat /var/lib/dhcp/dhclient.leases 2>/dev/null || ip addr show
```

## 11. Commandes / outils

| Commande | Rôle | Exemple |
|---|---|---|
| `dig` | Interroger DNS avec un contrôle fin | `dig example.com A` |
| `dig +trace` | Observer la résolution itérative complète, étape par étape | `dig +trace example.com` |
| `nslookup` | Interroger DNS (interface plus simple, historique) | `nslookup example.com` |
| `host` | Résolution DNS rapide en ligne de commande | `host example.com` |
| `dig -x` | Résolution inverse (PTR) | `dig -x 93.184.216.34` |
| `tcpdump port 67 or port 68` | Capturer le trafic DHCP (ports serveur/client) | `sudo tcpdump -i any port 67 or port 68` |
| `ip addr show` | Vérifier l'adresse obtenue par DHCP | `ip addr show` |

**Erreur fréquente :** tester une résolution DNS avec `ping <nom>` uniquement — si le `ping` échoue, l'étudiant conclut à tort à un problème DNS, alors que la résolution de nom a pu parfaitement réussir (visible en ajoutant `-v` ou en comparant avec `dig`) et que le problème se situe en réalité dans la joignabilité ICMP de l'adresse résolue (rappel module 11) — toujours isoler la résolution de nom elle-même (`dig`) de la joignabilité réseau (`ping`) comme deux tests distincts.

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** une entreprise migre son site web vers une nouvelle infrastructure avec une nouvelle adresse IP, et met à jour l'enregistrement DNS A en conséquence. Deux heures après la migration, la majorité des utilisateurs accèdent normalement au nouveau site, mais un sous-ensemble d'utilisateurs continue d'être dirigé vers l'ancienne adresse, dont le serveur a depuis été éteint, provoquant des erreurs de connexion pour ces utilisateurs.

**Analyse attendue :** ce symptôme s'explique directement par le mécanisme de **cache et TTL** vu en cours : les résolveurs récursifs utilisés par les utilisateurs concernés avaient mis en cache l'ancienne réponse DNS avant la migration, pour une durée définie par le TTL configuré sur l'ancien enregistrement (probablement plusieurs heures, voire davantage) — ces résolveurs continuent de servir l'ancienne réponse depuis leur cache jusqu'à expiration de ce TTL, indépendamment du changement déjà effectif côté serveur faisant autorité. Le problème n'est pas une panne actuelle du DNS, mais un effet de propagation attendu et prévisible. Correction pour l'incident en cours : attendre l'expiration naturelle des caches concernés (rien à corriger côté infrastructure, le changement est déjà correctement en place) ; mesure préventive pour l'avenir : réduire le TTL de l'enregistrement plusieurs heures avant toute migration planifiée future (rappel section 7.4), pour limiter ce délai de propagation lors du prochain changement.

## 15. Sécurité

- **DNS spoofing / cache poisoning** : un attaquant peut tenter d'injecter une fausse réponse DNS dans le cache d'un résolveur, redirigeant les utilisateurs vers une adresse malveillante — contre-mesure principale : **DNSSEC**, qui ajoute une signature cryptographique aux enregistrements DNS, permettant de vérifier leur authenticité et leur intégrité
- **DHCP spoofing / rogue DHCP server** : un serveur DHCP non autorisé installé sur le réseau (accidentellement ou malveillamment) peut répondre aux requêtes DISCOVER avant le serveur légitime, distribuant une configuration réseau malveillante (ex : une passerelle ou un serveur DNS contrôlé par l'attaquant, permettant une interception de trafic) — contre-mesure : **DHCP snooping**, fonctionnalité de switch manageable qui n'autorise les réponses DHCP (OFFER/ACK) qu'en provenance de ports explicitement désignés comme fiables
- Les enregistrements TXT de type SPF/DKIM (7.3) participent à la lutte contre l'usurpation d'expéditeur email, un exemple concret d'usage de DNS pour la sécurité applicative au-delà de la simple résolution de nom
- Un serveur DHCP mal sécurisé (accessible sans authentification à quiconque sur le réseau local) constitue une surface d'attaque simple pour une reconnaissance ou une prise de contrôle partielle du trafic réseau local

## 16. Automatisation

Script Bash de vérification de cohérence DNS avant une migration planifiée (confirme que le TTL a bien été abaissé et que l'ancienne/nouvelle réponse sont cohérentes avec ce qui est attendu).

```bash
#!/usr/bin/env bash
# Vérifie le TTL actuel d'un enregistrement DNS avant une migration planifiée
set -euo pipefail

DOMAINE="${1:?Usage: $0 <domaine> <ttl_max_attendu_secondes>}"
TTL_MAX_ATTENDU="${2:?Usage: $0 <domaine> <ttl_max_attendu_secondes>}"

TTL_ACTUEL=$(dig +noall +answer "$DOMAINE" A | awk 'NR==1 {print $2}')

if [[ -z "$TTL_ACTUEL" ]]; then
    echo "ERREUR: impossible de résoudre $DOMAINE"
    exit 2
fi

echo "TTL actuel pour $DOMAINE : ${TTL_ACTUEL}s"

if (( TTL_ACTUEL > TTL_MAX_ATTENDU )); then
    echo "ALERTE: TTL trop élevé pour une migration imminente (attendu <= ${TTL_MAX_ATTENDU}s)."
    echo "Abaissez le TTL et attendez sa propagation avant de procéder à la migration."
    exit 1
fi

echo "OK: TTL suffisamment bas pour procéder à la migration."
exit 0
```

## 17. Troubleshooting

Scénario court : une nouvelle machine connectée à un réseau d'entreprise n'obtient aucune adresse IP (reste en configuration APIPA/link-local `169.254.x.x`, une adresse d'auto-configuration utilisée par certains systèmes en l'absence de réponse DHCP). Démarche : capturer le trafic DHCP (`tcpdump port 67 or port 68`) pendant une tentative de connexion → observer si un DHCPDISCOVER part bien de la machine (confirme que la machine tente correctement le processus) → observer si un DHCPOFFER revient → si aucune offre ne revient, suspecter soit l'absence de serveur DHCP joignable sur ce segment (VLAN sans relais configuré, rappel section 7.6), soit un problème de couche 2 empêchant le broadcast d'atteindre le serveur (port switch mal configuré) → vérifier la configuration du relais DHCP sur le routeur du segment concerné si l'architecture en utilise un.

## 18. Synthèse

- DNS est organisé en hiérarchie déléguée (racine → TLD → domaine → sous-domaine) ; le client fait une requête récursive, le résolveur effectue ensuite des requêtes itératives
- Les enregistrements DNS ont des types spécifiques (A, AAAA, CNAME, MX, NS, PTR, TXT), chacun avec un rôle distinct
- Le TTL contrôle la durée de mise en cache ; un TTL mal anticipé avant une migration explique des symptômes de propagation lente, pas une panne réelle
- DHCP attribue une configuration réseau via le processus DORA (Discover, Offer, Request, Ack), avec DISCOVER et REQUEST en broadcast pour des raisons précises liées à l'absence initiale de configuration IP et à la coordination entre serveurs potentiellement multiples
- Un bail DHCP est temporaire et doit être renouvelé ; un relais DHCP permet à un serveur unique de servir plusieurs segments réseau
- DNS et DHCP partagent des enjeux de sécurité analogues (usurpation de réponse) avec des contre-mesures dédiées (DNSSEC, DHCP snooping)
- Toujours isoler le test de résolution de nom (`dig`) du test de joignabilité réseau (`ping`) — ce sont deux vérifications distinctes, souvent confondues à tort

## 19. Questions d'évaluation

**QCM**
1. Un résolveur DNS récursif, du point de vue du client, effectue :
 a) uniquement une requête itérative b) tout le travail nécessaire et retourne la réponse finale en une seule requête côté client c) aucune requête, la réponse est toujours en cache d) une requête TCP obligatoire

2. Le message DHCP DISCOVER est envoyé en broadcast car :
 a) c'est plus rapide qu'un envoi unicast b) le client ne dispose pas encore d'une configuration IP lui permettant de cibler un serveur précis c) le protocole DHCP n'accepte pas l'unicast d) c'est une exigence de sécurité

**Questions ouvertes**
- Expliquez pourquoi un enregistrement DNS modifié peut mettre plusieurs heures à être visible par tous les utilisateurs, malgré un changement déjà effectif côté serveur faisant autorité.
- Décrivez les 4 étapes du processus DHCP DORA et le rôle de chacune.

**Exercice technique**
- Donnez la commande permettant d'observer la résolution DNS complète, étape par étape (requêtes itératives), pour un domaine donné.

**Problème de diagnostic**
- Une machine ne parvient pas à joindre un site web par son nom, mais y parvient normalement en utilisant directement son adresse IP. Que peut-on en conclure sur la localisation probable du problème ?

## 20. Mini-projet

Développer un script (Bash ou Python) qui :
1. Surveille en continu la résolution d'une liste configurable de domaines critiques, en comparant la réponse actuelle à une réponse de référence connue
2. Détecte un changement inattendu de l'enregistrement A/AAAA d'un domaine surveillé (signature possible de cache poisoning ou de modification légitime non communiquée)
3. Vérifie et journalise le TTL actuel de chaque domaine surveillé, alertant si un TTL anormalement élevé est détecté peu avant une fenêtre de maintenance déclarée (configuration manuelle de la fenêtre)
4. Journalise chaque détection avec horodatage
5. Génère un rapport quotidien de synthèse sur l'état de résolution de l'ensemble des domaines surveillés

---

# PARTIE II — TD

### Niveau 1 — Compréhension

**Exercice 1.1**
Décrivez la hiérarchie DNS de la racine jusqu'à un sous-domaine, avec un exemple concret.

**Exercice 1.2**
Expliquez la différence entre résolution récursive et résolution itérative, en précisant qui effectue chacune.

**Exercice 1.3**
Décrivez les 4 étapes du processus DHCP DORA.

### Niveau 2 — Application

**Exercice 2.1**
Donnez le type d'enregistrement DNS approprié pour chacun des besoins suivants : associer un nom à une adresse IPv4 ; créer un alias vers un autre nom ; indiquer le serveur de messagerie d'un domaine.

**Exercice 2.2**
Donnez la commande permettant d'observer la résolution DNS complète, étape par étape, révélant chaque serveur interrogé (racine, TLD, autorité).

**Exercice 2.3**
Donnez la commande permettant de capturer spécifiquement le trafic DHCP sur une interface réseau.

### Niveau 3 — Analyse

**Exercice 3.1**
Un enregistrement DNS A est modifié pour pointer vers une nouvelle adresse IP à 10h00. À 11h30, certains utilisateurs continuent d'être dirigés vers l'ancienne adresse. L'équipe technique confirme que la modification a été correctement appliquée côté serveur faisant autorité dès 10h00. Expliquez ce décalage sans invoquer une erreur de configuration.

**Exercice 3.2**
Sur un réseau segmenté en plusieurs VLAN, un seul serveur DHCP physique existe, situé sur le VLAN d'administration. Les machines des autres VLAN parviennent pourtant à obtenir une adresse IP automatiquement. Expliquez le mécanisme qui rend cela possible, sans qu'un serveur DHCP dédié soit présent sur chaque VLAN.

### Niveau 4 — Problème complexe

**Exercice 4.1**
Dans un bureau, un nouvel équipement réseau est installé par erreur avec un serveur DHCP intégré activé par défaut (fonctionnalité souvent présente sur certains routeurs grand public), sur un réseau d'entreprise qui dispose déjà d'un serveur DHCP légitime et correctement configuré. Quelques utilisateurs commencent à rapporter des problèmes de connectivité Internet intermittents, tandis que d'autres n'ont aucun problème. En vous appuyant sur le processus DORA vu en cours (notamment le rôle du broadcast à chaque étape), expliquez pourquoi seuls certains utilisateurs sont affectés, ce que cela révèle sur la façon dont un client DHCP choisit entre plusieurs offres reçues, et quelle mesure de sécurité de couche 2 (vue en partie au cours) permettrait de prévenir ce type d'incident à l'avenir.

---

## Correction du TD

**1.1** — La hiérarchie part de la racine (`.`, implicite, gérée par des serveurs racine faisant autorité sur l'existence des TLD), puis descend vers un TLD (ex : `.com`, géré par un registre délégué), puis vers un domaine enregistré au sein de ce TLD (ex : `example.com`, avec ses propres serveurs faisant autorité), puis éventuellement vers un sous-domaine défini librement par le propriétaire du domaine (ex : `www.example.com`). Chaque niveau délègue l'autorité du niveau suivant.

**1.2** — La résolution récursive est celle vue par le **client** : il envoie une seule requête à son résolveur configuré, qui se charge de tout le travail nécessaire et retourne directement la réponse finale complète. La résolution itérative est celle effectuée par le **résolveur** lui-même : il interroge successivement la racine, puis le TLD concerné, puis le serveur faisant autorité, chaque serveur intermédiaire renvoyant soit la réponse finale, soit une référence vers le serveur suivant à contacter — le résolveur assemble ainsi la réponse complète avant de la transmettre au client.

**1.3** — 1. DHCPDISCOVER : le client, sans configuration IP, diffuse en broadcast une demande de configuration. 2. DHCPOFFER : un serveur DHCP disponible répond avec une proposition d'adresse et de paramètres réseau. 3. DHCPREQUEST : le client accepte formellement cette proposition, également en broadcast, pour informer d'éventuels autres serveurs DHCP que leur offre n'est pas retenue. 4. DHCPACK : le serveur confirme définitivement l'attribution, complétant le bail.

**2.1** — Association nom→IPv4 : enregistrement **A**. Alias vers un autre nom : enregistrement **CNAME**. Serveur de messagerie d'un domaine : enregistrement **MX**.

**2.2** — `dig +trace <domaine>`.

**2.3** — `sudo tcpdump -i any port 67 or port 68` (ports standards du protocole DHCP, serveur et client respectivement).

**3.1** — Ce décalage s'explique par le mécanisme de **cache et TTL** vu en cours : avant la modification effectuée à 10h00, l'ancien enregistrement DNS portait un TTL qui a pu être mis en cache par différents résolveurs récursifs (utilisés par les utilisateurs concernés) à des moments différents, chacun avec sa propre durée de validité restante au moment de la modification. Ces résolveurs continuent de servir la réponse mise en cache jusqu'à expiration de ce TTL, indépendamment du fait que le serveur faisant autorité ait bien été mis à jour dès 10h00 — ce n'est pas une erreur de configuration, mais un comportement attendu et prévisible du système de cache DNS, dont la durée résiduelle dépend du TTL configuré avant la migration et du moment de la dernière mise en cache par chaque résolveur concerné.

**3.2** — Ce mécanisme est rendu possible par un **agent relais DHCP (DHCP relay)**, configuré sur le routeur de chaque VLAN ne disposant pas de serveur DHCP local. Cet agent relais reçoit les requêtes DHCP broadcast émises localement sur son VLAN (DISCOVER, REQUEST) et les retransmet en unicast vers le serveur DHCP central situé sur le VLAN d'administration, puis achemine la réponse du serveur (OFFER, ACK) de retour vers le client d'origine sur son VLAN — permettant à un unique serveur DHCP physique de servir l'ensemble des segments du réseau, sans qu'un serveur dédié ne soit nécessaire sur chacun d'eux.

**4.1** — Le fait que seuls certains utilisateurs soient affectés s'explique par le comportement du **broadcast DHCPDISCOVER** (rappel section 7.5 et TD 1.3) : ce message atteint tous les serveurs DHCP présents sur le même domaine de diffusion, et **plusieurs serveurs peuvent répondre simultanément** avec chacun leur propre DHCPOFFER. Le client ne contacte généralement qu'**un seul** de ces serveurs (habituellement le premier dont l'offre est reçue, ou selon un critère propre à l'implémentation du client), de façon largement imprévisible et différente d'un client à l'autre selon le timing réel de réception des offres concurrentes. Les utilisateurs dont le client a retenu l'offre du serveur DHCP légitime obtiennent une configuration réseau correcte et fonctionnent normalement ; ceux dont le client a retenu par malchance l'offre du nouvel équipement mal configuré reçoivent une configuration incorrecte (passerelle, DNS, ou plage d'adresses erronée), provoquant les problèmes de connectivité intermittents rapportés — "intermittents" au sens où redémarrer leur connexion réseau peut faire basculer aléatoirement leur client vers l'un ou l'autre des deux serveurs concurrents lors d'un nouveau DORA complet. Mesure de sécurité de couche 2 préventive : le **DHCP snooping** (vu en section 15 du cours), une fonctionnalité de switch manageable qui n'autorise les messages DHCP de type OFFER/ACK qu'en provenance de ports explicitement désignés comme fiables (typiquement uniquement le port relié au serveur DHCP légitime), bloquant ainsi toute réponse DHCP émise par un équipement non autorisé branché sur un port utilisateur standard.

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur réseau junior pour diagnostiquer des problèmes de résolution DNS et d'attribution DHCP sur un petit réseau de laboratoire, et pour développer un outil de vérification pré-migration.

## Architecture

Une machine Linux avec accès réseau sortant, `dig`, `nslookup` ou `host`, `tcpdump` disponibles. L'observation directe d'un échange DHCP complet nécessite un environnement permettant de relâcher/renouveler un bail (VM avec interface dédiée recommandée plutôt qu'un conteneur, dont la gestion réseau est souvent différente).

## Objectifs
- Observer une résolution DNS récursive et une résolution itérative complète
- Différencier les types d'enregistrements DNS sur un domaine réel
- Observer (ou simuler par capture documentée) un échange DHCP complet
- Développer et tester un script de vérification de TTL avant migration

## Préparation

```bash
which dig nslookup host tcpdump 2>/dev/null
```

## Étapes

### Étape 1 — Résolution DNS simple et détaillée

```bash
dig example.com A
dig example.com MX
dig example.com NS
```
*Question :* pour chaque type interrogé, identifiez dans la sortie le TTL retourné et la section ANSWER correspondante. Les TTL sont-ils identiques pour les trois types d'enregistrements ?

### Étape 2 — Observer la résolution itérative complète

```bash
dig +trace example.com
```
*Question :* combien de serveurs différents sont interrogés successivement avant d'obtenir la réponse finale ? Identifiez, dans la sortie, le passage de la racine vers le TLD, puis du TLD vers le serveur faisant autorité.

### Étape 3 — Observer (ou documenter) un échange DHCP

```bash
sudo tcpdump -i any -nn port 67 or port 68 -c 8 &
# Sur une VM disposant d'un client DHCP configurable :
sudo dhclient -r eth0 2>/dev/null   # relâche le bail actuel (ATTENTION : coupe temporairement le réseau)
sudo dhclient eth0                   # redemande un bail complet
wait
```
*Si cette manipulation n'est pas réalisable dans votre environnement (conteneur, accès restreint) :* documentez à la place, à partir de la section 7.5 du cours, les 4 messages attendus dans l'ordre, avec le type d'adresse (broadcast/unicast) de chacun, comme substitut d'observation directe.

*Question :* si vous avez pu observer l'échange réel, combien de messages DORA complets identifiez-vous ? Le DHCPACK final contient-il, d'après votre capture ou votre recherche documentaire, d'autres informations que la seule adresse IP (masque, passerelle, DNS) ?

### Étape 4 — Script de vérification de TTL avant migration

Reprendre et tester le script de la section 16 du cours (`verif_ttl_migration.sh`) :

```bash
bash verif_ttl_migration.sh example.com 300
bash verif_ttl_migration.sh example.com 100000
```

## Commandes de référence
`dig`, `dig +trace`, `nslookup`, `tcpdump port 67 or port 68`

## Résultats attendus
- Sorties `dig` pour les 3 types d'enregistrements de l'étape 1, avec les TTL identifiés
- Sortie complète de `dig +trace` (étape 2), avec les serveurs identifiés à chaque niveau
- Capture ou documentation de l'échange DHCP (étape 3)
- Sorties du script de vérification TTL sur les deux cas de l'étape 4

## Questions

1. Pourquoi les enregistrements A et NS d'un même domaine peuvent-ils avoir des TTL différents ? Qu'est-ce que cela implique pour la vitesse de propagation respective d'un changement d'adresse IP versus d'un changement de serveur faisant autorité ?
2. Dans l'étape 2, si le TLD `.com` avait été temporairement injoignable au moment de votre requête `+trace`, à quel niveau précis de la chaîne la résolution aurait-elle échoué, et quel type de message d'erreur DNS vous attendriez-vous à observer ?
3. En reliant à l'exercice 4.1 du TD, si vous deviez concevoir un test rapide permettant de détecter la présence d'un serveur DHCP non autorisé sur un réseau de laboratoire, quelle observation dans une capture `tcpdump` du trafic DHCP constituerait un signal d'alerte fiable ?

## Troubleshooting (panne à diagnostiquer)

**Situation fournie :** un étudiant exécute le script `verif_ttl_migration.sh` contre un domaine interne de laboratoire et obtient une erreur "impossible de résoudre", alors que `dig` en ligne de commande directe résout parfaitement ce même domaine sans problème apparent.

**Démarche attendue :**
1. Symptôme : échec du script alors qu'une résolution manuelle directe fonctionne
2. Hypothèse : le script s'appuie sur un parsing précis de la sortie de `dig +noall +answer` (section NR==1, colonne 2 pour le TTL) — si le domaine interrogé possède plusieurs enregistrements A (plusieurs adresses IP pour un même nom, cas fréquent en répartition de charge DNS), ou si un CNAME précède l'enregistrement A dans la réponse (rappel section 7.3 du cours), la structure de la sortie diffère de ce que le script attend en position fixe, provoquant une extraction incorrecte ou vide
3. Vérification : exécuter manuellement `dig +noall +answer <domaine> A` et examiner précisément le nombre de lignes retournées et leur contenu exact
4. Isolation : confirmer que le domaine de laboratoire concerné utilise effectivement un CNAME ou plusieurs enregistrements A, contrairement au cas simple utilisé lors du développement initial du script
5. Correction : adapter le script pour gérer explicitement le cas d'un CNAME (suivre la chaîne jusqu'au premier enregistrement A réel) et/ou pour choisir une ligne précise en cas d'enregistrements multiples, plutôt que de supposer implicitement une réponse à une seule ligne simple
6. Vérification : relancer le script corrigé sur le domaine de laboratoire et confirmer un résultat cohérent avec la résolution manuelle
7. Documentation : noter cette limite de robustesse de parsing (encore un exemple, après `lsof` et `ipcs` en UE1, et la sortie `ip neigh`/`ps` déjà rencontrées) comme un point de vigilance récurrent et transversal dans tout script s'appuyant sur la sortie texte d'une commande système — la structure réelle des données doit toujours être vérifiée sur des cas non triviaux, pas seulement sur le cas simple utilisé pendant le développement initial

## Validation

Le TP est validé si l'étudiant fournit :
- Les 4 résultats/captures/documentations demandés
- Des réponses argumentées aux 3 questions
- Une identification correcte de la cause du scénario de troubleshooting (parsing fragile face à un CNAME ou des enregistrements multiples), cohérente avec la leçon transversale déjà rencontrée dans les modules précédents

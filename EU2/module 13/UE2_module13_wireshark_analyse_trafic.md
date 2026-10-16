# UE2 — Module 13 : Analyse de Trafic Réseau avec Wireshark

*Module de synthèse de l'UE2 — mobilise l'ensemble des modules 7 à 12 (OSI/TCP-IP, Ethernet/ARP, IPv4/IPv6/routage, TCP, UDP/ICMP, DNS/DHCP).*

---

# PARTIE I — COURS

## 1. Titre du cours
Analyse de trafic réseau : maîtriser Wireshark pour le diagnostic professionnel

## 2. Niveau
Élève ingénieur — cycle ingénieur, clôture de l'UE2 réseaux (S1)

## 3. Durée
CM : 3h (module orienté pratique — +3h TD, +5h TP en Parties II et III, volume TP plus élevé car module de synthèse)

## 4. Prérequis
- L'intégralité de l'UE2 (modules 7 à 12) : ce module ne introduit aucun nouveau protocole, il apprend à **lire et corréler** ce qui a déjà été étudié

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Utiliser efficacement les filtres de capture et les filtres d'affichage Wireshark, et expliquer leur différence
- Suivre un flux TCP complet ("Follow TCP Stream") et en extraire le contenu applicatif
- Utiliser les outils statistiques de Wireshark (Conversations, Protocol Hierarchy, I/O Graphs) pour obtenir une vue d'ensemble d'une capture
- Identifier dans une capture les signatures d'anomalies vues dans les modules précédents (retransmissions TCP, ARP suspect, DNS lent, DHCP rogue)
- Mener une analyse de bout en bout sur un incident réseau simulé, en mobilisant plusieurs couches simultanément
- Documenter une analyse de capture de façon claire et actionnable pour une équipe non experte en réseau

## 6. Motivation

Tous les modules précédents de l'UE2 ont donné les connaissances théoriques : comment fonctionne Ethernet, ARP, IP, TCP, UDP, ICMP, DNS, DHCP. Mais en production, un ingénieur ne reçoit jamais un problème pré-étiqueté "c'est un problème de couche 4" — il reçoit une capture réseau brute, ou un accès pour en réaliser une, et doit lui-même identifier où chercher. Wireshark est l'outil qui rend visible tout ce qui a été appris de façon abstraite jusqu'ici : une trame Ethernet, un handshake TCP, une requête DNS, un échange DHCP — tout devient lisible, à condition de savoir où et comment regarder.

Ce module est délibérément un module de synthèse : peu de nouveaux concepts, mais une pratique intensive de lecture de captures réelles, la compétence la plus directement transférable vers un poste d'ingénieur système/réseau junior.

## 7. Concepts fondamentaux

### 7.1 Filtres de capture vs filtres d'affichage

Une confusion fréquente et importante à lever d'emblée :

- **Filtre de capture (capture filter)** : appliqué **avant** l'enregistrement, utilise la syntaxe BPF (Berkeley Packet Filter, la même que `tcpdump`) — détermine quels paquets sont réellement capturés et écrits, les autres étant définitivement perdus (non enregistrés). Exemple : `tcp port 443`.
- **Filtre d'affichage (display filter)** : appliqué **après** la capture, sur des données déjà entièrement enregistrées — n'affecte que ce qui est affiché à l'écran, sans perte de données ; on peut changer ce filtre autant de fois que nécessaire sans avoir à recapturer. Syntaxe propre à Wireshark, plus riche et plus lisible. Exemple : `tcp.port == 443`.

**Recommandation professionnelle :** en cas de doute sur ce qu'il faut chercher, **capturer largement (peu ou pas de filtre de capture) puis filtrer à l'affichage** — un paquet non capturé est perdu définitivement, alors qu'un filtre d'affichage mal choisi peut toujours être corrigé après coup sur les mêmes données.

### 7.2 Filtres d'affichage courants

| Filtre | Effet |
|---|---|
| `ip.addr == 192.168.1.10` | Trafic impliquant cette adresse (source ou destination) |
| `tcp.port == 443` | Trafic TCP sur le port 443 (source ou destination) |
| `tcp.flags.syn == 1 && tcp.flags.ack == 0` | Isoler uniquement les SYN initiaux (début de handshake, module 10) |
| `tcp.analysis.retransmission` | Segments identifiés par Wireshark comme des retransmissions |
| `dns` | Tout le trafic DNS |
| `arp` | Tout le trafic ARP |
| `icmp.type == 3` | Messages ICMP Destination Unreachable (module 11) |
| `bootp` ou `dhcp` | Trafic DHCP (le nom historique du protocole, BOOTP, apparaît encore dans certains filtres) |

**Combinaison de filtres :** `&&` (ET), `||` (OU), `!` (NON) — ex : `ip.addr == 10.0.0.5 && tcp.port == 80` isole précisément le trafic HTTP impliquant cette adresse.

### 7.3 "Follow TCP Stream"

Wireshark permet de reconstituer l'intégralité d'un flux TCP applicatif (clic droit sur un segment → "Follow → TCP Stream"), en réassemblant tous les segments dans l'ordre logique et en affichant le contenu échangé dans les deux sens (généralement colorié différemment client/serveur) — extrêmement utile pour lire directement une requête HTTP complète, une commande SSH en clair (avant chiffrement de session), ou tout protocole textuel, sans avoir à décortiquer segment par segment manuellement.

**Limite importante :** pour du trafic chiffré (HTTPS/TLS), le contenu applicatif réel n'est pas lisible de cette façon sans les clés de déchiffrement — Wireshark peut afficher les métadonnées de la négociation TLS (visibles en clair, ex : le nom de domaine demandé via SNI — Server Name Indication) mais pas le contenu chiffré lui-même, sauf configuration spécifique de déchiffrement (hors périmètre de ce module).

### 7.4 Outils statistiques

- **Statistics → Protocol Hierarchy** : vue d'ensemble de la répartition du trafic capturé par protocole (pourcentage de TCP, UDP, ARP, DNS...) — un premier réflexe utile face à une capture volumineuse et inconnue, pour orienter rapidement l'investigation.
- **Statistics → Conversations** : liste des échanges entre paires d'adresses (ou de ports), avec volumes de données, durée, débit — permet d'identifier rapidement "qui parle à qui" et les flux les plus volumineux ou les plus longs.
- **Statistics → I/O Graph** : représentation graphique du trafic dans le temps, permettant de repérer visuellement un pic, une coupure, ou un comportement périodique anormal.
- **Expert Information (Analyze → Expert Information)** : Wireshark signale automatiquement certaines anomalies détectées (retransmissions, ACK dupliqués, fenêtres nulles, checksums invalides...) — un point de départ rapide, mais qui ne remplace jamais une lecture critique du contexte complet (des faux positifs existent, notamment liés à des limitations de capture comme le "TCP checksum offload" qui peut faire apparaître de faux checksums invalides).

### 7.5 Signatures reconnaissables des anomalies étudiées dans l'UE2

Une capture réseau permet de **voir concrètement** ce qui a été décrit en théorie dans les modules précédents :

| Symptôme observé en capture | Module de référence | Interprétation |
|---|---|---|
| Retransmission de SYN, pas de SYN-ACK en retour | M10 (TCP) | Établissement de connexion qui échoue, cible injoignable ou filtrée sur ce port |
| Segments dupliqués avec même numéro de séquence | M10 | Perte détectée et retransmission (rappel fenêtre de congestion) |
| Fenêtre annoncée à 0 (`tcp.analysis.zero_window`) | M10 | Récepteur saturé, contrôle de flux actif |
| Requêtes ARP répétées sans réponse | M8 (Ethernet/ARP) | Cible injoignable au niveau 2, ou changement d'adresse suspect si les réponses varient |
| Une IP associée à plusieurs MAC différentes dans le temps | M8 | Possible ARP spoofing (ou conflit d'IP, cf. M9) |
| ICMP Time Exceeded en chaîne | M11 (ICMP) | Trace d'un `traceroute`, ou boucle de routage si en excès inhabituel |
| ICMP Destination Unreachable (port) | M11 | Port UDP fermé côté destination |
| Requête DNS sans réponse, ou réponse très tardive | M12 (DNS) | Résolveur lent, injoignable, ou timeout applicatif à investiguer |
| Deux DHCPOFFER pour un seul DHCPDISCOVER | M12 (DHCP) | Rogue DHCP server potentiel (rappel TD module 12) |

**C'est ce tableau, plus que tout autre contenu de ce cours, qui constitue le véritable objectif pédagogique du module** : transformer des symptômes visuels en capture en diagnostics précis, ancrés dans la théorie déjà acquise.

### 7.6 Méthodologie d'analyse d'une capture inconnue

1. **Vue d'ensemble** : Protocol Hierarchy pour identifier la nature générale du trafic
2. **Repérage des flux principaux** : Conversations pour identifier les échanges les plus significatifs (volume, durée)
3. **Filtrage ciblé** : une fois une piste identifiée, filtrer sur l'adresse/le port/le protocole concerné
4. **Lecture fine** : examen segment par segment (ou "Follow Stream" si applicatif) de la zone d'intérêt
5. **Corrélation multi-couches** : un symptôme de couche 4 (ex : connexion TCP qui échoue) peut avoir sa cause réelle à une couche différente (ARP non résolu, DNS erroné) — toujours vérifier les couches inférieures/adjacentes avant de conclure

## 8. Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Capture réseau (.pcap)                                     │
│  ┌────────────────────────────────────────────────────┐   │
│  │ Trame 1: ARP Request                                   │   │
│  │ Trame 2: ARP Reply                                     │   │
│  │ Trame 3: TCP SYN                                        │   │
│  │ Trame 4: TCP SYN-ACK                                     │   │
│  │ Trame 5: TCP ACK                                          │   │
│  │ Trame 6-N: Données applicatives (HTTP, etc.)                │   │
│  │ Trame N+1: TCP FIN                                          │   │
│  └────────────────────────────────────────────────────┘   │
│         │                    │                    │            │
│    Protocol Hierarchy   Conversations        Follow Stream       │
│    (vue globale)        (qui parle à qui)   (contenu applicatif) │
└─────────────────────────────────────────────────────────┘
```

## 9. Fonctionnement interne

Wireshark s'appuie, pour la capture elle-même, sur les mêmes mécanismes de capture bas niveau que `tcpdump` (libpcap sous Linux) — les deux outils peuvent d'ailleurs lire et écrire le même format de fichier (`.pcap`/`.pcapng`), permettant une capture en ligne de commande (`tcpdump -w capture.pcap`, utile sur un serveur distant sans interface graphique) suivie d'une analyse détaillée en interface graphique Wireshark en local, un flux de travail très courant en environnement professionnel où l'accès direct à une interface graphique sur le serveur de production est rarement disponible ou souhaitable.

## 10. Démonstration

```bash
# Capturer en ligne de commande (utile sur un serveur distant), à analyser ensuite dans Wireshark
sudo tcpdump -i any -w /tmp/capture_demo.pcap -c 50

# Alternative : utiliser tshark, la version ligne de commande de Wireshark, avec filtre d'affichage
tshark -r /tmp/capture_demo.pcap -Y "tcp.analysis.retransmission"
```

## 11. Commandes / outils

| Outil/Fonction | Rôle | Exemple |
|---|---|---|
| Wireshark (interface graphique) | Analyse visuelle complète | — |
| `tshark` | Version ligne de commande de Wireshark | `tshark -r fichier.pcap -Y "dns"` |
| `tcpdump -w` | Capturer vers un fichier `.pcap` exploitable par Wireshark | `sudo tcpdump -i eth0 -w capture.pcap` |
| Filtre `tcp.analysis.retransmission` | Isoler les retransmissions TCP | dans la barre de filtre d'affichage |
| Filtre `tcp.analysis.duplicate_ack` | Isoler les ACK dupliqués | dans la barre de filtre d'affichage |
| Statistics → Protocol Hierarchy | Vue d'ensemble par protocole | menu Wireshark |
| Statistics → Conversations | Vue des échanges par paire | menu Wireshark |

**Erreur fréquente :** confondre un filtre de capture et un filtre d'affichage dans la même barre de saisie — Wireshark distingue les deux via des champs séparés (capture au lancement, affichage une fois la capture en cours ou chargée), une confusion qui mène souvent un débutant à perdre des paquets qu'il pensait simplement "cacher" temporairement.

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** une équipe support reçoit une plainte utilisateur vague : "l'application est lente ce matin". Aucune information technique précise n'est fournie. Une capture réseau de 5 minutes est réalisée sur le poste de l'utilisateur pendant qu'il reproduit le problème.

**Analyse attendue (méthodologie appliquée) :** 1. Protocol Hierarchy révèle une proportion inhabituellement élevée de trafic DNS par rapport à d'habitude. 2. Conversations confirme de nombreuses requêtes DNS vers le même résolveur, avec une durée d'échange anormalement longue pour certaines. 3. Filtrage sur `dns` et tri par temps de réponse révèle plusieurs requêtes DNS avec un délai de plusieurs secondes avant réponse (alors que la majorité répond en quelques millisecondes). 4. Corrélation avec les modules précédents (M12) : ce n'est pas un problème applicatif, mais un résolveur DNS lent ou surchargé, ralentissant chaque nouvelle connexion sortante de l'application qui doit résoudre un nom avant chaque appel. Résolution : investiguer la charge ou la configuration du résolveur DNS utilisé, indépendamment du code applicatif — un exemple type de la valeur d'une capture réseau face à un symptôme vague, permettant de transformer "c'est lent" en une cause précise et actionnable.

## 15. Sécurité

- L'analyse de capture est un outil de diagnostic **et** un outil d'investigation de sécurité (forensics réseau) : une capture peut révéler une exfiltration de données, une communication avec une infrastructure malveillante connue, ou les traces d'une attaque en cours (scans, tentatives d'exploitation) — compétence directement réutilisée en UE8
- Une capture réseau contient potentiellement des données sensibles (identifiants en clair sur un protocole non chiffré, contenu de communications) — sa manipulation doit suivre les mêmes précautions de confidentialité que toute donnée sensible, et sa conservation doit être limitée à la durée strictement nécessaire au diagnostic
- Le filtrage `tcp.port == 443` révèle qu'une connexion utilise HTTPS, mais ne garantit rien sur la légitimité du certificat ou l'absence de compromission applicative — le chiffrement protège la confidentialité du contenu, pas la fiabilité de ce qui s'y passe

## 16. Automatisation

Script Bash d'analyse automatisée d'une capture, résumant les anomalies principales via `tshark` (utile pour un premier tri avant analyse manuelle approfondie).

```bash
#!/usr/bin/env bash
# Résumé automatisé d'une capture réseau : anomalies principales via tshark
set -euo pipefail

CAPTURE="${1:?Usage: $0 <fichier.pcap>}"

echo "=== Répartition par protocole ==="
tshark -r "$CAPTURE" -q -z io,phs 2>/dev/null | head -30

echo ""
echo "=== Retransmissions TCP ==="
NB_RETRANS=$(tshark -r "$CAPTURE" -Y "tcp.analysis.retransmission" 2>/dev/null | wc -l)
echo "Nombre de retransmissions détectées : $NB_RETRANS"

echo ""
echo "=== Requêtes DNS sans réponse visible dans la capture ==="
tshark -r "$CAPTURE" -Y "dns.flags.response == 0" -T fields -e dns.qry.name 2>/dev/null | sort | uniq -c | sort -rn | head -10

echo ""
echo "=== Messages ICMP Destination Unreachable ==="
NB_UNREACH=$(tshark -r "$CAPTURE" -Y "icmp.type == 3" 2>/dev/null | wc -l)
echo "Nombre détecté : $NB_UNREACH"
```

## 17. Troubleshooting

Scénario court : une capture réseau censée documenter un incident de connexion applicative ne contient, à l'examen, quasiment aucun trafic pertinent — l'application était pourtant активement utilisée pendant la capture. Démarche : vérifier l'**interface de capture** sélectionnée (une capture lancée sur la mauvaise interface réseau, par exemple une interface virtuelle inactive plutôt que l'interface physique réellement utilisée, est une erreur fréquente et invisible tant qu'on ne compare pas le volume de trafic capturé au trafic attendu) → vérifier qu'aucun **filtre de capture** trop restrictif n'a été appliqué par erreur, excluant silencieusement le trafic recherché (rappel section 7.1 : un paquet exclu au niveau capture est perdu définitivement, contrairement à un filtre d'affichage) → relancer une capture large sur la bonne interface pour confirmer.

## 18. Synthèse

- Filtre de capture (avant, BPF, perte définitive si mal réglé) vs filtre d'affichage (après, syntaxe Wireshark, réversible) — une distinction fondamentale à ne jamais confondre
- Follow TCP Stream reconstitue un échange applicatif complet, mais ne révèle rien du contenu chiffré (TLS) sans déchiffrement explicite
- Protocol Hierarchy, Conversations et I/O Graph offrent une vue d'ensemble avant toute analyse fine ligne par ligne
- Chaque anomalie théorique étudiée dans l'UE2 (retransmission, ARP suspect, DNS lent, rogue DHCP) a une signature visuelle reconnaissable en capture
- Une méthodologie d'analyse rigoureuse va toujours du général (vue d'ensemble) au particulier (filtrage ciblé), et considère systématiquement une corrélation multi-couches avant de conclure
- `tshark` permet d'automatiser l'analyse de capture en ligne de commande, complémentaire à l'interface graphique Wireshark

## 19. Questions d'évaluation

**QCM**
1. Un filtre de capture mal configuré, excluant un trafic recherché :
 a) peut être corrigé après coup sans recapturer b) entraîne une perte définitive de ce trafic non enregistré c) n'a aucun effet sur les données capturées d) s'applique uniquement à l'affichage

2. Le filtre d'affichage permettant d'isoler les segments TCP retransmis est :
 a) `tcp.retransmit` b) `tcp.analysis.retransmission` c) `tcp.flags.retrans` d) `retransmission.tcp`

**Questions ouvertes**
- Expliquez la différence entre filtre de capture et filtre d'affichage, avec un exemple de conséquence pratique d'une confusion entre les deux.
- Un utilisateur signale une application "lente" sans autre précision. Décrivez la méthodologie d'analyse de capture (du général au particulier) que vous appliqueriez pour investiguer.

**Exercice technique**
- Donnez le filtre d'affichage Wireshark permettant d'isoler tout le trafic DNS impliquant l'adresse IP `192.168.1.50`.

**Problème de diagnostic**
- Une capture destinée à documenter un incident applicatif ne contient presque aucun trafic pertinent, alors que l'application était activement utilisée. Proposez les deux premières vérifications à effectuer.

## 20. Mini-projet

Développer un script (Bash ou Python, s'appuyant sur `tshark`) qui, à partir d'un fichier de capture `.pcap` fourni en argument :
1. Produit un résumé de la répartition par protocole (Protocol Hierarchy en ligne de commande)
2. Détecte et compte les principales anomalies vues dans le tableau de la section 7.5 (retransmissions TCP, ARP sans réponse, ICMP Destination Unreachable, DNS sans réponse visible)
3. Identifie les 5 conversations les plus volumineuses ou les plus longues de la capture
4. Produit un rapport de synthèse structuré, orienté vers un lecteur non expert (formulations claires, pas seulement des chiffres bruts), à la manière de l'étude de cas industrielle du cours
5. Journalise l'exécution avec horodatage pour un historique d'analyses successives sur un même incident

---

# PARTIE II — TD

*Ce TD s'appuie sur la lecture de sorties de filtres et de statistiques Wireshark décrites textuellement — la pratique de capture réelle est réservée au TP.*

### Niveau 1 — Compréhension

**Exercice 1.1**
Expliquez la différence fondamentale entre un filtre de capture et un filtre d'affichage, en particulier sur la question de la réversibilité.

**Exercice 1.2**
À quoi sert la fonctionnalité "Follow TCP Stream", et quelle est sa principale limite face à du trafic chiffré ?

**Exercice 1.3**
Citez les trois outils statistiques Wireshark vus en cours et leur usage respectif en une phrase chacun.

### Niveau 2 — Application

**Exercice 2.1**
Donnez le filtre d'affichage Wireshark permettant d'isoler uniquement les segments TCP de type SYN initial (sans le flag ACK).

**Exercice 2.2**
Donnez le filtre d'affichage permettant d'isoler tout le trafic ARP dans une capture.

**Exercice 2.3**
Donnez la commande `tshark` permettant d'appliquer, en ligne de commande sur un fichier `capture.pcap` déjà enregistré, un filtre d'affichage isolant les messages ICMP de type 3.

### Niveau 3 — Analyse

**Exercice 3.1**
Une capture réseau, filtrée sur `arp`, montre une même adresse IP associée à trois adresses MAC différentes au cours des dix dernières minutes, avec des changements toutes les 30 secondes environ. Quelle hypothèse cette observation soutient-elle, et quel module de l'UE2 permet de la justifier précisément ?

**Exercice 3.2**
Une capture Protocol Hierarchy montre que 40% du trafic total capturé est constitué de messages ICMP Time Exceeded. Ce chiffre est-il en soi anormal ? Formulez une hypothèse expliquant ce volume avant de conclure à un problème, en tenant compte du contexte probable de la capture.

### Niveau 4 — Problème complexe

**Exercice 4.1**
Une équipe support reçoit une capture réseau de 10 minutes réalisée pendant un incident où "le site web ne se charge jamais complètement, il reste bloqué". L'analyse Protocol Hierarchy révèle un trafic majoritairement TCP, avec quelques requêtes DNS en tout début de capture. Le filtre `tcp.analysis.retransmission` révèle un nombre élevé de retransmissions, concentrées sur une poignée de connexions vers la même adresse IP serveur. Le filtre `tcp.analysis.zero_window` ne révèle aucun résultat. En vous appuyant sur les modules 10 et 11 de l'UE2, formulez une hypothèse cohérente sur la cause du blocage, en excluant explicitement au moins une cause plausible mais réfutée par les observations disponibles (justifiez pourquoi elle est réfutée), et proposez une vérification complémentaire pour affiner le diagnostic.

---

## Correction du TD

**1.1** — Le filtre de capture est appliqué **avant** l'enregistrement des paquets : tout paquet qui ne correspond pas au filtre n'est jamais écrit, sa perte est **définitive** et ne peut être corrigée qu'en relançant une nouvelle capture. Le filtre d'affichage est appliqué **après** la capture, sur des données déjà entièrement enregistrées : il peut être modifié, retiré ou remplacé autant de fois que nécessaire sans aucune perte de données, puisque l'intégralité de la capture reste disponible en arrière-plan.

**1.2** — "Follow TCP Stream" reconstitue et affiche l'intégralité du contenu échangé dans une connexion TCP, dans l'ordre logique et pour les deux sens de communication, facilitant la lecture d'un protocole applicatif textuel. Sa principale limite est qu'elle ne révèle pas le contenu réel d'un trafic **chiffré** (HTTPS/TLS) : les données affichées resteraient chiffrées et illisibles sans les clés de déchiffrement appropriées, bien que certaines métadonnées de la négociation (comme le SNI) restent visibles en clair.

**1.3** — Protocol Hierarchy : donne une vue d'ensemble de la répartition du trafic capturé par protocole, utile pour un premier diagnostic rapide sur une capture inconnue. Conversations : liste les échanges entre paires d'adresses/ports avec leurs volumes et durées, utile pour identifier rapidement les flux les plus significatifs. I/O Graph : représente le trafic dans le temps sous forme graphique, utile pour repérer visuellement un pic, une coupure, ou un comportement périodique anormal.

**2.1** — `tcp.flags.syn == 1 && tcp.flags.ack == 0`.

**2.2** — `arp`.

**2.3** — `tshark -r capture.pcap -Y "icmp.type == 3"`.

**3.1** — Cette observation soutient fortement l'hypothèse d'une **attaque ARP spoofing** (ou, à un degré moindre de suspicion, un conflit d'adresse IP si les changements sont irréguliers plutôt que périodiques) : dans un fonctionnement légitime du protocole ARP, l'association entre une adresse IP et une adresse MAC ne devrait changer que lors d'un événement matériel réel (remplacement de carte réseau), pas de façon répétée et régulière en quelques minutes. Le module qui permet de justifier précisément cette hypothèse est le **module 8 (Ethernet, ARP et commutation)**, en particulier sa section sur l'absence d'authentification native du protocole ARP et le script de détection de changement suspect développé dans ce module.

**3.2** — Non, ce chiffre n'est pas nécessairement anormal en soi — tout dépend du **contexte de la capture**. Si cette capture a été réalisée pendant l'exécution d'un ou plusieurs `traceroute` (module 11, mécanisme TTL/Time Exceeded), un volume élevé de messages ICMP Time Exceeded est **parfaitement attendu et normal**, puisque c'est précisément le mécanisme sur lequel `traceroute` repose délibérément pour révéler chaque saut du chemin réseau. Ce n'est que si ce volume est observé **en dehors** de tout contexte de diagnostic de routage volontaire (aucun `traceroute` en cours, trafic applicatif normal uniquement) qu'il faudrait suspecter une anomalie réelle, par exemple une boucle de routage générant un TTL épuisé de façon répétée et non désirée — d'où l'importance, rappelée en section 7.6 du cours, de toujours contextualiser une observation statistique avant de conclure à un problème.

**4.1** — Hypothèse cohérente : le nombre élevé de retransmissions TCP concentrées sur des connexions vers la même adresse serveur, en l'absence de toute fenêtre nulle observée (`tcp.analysis.zero_window` vide), oriente vers une **perte de paquets sur le chemin réseau vers ce serveur spécifique** (congestion, lien dégradé, ou éventuellement un problème de charge/traitement côté serveur provoquant des délais suffisants pour déclencher des retransmissions par timeout), plutôt qu'un problème de **contrôle de flux côté client** — cette dernière cause est explicitement **réfutée** par l'absence de fenêtre nulle : si le client (récepteur dans le sens serveur→client, pour du contenu web) était saturé et incapable de traiter les données reçues, on s'attendrait à observer des fenêtres annoncées proches de zéro (rappel module 10, section 7.4), ce qui n'est pas le cas ici. Le problème n'est donc probablement pas lié à une lenteur de traitement local du navigateur/client, mais bien au chemin réseau ou au serveur lui-même. Vérification complémentaire proposée : examiner le délai entre l'envoi initial d'un segment et sa retransmission (visible dans les détails de chaque paquet marqué comme retransmission par Wireshark) pour évaluer s'il s'agit de pertes ponctuelles sur un lien autrement fonctionnel (retransmissions occasionnelles, cohérent avec une perte de paquets classique) ou d'un blocage plus sévère et prolongé (cohérent avec une surcharge sévère du serveur distant) — cette distinction orienterait la suite de l'investigation soit vers une équipe réseau (qualité du lien), soit vers une équipe infrastructure serveur (charge, capacité).

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur réseau junior pour mener une analyse complète d'incidents simulés à partir de captures réseau, en appliquant la méthodologie du general au particulier vue en cours, et en mobilisant l'ensemble des connaissances de l'UE2.

## Architecture

Une machine Linux avec Wireshark (interface graphique) ou `tshark` (ligne de commande) installé, capable de générer et capturer du trafic réel (accès réseau sortant, privilèges de capture).

## Objectifs
- Réaliser une capture réelle et appliquer la méthodologie d'analyse du général au particulier
- Utiliser Follow TCP Stream sur un échange HTTP en clair
- Identifier au moins deux signatures d'anomalie du tableau de la section 7.5, provoquées volontairement
- Développer et tester un script d'analyse automatisée de capture

## Préparation

```bash
which wireshark tshark tcpdump 2>/dev/null
```

## Étapes

### Étape 1 — Capture et vue d'ensemble

```bash
sudo tcpdump -i any -w /tmp/tp_capture.pcap &
TCPDUMP_PID=$!
sleep 1

# Générer un trafic varié pendant la capture
dig example.com > /dev/null
curl -s http://example.com > /dev/null
ping -c 3 8.8.8.8 > /dev/null

sleep 2
sudo kill $TCPDUMP_PID

tshark -r /tmp/tp_capture.pcap -q -z io,phs
```
*Question :* d'après la répartition par protocole obtenue, quelle proportion représente approximativement le trafic DNS par rapport au trafic TCP/HTTP dans cette capture ? Est-ce cohérent avec les actions réalisées ?

### Étape 2 — Follow TCP Stream sur un échange HTTP

*Avec l'interface graphique Wireshark (ouvrir `/tmp/tp_capture.pcap`) :* filtrer sur `http`, clic droit sur une requête → Follow → TCP Stream.

*Alternative en ligne de commande avec tshark :*
```bash
tshark -r /tmp/tp_capture.pcap -Y "http" -T fields -e http.request.full_uri -e http.response.code
```
*Question :* pouvez-vous identifier, dans le flux reconstitué (graphique ou via les champs extraits), l'en-tête `Host:` de la requête HTTP ? À quel module de l'UE2 ce mécanisme d'identification de nom de domaine dans une requête applicative (utile notamment pour l'hébergement mutualisé sur une même adresse IP) se rattache-t-il indirectement ?

### Étape 3 — Provoquer et observer une anomalie du tableau 7.5

```bash
# Provoquer des retransmissions en interrogeant une adresse qui ne répondra pas
sudo tcpdump -i any -w /tmp/tp_anomalie.pcap &
TCPDUMP_PID=$!
sleep 1

timeout 5 curl -s --connect-timeout 4 http://10.255.255.1/ > /dev/null 2>&1 || true

sleep 1
sudo kill $TCPDUMP_PID

tshark -r /tmp/tp_anomalie.pcap -Y "tcp.flags.syn == 1"
```
*Résultat attendu :* plusieurs SYN identiques (même port source/destination) à intervalles croissants, sans jamais recevoir de SYN-ACK — signature de retransmission au niveau de l'établissement de connexion lui-même.

*Question :* combien de tentatives de SYN observez-vous avant l'abandon (`timeout`) ? Ce comportement est-il cohérent avec le mécanisme de retransmission décrit au module 10 ?

### Étape 4 — Script d'analyse automatisée

Reprendre et tester le script de la section 16 du cours (`analyse_capture.sh`) sur les deux fichiers de capture générés :

```bash
bash analyse_capture.sh /tmp/tp_capture.pcap
bash analyse_capture.sh /tmp/tp_anomalie.pcap
```

## Commandes de référence
`tcpdump -w`, `tshark -r`, `tshark -Y`, `tshark -q -z io,phs`, Wireshark (Follow Stream, Statistics)

## Résultats attendus
- Répartition par protocole de l'étape 1, avec commentaire de cohérence
- En-tête Host identifié à l'étape 2 (graphique ou extrait `tshark`)
- Capture des SYN répétés de l'étape 3, avec compte des tentatives
- Sorties du script d'analyse sur les deux captures (étape 4), avec comparaison des résultats entre la capture "normale" et la capture "anomalie"

## Questions

1. Dans l'étape 3, si vous aviez appliqué un filtre de **capture** `tcp port 80` avant de lancer `tcpdump`, plutôt que de tout capturer puis filtrer à l'affichage avec `tshark -Y`, quel risque cela aurait-il pu introduire par rapport à l'objectif de l'exercice ? Reliez votre réponse à la section 7.1 du cours.
2. Le script d'analyse automatisée (étape 4) fournit-il, à lui seul, un diagnostic complet et définitif de la cause d'un incident réseau ? Justifiez en vous appuyant sur la méthodologie en 5 étapes de la section 7.6 du cours.
3. Si l'étape 3 avait été réalisée contre une adresse dont le port 80 est explicitement fermé (plutôt qu'une adresse totalement injoignable comme utilisé ici), quelle différence de comportement observeriez-vous dans la capture (rappel module 10 : distinction refus explicite vs timeout) ?

## Troubleshooting (panne à diagnostiquer)

**Situation fournie :** un étudiant tente de reproduire l'étape 3 du TP mais observe, à sa grande surprise, un seul SYN dans sa capture, sans aucune retransmission visible, alors que la commande `curl` a effectivement mis plusieurs secondes à échouer comme attendu.

**Démarche attendue :**
1. Symptôme : comportement de retransmission attendu (plusieurs SYN) non observé dans la capture, alors que le délai d'échec constaté (`curl` qui prend du temps) suggère bien que quelque chose se passe au niveau réseau
2. Hypothèse : l'interface de capture utilisée (`any` dans la commande fournie) peut, selon la configuration système ou la version de `tcpdump`/`libpcap`, ne pas capturer fidèlement 100% du trafic généré localement dans certains environnements virtualisés ou conteneurisés — ou plus probablement, le délai constaté n'est pas dû à des retransmissions TCP au niveau réseau mais à un **timeout de résolution ou de connexion géré directement par `curl`** lui-même sans réellement générer de trafic réseau supplémentaire au-delà du premier SYN (par exemple si la route vers l'adresse cible est immédiatement rejetée localement par la table de routage, sans qu'aucun paquet ne soit réellement émis sur le réseau)
3. Vérification : comparer l'adresse cible utilisée avec la table de routage locale (`ip route get 10.255.255.1`) pour déterminer si un paquet est réellement censé être émis sur le réseau, ou si une route locale provoque un rejet immédiat sans émission réelle
4. Isolation : si la route locale explique un rejet immédiat (par exemple une route "unreachable" explicitement configurée, ou l'absence de route par défaut couvrant cette plage), le comportement observé (un seul SYN ou aucun trafic réseau réel) est parfaitement cohérent et n'est pas une anomalie de capture
5. Correction pédagogique : adapter l'adresse cible utilisée pour l'exercice à une adresse réellement routable mais volontairement non-répondante (par exemple une adresse dans un sous-réseau existant mais actuellement inutilisée, plutôt qu'une adresse arbitraire potentiellement rejetée localement), pour garantir un comportement de retransmission réellement observable sur le réseau
6. Vérification : relancer l'étape 3 avec une adresse cible corrigée, confirmer l'apparition des retransmissions attendues
7. Documentation : noter ce cas comme rappel qu'un comportement réseau "attendu en théorie" dépend toujours du contexte de routage local réel (module 9) — un exercice de laboratoire doit être conçu en tenant compte de cette réalité, pas seulement du protocole étudié en isolation

## Validation

Le TP est validé si l'étudiant fournit :
- Les 4 résultats/captures demandés
- Des réponses argumentées aux 3 questions
- Une identification correcte de la cause du scénario de troubleshooting (rejet local par la table de routage plutôt qu'un vrai comportement réseau de retransmission), avec correction proposée cohérente

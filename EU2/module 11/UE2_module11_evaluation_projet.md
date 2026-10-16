# UE2 — Module 11 : UDP et ICMP
## Compléments : ÉVALUATION / PROJET

---

# PARTIE I — ÉVALUATION

**Durée :** 1h30
**Modalités :** documents non autorisés sauf `man` local si épreuve sur machine
**Barème total :** /20

## Section A — QCM (4 points, 0,5 pt/question)

1. L'en-tête UDP fait au minimum :
 a) 20 octets b) 8 octets c) 4 octets d) 40 octets

2. UDP, contrairement à TCP, ne garantit pas :
 a) le multiplexage par port b) la livraison et l'ordre des données c) le calcul d'une somme de contrôle d) l'encapsulation dans IP

3. ICMP appartient à :
 a) la couche 4 (Transport), comme TCP et UDP b) la couche 3 (Réseau), comme IP c) la couche 7 (Application) d) la couche 2 (Liaison de données)

4. Le message ICMP "Time Exceeded" est émis quand :
 a) une connexion UDP expire b) le TTL d'un paquet atteint zéro avant d'arriver à destination c) un port TCP est fermé d) le DNS ne répond pas

5. `traceroute` exploite le mécanisme TTL/Time Exceeded en :
 a) envoyant un seul paquet avec un TTL maximal b) envoyant des paquets avec un TTL croissant pour forcer une réponse de chaque routeur intermédiaire c) désactivant totalement le TTL d) utilisant exclusivement TCP

6. Un `ping` sans réponse vers un serveur dont le service web fonctionne normalement s'explique le plus probablement par :
 a) une panne réseau généralisée b) un filtrage ICMP délibéré, sans rapport avec la disponibilité du service web c) une erreur de DNS d) un problème de couche 2 uniquement

7. Un blocage ICMP total en bordure de réseau peut casser silencieusement :
 a) le routage IP lui-même b) la découverte de MTU par chemin (Path MTU Discovery) c) la résolution ARP d) le protocole TCP entièrement

8. Les attaques par amplification DNS exploitent principalement :
 a) une faille du protocole TCP b) l'absence de vérification de l'adresse source propre à UDP c) une vulnérabilité d'ICMP uniquement d) le protocole ARP

## Section B — Questions ouvertes (6 points)

**Q1 (2 pts).** Comparez UDP et TCP sur trois critères précis (fiabilité, overhead d'en-tête, établissement de connexion), et donnez un exemple d'application justifiant le choix d'UDP.

**Q2 (2 pts).** Expliquez le mécanisme par lequel `traceroute` révèle chaque routeur intermédiaire d'un chemin réseau, en citant le champ IP et le message ICMP impliqués.

**Q3 (2 pts).** Expliquez pourquoi une attaque par amplification DNS est structurellement plus facile à mener via UDP que via TCP, en vous appuyant sur le mécanisme du three-way handshake (rappel module 10).

## Section C — Exercice technique (4 points)

Donnez la commande permettant de capturer spécifiquement le trafic ICMP sur une interface réseau, ainsi que la commande permettant de lister les sockets UDP actuellement en écoute sur une machine Linux.

## Section D — Problème de diagnostic (6 points)

Après le déploiement d'un pare-feu bloquant tout ICMP entrant, des transferts de fichiers volumineux entre deux sites distants commencent à échouer silencieusement, tandis que la navigation web légère continue de fonctionner normalement.

**a) (2 pts)** Formulez l'hypothèse la plus probable expliquant ce phénomène, en nommant précisément le mécanisme cassé.

**b) (2 pts)** Expliquez pourquoi ce phénomène n'affecte que les transferts volumineux et pas la navigation web légère.

**c) (2 pts)** Proposez une correction qui rétablit la fonctionnalité cassée sans revenir à un blocage ICMP totalement désactivé.

## Grille de correction résumée

| Section | Points | Critères |
|---|---|---|
| A — QCM | /4 | Réponse exacte uniquement |
| B — Questions ouvertes | /6 | Précision technique, comparaison structurée, mécanisme correctement nommé |
| C — Exercice technique | /4 | Deux commandes exactes et complètes |
| D — Diagnostic | /6 | Hypothèse cohérente (PMTU), explication du caractère sélectif du symptôme, correction ciblée plutôt que binaire |

---

# PARTIE II — MINI-PROJET (échelle module)

## Cahier des charges

**Titre :** Diagnostiqueur ICMP/UDP contextuel (Contextual Reachability Diagnoser)

**Contexte :** Suite au constat du TP (les tests ICMP génériques et les tests UDP bruts ont des limites structurelles bien identifiées), l'équipe réseau souhaite un outil de diagnostic plus rigoureux que le script de démonstration du cours, capable d'exploiter des tests applicatifs spécifiques quand ils existent, et de signaler honnêtement ses limites quand ce n'est pas le cas.

### Objectifs du livrable

Développer un outil (Bash ou Python, au choix justifié) qui, pour une cible donnée :
1. Effectue un test ICMP (ping) sans jamais l'interpréter isolément comme preuve définitive de panne ou de disponibilité
2. Effectue un `traceroute` limité et identifie précisément le ou les sauts où les réponses ICMP s'interrompent, en distinguant un saut muet isolé (suivi de sauts qui répondent, cf. TP) d'un arrêt complet en fin de chemin
3. Dispose d'un système de **plugins de test applicatif** pour au moins deux protocoles UDP connus (DNS obligatoire ; un second au choix parmi NTP, SNMP simple ou un protocole applicatif interne documenté), avec un résultat fiable pour ces cas précis
4. Pour tout autre port/service UDP sans plugin dédié, signale explicitement l'incertitude structurelle du test (pas de fausse certitude, conformément à la limite documentée en cours et en TP)
5. Produit un rapport final distinguant clairement trois catégories : "confirmé disponible" (test applicatif concluant), "confirmé indisponible" (refus explicite ou timeout applicatif concluant), "indéterminé" (ICMP seul, ou UDP sans plugin)
6. Journalise chaque exécution avec horodatage

### Contraintes techniques
- Ne jamais présenter un résultat "indéterminé" comme "confirmé" — la distinction des trois catégories doit être stricte et visible dans le rapport
- Architecture extensible : ajouter un nouveau plugin de test applicatif UDP ne doit pas nécessiter de modifier la logique centrale de l'outil
- Gestion d'erreurs robuste (cible invalide, timeout, permissions insuffisantes pour `ping`/`traceroute` bruts)
- Documentation d'installation et d'utilisation (README), incluant comment ajouter un nouveau plugin
- Tests démontrés sur au moins 3 scénarios : service DNS (confirmé disponible), cible avec ICMP filtré mais service applicatif fonctionnel (reproduisant le TD 3.1/TP), service UDP sans plugin dédié (doit être rapporté "indéterminé", pas faussement "disponible" ou "indisponible")

### Livrables attendus
- Code source commenté, avec au moins un plugin DNS fonctionnel et un second plugin
- README (installation, utilisation, ajout d'un plugin, format du rapport)
- Journal de tests montrant les 3 scénarios validés, avec captures
- Rapport court (1 page) sur l'architecture de plugins retenue et les limites structurelles assumées de l'outil (catégorie "indéterminé")

### Grille d'évaluation

| Critère | Pondération |
|---|---|
| Fiabilité des tests ICMP/traceroute avec interprétation prudente | 20% |
| Architecture de plugins extensible, au moins 2 protocoles couverts | 25% |
| Distinction stricte confirmé/indéterminé dans le rapport | 25% |
| Robustesse et gestion d'erreurs | 15% |
| Documentation et tests démontrés (3 scénarios) | 15% |

### Modalité
Travail individuel ou en binôme, durée indicative : 1 semaine. Cet outil complète la série de diagnostic réseau initiée au module 7 (sonde multi-couches) — l'architecture de plugins pourra servir de base à un outil de diagnostic applicatif plus large dans les modules réseau suivants (DNS, DHCP).

# UE2 — Module 7 : Modèle OSI et TCP/IP
## Compléments : ÉVALUATION / PROJET

---

# PARTIE I — ÉVALUATION

**Durée :** 1h30
**Modalités :** documents non autorisés sauf `man` local si épreuve sur machine
**Barème total :** /20

## Section A — QCM (4 points, 0,5 pt/question)

1. La couche OSI responsable de l'adressage physique local (MAC) est :
 a) couche 1 (Physique) b) couche 2 (Liaison de données) c) couche 3 (Réseau) d) couche 4 (Transport)

2. Le modèle TCP/IP, comparé au modèle OSI :
 a) comporte davantage de couches b) regroupe les couches 5, 6 et 7 d'OSI en une seule couche Application c) n'a pas de couche Transport d) est purement théorique, jamais implémenté

3. La PDU manipulée à la couche Liaison de données est appelée :
 a) segment b) paquet c) trame d) datagramme

4. Un message "Connection refused" indique typiquement que :
 a) le câble réseau est débranché b) le paquet a atteint la cible mais aucun service n'écoute sur le port demandé c) le DNS est en panne d) le chiffrement TLS a échoué

5. Sur un trajet A→R1→R2→B (deux routeurs intermédiaires), l'adresse qui reste identique de bout en bout est :
 a) l'adresse MAC source b) l'adresse IP source c) les deux changent à chaque saut d) aucune des deux ne change

6. Un test `ping` qui échoue alors que le service web de la même machine est parfaitement accessible en HTTP indique le plus probablement :
 a) une panne réseau généralisée b) un filtrage spécifique du protocole ICMP, sans rapport avec une vraie panne c) un problème de couche 7 uniquement d) une erreur DNS

7. Un pare-feu qui bloque silencieusement (drop) un port, sans réponse explicite, provoque côté client :
 a) un message "Connection refused" b) un timeout (absence de réponse) c) une erreur DNS d) un message applicatif HTTP 403

8. Le principal intérêt du modèle OSI aujourd'hui, malgré son écart avec les implémentations réelles, est :
 a) sa conformité stricte à toute pile réseau moderne b) un vocabulaire de diagnostic partagé entre ingénieurs c) le remplacement du modèle TCP/IP d) la suppression du besoin d'encapsulation

## Section B — Questions ouvertes (6 points)

**Q1 (2 pts).** Expliquez le principe d'encapsulation en citant, dans l'ordre, les en-têtes ajoutés lorsqu'une application envoie des données via TCP/IP jusqu'à la transmission physique.

**Q2 (2 pts).** Un ingénieur junior conclut à une panne réseau généralisée sur la seule base d'un `ping` en échec. Expliquez pourquoi cette conclusion peut être erronée, et quelle vérification complémentaire aurait dû être effectuée avant de conclure.

**Q3 (2 pts).** Expliquez pourquoi l'adresse MAC change à chaque saut de routeur alors que l'adresse IP reste identique, en reliant votre réponse à la portée respective de ces deux systèmes d'adressage (locale vs globale).

## Section C — Exercice technique (4 points)

Un `tcpdump -X` capture une trame contenant successivement : un en-tête Ethernet, un en-tête IP, un en-tête TCP, puis des données applicatives HTTP. Nommez, dans l'ordre où elles apparaîtraient dans le flux d'octets capturé, les quatre couches (au sens TCP/IP) traversées par ces données, et précisez laquelle contient l'information de port de destination.

## Section D — Problème de diagnostic (6 points)

Un client tente de se connecter à un serveur applicatif sur le port 443 (HTTPS) et sur le port 22 (SSH). Le port 443 répond par un timeout (aucune réponse), tandis que le port 22 répond par "Connection refused" explicite.

**a) (2 pts)** Que peut-on déjà conclure avec certitude sur l'état des couches basses (1 à 3) entre le client et le serveur ? Justifiez.

**b) (2 pts)** Formulez l'hypothèse la plus probable expliquant la différence de comportement entre les deux ports, en vous appuyant sur la distinction refus explicite / silence vue en cours.

**c) (2 pts)** Proposez une commande ou un test permettant de confirmer ou d'infirmer cette hypothèse.

## Grille de correction résumée

| Section | Points | Critères |
|---|---|---|
| A — QCM | /4 | Réponse exacte uniquement |
| B — Questions ouvertes | /6 | Précision technique, raisonnement multi-couches correctement articulé |
| C — Exercice technique | /4 | Ordre des couches correct, identification correcte de la couche transportant le port |
| D — Diagnostic | /6 | Raisonnement cohérent avec les mécanismes du cours, hypothèse plausible et testable |

---

# PARTIE II — MINI-PROJET (échelle module)

## Cahier des charges

**Titre :** Sonde de diagnostic multi-couches (Layered Connectivity Probe)

**Contexte :** L'équipe réseau souhaite un outil de première intervention permettant à n'importe quel technicien, même peu expérimenté, d'obtenir rapidement un diagnostic structuré par couche lors d'un incident de connectivité — pour éviter le piège méthodologique observé en TP (conclure à tort à une panne réseau généralisée sur la base d'un simple `ping`).

### Objectifs du livrable

Développer un outil (Bash ou Python, au choix justifié) qui, pour une cible (IP/nom d'hôte) et un port fournis :
1. Teste la résolution de nom si une adresse non-IP est fournie (anticipation du module DNS à venir — peut être une simple résolution basique)
2. Teste la joignabilité couche 3 (`ping`), sans jamais interpréter un échec ICMP comme une conclusion définitive
3. Teste l'ouverture du port applicatif couche 4, indépendamment du résultat du test couche 3
4. Si couche 3 échoue mais couche 4 réussit, signale explicitement le cas "ICMP probablement filtré, service accessible" plutôt que de considérer cela comme une anomalie
5. Si couche 4 échoue, distingue explicitement dans le rapport "Connection refused" (service arrêté ou pare-feu actif explicite) de "timeout" (pare-feu silencieux ou véritable problème de routage/couche basse)
6. Produit un rapport lisible identifiant clairement la couche la plus probable en cause, avec le niveau de certitude associé
7. Journalise chaque exécution avec horodatage

### Contraintes techniques
- Ne jamais formuler de conclusion catégorique sur la seule base d'un test ICMP
- Gestion robuste des erreurs (cible invalide, timeout réseau, permissions insuffisantes pour `ping` brut si applicable)
- Configuration externalisée pour les timeouts (valeurs par défaut raisonnables, ajustables)
- Documentation d'installation et d'utilisation (README)
- Tests démontrés sur au moins 3 scénarios distincts : cible pleinement accessible, cible avec ICMP filtré mais service accessible (reproduisant le piège du TP), cible avec port explicitement refusé

### Livrables attendus
- Code source commenté
- README (installation, utilisation, interprétation du rapport produit)
- Journal de tests montrant les 3 scénarios validés avec captures
- Rapport court (1 page) expliquant comment l'outil évite structurellement le piège méthodologique identifié en cours et en TP, et ses limites connues (ex : ne distingue pas encore un pare-feu applicatif d'un pare-feu réseau, sujet des modules suivants)

### Grille d'évaluation

| Critère | Pondération |
|---|---|
| Tests multi-couches corrects (3, 4, distinction refus/timeout) | 30% |
| Non-interprétation abusive d'un échec ICMP isolé | 20% |
| Clarté et pertinence du rapport produit | 20% |
| Robustesse et configuration externalisée | 15% |
| Documentation et rapport | 10% |
| Tests démontrés (3 scénarios) | 5% |

### Modalité
Travail individuel ou en binôme, durée indicative : 1 semaine. Cet outil marque le début d'une nouvelle série d'outils réseau qui s'enrichira au fil des modules UE2 suivants (résolution DNS avancée, analyse de trafic Wireshark, diagnostic de routage) — une architecture pensée pour être étendue est valorisée mais non obligatoire à ce stade.

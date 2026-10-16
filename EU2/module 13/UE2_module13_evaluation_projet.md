# UE2 — Module 13 : Analyse de Trafic Réseau avec Wireshark
## Compléments : ÉVALUATION / PROJET

---

# PARTIE I — ÉVALUATION

**Durée :** 2h (module de synthèse — durée allongée, épreuve possiblement basée sur l'analyse d'une capture fournie)
**Modalités :** documents non autorisés sauf `man` local ; si épreuve sur machine, un fichier `.pcap` peut être fourni pour la Section D
**Barème total :** /20

## Section A — QCM (4 points, 0,5 pt/question)

1. Un filtre de capture mal configuré, excluant un trafic recherché :
 a) peut être corrigé après coup sans recapturer b) entraîne une perte définitive de ce trafic non enregistré c) n'affecte que l'affichage d) double automatiquement le volume capturé

2. Le filtre d'affichage isolant les segments TCP retransmis est :
 a) `tcp.retransmit == 1` b) `tcp.analysis.retransmission` c) `tcp.flags.retrans` d) `retransmission == true`

3. "Follow TCP Stream" appliqué à du trafic HTTPS/TLS :
 a) révèle systématiquement le contenu applicatif en clair b) ne révèle pas le contenu chiffré sans déchiffrement explicite c) est impossible techniquement d) ne fonctionne que pour DNS

4. Statistics → Conversations sert principalement à :
 a) modifier le contenu des paquets b) identifier les échanges les plus significatifs entre paires d'adresses/ports c) désactiver un protocole d) chiffrer une capture

5. Une IP associée à plusieurs adresses MAC différentes, changeant régulièrement en quelques minutes dans une capture, oriente vers :
 a) un fonctionnement DHCP normal b) une possible attaque ARP spoofing ou un conflit d'adresse IP c) un problème de couche 7 uniquement d) une anomalie DNS

6. Un volume élevé de messages ICMP Time Exceeded dans une capture est :
 a) toujours le signe d'une boucle de routage b) à contextualiser, normal si la capture couvre un ou plusieurs traceroute c) impossible en pratique d) toujours lié à un problème DHCP

7. L'absence de `tcp.analysis.zero_window` dans une capture montrant des retransmissions permet d'exclure :
 a) toute cause réseau b) un problème de contrôle de flux côté récepteur comme cause de ces retransmissions c) toute possibilité de perte de paquets d) l'existence même de la connexion

8. La méthodologie d'analyse de capture recommandée en cours va :
 a) toujours du particulier vers le général b) du général (vue d'ensemble) vers le particulier (filtrage ciblé), avec corrélation multi-couches c) uniquement par ordre chronologique des paquets d) uniquement par adresse IP croissante

## Section B — Questions ouvertes (6 points)

**Q1 (2 pts).** Expliquez la différence entre filtre de capture et filtre d'affichage, avec un exemple concret de conséquence pratique d'une confusion entre les deux dans un scénario de diagnostic réel.

**Q2 (2 pts).** Décrivez la méthodologie d'analyse d'une capture inconnue en 5 étapes vue en cours, en expliquant pourquoi l'ordre proposé (du général au particulier) est préférable à un examen paquet par paquet dès le départ.

**Q3 (2 pts).** Une capture montre des retransmissions TCP concentrées sur les connexions vers un seul serveur, sans aucune fenêtre nulle observée. Expliquez pourquoi l'absence de fenêtre nulle permet d'exclure une hypothèse précise, et laquelle.

## Section C — Exercice technique (4 points)

Donnez les filtres d'affichage Wireshark permettant respectivement de : isoler tout le trafic DNS ; isoler les segments TCP correspondant à un SYN initial (sans ACK) ; isoler les messages ICMP de type 3 (Destination Unreachable) ; isoler le trafic impliquant l'adresse IP `10.0.0.5`, tous protocoles confondus.

## Section D — Analyse de capture / problème de diagnostic (6 points)

*(Si épreuve sur machine avec fichier `.pcap` fourni : appliquer directement à la capture. Sinon, traiter comme un cas décrit ci-dessous.)*

Une capture de 10 minutes est fournie pour un incident où un utilisateur rapporte "je n'arrive pas à me connecter au serveur de fichiers ce matin". L'analyse Protocol Hierarchy révèle une majorité de trafic ARP et DHCP, avec très peu de trafic TCP. Le filtre `arp` révèle de nombreuses requêtes ARP Request pour l'adresse de la passerelle, sans qu'aucune réponse ARP Reply ne soit visible dans la capture.

**a) (2 pts)** Formulez une hypothèse cohérente avec ces observations, en identifiant à quelle couche se situe le problème.

**b) (2 pts)** Cette observation exclut-elle un problème purement applicatif (couche 7, configuration du client du serveur de fichiers) ? Justifiez.

**c) (2 pts)** Proposez une vérification complémentaire, hors de cette capture, permettant de confirmer l'hypothèse formulée en (a).

## Grille de correction résumée

| Section | Points | Critères |
|---|---|---|
| A — QCM | /4 | Réponse exacte uniquement |
| B — Questions ouvertes | /6 | Précision technique, méthodologie correctement articulée |
| C — Exercice technique | /4 | 4 filtres exacts |
| D — Analyse/diagnostic | /6 | Hypothèse cohérente avec les observations, raisonnement d'exclusion correct, vérification pertinente |

---

# PARTIE II — MINI-PROJET (échelle module — projet de clôture de l'UE2)

## Cahier des charges

**Titre :** Analyseur de captures et rapport d'incident (Capture Triage & Incident Report Tool)

**Contexte :** Ce mini-projet clôt l'UE2. L'équipe support reçoit régulièrement des captures réseau liées à des incidents vagues ("c'est lent", "je n'arrive pas à me connecter"), et souhaite un outil de premier tri automatisé, produisant un rapport lisible par une personne non experte en réseau, à la manière de l'étude de cas industrielle du cours.

### Objectifs du livrable

Développer un outil (Bash ou Python, `tshark` recommandé comme moteur d'extraction) qui, à partir d'un fichier `.pcap` fourni :
1. Produit une vue d'ensemble (répartition par protocole, durée de la capture, nombre total de paquets)
2. Détecte et quantifie chacune des anomalies du tableau de la section 7.5 du cours (retransmissions TCP, fenêtres nulles, ARP sans réponse, IP associée à plusieurs MAC, ICMP Destination Unreachable en excès, DNS sans réponse, DHCPOFFER multiples pour un DISCOVER)
3. Identifie les 5 conversations (paires d'adresses) les plus volumineuses ou les plus longues
4. Pour chaque anomalie détectée, produit une explication en langage clair (pas seulement un chiffre), reliant l'observation à une cause probable, à la manière du tableau du cours
5. Distingue explicitement les résultats "anomalie confirmée avec haute confiance" des résultats "signal à interpréter avec prudence selon le contexte" (rappel TD 3.2 : un volume élevé d'ICMP Time Exceeded n'est pas anormal en soi)
6. Produit un rapport final structuré, dans un format lisible par un non-expert, avec un résumé en tête de document et le détail technique en annexe

### Contraintes techniques
- Aucune anomalie ne doit être présentée comme certaine si le contexte ne permet pas de trancher (cohérence avec la distinction de l'objectif 5)
- Gestion d'erreurs robuste (fichier `.pcap` invalide ou vide, `tshark` non disponible)
- Documentation d'installation et d'utilisation (README)
- Tests démontrés sur au moins 3 captures distinctes : une capture "normale" sans anomalie notable, une capture reproduisant un scénario de retransmission (comme au TP), une capture reproduisant un scénario ARP suspect (comme au module 8) — le rapport produit pour chacune doit être cohérent avec le contenu réel de la capture

### Livrables attendus
- Code source commenté
- README (installation, utilisation, format du rapport, limites connues)
- Journal de tests montrant les 3 scénarios validés, avec le rapport complet généré pour chacun
- Rapport court (1 page) sur les choix techniques (extraction via `tshark`, structuration du rapport final) et une réflexion sur les limites de l'automatisation en diagnostic réseau — pourquoi cet outil ne remplace jamais un ingénieur, mais accélère son premier tri

### Grille d'évaluation

| Critère | Pondération |
|---|---|
| Détection correcte des anomalies du tableau 7.5 | 30% |
| Distinction confirmé / à interpréter avec prudence | 20% |
| Clarté du rapport pour un lecteur non expert | 20% |
| Robustesse et gestion d'erreurs | 15% |
| Documentation et tests démontrés (3 captures) | 15% |

### Modalité
Travail individuel ou en binôme, durée indicative : 1,5 à 2 semaines (projet de clôture, volume plus important).

**Clôture de l'UE2 :** ce projet peut, à la discrétion de l'enseignant, être proposé comme brique finale d'un projet intégrateur combinant l'ensemble de l'UE2 — reprenant les captures générées lors des scénarios des modules 7 à 12 et les analysant avec cet outil, démontrant la cohérence de bout en bout des compétences acquises sur l'unité. Ce projet fait écho au **Projet 3** du programme général de la formation (infrastructure réseau et analyse Wireshark).

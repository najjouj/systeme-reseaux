# UE2 — Module 8 : Ethernet, ARP et Commutation
## Compléments : ÉVALUATION / PROJET

---

# PARTIE I — ÉVALUATION

**Durée :** 1h30
**Modalités :** documents non autorisés sauf `man` local si épreuve sur machine
**Barème total :** /20

## Section A — QCM (4 points, 0,5 pt/question)

1. Le champ FCS d'une trame Ethernet sert à :
 a) chiffrer les données b) détecter (sans corriger) une corruption de la trame c) indiquer le protocole encapsulé d) router la trame vers un autre réseau

2. Un switch qui reçoit une trame dont l'adresse MAC destination est inconnue de sa table :
 a) la rejette b) l'inonde sur tous les ports sauf celui d'origine c) envoie une requête ARP d) attend indéfiniment

3. ARP traduit :
 a) un nom de domaine en IP b) une adresse IP en adresse MAC c) un port en service d) une adresse MAC en IP

4. La principale faiblesse native du protocole ARP est :
 a) sa lenteur b) l'absence d'authentification des réponses c) son incompatibilité avec IPv6 d) sa consommation excessive de bande passante

5. Deux machines connectées au même switch physique mais sur deux VLAN différents :
 a) peuvent communiquer directement en ping b) ne peuvent pas communiquer sans routage inter-VLAN c) partagent automatiquement la même table MAC d) sont dans le même domaine de collision uniquement

6. Une boucle physique entre deux switchs sans protection STP provoque typiquement :
 a) une simple lenteur mineure b) une tempête de diffusion pouvant paralyser tout le réseau c) un changement automatique d'adresses MAC d) aucun effet si le trafic est faible

7. Le trunking 802.1Q permet :
 a) de chiffrer le trafic entre deux switchs b) de faire transiter plusieurs VLAN sur un seul lien physique c) de remplacer ARP d) de fusionner deux domaines de collision

8. Une entrée de cache ARP en état `STALE` sous Linux signifie :
 a) une résolution qui a définitivement échoué b) une association potentiellement obsolète, qui sera revérifiée au prochain usage c) une entrée protégée contre toute modification d) une entrée appartenant à un autre VLAN

## Section B — Questions ouvertes (6 points)

**Q1 (2 pts).** Décrivez les trois mécanismes de fonctionnement d'un switch (apprentissage, inondation, commutation) et expliquez pourquoi ce comportement le différencie fondamentalement d'un simple hub.

**Q2 (2 pts).** Expliquez pourquoi ARP est vulnérable à l'usurpation, en identifiant précisément la propriété du protocole qui rend cela possible, et citez une conséquence pratique de cette vulnérabilité.

**Q3 (2 pts).** Un administrateur ajoute un lien physique redondant entre deux switchs sans avoir vérifié que STP était actif. Expliquez le phénomène qui peut en résulter, en décrivant le mécanisme de multiplication à l'œuvre.

## Section C — Exercice technique (4 points)

Donnez la séquence de commandes Linux permettant de : vider le cache ARP local, effectuer un ping vers une adresse donnée pour déclencher une nouvelle résolution, puis afficher le cache ARP mis à jour.

## Section D — Problème de diagnostic (6 points)

Une machine nouvellement connectée à un réseau d'entreprise, avec une interface `UP` et une adresse IP valide, ne parvient à joindre aucune autre machine, y compris celles du même sous-réseau physique.

**a) (2 pts)** Donnez la première commande à exécuter pour orienter le diagnostic, et précisez ce que vous en attendez.

**b) (2 pts)** En supposant que le `ping` échoue et qu'aucune requête ARP ne semble partir sur le réseau (confirmé par capture), formulez une hypothèse cohérente avec les mécanismes vus en cours, distincte d'une simple panne de câble.

**c) (2 pts)** Proposez une vérification côté infrastructure (switch) permettant de confirmer ou infirmer cette hypothèse.

## Grille de correction résumée

| Section | Points | Critères |
|---|---|---|
| A — QCM | /4 | Réponse exacte uniquement |
| B — Questions ouvertes | /6 | Précision technique, mécanisme correctement expliqué |
| C — Exercice technique | /4 | Séquence de commandes correcte et complète |
| D — Diagnostic | /6 | Démarche méthodique, hypothèse plausible orientée VLAN, vérification pertinente |

---

# PARTIE II — MINI-PROJET (échelle module)

## Cahier des charges

**Titre :** Surveillant de cache ARP (ARP Cache Sentinel)

**Contexte :** Suite à la sonde de diagnostic multi-couches développée au module 7, l'équipe réseau souhaite un second outil, dédié spécifiquement à la surveillance de la couche 2 — capable de détecter des changements suspects d'association IP↔MAC, tout en évitant les pièges méthodologiques rencontrés en TP (faux positifs au redémarrage dus à un état non persistant).

### Objectifs du livrable

Développer un outil (Bash ou Python, au choix justifié) qui :
1. Surveille en continu le cache ARP local et détecte tout changement d'association IP↔MAC
2. Stocke son état de référence dans un emplacement **persistant** (pas `/tmp`), corrigeant explicitement le piège identifié au TP
3. Maintient une liste blanche configurable d'associations IP/MAC critiques (au minimum la passerelle par défaut), pour laquelle toute déviation déclenche une alerte de priorité plus élevée que pour une IP quelconque
4. Distingue, dans son rapport, une "première apparition" légitime (IP jamais vue auparavant) d'un "changement réel" (IP déjà connue avec un MAC différent) — ne traite jamais les deux cas de la même façon
5. Journalise chaque détection avec horodatage, ancienne et nouvelle adresse MAC le cas échéant
6. Documente explicitement ses limites (faux positifs possibles lors d'un remplacement matériel légitime, portée limitée à la machine locale)

### Contraintes techniques
- Gestion robuste des erreurs (premier lancement sans état de référence, interface réseau absente)
- Configuration externalisée (liste blanche, intervalle de surveillance, emplacement de stockage d'état)
- Documentation d'installation et d'utilisation (README)
- Tests démontrés sur au moins 3 scénarios : premier lancement (aucune fausse alerte), changement légitime simulé sur une IP quelconque, changement simulé sur une IP de la liste blanche (alerte prioritaire)

### Livrables attendus
- Code source commenté
- README (installation, configuration, format du rapport, limites connues)
- Journal de tests montrant les 3 scénarios validés avec captures
- Rapport court (1 page) sur les choix techniques, notamment la correction du piège de persistance identifié en TP, et une réflexion sur ce que l'outil ne peut pas détecter (attaque visant une autre machine que celle surveillée)

### Grille d'évaluation

| Critère | Pondération |
|---|---|
| Détection correcte des changements ARP | 20% |
| Persistance correcte de l'état (pas de faux positif au redémarrage) | 20% |
| Distinction première apparition / changement réel | 15% |
| Gestion de la liste blanche avec priorité différenciée | 15% |
| Documentation, limites explicitées | 15% |
| Tests démontrés (3 scénarios) | 15% |

### Modalité
Travail individuel ou en binôme, durée indicative : 1 semaine. Cet outil s'inscrit dans la série d'outils réseau initiée au module 7 (Layered Connectivity Probe) — une architecture permettant de partager la configuration ou le format de rapport entre les deux outils est valorisée mais non obligatoire.

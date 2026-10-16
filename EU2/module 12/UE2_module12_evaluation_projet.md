# UE2 — Module 12 : DNS et DHCP
## Compléments : ÉVALUATION / PROJET

---

# PARTIE I — ÉVALUATION

**Durée :** 1h30
**Modalités :** documents non autorisés sauf `man` local si épreuve sur machine
**Barème total :** /20

## Section A — QCM (4 points, 0,5 pt/question)

1. Un résolveur DNS récursif, vu du côté client, retourne :
 a) uniquement une référence vers le serveur suivant b) la réponse finale complète en une seule requête côté client c) systématiquement une erreur si le TLD est inconnu d) uniquement des enregistrements MX

2. Un enregistrement CNAME :
 a) associe directement un nom à une adresse IPv4 b) associe un nom à un autre nom c) indique le serveur de messagerie d'un domaine d) sert exclusivement à la résolution inverse

3. Le TTL d'un enregistrement DNS détermine :
 a) la vitesse de la connexion réseau b) la durée pendant laquelle un résolveur peut conserver la réponse en cache c) le nombre de sauts autorisés d) la taille maximale de la réponse

4. Le message DHCPDISCOVER est envoyé en broadcast car :
 a) c'est une exigence esthétique du protocole b) le client ne dispose pas encore d'une configuration IP pour cibler un serveur précis c) cela accélère artificiellement le débit d) DHCP n'existe qu'en broadcast

5. Un bail DHCP :
 a) est attribué de façon permanente et définitive b) est attribué pour une durée limitée et doit être renouvelé c) ne concerne que l'adresse IP, jamais la passerelle ni le DNS d) est toujours identique pour tous les clients

6. Une propagation lente d'un changement DNS après une migration, malgré une modification déjà effective côté serveur faisant autorité, s'explique généralement par :
 a) une panne du serveur faisant autorité b) le cache DNS des résolveurs, lié au TTL précédemment servi c) une erreur de type d'enregistrement d) un problème de couche 2

7. Un serveur DHCP non autorisé branché sur un réseau d'entreprise (rogue DHCP) peut affecter :
 a) tous les utilisateurs de façon identique et systématique b) seulement une partie des utilisateurs, de façon dépendante du client DHCP ayant retenu son offre c) uniquement les serveurs, jamais les postes clients d) uniquement le trafic HTTPS

8. Le DHCP snooping, activé sur un switch manageable, permet de :
 a) accélérer la résolution DNS b) n'autoriser les réponses DHCP (OFFER/ACK) que depuis des ports désignés comme fiables c) remplacer entièrement DNSSEC d) supprimer le besoin de bail DHCP

## Section B — Questions ouvertes (6 points)

**Q1 (2 pts).** Expliquez la différence entre résolution récursive et résolution itérative, en précisant qui effectue chacune dans la chaîne de résolution DNS.

**Q2 (2 pts).** Décrivez le processus DHCP DORA en 4 étapes, en expliquant pourquoi les deux premières étapes utilisent le broadcast.

**Q3 (2 pts).** Un rogue DHCP server est branché sur un réseau d'entreprise disposant déjà d'un serveur légitime. Expliquez pourquoi seuls certains utilisateurs sont affectés, et non l'ensemble du réseau de façon uniforme.

## Section C — Exercice technique (4 points)

Donnez la commande permettant d'observer la résolution DNS complète et itérative (racine → TLD → autorité) pour un domaine donné, ainsi que la commande permettant de capturer spécifiquement le trafic DHCP sur une interface réseau.

## Section D — Problème de diagnostic (6 points)

Après une migration planifiée d'un service vers une nouvelle adresse IP, avec mise à jour immédiate de l'enregistrement DNS A, une partie des utilisateurs continue d'être dirigée vers l'ancienne adresse plusieurs heures après la migration, malgré une confirmation que le changement est bien appliqué côté serveur faisant autorité.

**a) (2 pts)** Formulez l'hypothèse la plus probable expliquant ce phénomène, sans invoquer d'erreur de configuration côté serveur.

**b) (2 pts)** Proposez une commande permettant de vérifier cette hypothèse en interrogeant directement le comportement de cache observé.

**c) (2 pts)** Proposez une mesure préventive qui aurait dû être appliquée avant la migration pour limiter ce phénomène.

## Grille de correction résumée

| Section | Points | Critères |
|---|---|---|
| A — QCM | /4 | Réponse exacte uniquement |
| B — Questions ouvertes | /6 | Précision technique, mécanismes DORA/récursif-itératif correctement distingués |
| C — Exercice technique | /4 | Deux commandes exactes et complètes |
| D — Diagnostic | /6 | Hypothèse cohérente (TTL/cache), vérification pertinente, mesure préventive correcte (réduction TTL en amont) |

---

# PARTIE II — MINI-PROJET (échelle module)

## Cahier des charges

**Titre :** Surveillant DNS/DHCP (Name & Address Sentinel)

**Contexte :** Suite aux outils réseau développés aux modules précédents, l'équipe souhaite un outil dédié à la santé des services DNS et DHCP — capables tous deux de provoquer des incidents "invisibles" difficiles à diagnostiquer sans outillage (propagation de cache, rogue DHCP), comme démontré en cours et en TD.

### Objectifs du livrable

Développer un outil (Bash ou Python, au choix justifié) qui :
1. **Volet DNS** : surveille une liste configurable de domaines critiques, compare la réponse actuelle à une valeur de référence connue, et détecte tout changement inattendu de l'enregistrement A/AAAA — en gérant explicitement le cas d'un CNAME intermédiaire et d'enregistrements multiples (correction directe du piège de parsing identifié en TP)
2. **Volet DNS — pré-migration** : reprend et étend le script de vérification de TTL du cours, avec un mode "compte à rebours" avertissant si le TTL n'a pas été suffisamment abaissé dans le délai précédant une migration planifiée déclarée
3. **Volet DHCP** : capture et analyse le trafic DHCP sur une interface donnée pendant une fenêtre de temps configurable, et détecte la présence de **plusieurs sources distinctes** émettant des DHCPOFFER pour une même requête DHCPDISCOVER — signature possible d'un rogue DHCP server (référence directe à l'exercice 4.1 du TD)
4. Journalise chaque détection (des deux volets) avec horodatage et détails
5. Génère un rapport quotidien de synthèse séparant clairement les deux volets

### Contraintes techniques
- Le parsing des sorties `dig` doit gérer explicitement CNAME et enregistrements multiples (test dédié obligatoire, reproduisant le bug du TP)
- La détection de rogue DHCP ne doit produire aucun faux positif dans le cas normal d'un seul serveur DHCP légitime répondant à une requête (test dédié obligatoire)
- Gestion d'erreurs robuste (domaine invalide, interface réseau absente, permissions insuffisantes pour la capture)
- Configuration externalisée (liste de domaines, valeurs de référence, fenêtre de capture DHCP)
- Documentation d'installation et d'utilisation (README)
- Tests démontrés sur au moins 3 scénarios : changement DNS légitime détecté correctement malgré un CNAME, environnement DHCP normal (pas de fausse alerte), simulation d'un second répondeur DHCP (peut être simulée par un second processus répondant artificiellement, à documenter comme limite si un vrai second serveur DHCP n'est pas disponible en laboratoire)

### Livrables attendus
- Code source commenté
- README (installation, configuration, format du rapport, limites connues)
- Journal de tests montrant les 3 scénarios validés avec captures
- Rapport court (1 page) sur les choix techniques, notamment la robustesse du parsing DNS (CNAME/multi-A) et la méthode de détection de rogue DHCP retenue, ainsi que les limites connues (ex : ne détecte pas un rogue DHCP répondant délibérément avec un léger délai pour éviter la fenêtre de capture)

### Grille d'évaluation

| Critère | Pondération |
|---|---|
| Détection DNS robuste (gestion CNAME/multi-A) | 25% |
| Vérification TTL pré-migration avec mode compte à rebours | 15% |
| Détection de rogue DHCP sans faux positif en environnement normal | 30% |
| Robustesse et configuration externalisée | 10% |
| Documentation et tests démontrés (3 scénarios) | 20% |

### Modalité
Travail individuel ou en binôme, durée indicative : 1,5 semaine (volume plus important, deux volets distincts). Cet outil complète la série de supervision réseau initiée au module 7 — la clôture de l'UE2 pourra être l'occasion d'assembler l'ensemble des outils réseau développés (modules 7 à 12) en une suite cohérente, sur le modèle de ce qui a été fait en fin d'UE1.

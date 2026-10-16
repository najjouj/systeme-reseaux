# UE2 — Module 9 : IPv4/IPv6, Subnetting, CIDR et Routage Statique
## Compléments : ÉVALUATION / PROJET

---

# PARTIE I — ÉVALUATION

**Durée :** 2h (module calculatoire, durée allongée par rapport aux modules précédents)
**Modalités :** calculatrice simple autorisée (pas de calculatrice programmable), documents non autorisés
**Barème total :** /20

## Section A — QCM (4 points, 0,5 pt/question)

1. Pour un préfixe `/25`, le nombre d'hôtes utilisables est :
 a) 128 b) 126 c) 64 d) 62

2. La première adresse d'un sous-réseau désigne :
 a) la passerelle par défaut obligatoirement b) le réseau lui-même, non attribuable à un hôte c) toujours le serveur DNS d) l'adresse de diffusion

3. Le VLSM permet :
 a) de fusionner plusieurs réseaux en un seul b) de découper un bloc en sous-réseaux de tailles différentes selon les besoins c) de remplacer IPv6 par IPv4 d) d'éviter tout calcul de sous-réseau

4. Une adresse IPv6 link-local utilise le préfixe :
 a) 2001::/16 b) fe80::/10 c) ff00::/8 d) ::1/128

5. IPv6, comparé à IPv4, supprime :
 a) le routage b) l'adresse de diffusion (broadcast), remplacée par le multicast c) les adresses privées d) le masque de sous-réseau

6. Le principe du "longest prefix match" signifie que la table de routage choisit :
 a) la première route ajoutée chronologiquement b) la route dont le préfixe est le plus spécifique parmi celles qui correspondent c) toujours la route par défaut d) la route la plus courte en nombre de caractères

7. Une machine configurée par erreur avec un masque `/32` au lieu de `/24` :
 a) fonctionne normalement, le masque n'a pas d'impact b) se retrouve isolée, incapable de joindre directement les autres machines de son réseau local c) obtient automatiquement plus d'adresses disponibles d) bascule automatiquement en IPv6

8. Une route statique ajoutée avec `ip route add`, sans configuration persistante complémentaire :
 a) survit à tout redémarrage b) disparaît au redémarrage de la machine c) se réplique automatiquement sur toutes les interfaces d) devient une route par défaut

## Section B — Questions ouvertes / calcul (6 points)

**Q1 (2 pts).** Calculez l'adresse réseau, l'adresse de diffusion et la plage d'hôtes utilisables pour `192.168.50.90/27`. Montrez votre calcul.

**Q2 (2 pts).** Une entreprise dispose du bloc `10.0.0.0/23` et doit créer 2 sous-réseaux de taille égale. Donnez le préfixe CIDR résultant et l'adresse réseau de chacun des 2 sous-réseaux.

**Q3 (2 pts).** Expliquez pourquoi IPv6 n'a pas besoin d'une adresse de diffusion, en reliant votre réponse au problème de tempête de diffusion vu au module 8.

## Section C — Exercice technique (4 points)

Une machine dispose de la table de routage suivante :
```
172.16.0.0/16 via 10.0.0.1
172.16.5.0/24 via 10.0.0.2
0.0.0.0/0 via 10.0.0.1
```
Pour chacune des destinations suivantes, indiquez précisément quelle passerelle sera utilisée, en justifiant par le principe vu en cours : `172.16.5.50`, `172.16.9.10`, `8.8.8.8`.

## Section D — Problème de diagnostic (6 points)

Une machine nouvellement configurée, avec une interface `UP` confirmée, ne parvient à joindre aucune autre machine de son réseau local, alors que sa configuration IP semble à première vue correcte (adresse dans la bonne plage attendue).

**a) (2 pts)** Donnez la première commande à exécuter pour vérifier la cohérence complète de la configuration IP (adresse ET masque), et précisez ce qu'une incohérence de masque révélerait.

**b) (2 pts)** En supposant que le masque configuré soit `/32` au lieu du `/24` attendu, expliquez précisément pourquoi cela isole la machine, en vous appuyant sur la notion de partie réseau/partie hôte.

**c) (2 pts)** Proposez un second scénario de cause distinct (n'impliquant pas un masque incorrect) pouvant produire un symptôme similaire, et une commande permettant de le distinguer du cas du masque erroné.

## Grille de correction résumée

| Section | Points | Critères |
|---|---|---|
| A — QCM | /4 | Réponse exacte uniquement |
| B — Questions ouvertes/calcul | /6 | Calcul exact et méthode montrée, pas seulement le résultat final |
| C — Exercice technique | /4 | Application correcte du longest prefix match sur les 3 cas |
| D — Diagnostic | /6 | Raisonnement méthodique, deux causes distinctes correctement différenciées |

---

# PARTIE II — MINI-PROJET (échelle module)

## Cahier des charges

**Titre :** Planificateur d'adressage VLSM (VLSM Address Planner)

**Contexte :** L'équipe réseau souhaite un outil permettant de générer automatiquement un plan d'adressage VLSM cohérent à partir d'un bloc de départ et d'une liste de besoins, pour fiabiliser une tâche aujourd'hui réalisée manuellement et sujette à erreur (chevauchements, sous-dimensionnement).

### Objectifs du livrable

Développer un outil (Bash ou Python, au choix justifié — Python recommandé pour la manipulation d'adresses via le module `ipaddress`) qui :
1. Prend en entrée un bloc CIDR de départ et une liste de besoins en nombre d'hôtes par segment nommé (ex : `serveurs:50, utilisateurs:90, invites:20, liaison:2`)
2. Calcule automatiquement un découpage VLSM valide, en respectant l'ordre décroissant de taille (méthode vue en cours)
3. Vérifie et garantit l'absence de tout chevauchement entre les sous-réseaux calculés
4. Vérifie que le bloc de départ est suffisant ; si ce n'est pas le cas, produit un message d'erreur clair indiquant le déficit d'adresses, plutôt qu'un résultat silencieusement incorrect
5. Produit un rapport listant, pour chaque segment : nom, adresse réseau, préfixe CIDR, masque décimal, adresse de diffusion, plage d'hôtes utilisables, nombre d'hôtes réellement disponibles
6. Exporte ce rapport en deux formats : lisible (texte structuré) et exploitable (CSV ou JSON), pour intégration dans une documentation d'infrastructure ou un futur outil de configuration automatique

### Contraintes techniques
- Gestion d'erreurs robuste (bloc de départ invalide, besoin de taille zéro ou négatif, format d'entrée incorrect)
- Le calcul doit être démontrablement correct sur le cas d'exemple du cours (192.168.100.0/24 avec les 4 segments de l'exercice 4.1 du TD) — comparaison directe attendue avec la correction du TD
- Documentation d'installation et d'utilisation (README)
- Tests démontrés sur au moins 3 scénarios : le cas du TD (doit produire exactement le même résultat), un cas où le bloc de départ est insuffisant (doit échouer proprement), un cas avec un nombre de segments plus élevé (au moins 6)

### Livrables attendus
- Code source commenté
- README (installation, format d'entrée, format de sortie, limites connues)
- Journal de tests montrant les 3 scénarios validés, avec comparaison explicite au résultat attendu du TD pour le premier
- Rapport court (1 page) sur les choix techniques (notamment l'algorithme de vérification de non-chevauchement retenu) et les limites connues (ex : pas de gestion d'IPv6 dans cette première version)

### Grille d'évaluation

| Critère | Pondération |
|---|---|
| Exactitude du calcul VLSM (conformité au cas du TD) | 30% |
| Détection correcte d'un bloc insuffisant | 20% |
| Vérification de non-chevauchement | 20% |
| Formats de sortie (lisible + exploitable) | 15% |
| Documentation et tests démontrés | 15% |

### Modalité
Travail individuel ou en binôme, durée indicative : 1 semaine. Ce planificateur pourra, dans un module ultérieur (automatisation Python, UE7), être étendu pour générer directement des commandes de configuration réseau — une architecture anticipant cette extension (séparation claire entre calcul et présentation du résultat) est valorisée mais non obligatoire à ce stade.

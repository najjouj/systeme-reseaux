# UE2 — Projet Intégrateur de Clôture
## Conception, déploiement, sécurisation et analyse d'une infrastructure réseau complète

*Correspond au **Projet 3** du programme général : "Mettre en place une infrastructure réseau et analyser son trafic avec Wireshark". Mobilise l'intégralité des modules 7 à 13 de l'UE2.*

---

## 1. Titre du projet
De l'adressage au diagnostic : concevoir, déployer et auditer une infrastructure réseau multi-segments

## 2. Niveau
Élève ingénieur — cycle ingénieur, clôture de l'UE2 réseaux (S1)

## 3. Durée
2 à 3 semaines de travail en autonomie (binôme recommandé), avec 3 points de passage encadrés (voir section 10) et une soutenance finale de 20 minutes par binôme.

## 4. Prérequis
L'intégralité de l'UE2 validée (modules 7 à 13) : modèle OSI/TCP-IP, Ethernet/ARP/commutation, IPv4/IPv6/subnetting/routage, TCP en profondeur, UDP/ICMP, DNS/DHCP, analyse Wireshark.

## 5. Contexte du projet

Une PME fictive, **Atelier Nomia**, compte une trentaine d'employés répartis en trois services (Direction/Administration, Production, Support technique) et souhaite structurer son réseau interne, jusque-là composé d'un unique switch non managé où toutes les machines partagent le même sous-réseau plat, sans plan d'adressage documenté, sans serveur DHCP (adresses configurées manuellement), et sans supervision.

Vous intervenez comme équipe d'ingénieurs réseau pour concevoir, déployer (en environnement de laboratoire virtualisé) et documenter la nouvelle infrastructure, puis produire un audit de trafic démontrant sa validité.

## 6. Objectifs pédagogiques

Ce projet doit démontrer la maîtrise intégrée de :
- La conception d'un plan d'adressage IPv4 cohérent par VLSM (module 9)
- La segmentation logique en VLAN et la configuration du routage inter-VLAN (module 8, 9)
- La configuration et le diagnostic de services DHCP avec relais multi-segments (module 12)
- La configuration d'un service DNS interne minimal (module 12)
- La compréhension et la vérification du comportement TCP/UDP/ICMP réel sur l'infrastructure déployée (modules 10, 11)
- L'analyse de trafic pour valider le fonctionnement et détecter des anomalies volontairement introduites (module 13)
- La production d'une documentation technique professionnelle et d'une présentation orale claire

## 7. Cahier des charges

### 7.1 Contraintes fonctionnelles

L'infrastructure doit comporter au minimum :
- **3 segments logiques (VLAN)** correspondant aux 3 services de l'entreprise, plus un segment dédié à l'infrastructure elle-même (serveurs DHCP/DNS)
- Un plan d'adressage IPv4 complet, calculé par VLSM à partir d'un unique bloc de départ `10.10.0.0/22`, dimensionné selon les besoins suivants (à adapter/justifier si l'équipe estime un besoin différent, avec argumentation) :
  - Direction/Administration : 15 machines
  - Production : 80 machines
  - Support technique : 25 machines
  - Infrastructure (serveurs) : 6 adresses
- **Routage inter-VLAN** fonctionnel, permettant aux trois segments utilisateurs d'atteindre le segment infrastructure et, via une route par défaut, un réseau externe simulé (Internet fictif)
- **Un serveur DHCP central** (sur le segment infrastructure) servant les trois VLAN utilisateurs via relais DHCP, avec des plages et options cohérentes (passerelle, DNS) par VLAN
- **Un serveur DNS interne minimal** résolvant au moins un nom de domaine interne fictif (ex : `.atelier-nomia.lan`) vers les adresses des serveurs internes
- Un **pare-feu de base** distinguant au minimum le trafic autorisé entre VLAN (ex : Support technique peut atteindre Production pour maintenance, mais pas l'inverse — à justifier par l'équipe selon une politique de sécurité qu'elle définit elle-même)

### 7.2 Contraintes de validation

L'équipe doit produire, pour chaque brique de l'infrastructure, une preuve de fonctionnement par capture réseau (Wireshark/tcpdump), et non une simple affirmation :
- Capture d'un cycle DHCP DORA complet pour au moins un client de chaque VLAN utilisateur, prouvant le bon fonctionnement du relais
- Capture d'une résolution DNS réussie pour le nom de domaine interne
- Capture d'un `traceroute` (ou équivalent) entre deux VLAN différents, prouvant le passage effectif par le routeur inter-VLAN
- Capture démontrant l'application effective de la politique de pare-feu définie (une tentative de connexion autorisée qui réussit, une tentative interdite qui échoue explicitement)

### 7.3 Contrainte d'incident volontaire (audit)

Une fois l'infrastructure fonctionnelle et documentée, l'équipe doit **introduire volontairement une anomalie** parmi la liste suivante (au choix, une seule, non révélée dans le rapport avant la section d'audit), puis produire un rapport d'audit démontrant sa capacité à la diagnostiquer par analyse de capture, comme si elle recevait ce ticket d'un tiers :
- Un second serveur DHCP non autorisé sur un des VLAN (rogue DHCP, cf. module 12)
- Un TTL DNS anormalement élevé juste avant simulation d'une migration (cf. module 12)
- Une route mal configurée provoquant un routage asymétrique entre deux VLAN (cf. module 9)
- Un filtrage ICMP total sur le pare-feu inter-VLAN, avec un test de transfert de fichier volumineux révélant un symptôme de PMTU blackhole (cf. module 11)
- Une boucle de commutation non protégée sur un segment (si l'environnement de laboratoire le permet techniquement), provoquant une tempête de diffusion (cf. module 8)

Le rapport d'audit doit suivre la méthodologie du module 13 (vue d'ensemble → repérage → filtrage ciblé → lecture fine → corrélation multi-couches) et aboutir à un diagnostic correct et argumenté, avec la capture à l'appui.

## 8. Architecture cible (schéma de référence)

```
                         ┌─────────────────────┐
                         │   Routeur central     │
                         │  (routage inter-VLAN,  │
                         │   route par défaut)     │
                         └──────────┬────────────┘
                                     │
        ┌────────────────┬────────────┬────────────────┐
        │                  │              │                  │
   VLAN 10                VLAN 20        VLAN 30            VLAN 99
  Direction              Production      Support           Infrastructure
  /27 environ            /25 environ    /27 environ         /29 environ
        │                  │              │                  │
    Postes Direction    Postes Prod.   Postes Support    ┌─────────────┐
                                                              │ Serveur DHCP │
                                                              │ Serveur DNS  │
                                                              └─────────────┘
```

*Ce schéma est indicatif — l'équipe doit produire son propre schéma détaillé avec les adresses réellement calculées, en annexe du rapport.*

## 9. Environnement de laboratoire

**Option recommandée :** ensemble de conteneurs/VM Linux avec interfaces réseau virtuelles (`veth`, bridges Linux, ou VM avec plusieurs interfaces réseau selon l'outil de virtualisation disponible), simulant les VLAN par des sous-réseaux ou bridges distincts si un vrai support 802.1Q n'est pas disponible dans l'environnement pédagogique. Un routeur logiciel (une VM/conteneur Linux avec `ip forward` activé et des routes configurées, ou un outil dédié comme `vyos` si disponible) fait office de routeur inter-VLAN. Un serveur DHCP (`isc-dhcp-server` ou équivalent) et un serveur DNS (`bind9`, `dnsmasq`, ou équivalent léger) sont déployés sur le segment infrastructure.

*Si l'environnement pédagogique dispose d'un vrai accès à des switchs manageables ou d'un simulateur réseau (GNS3, EVE-NG, Packet Tracer), leur usage est encouragé pour une fidélité accrue à un contexte professionnel réel — l'enseignant précise l'outil disponible en début de projet.*

## 10. Étapes et points de passage encadrés

### Phase 1 — Conception (points de passage 1, fin de semaine 1)
- Plan d'adressage VLSM complet et justifié (calculs montrés, pas seulement le résultat)
- Schéma d'architecture cible
- Politique de pare-feu inter-VLAN définie et justifiée
- Validation par l'enseignant avant de poursuivre le déploiement

### Phase 2 — Déploiement et validation fonctionnelle (points de passage 2, fin de semaine 2)
- Infrastructure déployée en laboratoire
- Toutes les captures de validation de la section 7.2 produites
- Point de passage : démonstration en direct du fonctionnement de bout en bout (DHCP, DNS, routage, pare-feu)

### Phase 3 — Audit et clôture (fin de semaine 2 ou 3 selon durée retenue)
- Introduction de l'anomalie choisie (section 7.3)
- Rédaction du rapport d'audit final
- Préparation de la soutenance

### Soutenance finale (20 minutes par binôme)
- 10 minutes de présentation : architecture, choix techniques, démonstration de captures clés
- 5 minutes : présentation du diagnostic d'audit (sans révéler l'anomalie à l'avance — l'enseignant peut poser des questions comme si le binôme recevait le ticket en direct)
- 5 minutes de questions

## 11. Livrables attendus

- **Rapport de conception** (Phase 1) : plan d'adressage VLSM avec calculs, schéma d'architecture, politique de pare-feu justifiée
- **Documentation de déploiement** : configuration complète de chaque brique (DHCP, DNS, routage, pare-feu), avec captures d'écran/fichiers de configuration réels utilisés
- **Preuves de validation fonctionnelle** : les 4 captures minimales de la section 7.2, chacune annotée expliquant ce qu'elle démontre
- **Rapport d'audit** : anomalie introduite (révélée uniquement à ce stade du document), démarche de diagnostic suivie (méthodologie module 13), captures à l'appui, diagnostic final et correction proposée
- **Support de soutenance** (slides ou équivalent)

## 12. Grille d'évaluation

| Critère | Pondération | Détail |
|---|---|---|
| Plan d'adressage VLSM | 15% | Exactitude des calculs, absence de chevauchement, dimensionnement cohérent avec les besoins |
| Architecture et segmentation VLAN | 10% | Cohérence de la segmentation, justification des choix |
| Déploiement DHCP/DNS/routage | 20% | Fonctionnement réel démontré par capture, pas seulement affirmé |
| Politique de pare-feu | 10% | Cohérence avec une politique de sécurité définie et justifiée, preuve d'application effective |
| Qualité des captures de validation | 10% | Captures pertinentes, correctement annotées, prouvant réellement ce qu'elles prétendent démontrer |
| Rapport d'audit (diagnostic de l'anomalie) | 20% | Méthodologie correctement appliquée, diagnostic exact et bien argumenté, pas seulement la bonne réponse par chance |
| Documentation et qualité de rédaction globale | 10% | Clarté, structure professionnelle, schémas lisibles |
| Soutenance | 5% | Clarté de la présentation, capacité à répondre aux questions, y compris en diagnostic live |

**Note additionnelle sur le rapport d'audit :** un diagnostic correct obtenu sans méthodologie visible (par exemple une affirmation directe de la cause sans capture ni démarche d'exclusion d'hypothèses alternatives) est pénalisé même si la conclusion finale est juste — l'évaluation porte sur la **démarche d'ingénieur**, cohérente avec l'esprit de l'ensemble du programme UE2 (méthodologie de troubleshooting rappelée à chaque module depuis M2 de l'UE1).

## 13. Contraintes importantes

- Toute décision technique non conforme aux bonnes pratiques vues en cours (ex : bloc VLSM mal calculé, chevauchement de sous-réseaux, filtrage ICMP total sans discernement en dehors du scénario d'audit volontaire) doit être justifiée explicitement si l'équipe s'en écarte délibérément, sinon elle est considérée comme une erreur
- Les captures fournies comme preuves doivent être réelles (produites par l'équipe sur son propre déploiement), jamais reconstituées ou simulées après coup
- Le choix de l'anomalie d'audit (section 7.3) doit rester inconnu de l'enseignant jusqu'à la remise du rapport d'audit, pour préserver l'authenticité de l'exercice de diagnostic lors de la soutenance

## 14. Prolongement possible

Les outils développés au fil des modules 7 à 12 (sonde multi-couches, surveillant ARP, planificateur VLSM, superviseur d'états TCP, diagnostiqueur ICMP/UDP, surveillant DNS/DHCP) peuvent être réutilisés et intégrés dans ce projet comme outillage de supervision de l'infrastructure déployée — une intégration cohérente de ces outils dans le projet est valorisée en bonus dans l'évaluation de la documentation, sans être une exigence du cahier des charges de base.

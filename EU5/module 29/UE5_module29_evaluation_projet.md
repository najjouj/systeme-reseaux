# UE5 — Module 29 : DNS et DHCP Côté Windows Server
## Compléments : ÉVALUATION / PROJET

---

# PARTIE I — ÉVALUATION

**Durée :** 1h30
**Modalités :** documents non autorisés sauf aide PowerShell locale si épreuve sur machine
**Barème total :** /20

## Section A — QCM (4 points, 0,5 pt/question)

1. Une zone DNS intégrée à Active Directory se réplique via :
 a) un transfert de zone dédié b) la réplication Active Directory elle-même c) manuellement uniquement d) le protocole DHCP

2. Les mises à jour dynamiques sécurisées DNS nécessitent :
 a) une zone primaire/secondaire classique b) une zone intégrée à Active Directory, avec authentification Kerberos c) uniquement IPv6 d) la désactivation du DHCP

3. Un serveur DHCP Windows Server non autorisé dans Active Directory :
 a) fonctionne normalement sans restriction b) se met en veille fonctionnelle même si le service Windows apparaît démarré c) est automatiquement supprimé du domaine d) devient automatiquement le serveur DNS principal

4. L'autorisation DHCP dans Active Directory protège contre :
 a) tout rogue DHCP, y compris les équipements non-Windows b) un rogue DHCP spécifiquement issu d'un serveur Windows Server membre du domaine c) les attaques de couche 2 en général d) rien de particulier

5. Une réservation DHCP, comparée à une adresse IP statique configurée manuellement :
 a) fonctionne de façon identique sans différence b) combine une adresse stable avec la gestion centralisée de DHCP c) nécessite une configuration sur chaque machine individuellement d) ne peut pas être modifiée une fois créée

6. Le basculement DHCP (failover), comparé à un split-scope manuel :
 a) sont strictement équivalents b) coordonne automatiquement l'état des baux entre deux serveurs, contrairement au split-scope non coordonné c) désactive la redondance d) nécessite Active Directory obligatoirement

7. Un service DHCP Windows Server affichant l'état "Running" :
 a) garantit qu'il distribue effectivement des adresses b) peut être non fonctionnel s'il n'est pas autorisé dans Active Directory, malgré cet état apparent c) est toujours non fonctionnel d) indique automatiquement une autorisation valide

8. La protection complète contre le rogue DHCP dans un environnement mixte (Windows et équipements divers) nécessite :
 a) l'autorisation Active Directory seule b) le DHCP snooping seul c) la combinaison de l'autorisation Active Directory ET du DHCP snooping, chacun couvrant un périmètre différent d) aucune des deux mesures, le risque étant négligeable

## Section B — Questions ouvertes (6 points)

**Q1 (2 pts).** Expliquez le mécanisme d'autorisation DHCP dans Active Directory et pourquoi un service "Running" ne garantit pas qu'il soit fonctionnel.

**Q2 (2 pts).** Expliquez pourquoi l'autorisation Active Directory ne remplace pas le DHCP snooping, en donnant un exemple précis de scénario que seul le second couvre.

**Q3 (2 pts).** Expliquez l'avantage d'une zone DNS intégrée à Active Directory par rapport à une zone primaire/secondaire classique, en citant les deux bénéfices principaux vus en cours.

## Section C — Exercice technique (4 points)

Donnez la commande PowerShell permettant d'autoriser un serveur DHCP nommé "dhcp03.nomia.local" (192.168.40.5), celle permettant de vérifier la liste des serveurs autorisés, et celle permettant de créer une réservation pour l'adresse MAC "AA-11-BB-22-CC-33" sur l'adresse IP 192.168.40.75.

## Section D — Problème de diagnostic (6 points)

Un nouveau serveur DHCP Windows Server est déployé et configuré avec une étendue apparemment correcte, mais aucun client ne reçoit d'adresse depuis ce serveur.

**a) (2 pts)** Donnez la première vérification à effectuer, en expliquant pourquoi elle est souvent négligée.

**b) (2 pts)** Donnez la commande de correction si cette vérification révèle la cause du problème.

**c) (2 pts)** Si le problème persiste après correction, proposez une seconde piste de vérification, en vous appuyant sur la nuance pratique découverte en TP (au-delà de la simple théorie).

## Grille de correction résumée

| Section | Points | Critères |
|---|---|---|
| A — QCM | /4 | Réponse exacte uniquement |
| B — Questions ouvertes | /6 | Précision technique, complémentarité autorisation AD/DHCP snooping correctement articulée |
| C — Exercice technique | /4 | 3 commandes exactes |
| D — Diagnostic | /6 | Vérification d'autorisation en premier réflexe, commande de correction exacte, seconde piste pertinente |

---

# PARTIE II — MINI-PROJET (échelle module)

## Cahier des charges

**Titre :** Auditeur DNS/DHCP Active Directory (AD DNS/DHCP Compliance Auditor)

**Contexte :** Suite aux outils des modules 27 et 28, l'équipe système souhaite un troisième outil PowerShell dédié à la conformité DNS/DHCP dans l'environnement Active Directory — avec une attention particulière à la distinction entre "service démarré" et "service réellement fonctionnel" mise en évidence dans ce module.

### Objectifs du livrable

Développer un script PowerShell qui :
1. Audite l'ensemble des serveurs DHCP du réseau connu et vérifie leur statut d'autorisation dans Active Directory, signalant explicitement tout serveur dont le service est démarré mais non autorisé (reproduction directe du piège du cours/TP)
2. Vérifie l'état de toutes les étendues DHCP configurées (actives, taux d'occupation d'adresses proche de saturation, réservations orphelines pointant vers une adresse hors plage)
3. Audite les zones DNS et signale celles qui ne sont pas intégrées à Active Directory alors qu'elles pourraient l'être (candidates à une migration), ainsi que la configuration `DynamicUpdate` de chaque zone intégrée AD
4. Vérifie l'état du basculement DHCP si configuré, signalant toute désynchronisation
5. Produit un rapport de synthèse en catégories distinctes (autorisation, étendues, zones DNS, basculement), sur le modèle des rapports multi-catégories déjà développés
6. Documente l'exécution avec horodatage

### Contraintes techniques
- La détection du piège "Running mais non autorisé" (objectif 1) doit être fiable et testée explicitement (reproduction du scénario du cours)
- Gestion d'erreurs robuste (modules PowerShell DHCP/DNS non installés, serveur injoignable)
- Documentation d'installation et d'utilisation (README)
- Tests démontrés sur au moins 2 scénarios : serveur DHCP autorisé et fonctionnel (pas d'alerte), serveur DHCP démarré mais non autorisé (alerte correctement déclenchée, reproduction du TP)

### Livrables attendus
- Code source commenté
- README (installation, utilisation, format du rapport, limites connues)
- Journal de tests montrant les 2 scénarios validés, avec captures
- Rapport court (1 page) sur les choix techniques et le lien explicite avec le Name & Address Sentinel développé en UE2-M12 pour Linux — comparaison des approches de détection entre les deux écosystèmes

### Grille d'évaluation

| Critère | Pondération |
|---|---|
| Détection du piège "Running mais non autorisé" | 30% |
| Audit des étendues DHCP (saturation, réservations orphelines) | 25% |
| Audit des zones DNS intégrées et DynamicUpdate | 20% |
| Vérification du basculement DHCP | 10% |
| Documentation et tests démontrés (2 scénarios) | 15% |

### Modalité
Travail individuel ou en binôme, durée indicative : 1 semaine. Cet outil complète la suite d'audit Windows initiée aux modules 27 et 28, enrichie aux modules suivants (GPO, NTFS).

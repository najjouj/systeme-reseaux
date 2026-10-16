# UE5 — Module 29 : DNS et DHCP Côté Windows Server

*Ce module applique et enrichit la théorie DNS/DHCP déjà posée en UE2-Module 12, en l'intégrant à Active Directory — avec un lien direct et structurel vers le scénario de rogue DHCP déjà étudié en UE2.*

---

# PARTIE I — COURS

## 1. Titre du cours
DNS et DHCP intégrés à Active Directory : zones intégrées, mises à jour dynamiques et haute disponibilité

## 2. Niveau
Élève ingénieur — cycle ingénieur, UE5 administration Windows (S2)

## 3. Durée
CM : 4h (module d'application directe — +3h TD, +5h TP en Parties II et III)

## 4. Prérequis
- UE2-Module 12 : DNS et DHCP en profondeur (hiérarchie, types d'enregistrements, processus DORA) — théorie mobilisée intensivement ici
- UE5-Module 28 : Active Directory, réplication multi-maître entre contrôleurs de domaine

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Expliquer l'avantage d'une zone DNS intégrée à Active Directory par rapport à une zone primaire/secondaire classique
- Configurer les mises à jour dynamiques sécurisées DNS liées à l'authentification Active Directory
- Configurer une étendue (scope) DHCP Windows Server, avec réservations et options
- Expliquer et configurer le mécanisme d'autorisation DHCP dans Active Directory, et son rôle dans la prévention structurelle du rogue DHCP
- Configurer une solution de haute disponibilité DHCP (basculement/split-scope)
- Diagnostiquer un problème de résolution DNS ou d'attribution DHCP dans un environnement Windows Server/Active Directory

## 6. Motivation

Rappel direct de l'UE2-M12, exercice 4.1 du TD : un serveur DHCP non autorisé (rogue DHCP), branché par erreur ou malveillance sur un réseau d'entreprise, peut distribuer une configuration réseau incorrecte ou malveillante à une partie aléatoire des clients — un incident aux symptômes intermittents et déroutants, dont la seule contre-mesure évoquée à l'époque était le **DHCP snooping** au niveau des switchs (une mesure de couche 2, indépendante de Windows).

Ce module révèle une seconde ligne de défense, structurelle et propre à l'écosystème Windows Server/Active Directory : l'**autorisation DHCP dans Active Directory**, qui empêche un serveur DHCP Windows non explicitement autorisé de démarrer son service de distribution d'adresses au sein d'un domaine — une protection complémentaire au DHCP snooping, opérant à un niveau différent (annuaire plutôt que réseau), démontrant qu'un même problème de sécurité peut souvent être adressé par plusieurs mécanismes complémentaires à différentes couches de l'infrastructure.

## 7. Concepts fondamentaux

### 7.1 Zones DNS intégrées à Active Directory — l'avantage de la réplication AD

Une zone DNS classique (primaire/secondaire, modèle déjà entrevu conceptuellement en UE2-M12 et concrètement avec BIND en UE3) se réplique via un mécanisme de **transfert de zone** entre un serveur primaire et ses serveurs secondaires. Une **zone intégrée à Active Directory** abandonne ce mécanisme dédié au profit de la **réplication Active Directory elle-même** (rappel direct UE5-M28, réplication multi-maître) : les données de zone DNS sont stockées comme des objets dans l'annuaire AD, répliqués automatiquement avec le reste de l'annuaire vers tous les contrôleurs de domaine hébergeant également le rôle DNS.

**Avantages directs de cette intégration :**
- **Multi-maître** : n'importe quel contrôleur de domaine hébergeant le DNS peut accepter une modification de zone (contrairement au modèle primaire/secondaire classique, où seul le serveur primaire accepte les écritures) — cohérent avec le modèle de réplication AD déjà étudié
- **Sécurité renforcée** : les mises à jour dynamiques sécurisées (7.2) ne sont possibles qu'avec une zone intégrée AD, s'appuyant sur l'authentification Kerberos du domaine (approfondi au module 30)
- **Simplicité de réplication** : pas de configuration de transfert de zone séparée à maintenir, la réplication suit automatiquement la topologie de réplication AD déjà en place

### 7.2 Mises à jour dynamiques sécurisées

Rappel direct UE2-M12 : DNS associe des noms à des adresses IP. Dans un environnement d'entreprise avec de nombreux postes obtenant leur adresse via DHCP (adresse potentiellement différente à chaque bail, rappel UE2-M12 section DORA), il serait impraticable de mettre à jour manuellement chaque enregistrement DNS correspondant à chaque changement d'adresse.

Les **mises à jour dynamiques DNS** permettent à un client (ou au serveur DHCP en son nom) d'enregistrer/mettre à jour automatiquement son propre enregistrement DNS lors de l'obtention d'un bail. Sans sécurisation, ce mécanisme serait vulnérable à une usurpation (n'importe quelle machine pourrait revendiquer et modifier l'enregistrement DNS d'un nom quelconque) — d'où l'importance des **mises à jour dynamiques sécurisées**, disponibles uniquement sur une zone intégrée AD (7.1), qui exigent une authentification Kerberos du domaine avant d'accepter une modification, empêchant qu'une machine non membre du domaine (ou usurpant l'identité d'une autre) ne puisse falsifier un enregistrement DNS.

### 7.3 DHCP Windows Server — étendues et réservations

Rappel direct UE2-M12 (processus DORA, déjà étudié en détail) : DHCP Windows Server implémente ce même protocole, avec une terminologie et des outils spécifiques :

```powershell
Add-DhcpServerv4Scope -Name "Reseau-Production" -StartRange 192.168.20.10 -EndRange 192.168.20.200 -SubnetMask 255.255.255.0

Set-DhcpServerv4OptionValue -ScopeId 192.168.20.0 -Router 192.168.20.1 -DnsServer 192.168.20.5

Add-DhcpServerv4Reservation -ScopeId 192.168.20.0 -IPAddress 192.168.20.50 -ClientId "AA-BB-CC-DD-EE-FF" -Description "Imprimante service compta"
```

**Réservation DHCP — comparaison directe avec le concept déjà vu :** une réservation associe une adresse MAC précise à une adresse IP toujours identique, garantissant qu'un équipement particulier (imprimante, serveur applicatif) obtient systématiquement la même adresse via DHCP plutôt qu'une adresse potentiellement différente à chaque renouvellement de bail — un compromis entre la flexibilité complète de DHCP et la stabilité d'une adresse statique configurée manuellement (rappel UE2-M9), sans les inconvénients de gestion manuelle d'un adressage statique classique.

### 7.4 Autorisation DHCP dans Active Directory — la protection structurelle contre le rogue DHCP

**Mécanisme central de ce module :** dans un domaine Active Directory, un serveur **Windows Server** exécutant le rôle DHCP doit être explicitement **autorisé** dans l'annuaire avant de pouvoir effectivement répondre aux requêtes DHCP des clients — un serveur DHCP Windows non autorisé, même correctement configuré et démarré, se met automatiquement en veille et ne distribue aucune adresse.

```powershell
Add-DhcpServerInDC -DnsName "dhcp01.nomia.local" -IPAddress 192.168.20.5
Get-DhcpServerInDC   # liste les serveurs DHCP autorisés dans le domaine
```

**Rappel direct et résolution structurelle du scénario UE2-M12 (exercice 4.1 du TD) :** ce mécanisme d'autorisation apporte une réponse structurelle, complémentaire au DHCP snooping déjà étudié, spécifiquement pour les serveurs DHCP **Windows Server** au sein d'un domaine — un nouvel équipement Windows Server installé par erreur avec le rôle DHCP activé, s'il n'est pas explicitement autorisé dans l'annuaire, ne posera jamais le problème de rogue DHCP décrit à l'époque, car son service DHCP se désactivera automatiquement en détectant sa présence dans un domaine où il n'est pas autorisé.

**Limite importante à ne pas négliger :** cette protection ne s'applique **qu'aux serveurs Windows Server membres du domaine** — un routeur grand public ou un appareil non-Windows avec un serveur DHCP intégré (le scénario exact de l'exercice UE2-M12) n'est absolument pas concerné par ce mécanisme d'autorisation AD, et continue de nécessiter le DHCP snooping (une mesure de couche 2, indépendante du système d'exploitation de l'équipement) comme seule protection efficace contre ce cas précis. Les deux mécanismes sont donc **complémentaires, pas substituables** : l'autorisation AD protège contre un déploiement Windows Server non coordonné au sein de l'équipe, le DHCP snooping protège contre tout équipement, quel qu'il soit, y compris non-Windows.

### 7.5 Haute disponibilité DHCP — basculement (failover) et split-scope

**Rappel direct du principe de redondance déjà établi (UE2-M12, UE5-M28) :** un service DHCP unique constitue un point de défaillance unique — si ce serveur devient indisponible, plus aucune nouvelle machine ne peut obtenir d'adresse (bien que les baux déjà attribués restent valides jusqu'à leur expiration).

**Basculement DHCP (failover)** : fonctionnalité Windows Server permettant à deux serveurs DHCP de partager la responsabilité d'une même étendue, avec réplication automatique de l'état des baux entre eux — si l'un devient indisponible, l'autre prend automatiquement le relais avec une connaissance à jour des baux déjà attribués, sans risque de conflit d'adresse (contrairement à une configuration naïve avec deux serveurs DHCP indépendants gérant chacun une portion arbitraire sans coordination).

```powershell
Add-DhcpServerv4Failover -Name "Failover-Production" -ScopeId 192.168.20.0 `
    -PartnerServer "dhcp02.nomia.local" -Mode LoadBalance
```

**Split-scope (approche alternative plus ancienne) :** diviser manuellement une même étendue logique en deux plages d'adresses distinctes, réparties entre deux serveurs indépendants (par exemple 80%/20%), sans réplication d'état entre eux — une solution plus simple à comprendre mais moins élégante que le basculement moderne, qui gère la coordination de façon transparente et automatique.

## 8. Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Contrôleur de domaine (AD)                                    │
│  ┌────────────────────────────────────────────────┐       │
│  │  Zone DNS intégrée AD                                │       │
│  │  (répliquée via la réplication AD multi-maître)         │       │
│  └────────────────────────────────────────────────┘       │
│  ┌────────────────────────────────────────────────┐       │
│  │  Liste des serveurs DHCP autorisés (Get-DhcpServerInDC) │       │
│  └────────────────────────────────────────────────┘       │
└──────────────┬──────────────────────┬───────────────────┘
                │                        │
        ┌──────────────┐         ┌──────────────┐
        │ DHCP01          │◀─────▶│ DHCP02          │
        │ (autorisé)       │failover│ (autorisé)       │
        └──────────────┘         └──────────────┘
                │                        │
        Clients DHCP (DORA, rappel UE2-M12) obtiennent une adresse
        + mise à jour dynamique sécurisée DNS automatique
```

## 9. Fonctionnement interne

Le mécanisme d'autorisation DHCP (7.4) s'appuie sur un objet spécifique de l'annuaire Active Directory (dans le conteneur des services réseau), consulté par tout service DHCP Windows Server au démarrage : s'il détecte qu'il est membre d'un domaine mais que sa propre adresse IP n'apparaît pas dans la liste des serveurs autorisés, il refuse de répondre aux requêtes DHCP (bien que le service Windows lui-même puisse apparaître comme démarré, une distinction subtile mais importante pour le diagnostic, rappel indirect de la distinction "actif" vs "réellement fonctionnel" déjà rencontrée en UE3-M20 pour le noyau actif vs installé). Les mises à jour dynamiques sécurisées (7.2) s'appuient sur le protocole Kerberos (approfondi au module 30) pour authentifier la machine ou le service demandant la mise à jour, avant que le contrôleur de domaine hébergeant la zone n'accepte la modification de l'enregistrement DNS correspondant.

## 10. Démonstration

```powershell
# Vérifier l'autorisation DHCP
Get-DhcpServerInDC

# Observer une zone DNS intégrée AD et sa réplication
Get-DnsServerZone | Where-Object {$_.ZoneType -eq "Primary" -and $_.IsDsIntegrated -eq $true}

# Observer les étendues DHCP actives
Get-DhcpServerv4Scope
```

## 11. Commandes / outils

| Commande PowerShell | Rôle | Exemple |
|---|---|---|
| `Add-DhcpServerInDC` / `Get-DhcpServerInDC` | Autoriser/lister les serveurs DHCP dans AD | `Get-DhcpServerInDC` |
| `Add-DhcpServerv4Scope` | Créer une étendue DHCP | `Add-DhcpServerv4Scope -Name "Prod" -StartRange ...` |
| `Add-DhcpServerv4Reservation` | Créer une réservation | `Add-DhcpServerv4Reservation -ScopeId ... -ClientId ...` |
| `Add-DhcpServerv4Failover` | Configurer le basculement DHCP | `Add-DhcpServerv4Failover -PartnerServer ...` |
| `Get-DnsServerZone` | Lister les zones DNS et leur type | `Get-DnsServerZone` |
| `Get-DnsServerResourceRecord` | Lister les enregistrements d'une zone | `Get-DnsServerResourceRecord -ZoneName nomia.local` |

**Erreur fréquente :** installer et configurer un serveur DHCP Windows Server complet, puis constater que les clients n'obtiennent jamais d'adresse, sans avoir vérifié en premier lieu si le serveur est bien **autorisé** dans Active Directory (7.4) — une étape facile à oublier car elle ne fait pas partie de la configuration DHCP elle-même (étendues, options), mais d'une action distincte et spécifique à l'écosystème Active Directory.

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** une équipe déploie un nouveau serveur DHCP Windows Server pour remplacer progressivement l'ancien, en migrant les étendues configurées. Après le déploiement, aucun client ne reçoit d'adresse depuis ce nouveau serveur, alors que sa configuration (étendues, options, plage d'adresses) est vérifiée et semble parfaitement correcte à plusieurs reprises.

**Analyse attendue :** ce scénario est la conséquence directe et classique de l'oubli de la section 7.4 : l'équipe a probablement configuré intégralement le rôle DHCP (étendues, réservations, options) mais a omis l'étape d'**autorisation** du nouveau serveur dans Active Directory — une étape technique distincte de la configuration DHCP proprement dite, facilement oubliée précisément parce qu'elle n'apparaît dans aucun assistant de configuration DHCP standard, mais dans la console de gestion Active Directory ou via `Add-DhcpServerInDC`. Diagnostic : `Get-DhcpServerInDC` révélerait immédiatement l'absence du nouveau serveur dans la liste des serveurs autorisés — un service DHCP Windows Server correctement configuré mais non autorisé se comporte silencieusement, sans message d'erreur explicite pour l'administrateur non averti de ce mécanisme, un piège de diagnostic comparable en esprit à d'autres cas déjà rencontrés dans le programme où un composant "actif" au sens du service ne garantit pas qu'il soit réellement fonctionnel (rappel UE3-M20, noyau installé vs actif).

## 15. Sécurité

- L'autorisation DHCP (7.4) est elle-même une mesure de sécurité structurelle contre un déploiement non coordonné, complémentaire (jamais substituable) au DHCP snooping déjà étudié en UE2-M12 pour les équipements non-Windows
- Les mises à jour dynamiques sécurisées (7.2) empêchent une usurpation d'enregistrement DNS par une machine non authentifiée — une protection directement liée au risque de DNS spoofing déjà évoqué en UE2-M12, ici renforcée par l'intégration à l'authentification du domaine
- La liste des serveurs DHCP autorisés dans Active Directory devrait elle-même faire l'objet d'un audit périodique (rappel du principe général d'audit déjà appliqué à de nombreux domaines dans le programme) — un serveur autorisé par le passé mais retiré du service devrait être désautorisé, pour éviter qu'une réactivation accidentelle de cet ancien serveur ne redevienne immédiatement opérationnelle sans contrôle
- Le basculement DHCP (7.5), en plus de la disponibilité, contribue indirectement à la sécurité opérationnelle : un service DHCP fiable réduit la tentation, pour un utilisateur frustré par des pannes fréquentes, de configurer manuellement et de façon non coordonnée un équipement de secours (risque direct de rogue DHCP par improvisation)

## 16. Automatisation

Script PowerShell d'audit combinant autorisation DHCP, cohérence de zone DNS intégrée et état du basculement — un pendant direct des scripts d'audit DNS/DHCP déjà développés pour Linux en UE2-M12.

```powershell
Import-Module DhcpServer, DnsServer

Write-Output "=== Serveurs DHCP autorisés dans le domaine ==="
Get-DhcpServerInDC | Format-Table

Write-Output "=== Étendues DHCP et leur état ==="
Get-DhcpServerv4Scope | Select-Object Name, State, StartRange, EndRange

Write-Output "=== Zones DNS intégrées AD ==="
Get-DnsServerZone | Where-Object {$_.IsDsIntegrated -eq $true} | Select-Object ZoneName, ZoneType

Write-Output "=== État du basculement DHCP configuré ==="
Get-DhcpServerv4Failover | Select-Object Name, PartnerServer, State
```

## 17. Troubleshooting

Scénario court : un nouveau serveur DHCP Windows Server, apparemment démarré et configuré correctement (`Get-Service DHCPServer` indique "Running"), ne répond jamais aux requêtes clientes. Démarche : vérifier en priorité l'autorisation dans Active Directory (`Get-DhcpServerInDC`, rappel direct de la section 7.4 et de l'étude de cas) → si le serveur n'apparaît pas dans la liste, l'autoriser (`Add-DhcpServerInDC`) puis vérifier que le service redevient actif du point de vue fonctionnel (pas seulement l'état du service Windows lui-même, qui peut apparaître "Running" même en l'absence d'autorisation, une distinction déjà soulignée) → si le serveur est bien autorisé mais le problème persiste, basculer vers la méthodologie de diagnostic réseau plus générale déjà étudiée en UE2 (couche 2/3, pare-feu) pour investiguer une cause différente.

## 18. Synthèse

- Une zone DNS intégrée à Active Directory bénéficie de la réplication multi-maître d'AD, sans transfert de zone dédié, et permet les mises à jour dynamiques sécurisées
- Les mises à jour dynamiques sécurisées empêchent l'usurpation d'enregistrement DNS via l'authentification Kerberos du domaine
- DHCP Windows Server reprend le processus DORA déjà étudié, avec réservations pour des adresses stables sans configuration statique manuelle
- L'autorisation DHCP dans Active Directory est une protection structurelle contre le rogue DHCP, complémentaire (jamais substituable) au DHCP snooping déjà étudié, et limitée aux serveurs Windows Server membres du domaine
- Un serveur DHCP peut apparaître "actif" au sens du service Windows tout en étant non fonctionnel faute d'autorisation — une distinction cruciale pour le diagnostic
- Le basculement DHCP offre une haute disponibilité coordonnée, supérieure à un split-scope manuel non coordonné

## 19. Questions d'évaluation

**QCM**
1. Une zone DNS intégrée à Active Directory se réplique via :
 a) un transfert de zone dédié b) la réplication Active Directory elle-même c) manuellement uniquement d) DHCP

2. L'autorisation DHCP dans Active Directory protège contre :
 a) tout rogue DHCP, quel que soit l'équipement b) un rogue DHCP spécifiquement issu d'un serveur Windows Server membre du domaine c) les attaques réseau de couche 2 en général d) rien de particulier

**Questions ouvertes**
- Expliquez pourquoi l'autorisation DHCP dans Active Directory ne remplace pas le DHCP snooping déjà étudié en UE2, en précisant leur complémentarité.
- Expliquez pourquoi les mises à jour dynamiques sécurisées nécessitent une zone intégrée à Active Directory.

**Exercice technique**
- Donnez la commande PowerShell permettant de vérifier la liste des serveurs DHCP actuellement autorisés dans le domaine.

**Problème de diagnostic**
- Un serveur DHCP Windows Server correctement configuré ne distribue aucune adresse, bien que le service apparaisse démarré. Proposez la première vérification à effectuer.

## 20. Mini-projet

Développer un script PowerShell qui :
1. Audite la cohérence entre les serveurs DHCP configurés sur le réseau et la liste des serveurs autorisés dans Active Directory (reprise et extension du script de la section 16)
2. Vérifie l'état du basculement DHCP si configuré, signalant toute désynchronisation entre partenaires
3. Vérifie la cohérence des zones DNS intégrées AD (présence d'enregistrements orphelins ou anormalement anciens, en écho à la logique déjà appliquée pour l'audit DNS en UE2-M12)
4. Produit un rapport de synthèse périodique, structuré en catégories distinctes (autorisation, basculement, cohérence de zone)
5. Documente les parallèles directs avec le module UE2-M12 (Name & Address Sentinel), notamment sur la distinction entre les deux mécanismes de protection contre le rogue DHCP

---

# PARTIE II — TD

### Niveau 1 — Compréhension

**Exercice 1.1**
Expliquez l'avantage d'une zone DNS intégrée à Active Directory par rapport à une zone primaire/secondaire classique.

**Exercice 1.2**
Expliquez le mécanisme d'autorisation DHCP dans Active Directory et ce qu'il empêche.

**Exercice 1.3**
Expliquez la différence entre une réservation DHCP et une adresse IP statique configurée manuellement.

### Niveau 2 — Application

**Exercice 2.1**
Donnez la commande PowerShell permettant d'autoriser un serveur DHCP nommé "dhcp02.nomia.local" (adresse 192.168.30.5) dans Active Directory.

**Exercice 2.2**
Donnez la commande PowerShell permettant de lister toutes les zones DNS intégrées à Active Directory.

**Exercice 2.3**
Donnez la commande PowerShell permettant de créer une réservation DHCP pour l'adresse MAC "11-22-33-44-55-66" sur l'adresse IP 192.168.20.100.

### Niveau 3 — Analyse

**Exercice 3.1**
Un serveur DHCP Windows Server, non membre d'un domaine Active Directory (serveur autonome, workgroup), est déployé sur un réseau isolé sans Active Directory. Le mécanisme d'autorisation décrit en cours s'applique-t-il à ce serveur ? Justifiez, et expliquez quelle protection contre le rogue DHCP resterait pertinente dans ce contexte.

**Exercice 3.2**
Une entreprise configure le basculement DHCP entre deux serveurs, mais un administrateur, par méconnaissance, configure également un split-scope manuel sur les deux mêmes serveurs pour la même étendue, en plus du basculement. Expliquez pourquoi cette configuration combinée est problématique, et laquelle des deux approches devrait être conservée seule.

### Niveau 4 — Problème complexe

**Exercice 4.1**
Une équipe migre son infrastructure DHCP Linux (isc-dhcp-server, comme évoqué en UE3) vers un environnement Windows Server intégré à Active Directory, dans le but explicite de renforcer la protection contre le rogue DHCP après un incident passé (similaire au scénario de l'exercice 4.1 du TD de l'UE2-Module 12, où un routeur grand public avait causé des perturbations). Un membre de l'équipe affirme que cette migration résout définitivement le risque de rogue DHCP grâce à l'autorisation Active Directory. Évaluez cette affirmation de façon critique, en identifiant précisément ce qui est effectivement résolu par la migration et ce qui reste un risque non couvert, et proposez la mesure complémentaire nécessaire pour une protection réellement complète.

---

## Correction du TD

**1.1** — Une zone DNS intégrée à Active Directory bénéficie de la réplication multi-maître déjà utilisée par Active Directory lui-même, sans nécessiter de mécanisme de transfert de zone dédié et distinct à configurer et maintenir séparément. Elle permet également les mises à jour dynamiques sécurisées, impossibles sur une zone primaire/secondaire classique, car ces mises à jour s'appuient sur l'authentification Kerberos du domaine.

**1.2** — L'autorisation DHCP dans Active Directory exige qu'un serveur DHCP Windows Server membre d'un domaine soit explicitement enregistré comme autorisé dans l'annuaire avant de pouvoir effectivement répondre aux requêtes DHCP des clients ; un serveur non autorisé se met automatiquement en veille fonctionnelle, même si le service Windows lui-même apparaît démarré. Ce mécanisme empêche qu'un serveur DHCP Windows Server déployé par erreur ou sans coordination au sein d'une équipe ne perturbe le réseau en distribuant des configurations concurrentes ou incorrectes.

**1.3** — Une réservation DHCP associe une adresse MAC précise à une adresse IP toujours identique, mais l'attribution continue de transiter par le processus DHCP normal (DORA), avec un bail renouvelable et une configuration centralisée modifiable à tout moment depuis le serveur DHCP. Une adresse IP statique configurée manuellement, à l'inverse, est directement définie sur la machine elle-même, sans dépendre du service DHCP ni bénéficier d'une gestion centralisée — la réservation combine la stabilité d'une adresse fixe avec les avantages de gestion centralisée de DHCP.

**2.1** — `Add-DhcpServerInDC -DnsName "dhcp02.nomia.local" -IPAddress 192.168.30.5`.

**2.2** — `Get-DnsServerZone | Where-Object {$_.IsDsIntegrated -eq $true}`.

**2.3** — `Add-DhcpServerv4Reservation -ScopeId <ID_de_l_etendue_concernee> -IPAddress 192.168.20.100 -ClientId "11-22-33-44-55-66"`.

**3.1** — Non, le mécanisme d'autorisation DHCP décrit en cours ne s'applique pas à ce serveur : ce mécanisme repose entièrement sur l'existence d'un domaine Active Directory et d'un annuaire consultable pour vérifier l'autorisation — un serveur DHCP Windows Server autonome, sans domaine, n'a aucun annuaire à interroger et démarre son service DHCP sans aucune vérification de ce type. Dans ce contexte sans Active Directory, la protection pertinente contre le rogue DHCP resterait exclusivement le **DHCP snooping** au niveau des switchs (rappel direct UE2-M12), une mesure de couche 2 totalement indépendante de la présence ou non d'un domaine Active Directory, et donc applicable indifféremment de la nature de l'infrastructure DHCP en place.

**3.2** — Cette configuration combinée est problématique car le basculement DHCP (failover) est déjà conçu pour permettre à deux serveurs de gérer conjointement et de façon coordonnée l'intégralité de la même étendue, avec réplication automatique de l'état des baux entre eux — ajouter un split-scope manuel par-dessus introduirait une double logique de répartition contradictoire (l'une automatique et dynamique via le basculement, l'autre statique et manuelle via la division de plage), risquant de provoquer des conflits d'attribution ou un comportement imprévisible selon la façon dont les deux mécanismes interagiraient réellement en pratique. Seul le **basculement DHCP** devrait être conservé : c'est la solution moderne et complète, rendant le split-scope manuel totalement redondant et potentiellement dangereux une fois le basculement en place.

**4.1** — L'affirmation doit être nuancée et non simplement validée : la migration vers Active Directory avec autorisation DHCP résout effectivement et structurellement le risque qu'un **serveur DHCP Windows Server**, déployé sans coordination au sein de l'équipe elle-même (par exemple, un collègue installant par erreur le rôle DHCP sur un serveur Windows qui ne devrait pas l'avoir), ne perturbe le réseau — ce serveur, non explicitement autorisé, resterait fonctionnellement inactif. **Cependant**, cette migration ne couvre absolument pas le scénario exact évoqué dans l'incident passé de référence (rappel UE2-M12, exercice 4.1) : un **routeur grand public** ou tout équipement non-Windows disposant d'un serveur DHCP intégré activé par erreur reste totalement en dehors du périmètre de protection de l'autorisation Active Directory, qui ne concerne que les serveurs Windows Server membres du domaine. Ce type d'équipement continuerait de poser exactement le même risque qu'avant la migration, si aucune mesure complémentaire n'était mise en place. La mesure complémentaire nécessaire pour une protection réellement complète, couvrant l'ensemble des scénarios possibles (serveurs Windows non autorisés ET équipements non-Windows quelconques), est le **DHCP snooping** au niveau des switchs de l'infrastructure réseau (rappel direct UE2-M12) — une mesure de couche 2 indépendante du système d'exploitation de l'équipement source, à déployer en complément de l'autorisation Active Directory, jamais à sa place, pour obtenir une couverture réellement exhaustive du risque de rogue DHCP dans toute sa diversité de scénarios possibles.

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur système junior pour déployer et sécuriser l'infrastructure DNS/DHCP Windows Server intégrée à Active Directory d'un environnement de laboratoire, en reproduisant explicitement le scénario de l'oubli d'autorisation DHCP de l'étude de cas du cours.

## Architecture

Une VM Windows Server déjà contrôleur de domaine (reprise du TP du module 28), avec les rôles DNS et DHCP disponibles à installer.

## Objectifs
- Configurer une étendue DHCP avec réservation
- Observer et comprendre l'effet de l'autorisation (ou son absence) dans Active Directory
- Observer une zone DNS intégrée AD et une mise à jour dynamique
- Diagnostiquer le scénario exact de l'étude de cas du cours

## Préparation

```powershell
Install-WindowsFeature DHCP -IncludeManagementTools
Import-Module DhcpServer, DnsServer, ActiveDirectory
```

## Étapes

### Étape 1 — Configurer une étendue DHCP sans l'autoriser (reproduction volontaire du piège)

```powershell
Add-DhcpServerv4Scope -Name "Lab-Reseau" -StartRange 192.168.99.10 -EndRange 192.168.99.100 -SubnetMask 255.255.255.0 -State Active
Set-DhcpServerv4OptionValue -ScopeId 192.168.99.0 -Router 192.168.99.1 -DnsServer 192.168.99.5

Get-DhcpServerv4Scope
Get-Service DhcpServer   # observer : le service apparaît "Running"
Get-DhcpServerInDC        # observer : ce serveur n'apparaît PAS dans la liste
```
*Question :* d'après ce que vous observez, le service DHCP est-il réellement fonctionnel à ce stade ? Sur quels indices précis basez-vous votre réponse ?

### Étape 2 — Autoriser le serveur et vérifier l'effet

```powershell
$monServeur = (Get-DhcpServerInDC -ErrorAction SilentlyContinue)
Add-DhcpServerInDC -DnsName $env:COMPUTERNAME
Get-DhcpServerInDC

# Vérifier l'état fonctionnel réel de l'étendue après autorisation
Get-DhcpServerv4Scope -ScopeId 192.168.99.0 | Select-Object Name, State
```
*Question :* qu'est-ce qui a changé entre l'étape 1 et l'étape 2 ? Le service Windows lui-même (`Get-Service DhcpServer`) a-t-il changé d'état à un moment donné, ou seulement le comportement fonctionnel du serveur ?

### Étape 3 — Réservation et zone DNS intégrée

```powershell
Add-DhcpServerv4Reservation -ScopeId 192.168.99.0 -IPAddress 192.168.99.50 -ClientId "AA-BB-CC-11-22-33" -Description "Poste de test réservé"

Get-DhcpServerv4Reservation -ScopeId 192.168.99.0

Get-DnsServerZone | Where-Object {$_.IsDsIntegrated -eq $true} | Select-Object ZoneName, ZoneType, DynamicUpdate
```
*Question :* la propriété `DynamicUpdate` de votre zone principale indique-t-elle "Secure" (mises à jour dynamiques sécurisées) ou une autre valeur ? Pourquoi cette valeur est-elle importante au regard de la section 7.2 du cours ?

### Étape 4 — Diagnostic du scénario complet de l'étude de cas

*Scénario fourni :* "Un collègue a configuré une étendue DHCP `Lab-Reseau-2` sur ce serveur pour un nouveau segment réseau, mais les postes de ce segment ne reçoivent jamais d'adresse. Il affirme avoir vérifié la configuration à trois reprises et être certain qu'elle est correcte."

```powershell
Add-DhcpServerv4Scope -Name "Lab-Reseau-2" -StartRange 192.168.98.10 -EndRange 192.168.98.100 -SubnetMask 255.255.255.0 -State Active
# Volontairement : ne pas ré-autoriser (déjà fait à l'étape 2, mais imaginez un nouveau serveur DHCP distinct dans un scénario réel)

# Appliquez la méthodologie de diagnostic vue en cours pour ce scénario
Get-DhcpServerInDC
Get-DhcpServerv4Scope
```
*À vous de jouer :* en supposant que ce soit un tout nouveau serveur DHCP (distinct de celui déjà autorisé), quelle serait votre démarche de diagnostic complète, étape par étape ?

## Commandes de référence
`Add-DhcpServerv4Scope`, `Get-DhcpServerInDC`, `Add-DhcpServerInDC`, `Add-DhcpServerv4Reservation`, `Get-DnsServerZone`

## Résultats attendus
- Observation de l'état "Running" du service malgré l'absence d'autorisation (étape 1)
- Confirmation du changement d'état fonctionnel après autorisation (étape 2)
- Réservation créée et propriété `DynamicUpdate` de la zone observée (étape 3)
- Démarche de diagnostic complète pour le scénario de l'étape 4

## Questions

1. Dans l'étape 1, pourquoi est-il pédagogiquement important d'observer que `Get-Service DhcpServer` indique "Running" **malgré** l'absence d'autorisation ? Quel piège de diagnostic cela illustre-t-il concrètement, et à quel autre module du programme ce piège fait-il directement écho ?
2. Si votre environnement de laboratoire ne disposait que d'un seul contrôleur de domaine, en quoi cela limiterait-il votre capacité à observer pleinement l'avantage de la réplication multi-maître pour la zone DNS intégrée AD (section 7.1) ?
3. Dans l'étape 4, si le scénario s'était avéré être un rogue DHCP provenant d'un routeur grand public plutôt qu'un oubli d'autorisation sur un serveur Windows légitime, votre démarche de diagnostic aurait-elle été différente ? Sur quel(s) outil(s) vous seriez-vous appuyé, en faisant le lien avec l'UE2 ?

## Troubleshooting (panne à diagnostiquer)

**Situation fournie :** après avoir autorisé le serveur DHCP à l'étape 2, un étudiant constate que les clients de test continuent, plusieurs minutes après, à ne recevoir aucune adresse — contrairement au résultat attendu suite à l'autorisation.

**Démarche attendue :**
1. Symptôme : absence persistante de distribution d'adresses malgré l'autorisation déjà effectuée
2. Hypothèse : l'autorisation dans Active Directory peut nécessiter un court délai de propagation avant d'être pleinement effective côté service DHCP local (rappel possible d'un délai similaire déjà rencontré pour la réplication AD en général, module 28) ; alternativement, le service DHCP local pourrait nécessiter un redémarrage explicite pour reconnaître son nouvel état d'autorisation, plutôt qu'une prise en compte totalement automatique et instantanée
3. Vérification : consulter le Journal des événements Système (rappel direct module 27) à la recherche d'un événement relatif au service DHCP et à son état d'autorisation, qui pourrait indiquer explicitement s'il a détecté son autorisation ou non
4. Isolation : tenter un redémarrage du service DHCP (`Restart-Service DhcpServer`) pour forcer une nouvelle vérification de son état d'autorisation
5. Correction : si le redémarrage du service résout le problème, documenter que l'autorisation, bien qu'immédiatement visible dans l'annuaire (`Get-DhcpServerInDC`), peut nécessiter un redémarrage du service DHCP local pour être pleinement prise en compte selon la version de Windows Server utilisée
6. Vérification : confirmer qu'un client de test obtient désormais correctement une adresse
7. Documentation : noter cette nuance comme un complément pratique et empirique à la théorie du cours — un mécanisme documenté comme automatique peut, en pratique, nécessiter une action complémentaire simple (redémarrage de service) pour être pleinement effectif, une leçon de prudence générale déjà rencontrée sous d'autres formes dans le programme (`daemon-reload` après modification d'unité systemd, UE3-M16)

## Validation

Le TP est validé si l'étudiant fournit :
- Les 4 résultats/réflexions demandés
- Des réponses argumentées aux 3 questions
- Une démarche de diagnostic cohérente pour le scénario de troubleshooting, mobilisant à bon escient les outils déjà pratiqués au fil du module

# UE5 — Module 30 : Group Policy, Kerberos et LDAP

*Ce module compare systématiquement Kerberos à l'authentification par clé SSH déjà étudiée (UE3-M18), et LDAP au modèle de résolution DNS déjà maîtrisé (UE2-M12).*

---

# PARTIE I — COURS

## 1. Titre du cours
Group Policy, Kerberos et LDAP : configuration centralisée, authentification et annuaire

## 2. Niveau
Élève ingénieur — cycle ingénieur, UE5 administration Windows (S2)

## 3. Durée
CM : 4h30 (module triple thématique — +4h TD, +5h TP en Parties II et III)

## 4. Prérequis
- UE5-Module 28 : Active Directory, structure en OU, réplication
- UE5-Module 29 : DNS intégré AD, mises à jour dynamiques sécurisées (déjà mentionné comme reposant sur Kerberos)
- UE3-Module 18 : authentification par clé SSH (comparaison directe avec Kerberos)
- UE2-Module 12 : DNS comme service de résolution distribué (comparaison avec LDAP)

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Expliquer le mécanisme des stratégies de groupe (GPO) et leur ordre d'application/héritage
- Expliquer le fonctionnement de Kerberos (ticket-granting ticket, ticket de service) et pourquoi il ne transmet jamais le mot de passe sur le réseau
- Comparer explicitement Kerberos à l'authentification par clé SSH déjà étudiée
- Expliquer le rôle de LDAP comme protocole d'annuaire sous-jacent à Active Directory, et le comparer à DNS
- Lire et construire un nom distinctif (Distinguished Name) LDAP
- Diagnostiquer un problème d'application de GPO ou d'authentification Kerberos

## 6. Motivation

Ce module aborde trois piliers techniques distincts mais profondément interconnectés d'Active Directory : comment une configuration s'applique uniformément à des milliers de postes (GPO), comment l'authentification garantit qu'un mot de passe ne transite jamais en clair ni même chiffré de façon interceptable sur le réseau (Kerberos), et comment l'annuaire lui-même est interrogé de façon standardisée (LDAP, un protocole ouvert, pas propriétaire à Windows).

Ce dernier point mérite d'être souligné : LDAP est un **standard ouvert**, utilisé aussi bien par Active Directory que par des annuaires Linux (OpenLDAP) — une passerelle technique directe entre les deux écosystèmes déjà entrevue avec WSL en UE5-M27, ici du côté de la gestion d'identités plutôt que de l'exécution de code.

## 7. Concepts fondamentaux

### 7.1 Group Policy Objects (GPO) — configuration centralisée à grande échelle

Un **GPO (Group Policy Object)** est un ensemble de paramètres de configuration (sécurité, restrictions applicatives, déploiement logiciel, scripts de démarrage/connexion) appliqué automatiquement aux ordinateurs et/ou utilisateurs d'un périmètre défini — un mécanisme qui répond au même besoin que la configuration centralisée déjà recherchée dans d'autres contextes du programme, mais à l'échelle de milliers de postes simultanément.

**Rappel direct et comparaison utile :** un GPO joue, pour la configuration système et de sécurité à l'échelle d'un domaine, un rôle comparable en intention à ce qu'un outil de gestion de configuration (Ansible, Puppet — mentionnés en fin de programme général) réalise pour des serveurs Linux, ou à ce qu'un ensemble cohérent de scripts d'automatisation déployés via cron/systemd timers (UE3, UE4) pourrait accomplir de façon plus artisanale — GPO offre nativement cette capacité de déploiement de configuration à grande échelle, intégrée directement à Active Directory.

**Liaison (linking) et portée :** un GPO peut être lié à un **site**, un **domaine**, ou une **unité d'organisation (OU)** — rappel direct de la structure en OU déjà étudiée en UE5-M28, ici mobilisée comme périmètre d'application d'une politique.

**Ordre d'application et héritage — un point souvent source de confusion :** les GPO s'appliquent dans un ordre précis, résumé par l'acronyme **LSDOU** (Local, Site, Domaine, OU — de la portée la plus large à la plus spécifique), les paramètres des GPO appliqués **plus tard** dans cet ordre prenant généralement le dessus en cas de conflit (le paramètre le plus spécifique à l'OU l'emporte sur le paramètre plus général du domaine, sauf configuration explicite contraire).

```
Local (paramètres locaux de la machine)
    ↓
Site (GPO liés au site AD)
    ↓
Domaine (GPO liés à l'ensemble du domaine)
    ↓
OU (GPO liés à l'unité d'organisation, la plus spécifique généralement prioritaire)
```

**Options de contrôle de l'héritage :**
- **Blocage d'héritage** (sur une OU) : empêche les GPO des niveaux supérieurs (domaine, site) de s'appliquer aux objets de cette OU
- **Application forcée** (Enforced, sur un GPO) : garantit qu'un GPO s'applique malgré un blocage d'héritage éventuel en aval — un mécanisme de priorité absolue à utiliser avec parcimonie, réservé aux politiques de sécurité critiques que l'organisation souhaite garantir sans exception

### 7.2 Application des GPO — `gpupdate` et cycle de rafraîchissement

Les GPO ne s'appliquent pas instantanément dès leur modification sur un contrôleur de domaine — les postes clients rafraîchissent périodiquement leur configuration (par défaut, environ toutes les 90 minutes avec un délai aléatoire pour éviter une surcharge simultanée du réseau, rappel indirect du principe déjà rencontré pour éviter une congestion coordonnée). Pour forcer une application immédiate :

```powershell
gpupdate /force
gpresult /r          # affiche les GPO effectivement appliqués et leur origine (diagnostic)
```

**`gpresult` — un outil de diagnostic essentiel :** face à un comportement de poste qui ne correspond pas à la politique attendue, `gpresult /r` (ou `/h` pour un rapport HTML détaillé) révèle précisément quels GPO ont été appliqués, dans quel ordre, et lesquels ont été filtrés ou bloqués — un réflexe de diagnostic comparable en esprit à `systemctl cat`/`systemctl status` déjà pratiqué pour comprendre l'état effectif d'une unité systemd (UE3-M16).

### 7.3 Kerberos — authentification sans transmission de secret

**Rappel direct de l'UE3-M18 :** l'authentification par clé SSH permet à un client de prouver son identité sans jamais transmettre sa clé privée sur le réseau, via un défi cryptographique résolu localement. **Kerberos**, le protocole d'authentification par défaut d'Active Directory, poursuit un objectif similaire par un mécanisme différent : garantir qu'un mot de passe (ou son équivalent cryptographique) ne transite jamais sur le réseau, même de façon chiffrée interceptable, en s'appuyant sur un tiers de confiance et un système de **tickets** à durée de vie limitée.

**Déroulement simplifié (analogie avec un système de tickets d'événement) :**

1. **Authentification initiale (AS-REQ/AS-REP)** : l'utilisateur s'authentifie une seule fois auprès du **Centre de Distribution de Clés (KDC — Key Distribution Center**, un rôle assuré par chaque contrôleur de domaine), obtenant en retour un **Ticket-Granting Ticket (TGT)** — un ticket chiffré prouvant que l'authentification initiale a réussi, sans que le mot de passe lui-même ne soit jamais transmis en clair sur le réseau à aucun moment de cet échange
2. **Obtention d'un ticket de service (TGS-REQ/TGS-REP)** : lorsque l'utilisateur souhaite accéder à une ressource précise (un partage de fichiers, une application), il présente son TGT au KDC et reçoit en retour un **ticket de service** spécifique à cette ressource, sans avoir à se réauthentifier avec son mot de passe
3. **Accès au service** : l'utilisateur présente ce ticket de service directement à la ressource souhaitée, qui le valide sans jamais avoir besoin de contacter le contrôleur de domaine elle-même pour cette vérification précise (le ticket contient les informations cryptographiques nécessaires à sa propre validation)

**Durée de vie limitée des tickets — une différence notable avec une clé SSH :** un TGT a une durée de vie limitée (typiquement 10 heures par défaut, configurable), après laquelle une nouvelle authentification est nécessaire — contrairement à une clé SSH qui reste valide indéfiniment tant qu'elle n'est pas explicitement révoquée. Ce choix de conception limite la fenêtre d'exploitation possible si un ticket venait à être compromis, un compromis sécurité/commodité différent de celui de SSH.

**Authentification mutuelle :** Kerberos permet également au **serveur** de prouver son identité au client (pas seulement l'inverse), une protection contre un serveur usurpé qui tenterait de se faire passer pour le service légitime — une garantie plus poussée que le modèle SSH classique, où l'authentification du serveur repose sur la vérification manuelle ou préalable de son empreinte de clé hôte (`known_hosts`), un mécanisme moins intégré nativement.

**Pourquoi les mises à jour dynamiques DNS sécurisées (UE5-M29) dépendent de Kerberos :** l'authentification Kerberos d'une machine membre du domaine, déjà réalisée pour d'autres besoins (accès à des ressources), est directement réutilisée pour authentifier une demande de mise à jour DNS dynamique — une seule infrastructure d'authentification centralisée servant plusieurs besoins, plutôt qu'un mécanisme dédié et distinct pour chaque service.

### 7.4 LDAP — protocole d'annuaire standard

**LDAP (Lightweight Directory Access Protocol)** est le protocole standard (ouvert, pas propriétaire à Microsoft) utilisé pour interroger et modifier un annuaire hiérarchique — Active Directory expose son annuaire via LDAP, mais LDAP est également utilisé par des annuaires Linux indépendants (OpenLDAP, FreeIPA), permettant une interopérabilité entre écosystèmes.

**Comparaison structurelle avec DNS (rappel direct UE2-M12) :** les deux sont des services de **résolution/consultation hiérarchique distribués**, mais répondant à des besoins différents — DNS résout un nom vers une adresse IP (et quelques autres types d'enregistrements limités, rappel des types A/AAAA/CNAME/MX étudiés), tandis que LDAP interroge un annuaire riche et arbitrairement structuré (utilisateurs, groupes, ordinateurs, avec de nombreux attributs par objet) — DNS est optimisé pour une résolution simple et extrêmement rapide à très grande échelle (l'ensemble d'Internet), LDAP pour des requêtes plus riches sur un ensemble de données plus structuré mais généralement de portée plus restreinte (une organisation).

**Nom distinctif (Distinguished Name, DN)** : l'identifiant complet et unique d'un objet dans l'annuaire LDAP, construit hiérarchiquement du plus spécifique au plus général (l'inverse de la notation d'un nom de domaine DNS, qui va du plus spécifique au plus général de gauche à droite mais avec une structure différente) :

```
CN=Alice Dupont,OU=Production,DC=nomia,DC=local
```

- `CN` (Common Name) : le nom de l'objet lui-même
- `OU` (Organizational Unit) : l'unité d'organisation contenant l'objet (rappel direct UE5-M28)
- `DC` (Domain Component) : chaque composant du nom de domaine, un par niveau (ici `nomia.local` devient `DC=nomia,DC=local`)

**Requête LDAP simple depuis PowerShell (déjà effectuée indirectement via le module ActiveDirectory) :**
```powershell
Get-ADUser -Filter "Name -eq 'Alice Dupont'" -SearchBase "OU=Production,DC=nomia,DC=local"
# Les cmdlets ActiveDirectory encapsulent en réalité des requêtes LDAP sous-jacentes
```

### 7.5 Synthèse des trois piliers — comment ils s'articulent

```
Un utilisateur ouvre une session sur un poste du domaine
    │
    ▼
Kerberos (7.3) authentifie l'utilisateur, sans transmettre son mot de passe
    │
    ▼
Le poste applique les GPO (7.1) correspondant à sa position dans la structure
en OU (rappel M28), résolues via des requêtes LDAP (7.4) à l'annuaire
    │
    ▼
L'utilisateur accède aux ressources, chaque accès validé par un ticket
de service Kerberos obtenu sans nouvelle saisie de mot de passe
```

## 8. Architecture

```
┌────────────────────────────────────────────────────────────┐
│  Contrôleur de domaine                                            │
│  ┌────────────┐  ┌────────────┐  ┌────────────────┐         │
│  │  KDC          │  │  Annuuaire   │  │  GPO stockés         │         │
│  │  (Kerberos)    │  │  (LDAP)       │  │  (SYSVOL + AD)          │         │
│  └────────────┘  └────────────┘  └────────────────┘         │
└───────┬──────────────┬──────────────────┬───────────────────┘
        │ TGT/tickets    │ requêtes LDAP     │ gpupdate
        ▼                 ▼                    ▼
┌────────────────────────────────────────────────────────────┐
│  Poste client (membre du domaine)                                 │
│  Session authentifiée, GPO appliqués, annuaire consultable          │
└────────────────────────────────────────────────────────────┘
```

## 9. Fonctionnement interne

Les GPO sont physiquement stockés en deux parties complémentaires : un objet dans l'annuaire Active Directory (le "Group Policy Container", contenant des métadonnées) et un ensemble de fichiers dans le volume partagé **SYSVOL** (répliqué entre contrôleurs de domaine via un mécanisme de réplication de fichiers, distinct mais complémentaire à la réplication de l'annuaire lui-même déjà étudiée en UE5-M28) — les deux parties doivent rester cohérentes entre elles pour qu'un GPO s'applique correctement, une source occasionnelle de dysfonctionnement si la réplication de l'un des deux composants prend du retard par rapport à l'autre. Kerberos repose sur des mécanismes cryptographiques symétriques (le KDC et chaque partie partagent un secret dérivé de leurs informations d'authentification respectives), avec une synchronisation temporelle stricte entre les machines participantes (les tickets incluent un horodatage, et un écart d'horloge excessif entre un client et le KDC peut faire échouer l'authentification — d'où l'importance, dans un domaine Active Directory, d'une synchronisation horaire fiable entre tous les membres, un service généralement assuré par les contrôleurs de domaine eux-mêmes).

## 10. Démonstration

```powershell
# Observer les GPO effectivement appliqués sur un poste
gpresult /r

# Observer les tickets Kerberos actuellement détenus par la session
klist

# Observer une requête LDAP simple encapsulée
Get-ADUser -Filter * -SearchBase "OU=Production,DC=nomia,DC=local"
```

## 11. Commandes / outils

| Commande | Rôle | Exemple |
|---|---|---|
| `gpupdate /force` | Forcer l'application immédiate des GPO | `gpupdate /force` |
| `gpresult /r` | Diagnostiquer les GPO appliqués | `gpresult /r` |
| `klist` | Lister les tickets Kerberos actuels de la session | `klist` |
| `New-GPO` / `New-GPLink` | Créer et lier un GPO (PowerShell) | `New-GPO -Name "Politique-Securite"` |
| `Get-ADUser -SearchBase` | Requête LDAP encapsulée via le module AD | `Get-ADUser -Filter * -SearchBase "OU=..."` |

**Erreur fréquente :** oublier qu'un GPO modifié met un temps de propagation (réplication AD + SYSVOL + cycle de rafraîchissement client) avant d'être visible sur un poste, et conclure hâtivement à un GPO mal configuré alors qu'un simple `gpupdate /force` (ou l'attente du cycle naturel) suffirait — un piège directement comparable en esprit au délai de réplication déjà rencontré pour Active Directory en général (UE5-M28) et pour le cache DNS (UE2-M12).

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** une équipe configure un GPO de sécurité au niveau du domaine, imposant une politique de verrouillage de compte après 5 tentatives de connexion échouées. Quelques semaines plus tard, elle découvre qu'une OU spécifique (celle des comptes de service informatique) ne respecte pas cette politique, sans qu'aucune erreur de configuration évidente ne soit détectée dans le GPO lui-même.

**Analyse attendue :** ce scénario oriente directement vers un mécanisme de **blocage d'héritage** (section 7.1) actif sur cette OU spécifique, ou un GPO plus spécifique lié directement à cette OU et définissant une politique différente (voire l'absence de politique), qui l'emporte sur le GPO de domaine du fait de sa position plus spécifique dans l'ordre LSDOU. Diagnostic : `gpresult /r` exécuté sur un poste de cette OU révélerait précisément quels GPO sont effectivement appliqués et lesquels ont été filtrés — la démarche de diagnostic directe recommandée en section 7.2, plutôt que de supposer une erreur dans le GPO de domaine lui-même sans vérifier son application réelle sur le périmètre concerné. Ce cas illustre l'importance de toujours vérifier l'état **effectivement appliqué** plutôt que de se fier uniquement à la configuration telle que définie, un principe déjà rencontré à de multiples reprises dans le programme (configuration systemd non rechargée, UE3-M16 ; noyau installé vs actif, UE3-M20).

## 15. Sécurité

- Kerberos, bien que structurellement plus robuste qu'une authentification par mot de passe classique transmise sur le réseau, n'est pas invulnérable : des attaques spécifiques (comme le "Golden Ticket", exploitant une compromission du compte de service KDC lui-même) existent et sont documentées, approfondies en UE8 sécurité — la sécurité de Kerberos dépend en dernier ressort de la protection du contrôleur de domaine lui-même, rappel du principe de centralisation de risque déjà souligné en UE5-M28
- Un GPO mal conçu ou trop permissif peut affaiblir la sécurité de l'ensemble d'un périmètre organisationnel en une seule modification malencontreuse — la puissance de la configuration centralisée est aussi, comme toujours dans ce programme, un facteur d'amplification du risque en cas d'erreur
- La configuration de GPO de durcissement (restriction des privilèges locaux, désactivation de fonctionnalités non nécessaires) applique à l'échelle d'un domaine entier les mêmes principes de moindre privilège déjà enseignés localement pour une seule machine (UE3-M14) — GPO est un vecteur naturel pour industrialiser ces bonnes pratiques de sécurité déjà connues
- L'accès en lecture à l'annuaire LDAP lui-même doit être contrôlé avec soin : un annuaire trop ouvert en lecture peut révéler une cartographie complète de l'organisation (utilisateurs, groupes, structure) à un attaquant ayant obtenu un accès limité mais suffisant pour interroger l'annuaire

## 16. Automatisation

Script PowerShell d'audit des GPO liés à chaque OU, avec vérification de cohérence entre GPO de domaine et exceptions locales — un outil de diagnostic proactif complémentaire à `gpresult` réactif.

```powershell
Import-Module GroupPolicy, ActiveDirectory

$toutesLesOU = Get-ADOrganizationalUnit -Filter *

foreach ($ou in $toutesLesOU) {
    $gpoLies = Get-GPInheritance -Target $ou.DistinguishedName

    Write-Output "=== OU: $($ou.Name) ==="
    Write-Output "Héritage bloqué: $($gpoLies.GpoInheritanceBlocked)"
    $gpoLies.GpoLinks | ForEach-Object {
        Write-Output "  GPO lié: $($_.DisplayName) - Activé: $($_.Enabled) - Application forcée: $($_.Enforced)"
    }
}
```

## 17. Troubleshooting

Scénario court : un utilisateur signale qu'une nouvelle restriction de sécurité, censée s'appliquer immédiatement selon l'équipe informatique, n'affecte toujours pas son poste plusieurs heures après son déploiement annoncé. Démarche : exécuter `gpresult /r` sur le poste concerné pour observer précisément quels GPO sont effectivement appliqués → si le GPO attendu n'apparaît pas du tout, vérifier sa liaison correcte à l'OU appropriée (rappel section 7.1) et l'état de réplication SYSVOL/AD (rappel section 9) → si le GPO apparaît mais son paramètre spécifique ne semble pas actif, vérifier un éventuel conflit avec un GPO plus spécifique appliqué plus tard dans l'ordre LSDOU → en dernier recours, forcer `gpupdate /force` sur le poste et redémarrer si nécessaire (certains paramètres GPO ne s'appliquent qu'au redémarrage), avant de conclure à un problème de configuration plutôt qu'un simple délai de propagation normal.

## 18. Synthèse

- Les GPO appliquent une configuration centralisée selon l'ordre LSDOU (Local, Site, Domaine, OU), avec blocage d'héritage et application forcée comme mécanismes de contrôle fin
- `gpresult /r` est l'outil de diagnostic de référence pour observer ce qui est effectivement appliqué, comparable en esprit à `systemctl status`/`cat` déjà pratiqué sous Linux
- Kerberos authentifie sans jamais transmettre le mot de passe sur le réseau, via un système de tickets à durée de vie limitée (TGT puis tickets de service), avec authentification mutuelle — un objectif similaire à l'authentification par clé SSH mais un mécanisme et des garanties différents
- LDAP est le protocole standard et ouvert d'interrogation d'annuaire sous-jacent à Active Directory, comparable en intention (mais différent en usage) à DNS déjà étudié — le nom distinctif (DN) identifie un objet hiérarchiquement
- Un délai de propagation (réplication AD/SYSVOL, cycle de rafraîchissement client) doit toujours être envisagé avant de conclure à un GPO mal configuré
- La sécurité de l'ensemble de cette infrastructure centralisée dépend en dernier ressort de la protection des contrôleurs de domaine eux-mêmes

## 19. Questions d'évaluation

**QCM**
1. L'ordre d'application des GPO, du plus général au plus spécifique, est résumé par l'acronyme :
 a) DOSL b) LSDOU c) ULDOS d) SDLOU

2. Kerberos garantit notamment que :
 a) le mot de passe est chiffré mais transmis sur le réseau b) le mot de passe n'est jamais transmis sur le réseau, même chiffré c) l'authentification est valide indéfiniment d) aucune authentification n'est nécessaire

**Questions ouvertes**
- Comparez Kerberos et l'authentification par clé SSH déjà étudiée, en identifiant un point commun et une différence significative.
- Expliquez la différence de rôle entre DNS et LDAP, bien que les deux soient des services de résolution/consultation hiérarchique distribués.

**Exercice technique**
- Donnez la commande permettant de diagnostiquer précisément quels GPO sont effectivement appliqués sur un poste.

**Problème de diagnostic**
- Un GPO récemment modifié ne semble pas encore appliqué sur un poste. Proposez la démarche de diagnostic, en évitant une conclusion prématurée d'erreur de configuration.

## 20. Mini-projet

Développer un script PowerShell qui :
1. Audite l'ensemble des OU et leurs GPO liés, avec l'état d'héritage (bloqué ou non) et d'application forcée (reprise et extension du script de la section 16)
2. Détecte les conflits potentiels entre GPO (deux GPO liés à des niveaux différents définissant des valeurs contradictoires pour un même paramètre, dans la mesure du possible avec les cmdlets disponibles)
3. Vérifie la cohérence entre l'objet GPO dans l'annuaire et sa réplication SYSVOL (détection de désynchronisation potentielle, rappel section 9)
4. Produit un rapport de synthèse structuré, à la manière des outils d'audit déjà développés dans le programme
5. Documente les parallèles avec les outils de gestion de configuration déjà évoqués pour Linux (Ansible/Puppet, mentionnés en fin de programme général)

---

# PARTIE II — TD

### Niveau 1 — Compréhension

**Exercice 1.1**
Expliquez l'ordre d'application des GPO (LSDOU) et ce que signifie chaque lettre.

**Exercice 1.2**
Décrivez les trois étapes principales du fonctionnement de Kerberos (authentification initiale, obtention d'un ticket de service, accès au service).

**Exercice 1.3**
Expliquez la différence de rôle entre DNS et LDAP, bien que les deux soient des services de consultation hiérarchique distribués.

### Niveau 2 — Application

**Exercice 2.1**
Donnez la commande permettant de forcer l'application immédiate des GPO sur un poste.

**Exercice 2.2**
Donnez la commande permettant de lister les tickets Kerberos actuellement détenus par la session en cours.

**Exercice 2.3**
Donnez le nom distinctif (Distinguished Name) LDAP pour un utilisateur "Bob Martin" dans l'OU "SupportTechnique" du domaine "nomia.local".

### Niveau 3 — Analyse

**Exercice 3.1**
Un GPO de domaine impose un fond d'écran spécifique, mais les utilisateurs d'une OU précise continuent d'avoir un fond d'écran différent. Un administrateur vérifie le GPO de domaine et le trouve parfaitement configuré. Proposez deux hypothèses distinctes expliquant ce comportement, en vous appuyant sur les mécanismes vus en cours.

**Exercice 3.2**
Un utilisateur, après avoir changé son mot de passe, continue de pouvoir accéder normalement à des ressources réseau pendant les minutes suivantes, sans qu'aucune nouvelle saisie de mot de passe ne lui soit demandée pour ces accès. Expliquez ce comportement en vous appuyant sur le mécanisme des tickets Kerberos vu en cours.

### Niveau 4 — Problème complexe

**Exercice 4.1**
Une entreprise configure Kerberos avec des contrôleurs de domaine dont l'horloge système a dérivé de plusieurs minutes par rapport à l'heure réelle, suite à un problème de configuration du service de synchronisation horaire (NTP). Les utilisateurs commencent à rencontrer des échecs d'authentification intermittents et difficiles à corréler avec une cause évidente. En vous appuyant sur le fonctionnement interne de Kerberos vu en cours (section 9, synchronisation temporelle), expliquez précisément le lien entre la dérive d'horloge et les échecs d'authentification observés, et proposez à la fois une correction immédiate et une mesure préventive structurelle.

---

## Correction du TD

**1.1** — LSDOU signifie, dans l'ordre d'application (du plus général au plus spécifique) : Local (paramètres locaux de la machine elle-même), Site (GPO liés au site Active Directory), Domaine (GPO liés à l'ensemble du domaine), OU (GPO liés à l'unité d'organisation, la plus spécifique) — les paramètres appliqués plus tard dans cet ordre l'emportent généralement en cas de conflit avec un paramètre identique défini à un niveau plus général.

**1.2** — (1) Authentification initiale : l'utilisateur s'authentifie une seule fois auprès du KDC (assuré par un contrôleur de domaine), obtenant un Ticket-Granting Ticket (TGT) sans que son mot de passe ne transite sur le réseau. (2) Obtention d'un ticket de service : lorsque l'utilisateur souhaite accéder à une ressource précise, il présente son TGT au KDC et reçoit un ticket de service spécifique à cette ressource, sans nouvelle saisie de mot de passe. (3) Accès au service : l'utilisateur présente ce ticket de service directement à la ressource, qui le valide sans avoir besoin de recontacter le contrôleur de domaine pour cette vérification précise.

**1.3** — DNS résout un nom vers une adresse IP (et quelques autres types d'enregistrements limités), optimisé pour une résolution simple et rapide à très grande échelle (l'ensemble d'Internet). LDAP interroge un annuaire riche et structuré (utilisateurs, groupes, ordinateurs, avec de nombreux attributs par objet), optimisé pour des requêtes plus complexes sur un ensemble de données généralement de portée plus restreinte (une organisation) — les deux répondent à un besoin de consultation hiérarchique distribuée, mais avec des types de données et des cas d'usage très différents.

**2.1** — `gpupdate /force`.

**2.2** — `klist`.

**2.3** — `CN=Bob Martin,OU=SupportTechnique,DC=nomia,DC=local`.

**3.1** — Deux hypothèses distinctes : (1) un GPO plus spécifique, lié directement à cette OU (ou à une OU intermédiaire dans la hiérarchie menant à celle-ci), définit une valeur différente pour ce même paramètre de fond d'écran, et l'emporte sur le GPO de domaine du fait de sa position plus spécifique dans l'ordre LSDOU ; (2) un **blocage d'héritage** est actif sur cette OU (ou une OU parente), empêchant purement et simplement le GPO de domaine de s'appliquer aux objets qu'elle contient, indépendamment de tout autre GPO plus spécifique qui pourrait ou non être configuré. Dans les deux cas, la démarche de diagnostic recommandée (`gpresult /r` sur un poste de cette OU) permettrait de distinguer immédiatement laquelle des deux hypothèses est la cause réelle.

**3.2** — Ce comportement s'explique par le mécanisme des tickets Kerberos vu en cours : l'utilisateur avait déjà obtenu, avant son changement de mot de passe, un Ticket-Granting Ticket (TGT) et potentiellement des tickets de service encore valides pour les ressources concernées, à durée de vie limitée mais généralement de plusieurs heures. Ces tickets restent valides après un changement de mot de passe (le changement de mot de passe n'invalide pas rétroactivement les tickets déjà émis) jusqu'à leur expiration naturelle ou jusqu'à ce que l'utilisateur ferme sa session et s'authentifie à nouveau — expliquant pourquoi il continue d'accéder normalement aux ressources pendant un certain temps après son changement de mot de passe, sans nouvelle saisie nécessaire, tant que ses tickets existants n'ont pas expiré.

**4.1** — Kerberos repose sur une synchronisation temporelle stricte entre le client et le KDC (rappel section 9 du cours) : chaque ticket inclut un horodatage, et le protocole rejette une requête d'authentification si l'écart entre l'horloge du client et celle du KDC dépasse une tolérance configurée (typiquement de l'ordre de 5 minutes par défaut) — une mesure de sécurité destinée à limiter la fenêtre d'exploitation d'un ticket intercepté (rejeu d'un ticket capturé après un délai trop long). Une dérive d'horloge de plusieurs minutes sur les contrôleurs de domaine dépasse potentiellement cette tolérance pour certains postes clients (dont l'horloge propre peut elle-même avoir une dérive additionnelle, cumulative avec celle du contrôleur), provoquant des échecs d'authentification qui semblent intermittents et sans cause évidente car ils dépendent de l'écart cumulé exact entre chaque poste client et le contrôleur de domaine auquel il s'adresse à un instant donné, un facteur difficile à corréler visuellement sans investiguer spécifiquement la question de l'horloge système. Correction immédiate : corriger manuellement l'heure des contrôleurs de domaine affectés pour revenir à une heure exacte. Mesure préventive structurelle : vérifier et corriger la configuration du service de synchronisation horaire (NTP) sur l'ensemble de l'infrastructure, en s'assurant que les contrôleurs de domaine (généralement configurés pour se synchroniser eux-mêmes à une source de temps externe fiable, la plupart des autres machines du domaine se synchronisant alors sur eux par la hiérarchie de temps Windows) disposent d'une source de temps correctement configurée et fonctionnelle, avec une supervision de la dérive d'horloge comme mesure de détection précoce pour éviter la récurrence de ce type d'incident difficile à diagnostiquer a posteriori.

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur système junior pour créer et diagnostiquer des GPO, observer concrètement le mécanisme Kerberos, et effectuer une requête LDAP directe, en établissant systématiquement les parallèles avec les outils Linux déjà maîtrisés.

## Architecture

Une VM Windows Server contrôleur de domaine (reprise des TP précédents), avec le module GroupPolicy disponible, et idéalement un poste client joint au domaine pour observer l'application effective des GPO.

## Objectifs
- Créer et lier un GPO à une OU, observer son application
- Observer les tickets Kerberos d'une session active
- Effectuer une requête LDAP directe et via les cmdlets AD
- Diagnostiquer un cas de GPO non appliqué comme attendu

## Préparation

```powershell
Import-Module GroupPolicy, ActiveDirectory
Get-GPO -All | Select-Object DisplayName
```

## Étapes

### Étape 1 — Créer et lier un GPO

```powershell
New-GPO -Name "Lab-Politique-Ecran-Verrouillage"
Set-GPRegistryValue -Name "Lab-Politique-Ecran-Verrouillage" `
    -Key "HKCU\Software\Policies\Microsoft\Windows\Control Panel\Desktop" `
    -ValueName "ScreenSaveTimeOut" -Type DWord -Value 300

New-GPLink -Name "Lab-Politique-Ecran-Verrouillage" -Target "OU=Production,DC=nomia,DC=local"

Get-GPInheritance -Target "OU=Production,DC=nomia,DC=local"
```

### Étape 2 — Observer l'application (gpupdate/gpresult)

*Sur un poste client membre du domaine, dans l'OU Production (ou en simulant depuis le contrôleur lui-même en environnement de laboratoire simplifié) :*
```powershell
gpupdate /force
gpresult /r
```
*Question :* le GPO "Lab-Politique-Ecran-Verrouillage" apparaît-il dans la sortie de `gpresult /r` ? Si vous testez sur une machine hors de l'OU Production, quel résultat attendez-vous, et pourquoi ?

### Étape 3 — Observer Kerberos

```powershell
klist

# Provoquer une nouvelle authentification et observer le nouveau TGT
klist purge
whoami   # déclenche une vérification qui peut renouveler les tickets nécessaires
klist
```
*Question :* après `klist purge`, un nouveau ticket apparaît-il automatiquement lors de votre prochaine action nécessitant une authentification (comme l'accès à une ressource du domaine) ? Reliez votre observation au mécanisme du TGT vu en cours.

### Étape 4 — Requête LDAP directe et via cmdlets

```powershell
# Requête LDAP directe (nécessite le module ActiveDirectory, qui encapsule LDAP)
Get-ADObject -LDAPFilter "(objectClass=user)" -SearchBase "OU=Production,DC=nomia,DC=local" | Select-Object Name, DistinguishedName

# Comparaison avec la cmdlet de plus haut niveau déjà utilisée aux modules précédents
Get-ADUser -Filter * -SearchBase "OU=Production,DC=nomia,DC=local" | Select-Object Name, DistinguishedName
```
*Question :* les deux commandes produisent-elles un résultat cohérent ? Laquelle des deux exprime le plus directement une syntaxe de filtre LDAP brute, et laquelle offre une abstraction de plus haut niveau ?

## Commandes de référence
`New-GPO`, `New-GPLink`, `Get-GPInheritance`, `gpupdate`, `gpresult`, `klist`, `Get-ADObject -LDAPFilter`

## Résultats attendus
- GPO créé et lié, héritage confirmé (étape 1)
- Résultat de `gpresult /r` avec présence/absence du GPO selon la position de la machine testée (étape 2)
- Observation du cycle de vie des tickets Kerberos (étape 3)
- Comparaison requête LDAP directe / cmdlet de haut niveau (étape 4)

## Questions

1. Dans l'étape 1, pourquoi le paramètre configuré cible-t-il `HKCU` (HKEY_CURRENT_USER, rappel module 27) plutôt que `HKLM` ? Quel type de portée cela implique-t-il pour ce GPO (paramètre utilisateur vs paramètre ordinateur) ?
2. Dans l'étape 3, si vous n'aviez jamais exécuté `klist purge`, combien de temps vos tickets initiaux seraient-ils normalement restés valides selon la configuration par défaut évoquée en cours ?
3. Dans l'étape 4, quel est l'avantage pratique d'utiliser `Get-ADUser` plutôt que `Get-ADObject -LDAPFilter` pour un usage quotidien, même si les deux peuvent aboutir à un résultat similaire dans ce cas simple ?

## Troubleshooting (panne à diagnostiquer)

**Situation fournie :** après avoir créé et lié le GPO de l'étape 1, un étudiant exécute `gpupdate /force` puis `gpresult /r` sur un poste de test, mais le GPO n'apparaît toujours pas dans la liste des GPO appliqués, alors que la liaison a été confirmée par `Get-GPInheritance` à l'étape 1.

**Démarche attendue :**
1. Symptôme : GPO correctement lié selon `Get-GPInheritance`, mais absent de `gpresult /r` sur le poste testé
2. Hypothèses : (a) le poste testé n'appartient en réalité pas à l'OU "Production" ciblée par la liaison (erreur de placement de l'objet ordinateur) ; (b) un délai de réplication SYSVOL/AD (rappel section 9 du cours) empêche encore la pleine disponibilité du GPO sur le contrôleur de domaine consulté par ce poste ; (c) un filtrage de sécurité sur le GPO lui-même (les autorisations de lecture/application du GPO, configurables indépendamment de sa liaison à une OU) exclut ce poste ou cet utilisateur
3. Vérification : confirmer l'appartenance réelle du poste (ou du compte utilisateur testé) à l'OU Production (`Get-ADComputer`/`Get-ADUser` avec vérification du `DistinguishedName`) ; vérifier l'heure de création/modification du GPO comparée au moment du test, pour évaluer la plausibilité d'un délai de réplication encore en cours ; consulter les autorisations du GPO (`Get-GPPermission`)
4. Isolation : déterminer laquelle des trois hypothèses correspond à la situation observée
5. Correction : selon le cas, déplacer l'objet vers la bonne OU, attendre la réplication (ou la forcer en environnement de test avec les outils appropriés), ou ajuster les autorisations du GPO
6. Vérification : relancer `gpupdate /force` puis `gpresult /r`, confirmer l'apparition du GPO
7. Documentation : noter cette expérience comme illustration directe de la nécessité de vérifier méthodiquement plusieurs causes possibles (placement, réplication, permissions) plutôt que de supposer une seule explication, cohérent avec la méthodologie de formulation de plusieurs hypothèses déjà enseignée de façon transversale dans le programme (UE3-M22)

## Validation

Le TP est validé si l'étudiant fournit :
- Les 4 résultats/observations demandés
- Des réponses argumentées aux 3 questions
- Une démarche méthodique complète pour le scénario de troubleshooting, envisageant explicitement plusieurs hypothèses avant de conclure

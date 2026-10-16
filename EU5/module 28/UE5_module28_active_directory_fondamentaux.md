# UE5 — Module 28 : Active Directory — Fondamentaux

*Ce module compare systématiquement la gestion centralisée d'identités d'Active Directory avec la gestion locale par machine déjà étudiée sous Linux (UE3-M14).*

---

# PARTIE I — COURS

## 1. Titre du cours
Active Directory : domaines, contrôleurs de domaine et structure organisationnelle

## 2. Niveau
Élève ingénieur — cycle ingénieur, UE5 administration Windows (S2)

## 3. Durée
CM : 4h (module dense — +3h TD, +5h TP en Parties II et III)

## 4. Prérequis
- UE5-Module 27 : architecture Windows, services, registre
- UE3-Module 14 : gestion des utilisateurs et groupes sous Linux (comparaison directe et systématique)

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Expliquer le problème que résout Active Directory par rapport à une gestion d'identités machine par machine
- Décrire la hiérarchie domaine/arborescence/forêt et le rôle d'un contrôleur de domaine
- Créer et organiser des unités d'organisation (OU), utilisateurs, groupes et ordinateurs dans Active Directory
- Expliquer le mécanisme de réplication entre contrôleurs de domaine
- Comparer explicitement la gestion d'identités centralisée (AD) et locale (fichiers `/etc/passwd`/`/etc/group` par machine)
- Diagnostiquer un problème basique de réplication ou d'appartenance à un domaine

## 6. Motivation

Rappel direct de l'UE3-M14 : sous Linux, chaque machine gère ses propres comptes utilisateurs dans `/etc/passwd`/`/etc/shadow`/`/etc/group`, un modèle simple et efficace pour une machine isolée, mais qui devient rapidement ingérable à l'échelle d'une organisation de plusieurs dizaines ou centaines de postes — comment garantir qu'un employé qui quitte l'entreprise voit son accès révoqué **partout** simultanément, sans devoir intervenir manuellement sur chaque machine ? **Active Directory** répond précisément à ce problème : un annuaire centralisé d'identités et de ressources, consulté par l'ensemble des machines d'un réseau d'entreprise, où une seule modification (désactivation d'un compte, changement de mot de passe, retrait d'un groupe) se propage instantanément à toute l'infrastructure.

Ce module pose les fondations d'Active Directory, le socle sur lequel reposent la plupart des infrastructures Windows d'entreprise, et sur lequel s'appuieront directement les modules suivants (DNS/DHCP intégrés, GPO, Kerberos/LDAP).

## 7. Concepts fondamentaux

### 7.1 Le problème résolu — rappel comparatif avec Linux

| Aspect | Linux (gestion locale, UE3-M14) | Active Directory (gestion centralisée) |
|---|---|---|
| Stockage des comptes | `/etc/passwd`/`/etc/shadow` par machine | Base de données AD centralisée (NTDS.dit) sur les contrôleurs de domaine |
| Propagation d'un changement | Manuelle, machine par machine (ou via un outil externe comme Ansible/LDAP) | Automatique, répliquée à tous les contrôleurs de domaine, consultée par toutes les machines membres |
| Authentification | Locale à chaque machine (sauf intégration LDAP/AD externe) | Centralisée via le domaine (Kerberos, approfondi au module 30) |
| Scalabilité | Adaptée à un nombre limité de machines gérées individuellement | Conçue pour des organisations de toute taille |

**Point clé pédagogique :** Linux **peut** s'intégrer à un annuaire centralisé (via LDAP, SSSD, ou une intégration directe à Active Directory) — la gestion locale par `/etc/passwd` n'est pas une limitation technique insurmontable de Linux, mais le comportement par défaut pour une machine autonome. Ce module prépare d'ailleurs le module 30, où LDAP (protocole sous-jacent à Active Directory) sera présenté comme un standard ouvert, potentiellement utilisé pour fédérer des identités entre systèmes Windows et Linux.

### 7.2 Domaine, arborescence, forêt — la hiérarchie Active Directory

- **Domaine** : unité administrative de base d'Active Directory, regroupant des utilisateurs, groupes, ordinateurs et autres ressources partageant une même base de données d'annuaire et une politique de sécurité commune (ex : `nomia.local`)
- **Arborescence (tree)** : un ensemble de domaines liés par un espace de noms DNS contigu (ex : `nomia.local` et son sous-domaine `europe.nomia.local`)
- **Forêt (forest)** : le regroupement le plus large, pouvant contenir plusieurs arborescences ne partageant pas nécessairement un espace de noms contigu, mais partageant un schéma AD commun et une relation de confiance implicite entre tous les domaines qui la composent

**Pour la plupart des petites et moyennes organisations, un seul domaine dans une forêt à domaine unique** suffit amplement — la complexité multi-domaine/multi-forêt se justifie surtout pour de grandes organisations avec des besoins de séparation administrative forte (fusions d'entreprises, filiales autonomes, exigences réglementaires distinctes).

### 7.3 Contrôleur de domaine (DC)

Un **contrôleur de domaine** est un serveur Windows Server hébergeant une copie de la base de données Active Directory et assurant les services d'authentification pour le domaine. Une bonne pratique de production consiste à déployer **au moins deux contrôleurs de domaine** par domaine, pour la redondance : si un contrôleur de domaine devient indisponible, l'authentification et la consultation de l'annuaire continuent de fonctionner via l'autre.

**Rappel direct du principe de redondance déjà étudié :** cette recommandation rejoint directement la logique de redondance de serveurs DNS/DHCP déjà vue en UE2-M12, et plus généralement le principe de disponibilité déjà appliqué à travers tout le programme (RAID, sauvegardes multiples de la règle 3-2-1 en UE3-M21) — un point de défaillance unique (single point of failure) est systématiquement à éviter pour tout service critique.

### 7.4 Réplication entre contrôleurs de domaine

Lorsqu'un domaine dispose de plusieurs contrôleurs, toute modification effectuée sur l'un d'eux (création d'un utilisateur, changement de mot de passe) doit être **répliquée** vers les autres, pour que l'annuaire reste cohérent partout. Active Directory utilise un mécanisme de réplication multi-maître (multi-master) : contrairement à un modèle strictement maître-esclave, **n'importe quel contrôleur de domaine peut accepter une modification**, celle-ci étant ensuite répliquée vers les autres — un choix de conception offrant une résilience accrue (pas de contrôleur unique indispensable pour toute écriture), au prix d'une complexité de gestion des conflits potentiels si deux modifications contradictoires surviennent presque simultanément sur deux contrôleurs différents avant réplication complète.

**Le rôle FSMO (Flexible Single Master Operations) — une nuance importante :** malgré le modèle multi-maître général, certaines opérations spécifiques et sensibles d'Active Directory (par exemple, l'attribution d'identifiants uniques pour éviter les doublons, ou la gestion du schéma de l'annuaire) restent assurées par un seul contrôleur désigné à la fois, pour ces opérations précises uniquement — un compromis entre les bénéfices du multi-maître général et la nécessité d'éviter certains conflits particulièrement problématiques sur des opérations structurelles critiques.

### 7.5 Unités d'organisation (OU)

Une **unité d'organisation (Organizational Unit, OU)** est un conteneur logique au sein d'un domaine, permettant d'organiser hiérarchiquement les objets (utilisateurs, groupes, ordinateurs) selon une structure reflétant l'organisation réelle de l'entreprise (par département, par site géographique, ou toute autre logique pertinente).

```
nomia.local
├── OU=Direction
├── OU=Production
│   ├── OU=Ligne1
│   └── OU=Ligne2
├── OU=SupportTechnique
└── OU=Ordinateurs
    ├── OU=Postes_Direction
    └── OU=Postes_Production
```

**Pourquoi les OU sont-elles centrales pour l'administration ?** Elles permettent d'appliquer des **stratégies de groupe (GPO, approfondi au module 30)** et de **déléguer des droits d'administration** à un périmètre précis (par exemple, autoriser un responsable du support technique à réinitialiser les mots de passe uniquement des utilisateurs de son OU, sans lui accorder un accès administrateur complet sur tout le domaine) — une application directe et très fine du principe du moindre privilège déjà rencontré à de nombreuses reprises dans le programme (UE3-M14 notamment), ici appliquée à l'échelle d'une organisation entière plutôt qu'une seule machine.

**Distinction importante OU vs Groupe :** une OU est une structure d'**organisation administrative** (un objet ne peut appartenir qu'à une seule OU à la fois, comme un répertoire dans une arborescence de fichiers) ; un **groupe** est un regroupement logique pour l'attribution de **permissions ou de droits** (un utilisateur peut appartenir à plusieurs groupes simultanément, rappel direct des groupes secondaires multiples déjà étudiés sous Linux en UE3-M14) — les deux notions répondent à des besoins différents et se combinent, elles ne se substituent pas l'une à l'autre.

### 7.6 Types de groupes Active Directory

- **Groupes de sécurité** : utilisés pour attribuer des permissions d'accès à des ressources (fichiers, imprimantes) — l'équivalent fonctionnel direct des groupes Linux déjà étudiés (UE3-M14)
- **Groupes de distribution** : utilisés uniquement pour la messagerie électronique (listes de diffusion), sans capacité d'attribution de permissions — une distinction propre à l'écosystème Windows/Exchange, sans équivalent direct dans le modèle Linux étudié jusqu'ici

**Étendue des groupes** (portée de leur validité au sein de la forêt) : locale au domaine, globale, ou universelle — une subtilité supplémentaire du modèle AD sans équivalent direct dans la simplicité du modèle de groupes Linux local à une seule machine, mais dont la nécessité découle directement de la possibilité, pour Active Directory, de gérer plusieurs domaines au sein d'une même forêt (rappel section 7.2).

## 8. Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Forêt Active Directory                                        │
│  ┌─────────────────────────────────────────────────┐      │
│  │  Domaine: nomia.local                                  │      │
│  │                                                          │      │
│  │  ┌──────────────┐        ┌──────────────┐               │      │
│  │  │ DC1 (contrôleur)│◀──réplication──▶│ DC2 (contrôleur) │        │      │
│  │  └──────────────┘        └──────────────┘               │      │
│  │                                                          │      │
│  │  OU=Direction   OU=Production   OU=SupportTechnique         │      │
│  │  ├─ Utilisateurs  ├─ Utilisateurs  ├─ Utilisateurs             │      │
│  │  └─ Groupes         └─ Groupes         └─ Groupes                │      │
│  └─────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────┘
```

## 9. Fonctionnement interne

La base de données Active Directory est physiquement stockée dans un fichier `NTDS.dit` sur chaque contrôleur de domaine, une base de données structurée selon le schéma Active Directory (définissant les classes d'objets possibles — utilisateur, groupe, ordinateur — et leurs attributs). La réplication multi-maître (section 7.4) s'appuie sur un système de numéros de séquence de mise à jour (USN — Update Sequence Number) permettant à chaque contrôleur de déterminer précisément quelles modifications il a déjà reçues d'un autre contrôleur donné, et de ne répliquer que les changements manquants — un mécanisme d'efficacité comparable en esprit à la logique de synchronisation incrémentale déjà étudiée pour `rsync` en UE3-M21, bien que les mécanismes techniques sous-jacents diffèrent totalement.

## 10. Démonstration

```powershell
# Depuis un contrôleur de domaine, avec le module ActiveDirectory chargé
Import-Module ActiveDirectory

Get-ADDomain
Get-ADDomainController -Filter *
Get-ADOrganizationalUnit -Filter * | Select-Object Name, DistinguishedName
```

## 11. Commandes / outils

| Commande PowerShell | Rôle | Exemple |
|---|---|---|
| `Get-ADDomain` | Informations sur le domaine actuel | `Get-ADDomain` |
| `Get-ADDomainController` | Liste des contrôleurs de domaine | `Get-ADDomainController -Filter *` |
| `New-ADOrganizationalUnit` | Créer une OU | `New-ADOrganizationalUnit -Name "Production"` |
| `New-ADUser` | Créer un utilisateur AD | `New-ADUser -Name "Alice Dupont" -Path "OU=Production,DC=nomia,DC=local"` |
| `New-ADGroup` | Créer un groupe AD | `New-ADGroup -Name "GG_Production" -GroupScope Global` |
| `Add-ADGroupMember` | Ajouter un membre à un groupe | `Add-ADGroupMember -Identity "GG_Production" -Members "adupont"` |
| `Get-ADReplicationPartnerMetadata` | Vérifier l'état de réplication entre contrôleurs | `Get-ADReplicationPartnerMetadata -Target "DC1"` |

**Erreur fréquente :** confondre le nom d'affichage d'un objet AD (par exemple "Alice Dupont") avec son identifiant de connexion (`sAMAccountName`, par exemple "adupont") ou son nom distinctif complet (`DistinguishedName`, ex : `CN=Alice Dupont,OU=Production,DC=nomia,DC=local`) — les commandes PowerShell attendent généralement l'un de ces identifiants précis selon le contexte, pas le nom d'affichage informel.

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** une entreprise ayant grandi rapidement gère encore ses comptes utilisateurs manuellement sur chaque poste de travail Windows, sans Active Directory. Un audit révèle que plusieurs employés ayant quitté l'entreprise depuis plusieurs mois disposent toujours de comptes actifs sur certains postes, faute d'un processus centralisé garantissant leur désactivation systématique partout.

**Analyse attendue :** ce scénario illustre directement le problème fondamental que résout Active Directory (rappel section 7.1) : sans annuaire centralisé, chaque désactivation de compte doit être répliquée manuellement, poste par poste, un processus intrinsèquement sujet à l'oubli et à l'erreur humaine à mesure que l'organisation grandit — exactement le type de risque de sécurité (comptes orphelins actifs) déjà identifié comme prioritaire dans les audits développés au fil de l'UE3 (Permission & Account Auditor, UE3-M14). La migration vers Active Directory, avec une procédure de désactivation centralisée (désactiver un compte dans AD suffit, la modification étant immédiatement effective pour l'authentification sur toutes les machines membres du domaine), élimine structurellement ce risque récurrent. Mesure complémentaire recommandée : mettre en place un processus RH-IT formalisé garantissant qu'un départ d'employé déclenche systématiquement et rapidement la désactivation de son compte AD, plutôt que de dépendre uniquement de la capacité technique nouvellement acquise sans discipline organisationnelle associée.

## 15. Sécurité

- La centralisation d'Active Directory est une **arme à double tranchant** en matière de sécurité : elle élimine le risque de comptes orphelins dispersés (étude de cas), mais fait du contrôleur de domaine une cible de choix critique — sa compromission équivaut potentiellement à la compromission de l'ensemble de l'infrastructure qui en dépend, un exemple direct et extrême du principe qu'une centralisation de gestion est aussi une centralisation de risque
- La délégation de droits d'administration au niveau des OU (section 7.5) doit être appliquée avec la même rigueur de moindre privilège déjà enseignée pour `sudo` sous Linux (UE3-M14) — accorder par facilité des droits d'administration de domaine complets à un support technique qui n'a besoin que de réinitialiser des mots de passe reproduit exactement le même anti-pattern déjà dénoncé pour un `sudo ALL=(ALL) NOPASSWD: ALL`
- Le compte administrateur intégré du domaine (Domain Admin) doit être protégé avec une rigueur maximale (authentification forte, usage exceptionnel uniquement) — son niveau de privilège dans l'ensemble de l'infrastructure Windows dépasse largement celui d'un compte root sur une seule machine Linux, une différence d'échelle de risque à bien comprendre

## 16. Automatisation

Script PowerShell d'audit des comptes utilisateurs AD désactivés depuis longtemps mais toujours présents (candidats potentiels au nettoyage), un pendant direct de l'audit de comptes déjà réalisé pour Linux en UE3-M14.

```powershell
Import-Module ActiveDirectory

$seuilJours = 180
$dateSeuil = (Get-Date).AddDays(-$seuilJours)

$comptesADesactives = Get-ADUser -Filter {Enabled -eq $false} -Properties WhenChanged |
    Where-Object { $_.WhenChanged -lt $dateSeuil }

if ($comptesADesactives) {
    Write-Output "Comptes désactivés depuis plus de $seuilJours jours (candidats au nettoyage) :"
    $comptesADesactives | Select-Object Name, SamAccountName, WhenChanged | Format-Table -AutoSize
} else {
    Write-Output "Aucun compte désactivé ancien à signaler"
}
```

## 17. Troubleshooting

Scénario court : un utilisateur signale qu'une modification de son appartenance à un groupe, effectuée par un administrateur sur un contrôleur de domaine, ne semble pas encore prise en compte lorsqu'il tente d'accéder à une ressource depuis un poste géré par un autre contrôleur de domaine. Démarche : vérifier l'état de la réplication entre les contrôleurs concernés (`Get-ADReplicationPartnerMetadata` ou l'outil `repadmin` en ligne de commande) → si la réplication accuse un retard anormal, investiguer une cause possible (connectivité réseau entre sites, rappel direct de l'ensemble de l'UE2 sur le diagnostic réseau ; charge excessive sur un contrôleur) → si la réplication est simplement en cours normalement (un délai de propagation raisonnable existe toujours, comparable en esprit au délai de propagation DNS déjà étudié en UE2-M12), attendre sa complétion normale plutôt que de conclure prématurément à une panne.

## 18. Synthèse

- Active Directory résout le problème de la gestion d'identités à l'échelle d'une organisation, là où la gestion locale par machine (comme `/etc/passwd` sous Linux) devient rapidement ingérable
- La hiérarchie domaine/arborescence/forêt organise Active Directory à différentes échelles ; un seul domaine suffit pour la plupart des organisations
- Plusieurs contrôleurs de domaine par redondance, avec une réplication multi-maître (sauf pour les rôles FSMO spécifiques nécessitant un maître unique)
- Les unités d'organisation structurent administrativement les objets et permettent la délégation fine de droits, distinctes des groupes qui servent à l'attribution de permissions
- Le compte administrateur de domaine représente un niveau de risque bien supérieur à un compte root local, à protéger avec une rigueur proportionnelle
- Un problème de propagation de modification entre postes doit d'abord être diagnostiqué comme un possible délai de réplication normal, avant de conclure à une panne

## 19. Questions d'évaluation

**QCM**
1. Le regroupement le plus large dans la hiérarchie Active Directory est :
 a) le domaine b) l'arborescence c) la forêt d) l'unité d'organisation

2. Une unité d'organisation (OU) sert principalement à :
 a) attribuer des permissions d'accès à des fichiers b) organiser administrativement les objets et permettre la délégation de droits c) remplacer les groupes de sécurité d) gérer la réplication entre contrôleurs

**Questions ouvertes**
- Expliquez le problème que résout Active Directory par rapport à une gestion d'identités locale machine par machine, avec un exemple concret de risque évité.
- Expliquez la différence entre une OU et un groupe Active Directory, en précisant pourquoi ces deux notions ne sont pas interchangeables.

**Exercice technique**
- Donnez la commande PowerShell permettant de lister l'ensemble des contrôleurs de domaine du domaine actuel.

**Problème de diagnostic**
- Une modification de groupe effectuée sur un contrôleur de domaine ne semble pas encore visible depuis un autre contrôleur. Proposez la première vérification à effectuer avant de conclure à une panne.

## 20. Mini-projet

Développer un script PowerShell qui :
1. Audite l'ensemble des comptes désactivés depuis plus d'une durée configurable (reprise et extension du script de la section 16)
2. Vérifie l'état de réplication entre l'ensemble des contrôleurs de domaine du domaine, signalant tout retard anormal
3. Audite la structure des unités d'organisation et signale celles ne contenant aucun objet (candidates à un nettoyage de structure)
4. Produit un rapport de synthèse périodique, à la manière des outils d'audit déjà développés en Bash pour Linux au fil du programme
5. Documente les parallèles et différences avec les outils Linux équivalents déjà construits

---

# PARTIE II — TD

### Niveau 1 — Compréhension

**Exercice 1.1**
Expliquez le problème que résout Active Directory, en le comparant explicitement à la gestion locale des comptes utilisateurs sous Linux déjà étudiée en UE3-M14.

**Exercice 1.2**
Décrivez la hiérarchie domaine/arborescence/forêt d'Active Directory.

**Exercice 1.3**
Expliquez la différence entre une unité d'organisation (OU) et un groupe Active Directory.

### Niveau 2 — Application

**Exercice 2.1**
Donnez la commande PowerShell permettant d'afficher les informations du domaine actuel.

**Exercice 2.2**
Donnez la commande PowerShell permettant de créer une unité d'organisation nommée "Comptabilite".

**Exercice 2.3**
Donnez la commande PowerShell permettant d'ajouter un utilisateur "jdupont" au groupe "GG_Comptabilite".

### Niveau 3 — Analyse

**Exercice 3.1**
Une entreprise souhaite qu'un responsable du support technique puisse réinitialiser les mots de passe des utilisateurs de son propre service, sans lui accorder de droits d'administration sur l'ensemble du domaine. Expliquez comment la structure en OU d'Active Directory permet de répondre précisément à ce besoin, en reliant votre réponse au principe du moindre privilège déjà étudié pour `sudo` sous Linux.

**Exercice 3.2**
Un domaine dispose de deux contrôleurs de domaine, DC1 et DC2. Un administrateur modifie un mot de passe sur DC1. Quelques secondes plus tard, un utilisateur tente de s'authentifier via DC2 avec le nouveau mot de passe et échoue. Le lendemain, la même opération fonctionne normalement. Expliquez ce comportement en vous appuyant sur le mécanisme de réplication vu en cours, sans conclure à une panne du système.

### Niveau 4 — Problème complexe

**Exercice 4.1**
Une entreprise de taille moyenne, actuellement sans Active Directory, gère 80 postes de travail avec des comptes locaux créés manuellement sur chacun. Elle envisage une migration vers Active Directory et vous demande de justifier cette décision auprès de la direction, en anticipant une objection probable : "cela ajoute de la complexité et un point de défaillance unique (le contrôleur de domaine), pourquoi prendre ce risque ?". En vous appuyant sur les concepts vus en cours (redondance des contrôleurs, réplication multi-maître, délégation par OU), construisez une réponse argumentée qui reconnaît la validité partielle de l'objection tout en expliquant pourquoi les bénéfices l'emportent, pour une organisation de cette taille, sur les risques correctement mitigés.

---

## Correction du TD

**1.1** — Sous Linux, chaque machine gère ses propres comptes utilisateurs localement (`/etc/passwd`/`/etc/shadow`/`/etc/group`), un modèle simple mais qui devient ingérable à l'échelle d'une organisation avec de nombreuses machines : toute modification (création, désactivation, changement de mot de passe) doit être répliquée manuellement sur chaque machine concernée, un processus source d'oublis et d'incohérences. Active Directory résout ce problème en centralisant la gestion des identités dans un annuaire unique, consulté par toutes les machines membres du domaine — une seule modification effectuée dans l'annuaire est immédiatement effective pour l'authentification sur l'ensemble de l'infrastructure, sans intervention manuelle répétée machine par machine.

**1.2** — Le domaine est l'unité administrative de base, regroupant utilisateurs, groupes et ordinateurs partageant une même base d'annuaire. Une arborescence (tree) est un ensemble de domaines liés par un espace de noms DNS contigu. Une forêt (forest) est le regroupement le plus large, pouvant contenir plusieurs arborescences ne partageant pas nécessairement un espace de noms contigu, mais partageant un schéma commun et une relation de confiance implicite entre tous les domaines qu'elle contient.

**1.3** — Une OU est une structure d'organisation administrative : un conteneur logique dans lequel chaque objet (utilisateur, groupe, ordinateur) ne peut appartenir qu'à un seul endroit à la fois, utilisée notamment pour l'application de stratégies de groupe et la délégation de droits d'administration à un périmètre précis. Un groupe est un regroupement logique destiné à l'attribution de permissions ou de droits, auquel un même objet peut appartenir simultanément à plusieurs groupes différents — les deux notions répondent à des besoins distincts (organisation administrative vs attribution de permissions) et se combinent, sans se substituer l'une à l'autre.

**2.1** — `Get-ADDomain`.

**2.2** — `New-ADOrganizationalUnit -Name "Comptabilite"`.

**2.3** — `Add-ADGroupMember -Identity "GG_Comptabilite" -Members "jdupont"`.

**3.1** — La structure en OU permet de créer une OU dédiée regroupant les utilisateurs du service concerné, puis de **déléguer** à ce responsable du support technique un droit d'administration précisément limité à cette OU (par exemple, uniquement la capacité de réinitialiser les mots de passe des comptes qu'elle contient), via l'assistant de délégation de contrôle d'Active Directory — sans jamais lui accorder de droits d'administration sur l'ensemble du domaine. Ce mécanisme applique directement le principe du moindre privilège déjà étudié pour `sudo` sous Linux (UE3-M14) : de la même façon qu'une règle `sudo` peut restreindre un utilisateur à une commande précise (`sudo systemctl restart monservice`) plutôt que de lui accorder un accès root complet, la délégation par OU restreint un droit d'administration à un périmètre organisationnel précis plutôt que d'accorder un accès administrateur de domaine complet — le même principe de sécurité, transposé à une échelle organisationnelle plutôt qu'à une seule machine.

**3.2** — Ce comportement s'explique directement par le mécanisme de **réplication multi-maître** vu en cours (section 7.4) : la modification du mot de passe effectuée sur DC1 n'est pas instantanément propagée à DC2, mais nécessite un cycle de réplication qui prend un certain temps (généralement de l'ordre de quelques secondes à quelques minutes en conditions normales, potentiellement plus selon la topologie réseau et la configuration des intervalles de réplication entre sites). L'échec initial de l'utilisateur via DC2, quelques secondes après la modification sur DC1, correspond simplement à une tentative d'authentification survenue **avant** que la réplication n'ait eu le temps de se compléter — ce n'est pas une panne du système, mais un délai de propagation normal et attendu, cohérent avec le fonctionnement même du modèle multi-maître ; le lendemain, la réplication ayant eu largement le temps de se compléter, l'authentification fonctionne normalement via n'importe quel contrôleur.

**4.1** — Réponse argumentée possible : "Il est exact qu'Active Directory introduit un composant supplémentaire dans l'infrastructure, et que la disponibilité de l'authentification dépend désormais de la disponibilité d'au moins un contrôleur de domaine — mais ce risque est directement et efficacement mitigé par le déploiement d'**au moins deux contrôleurs de domaine** (recommandation standard vue en cours), avec une **réplication multi-maître** garantissant qu'aucun des deux n'est un point de défaillance unique pour les opérations courantes : si l'un devient indisponible, l'authentification et la consultation de l'annuaire continuent de fonctionner normalement via l'autre. Ce risque géré est à mettre en balance avec le risque **actuel et bien réel** de la situation présente : avec 80 postes gérés individuellement, chaque départ d'employé, chaque changement de politique de sécurité, chaque audit de conformité nécessite une intervention manuelle répétée sur chacun des postes concernés, un processus qui a déjà statistiquement de fortes chances de générer des oublis (comptes orphelins actifs, incohérences de configuration) à mesure que l'organisation continue de grandir — un risque de sécurité et de gouvernance bien plus diffus, bien plus difficile à auditer, et bien plus probable de se matérialiser silencieusement, comparé au risque, correctement mitigé par la redondance, d'une indisponibilité ponctuelle d'un composant centralisé. De plus, la délégation fine par unités d'organisation permettra de responsabiliser chaque service pour la gestion de ses propres comptes dans un périmètre limité et contrôlé, réduisant la charge actuellement portée entièrement par l'équipe informatique centrale pour chaque intervention sur chacun des 80 postes."

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur système junior pour créer la structure organisationnelle initiale d'Active Directory pour une entreprise de laboratoire, incluant utilisateurs, groupes, délégation, puis pour diagnostiquer un cas de délai de réplication.

## Architecture

Une VM Windows Server promue en contrôleur de domaine (`Install-WindowsFeature AD-Domain-Services` puis `Install-ADDSForest`, opération généralement pré-configurée par l'enseignant pour ce TP compte tenu de sa durée), avec le module PowerShell ActiveDirectory disponible. Si possible, un second contrôleur de domaine pour observer réellement la réplication ; sinon, cette partie sera traitée en réflexion argumentée.

## Objectifs
- Créer une structure d'OU reflétant une organisation réelle
- Créer des utilisateurs et des groupes, et gérer leur appartenance
- Déléguer un droit d'administration limité à une OU précise
- Observer (ou documenter en réflexion) le mécanisme de réplication

## Préparation

```powershell
Import-Module ActiveDirectory
Get-ADDomain
```

## Étapes

### Étape 1 — Créer la structure d'OU

```powershell
New-ADOrganizationalUnit -Name "Direction" -Path "DC=nomia,DC=local"
New-ADOrganizationalUnit -Name "Production" -Path "DC=nomia,DC=local"
New-ADOrganizationalUnit -Name "SupportTechnique" -Path "DC=nomia,DC=local"

Get-ADOrganizationalUnit -Filter * | Select-Object Name, DistinguishedName
```

### Étape 2 — Créer des utilisateurs et des groupes

```powershell
$motDePasseSecurise = ConvertTo-SecureString "MotDePasse!2024Lab" -AsPlainText -Force

New-ADUser -Name "Alice Dupont" -SamAccountName "adupont" `
    -Path "OU=Production,DC=nomia,DC=local" `
    -AccountPassword $motDePasseSecurise -Enabled $true

New-ADUser -Name "Bob Martin" -SamAccountName "bmartin" `
    -Path "OU=SupportTechnique,DC=nomia,DC=local" `
    -AccountPassword $motDePasseSecurise -Enabled $true

New-ADGroup -Name "GG_Production" -GroupScope Global -Path "OU=Production,DC=nomia,DC=local"
Add-ADGroupMember -Identity "GG_Production" -Members "adupont"

Get-ADGroupMember -Identity "GG_Production"
```
*Question :* pourquoi le mot de passe est-il fourni via `ConvertTo-SecureString` plutôt qu'en texte brut directement dans la commande ? Reliez votre réponse aux principes de sécurité déjà rencontrés dans le programme.

### Étape 3 — Délégation de droits sur une OU

```powershell
# Illustration de la démarche (l'assistant de délégation graphique est souvent utilisé en pratique,
# mais la commande dsacls permet une délégation en ligne de commande)
dsacls "OU=SupportTechnique,DC=nomia,DC=local" /I:S /G "NOMIA\bmartin:CA;Reset Password;user"
```
*Question :* que signifie cette commande en langage clair ? À quel principe déjà étudié en UE3-M14 (délégation de privilège précis plutôt qu'accès complet) cette opération correspond-elle directement ?

### Étape 4 — Réplication (observation ou réflexion documentée)

*Si un second contrôleur de domaine est disponible :*
```powershell
Get-ADReplicationPartnerMetadata -Target (Get-ADDomainController -Filter *).Name
```

*Si un seul contrôleur est disponible dans votre environnement de laboratoire :* documentez, à partir de la section 7.4 du cours, ce que cette commande révélerait dans un environnement à plusieurs contrôleurs (dernière réplication réussie, partenaire de réplication, éventuel retard), et pourquoi cette vérification serait le premier réflexe face à un symptôme de propagation lente d'une modification.

## Commandes de référence
`New-ADOrganizationalUnit`, `New-ADUser`, `New-ADGroup`, `Add-ADGroupMember`, `Get-ADReplicationPartnerMetadata`

## Résultats attendus
- Structure d'OU créée (étape 1)
- Utilisateurs et groupe créés avec appartenance confirmée (étape 2)
- Explication de la commande de délégation (étape 3)
- Résultat ou réflexion documentée sur la réplication (étape 4)

## Questions

1. Dans l'étape 2, si vous aviez oublié de spécifier `-Enabled $true`, quel aurait été l'état du compte créé par défaut ? Pourquoi ce comportement par défaut est-il cohérent avec une bonne pratique de sécurité (rappel possible d'un principe déjà vu dans le programme) ?
2. Dans l'étape 1, pourquoi est-il recommandé de réfléchir à la structure des OU **avant** de créer des utilisateurs, plutôt que de créer les utilisateurs d'abord et de les réorganiser après coup ?
3. Si l'entreprise venait à fusionner avec une autre société disposant de son propre domaine Active Directory, quel concept vu en section 7.2 du cours permettrait d'envisager une intégration progressive entre les deux infrastructures existantes ?

## Troubleshooting (panne à diagnostiquer)

**Situation fournie :** un étudiant crée un nouvel utilisateur sur un contrôleur de domaine (dans un environnement à deux contrôleurs), puis, quelques secondes plus tard, tente de l'observer via une commande PowerShell exécutée en se connectant explicitement à l'autre contrôleur de domaine, sans succès — l'utilisateur nouvellement créé n'apparaît pas encore.

**Démarche attendue :**
1. Symptôme : un objet créé sur un contrôleur n'est pas immédiatement visible depuis l'autre
2. Hypothèse : délai de réplication normal (rappel direct de l'exercice 3.2 du TD), pas une erreur de création ni une panne
3. Vérification : `Get-ADReplicationPartnerMetadata` sur les deux contrôleurs pour observer l'heure de la dernière réplication réussie entre eux, et la comparer à l'heure de création de l'utilisateur
4. Isolation : si la dernière réplication réussie est antérieure à la création de l'utilisateur, cela confirme qu'un cycle de réplication supplémentaire est simplement nécessaire et n'a pas encore eu lieu
5. Correction : attendre le prochain cycle de réplication normal (ou le déclencher manuellement en environnement de test avec `repadmin /syncall`, une commande utile en laboratoire mais à utiliser avec discernement en production où la réplication automatique est généralement suffisante et préférable)
6. Vérification : réinterroger le second contrôleur après le délai ou la synchronisation manuelle, confirmer que l'utilisateur est désormais visible
7. Documentation : noter ce comportement comme un rappel qu'un système distribué avec réplication (Active Directory, mais le principe est transversal et déjà rencontré ailleurs dans le programme sous d'autres formes, comme le cache DNS de l'UE2-M12) implique presque toujours un délai de cohérence entre les nœuds, à anticiper dans toute procédure de test ou de vérification immédiate après une modification

## Validation

Le TP est validé si l'étudiant fournit :
- Les 4 résultats/réflexions demandés
- Des réponses argumentées aux 3 questions
- Une identification correcte de la cause du scénario de troubleshooting (délai de réplication normal), cohérente avec le principe déjà établi dans le cours et le TD

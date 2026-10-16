# UE5 — Module 31 : Permissions NTFS et ACL

*Ce module compare systématiquement le modèle NTFS avec le modèle Unix (rwx, ACL étendues) déjà étudié en profondeur en UE3-Module 14.*

---

# PARTIE I — COURS

## 1. Titre du cours
Permissions NTFS et ACL : un modèle différent de la permission Unix

## 2. Niveau
Élève ingénieur — cycle ingénieur, UE5 administration Windows (S2)

## 3. Durée
CM : 3h30 (module orienté comparaison — +3h TD, +5h TP en Parties II et III)

## 4. Prérequis
- UE3-Module 14 : permissions Unix (rwx, octal, SUID/SGID, ACL étendues) — comparaison systématique tout au long de ce module
- UE5-Module 28 : groupes Active Directory (utilisés comme sujets de permission NTFS)

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Décrire le modèle de permissions NTFS (ACE, ACL, permissions de base et avancées)
- Expliquer la règle de priorité "Refus explicite l'emporte sur Autorisation", une différence fondamentale avec le modèle Unix
- Expliquer l'héritage de permissions NTFS et le comparer à l'héritage SGID déjà étudié sous Unix
- Calculer les permissions effectives résultant de la combinaison de permissions de partage et de permissions NTFS
- Utiliser `icacls` pour auditer et modifier des permissions en ligne de commande
- Diagnostiquer un problème d'accès à une ressource lié à une combinaison de permissions incohérente

## 6. Motivation

Rappel direct de l'UE3-M14 : le modèle Unix repose sur trois catégories fixes (propriétaire/groupe/autres) avec des permissions simples (rwx), complété par des ACL étendues pour des besoins plus fins. NTFS, le système de fichiers natif de Windows, adopte une approche structurellement différente dès sa conception : chaque fichier ou répertoire porte une **liste de contrôle d'accès (ACL)** composée d'un nombre arbitraire d'entrées, chacune associant un utilisateur ou un groupe à un ensemble précis de permissions, avec une possibilité explicite de **refus** (deny) — un concept qui n'a pas d'équivalent direct dans le modèle Unix de base (bien que les ACL étendues Unix s'en rapprochent partiellement).

Ce module ne réapprend pas la notion de permission depuis zéro — il transpose et confronte le modèle déjà maîtrisé à un système différent, révélant à la fois des convergences (le besoin fondamental de contrôle d'accès) et des divergences significatives (la sémantique du refus explicite notamment) qu'un ingénieur évoluant entre les deux écosystèmes doit maîtriser sans les confondre.

## 7. Concepts fondamentaux

### 7.1 ACE et ACL — structure de base

Une **ACL (Access Control List)** NTFS est composée d'une liste d'**ACE (Access Control Entries)**, chacune spécifiant :
- Un **sujet** (utilisateur ou groupe, généralement un objet Active Directory rappel UE5-M28)
- Un **type** : Autoriser (Allow) ou **Refuser (Deny)**
- Un **ensemble de permissions** (lecture, écriture, exécution, modification, contrôle total...)

**Comparaison directe avec le modèle Unix (UE3-M14) :** là où Unix limite nativement à trois sujets fixes (propriétaire/groupe/autres, une ACL étendue permettant d'ajouter des sujets supplémentaires), NTFS autorise nativement un nombre arbitraire de sujets dans son ACL dès la conception de base — le modèle NTFS "de base" est en réalité toujours, structurellement, ce que Unix appellerait une ACL étendue.

### 7.2 Permissions de base et avancées

**Permissions de base** (les plus couramment utilisées, visibles dans l'onglet Sécurité standard) :

| Permission | Effet |
|---|---|
| Contrôle total | Tous les droits, y compris modifier les permissions elles-mêmes |
| Modification | Lecture, écriture, exécution, suppression |
| Lecture et exécution | Lecture du contenu, exécution si applicable |
| Lecture seule | Lecture du contenu uniquement |
| Écriture | Ajout/modification de contenu |

**Permissions avancées** : un ensemble bien plus granulaire (Parcourir le dossier/Exécuter le fichier, Lister le dossier/Lire les données, Attributs de lecture, Créer des fichiers/Écrire des données, Supprimer, Autorisations de lecture, Modifier les autorisations, Prendre possession...) — chaque permission de base regroupe en réalité plusieurs permissions avancées, de façon comparable en esprit (bien que bien plus granulaire) à la façon dont une permission `rwx` Unix combine trois droits distincts en une notation compacte.

### 7.3 La règle du refus explicite — différence fondamentale avec Unix

**Le point le plus important de ce module, sans équivalent direct dans le modèle Unix de base :** lorsqu'une ACL NTFS contient à la fois une autorisation (Allow) et un refus (Deny) applicables à un même sujet pour une même permission (par exemple, un utilisateur autorisé via son appartenance à un groupe, mais explicitement refusé individuellement), **le refus l'emporte toujours**, indépendamment de l'ordre des entrées dans l'ACL.

**Pourquoi cette règle existe-t-elle ?** Elle permet un cas d'usage précis et utile : autoriser un accès large via un groupe (par simplicité de gestion), tout en excluant explicitement un ou quelques membres spécifiques de ce groupe pour cette ressource précise, sans avoir à restructurer l'appartenance aux groupes eux-mêmes. Un exemple concret : le groupe "Tous les employés" a accès en lecture à un dossier d'annonces internes, mais un employé spécifique en congé disciplinaire se voit refuser explicitement cet accès, sans devoir être retiré du groupe "Tous les employés" (qui lui donne accès à de nombreuses autres ressources légitimes).

**Comparaison avec Unix :** le modèle Unix de base n'a pas de notion de "refus explicite" — les permissions sont purement additives selon la catégorie applicable (propriétaire, groupe, ou autres), sans mécanisme natif pour exclure un utilisateur précis d'un groupe auquel il appartient légitimement pour une ressource donnée. Les ACL étendues Unix (`setfacl`, déjà étudiées en UE3-M14) permettent d'ajouter des entrées avec des permissions précises, mais ne proposent pas non plus, de façon aussi structurée, un mécanisme de "refus l'emportant sur autorisation" comparable à NTFS — une différence philosophique réelle entre les deux systèmes, pas seulement une différence de vocabulaire.

### 7.4 Héritage de permissions — comparaison avec SGID

Rappel direct UE3-M14 : sous Unix, le bit SGID sur un répertoire fait hériter le **groupe propriétaire** à tout nouveau fichier créé à l'intérieur — mais ce mécanisme ne propage pas les permissions elles-mêmes, seulement le groupe.

Sous NTFS, l'**héritage de permissions** est un mécanisme bien plus direct et complet : les ACE définies sur un dossier parent peuvent être configurées pour se propager automatiquement à tous les fichiers et sous-dossiers créés à l'intérieur, avec un marquage explicite distinguant les permissions **héritées** (grisées dans l'interface, non modifiables directement à ce niveau) des permissions **explicites** (définies directement sur l'objet lui-même, prioritaires sur les permissions héritées en cas de conflit).

**Interruption de l'héritage :** il est possible de désactiver l'héritage pour un objet précis (en conservant une copie des permissions héritées au moment de la désactivation, ou en les supprimant entièrement pour repartir sur des permissions explicites propres) — un mécanisme sans équivalent direct dans le modèle Unix, où le SGID influence la création future mais ne "bloque" jamais un héritage de permission déjà en place au sens NTFS.

### 7.5 Permissions de partage vs permissions NTFS — le piège de la combinaison

Une ressource accédée via le réseau (un partage SMB, `\\serveur\partage`) est soumise à **deux couches de permissions distinctes** :
- Les **permissions de partage** (Share permissions), configurées au niveau du partage réseau lui-même
- Les **permissions NTFS**, configurées sur le système de fichiers sous-jacent

**Règle de combinaison — la plus restrictive l'emporte :** l'accès effectif d'un utilisateur à une ressource partagée est déterminé par l'**intersection** des deux ensembles de permissions — même si les permissions NTFS autorisent le contrôle total, une permission de partage limitée à "Lecture seule" restreint effectivement l'accès à la lecture seule pour tout accès via ce partage réseau (bien qu'un accès local direct au fichier, sans passer par le partage, resterait soumis uniquement aux permissions NTFS).

**Bonne pratique largement répandue :** configurer les permissions de partage de façon large et permissive (souvent "Contrôle total" pour le groupe "Tout le monde" au niveau du partage), et gérer la granularité fine du contrôle d'accès **exclusivement** au niveau NTFS — évitant ainsi d'avoir à maintenir une cohérence entre deux systèmes de permissions distincts, une source classique de confusion et d'erreur de configuration si les deux sont utilisés de façon également restrictive et potentiellement contradictoire.

### 7.6 `icacls` — gestion en ligne de commande

```powershell
icacls "C:\Data\Production"                                    # afficher les permissions actuelles
icacls "C:\Data\Production" /grant "NOMIA\GG_Production:(OI)(CI)M"  # accorder Modification, héritée par sous-objets
icacls "C:\Data\Production" /deny "NOMIA\jdupont:(OI)(CI)W"          # refuser explicitement l'écriture à un utilisateur précis
icacls "C:\Data\Production" /remove "NOMIA\ancien_employe"             # retirer toute entrée pour ce sujet
```

**Codes d'héritage** : `(OI)` (Object Inherit — hérité par les fichiers), `(CI)` (Container Inherit — hérité par les sous-dossiers) — une syntaxe technique mais précise pour scripter des configurations de permissions reproductibles, un pendant direct des commandes `setfacl` déjà pratiquées sous Unix (UE3-M14).

## 8. Architecture

```
┌────────────────────────────────────────────────────────────┐
│  Fichier/Dossier NTFS                                            │
│  ┌────────────────────────────────────────────────┐       │
│  │  ACL                                                   │       │
│  │  ┌──────────────────────────────────────┐              │       │
│  │  │ ACE 1: Allow GG_Production - Modification (héritée) │              │       │
│  │  │ ACE 2: Deny jdupont - Write (explicite)               │              │       │
│  │  │ ACE 3: Allow Administrateurs - Contrôle total          │              │       │
│  │  └──────────────────────────────────────┘              │       │
│  └────────────────────────────────────────────────┘       │
│                                                                    │
│  Résultat pour jdupont (membre de GG_Production) :                   │
│  Deny l'emporte sur Allow → écriture refusée malgré               │
│  l'appartenance au groupe autorisé                                  │
└────────────────────────────────────────────────────────────┘
```

## 9. Fonctionnement interne

Le noyau Windows vérifie les permissions d'accès à chaque tentative d'ouverture de fichier en parcourant l'ACL de l'objet et en évaluant chaque ACE applicable au jeton d'accès (access token) du processus demandeur — ce jeton contenant l'identité de l'utilisateur ainsi que la liste de tous les groupes AD auxquels il appartient (obtenue lors de l'authentification Kerberos initiale, rappel direct UE5-M30). L'algorithme d'évaluation Windows traite les ACE dans un ordre spécifique où les refus explicites sont généralement évalués prioritairement (bien que l'implémentation exacte dépende de l'ordre canonique des ACE au sein de l'ACL, généralement maintenu automatiquement par le système pour placer les refus explicites en premier) — garantissant que la règle de la section 7.3 s'applique de façon cohérente et prévisible, indépendamment de l'ordre dans lequel les ACE auraient pu être ajoutées historiquement par un administrateur.

## 10. Démonstration

```powershell
# Observer les permissions d'un dossier
Get-Acl "C:\Data\Production" | Format-List

# Observer via icacls
icacls "C:\Data\Production"

# Observer l'effet combiné partage + NTFS
Get-SmbShare -Name "Production" | Get-SmbShareAccess
```

## 11. Commandes / outils

| Commande | Rôle | Exemple |
|---|---|---|
| `Get-Acl` / `Set-Acl` (PowerShell) | Lire/modifier une ACL | `Get-Acl "C:\Data"` |
| `icacls` | Gestion en ligne de commande, scriptable | `icacls "C:\Data" /grant User:M` |
| `Get-SmbShareAccess` | Permissions de partage (distinct de NTFS) | `Get-SmbShareAccess -Name "Production"` |
| Onglet Sécurité (interface graphique) | Gestion visuelle des ACL | — |
| `Get-ADPermission` (pour les objets AD, pas les fichiers) | Permissions sur les objets Active Directory eux-mêmes | — |

**Erreur fréquente :** oublier la règle de priorité du refus explicite (section 7.3) et s'étonner qu'un utilisateur membre d'un groupe autorisé continue de se voir refuser l'accès — un refus, même ancien ou apparemment mineur dans l'historique des modifications, l'emporte toujours sur toute autorisation, quelle que soit sa source (directe ou via un autre groupe).

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** un utilisateur, membre du groupe "GG_Production" disposant d'un accès en Modification sur un dossier partagé, signale ne pas pouvoir écrire dans ce dossier, malgré une vérification de son appartenance au groupe confirmée par l'administrateur. Un audit des permissions NTFS du dossier ne révèle, à première vue, aucune entrée de refus explicite visible.

**Analyse attendue :** ce scénario illustre un piège de diagnostic à deux niveaux, directement lié aux sections 7.3 et 7.5 du cours. Premièrement, l'absence apparente de refus explicite au niveau du dossier consulté ne garantit pas l'absence d'un refus hérité d'un niveau parent, ou d'un refus appliqué directement à l'utilisateur individuellement (à vérifier séparément de l'entrée du groupe, un audit superficiel se limitant à l'entrée du groupe pouvant manquer une entrée de refus individuelle distincte et moins visible). Deuxièmement, et souvent négligé, la vérification des permissions NTFS seules ne suffit pas si l'accès se fait via un partage réseau : une permission de partage restrictive (rappel section 7.5, "la plus restrictive l'emporte") pourrait limiter l'accès effectif indépendamment de permissions NTFS par ailleurs correctes. Démarche de diagnostic complète : vérifier systématiquement les permissions effectives (via l'onglet "Permissions effectives" de l'interface graphique, ou un audit `icacls` complet remontant la hiérarchie de dossiers), puis vérifier séparément les permissions de partage si l'accès problématique transite par un partage réseau — ne jamais se limiter à l'examen du seul niveau NTFS immédiat du dossier concerné.

## 15. Sécurité

- La règle du refus explicite (7.3) est un outil de sécurité précieux pour des exceptions ponctuelles, mais son usage excessif ou mal documenté peut rendre une structure de permissions extrêmement difficile à auditer et à comprendre — chaque refus explicite devrait être documenté avec sa justification, un principe de traçabilité déjà rencontré à plusieurs reprises dans le programme pour toute exception de sécurité
- La bonne pratique de permissions de partage larges combinée à un contrôle NTFS fin (7.5) simplifie l'audit de sécurité : un seul système de permissions à examiner en détail plutôt que deux systèmes potentiellement contradictoires à réconcilier mentalement
- L'héritage de permissions (7.4), bien que pratique pour une gestion cohérente à grande échelle, peut aussi propager une erreur de configuration à un niveau élevé de l'arborescence vers un très grand nombre de fichiers et sous-dossiers en aval — une modification de permission sur un dossier racine important mérite une vérification particulièrement attentive avant application, compte tenu de son impact potentiellement très large
- `icacls` et les scripts PowerShell de gestion d'ACL constituent, comme pour les scripts Bash étudiés en UE4, une surface où la rigueur (validation des entrées, tests avant application réelle) est essentielle — une erreur de script appliquée à grande échelle sur une arborescence de production peut avoir des conséquences bien plus larges qu'une erreur manuelle isolée

## 16. Automatisation

Script PowerShell d'audit des permissions NTFS d'une arborescence, signalant les entrées de refus explicite (à documenter) et les ruptures d'héritage (à justifier) — un pendant direct des audits de permissions déjà réalisés pour Unix en UE3-M14.

```powershell
function Auditer-PermissionsNTFS {
    param(
        [string]$Chemin
    )

    Get-ChildItem -Path $Chemin -Recurse -Directory | ForEach-Object {
        $acl = Get-Acl -Path $_.FullName

        $refusExplicites = $acl.Access | Where-Object { $_.AccessControlType -eq "Deny" -and -not $_.IsInherited }
        if ($refusExplicites) {
            Write-Warning "Refus explicite trouvé sur $($_.FullName) :"
            $refusExplicites | ForEach-Object { Write-Output "  $($_.IdentityReference) - $($_.FileSystemRights)" }
        }

        if ($acl.AreAccessRulesProtected) {
            Write-Warning "Héritage rompu sur $($_.FullName) - vérifier la justification"
        }
    }
}

Auditer-PermissionsNTFS -Chemin "C:\Data\Production"
```

## 17. Troubleshooting

Scénario court : après une réorganisation de la structure de dossiers d'un service, plusieurs utilisateurs signalent des pertes d'accès inattendues et incohérentes à divers sous-dossiers, sans schéma évident. Démarche : vérifier si l'héritage de permissions a été interrompu par erreur lors du déplacement ou de la copie de dossiers (rappel section 7.4 — un déplacement de dossier au sein d'un même volume NTFS conserve généralement les permissions existantes, mais une copie ou un déplacement entre volumes différents peut réinitialiser l'héritage aux permissions par défaut du dossier de destination, un piège fréquent et une distinction technique importante) → auditer systématiquement (via le script de la section 16 ou un examen manuel) les dossiers concernés pour repérer les ruptures d'héritage inattendues → rétablir l'héritage approprié ou reconfigurer explicitement les permissions manquantes selon l'intention réelle de la réorganisation.

## 18. Synthèse

- NTFS structure ses permissions en ACL composées d'ACE, nativement plus riches que le modèle Unix de base (mais comparable en esprit aux ACL étendues Unix déjà étudiées)
- Le refus explicite l'emporte toujours sur une autorisation, une règle sans équivalent direct dans le modèle Unix, utile pour des exceptions ponctuelles mais à documenter rigoureusement
- L'héritage NTFS propage les permissions elles-mêmes (pas seulement le groupe comme SGID), avec une possibilité d'interruption explicite
- Les permissions de partage et NTFS se combinent selon la règle "la plus restrictive l'emporte" — la bonne pratique consiste à garder le partage large et gérer la granularité au niveau NTFS uniquement
- Un déplacement/copie entre volumes NTFS différents peut réinitialiser l'héritage de permissions, un piège fréquent lors de réorganisations
- Un diagnostic d'accès refusé doit systématiquement vérifier les permissions effectives complètes (hiérarchie NTFS ET partage), jamais se limiter à un seul niveau ou une seule couche

## 19. Questions d'évaluation

**QCM**
1. Lorsqu'une ACL NTFS contient à la fois une autorisation et un refus pour un même sujet et une même permission :
 a) l'autorisation l'emporte toujours b) le refus l'emporte toujours c) le dernier ajouté l'emporte d) aucun accès n'est possible dans les deux cas

2. La combinaison de permissions de partage et de permissions NTFS suit la règle :
 a) la plus permissive l'emporte b) la plus restrictive l'emporte c) seules les permissions NTFS comptent d) seules les permissions de partage comptent

**Questions ouvertes**
- Expliquez la règle de priorité du refus explicite NTFS et donnez un cas d'usage légitime pour cette fonctionnalité.
- Expliquez pourquoi un déplacement de dossier entre deux volumes NTFS différents peut modifier les permissions héritées, contrairement à un déplacement au sein du même volume.

**Exercice technique**
- Donnez la commande `icacls` permettant de refuser explicitement l'accès en écriture à un utilisateur "NOMIA\test" sur un dossier "C:\Data\Confidentiel", avec héritage aux sous-fichiers et sous-dossiers.

**Problème de diagnostic**
- Un utilisateur membre d'un groupe autorisé en Modification ne parvient pas à écrire dans un dossier, sans refus explicite visible à ce niveau précis. Proposez deux pistes de vérification distinctes.

## 20. Mini-projet

Développer un script PowerShell qui :
1. Audite une arborescence complète pour détecter les refus explicites et les ruptures d'héritage (reprise et extension du script de la section 16)
2. Calcule les permissions effectives d'un utilisateur ou groupe donné sur un chemin précis, en tenant compte de l'héritage et de la règle de priorité du refus
3. Vérifie la cohérence entre permissions de partage et permissions NTFS pour les partages configurés sur le serveur, signalant toute combinaison où le partage serait plus restrictif que le NTFS (source probable de confusion pour les administrateurs, rappel section 7.5)
4. Produit un rapport de synthèse structuré
5. Documente les parallèles et différences avec l'audit de permissions Unix déjà développé en UE3-M14 (Permission & Account Auditor)

---

# PARTIE II — TD

### Niveau 1 — Compréhension

**Exercice 1.1**
Expliquez la structure d'une ACL NTFS (ACE, sujet, type, permissions), en la comparant au modèle Unix de base.

**Exercice 1.2**
Expliquez la règle de priorité entre autorisation et refus dans une ACL NTFS.

**Exercice 1.3**
Expliquez la différence entre l'héritage de permissions NTFS et le mécanisme SGID déjà étudié sous Unix.

### Niveau 2 — Application

**Exercice 2.1**
Donnez la commande `icacls` permettant d'afficher les permissions actuelles d'un dossier "C:\Data\Ventes".

**Exercice 2.2**
Donnez la commande `icacls` permettant d'accorder la permission de Modification (héritée par sous-fichiers et sous-dossiers) au groupe "NOMIA\GG_Ventes" sur ce même dossier.

**Exercice 2.3**
Donnez la commande PowerShell permettant d'observer les permissions de partage (distinctes des permissions NTFS) d'un partage nommé "Ventes".

### Niveau 3 — Analyse

**Exercice 3.1**
Un dossier a des permissions de partage limitées à "Lecture seule" pour le groupe "Tout le monde", mais des permissions NTFS accordant "Contrôle total" au même groupe. Un utilisateur accédant à ce dossier via le partage réseau (`\\serveur\dossier`) parvient-il à modifier un fichier ? Justifiez en vous appuyant sur la règle de combinaison vue en cours.

**Exercice 3.2**
Un administrateur copie un dossier contenant des permissions NTFS personnalisées depuis un volume `D:` vers un volume `E:` sur le même serveur. Après la copie, il constate que les permissions du dossier de destination ne correspondent plus à celles d'origine, mais à celles par défaut du dossier parent de destination. Expliquez ce comportement.

### Niveau 4 — Problème complexe

**Exercice 4.1**
Un utilisateur, membre du groupe "GG_Production" disposant d'un accès en Modification sur un dossier partagé "\\serveur\Production", signale ne pas pouvoir écrire dans ce dossier. Un audit NTFS du dossier lui-même ne révèle aucun refus explicite pour cet utilisateur ni pour le groupe GG_Production. En vous appuyant sur l'ensemble des mécanismes vus en cours (héritage, refus explicite à un niveau différent, permissions de partage), construisez une démarche de diagnostic complète en au moins 4 étapes, permettant d'identifier la cause réelle parmi plusieurs hypothèses possibles, sans vous arrêter à la première vérification superficielle qui semblerait ne rien révéler d'anormal.

---

## Correction du TD

**1.1** — Une ACL NTFS est composée d'une liste d'ACE (Access Control Entries), chacune associant un sujet (utilisateur ou groupe) à un type (Autoriser ou Refuser) et un ensemble de permissions précises. Contrairement au modèle Unix de base, limité nativement à trois catégories fixes (propriétaire/groupe/autres), NTFS autorise nativement un nombre arbitraire de sujets dans son ACL dès sa conception — le modèle NTFS de base est structurellement comparable à ce que Unix appellerait une ACL étendue, sans distinction entre un modèle "simple" et un modèle "étendu" comme c'est le cas sous Unix.

**1.2** — Lorsqu'une ACL contient à la fois une autorisation et un refus applicables à un même sujet pour une même permission, le **refus l'emporte toujours**, indépendamment de l'ordre des entrées dans l'ACL ou de leur source (directe ou via l'appartenance à un groupe). Cette règle n'a pas d'équivalent direct dans le modèle Unix de base, qui ne dispose pas nativement d'un mécanisme de refus explicite prioritaire sur une autorisation.

**1.3** — L'héritage NTFS propage directement les **permissions elles-mêmes** (l'ensemble complet des ACE) d'un dossier parent vers les fichiers et sous-dossiers créés à l'intérieur, avec une distinction visible entre permissions héritées et explicites. Le SGID sous Unix ne fait hériter que le **groupe propriétaire** d'un nouveau fichier créé dans le répertoire concerné, sans propager les permissions elles-mêmes — un nouveau fichier créé dans un répertoire SGID hérite du bon groupe, mais ses permissions initiales dépendent toujours du umask de l'utilisateur créateur, pas d'une copie des permissions du répertoire parent.

**2.1** — `icacls "C:\Data\Ventes"`.

**2.2** — `icacls "C:\Data\Ventes" /grant "NOMIA\GG_Ventes:(OI)(CI)M"`.

**2.3** — `Get-SmbShareAccess -Name "Ventes"`.

**3.1** — Non, l'utilisateur ne parvient pas à modifier le fichier via le partage réseau. La règle de combinaison vue en cours stipule que l'accès effectif est déterminé par l'**intersection** des permissions de partage et des permissions NTFS — la plus restrictive des deux l'emporte. Ici, la permission de partage "Lecture seule" est plus restrictive que la permission NTFS "Contrôle total", donc l'accès effectif via ce partage reste limité à la lecture seule, malgré des permissions NTFS par ailleurs très permissives. Un accès local direct au fichier (sans passer par le partage réseau, par exemple en étant physiquement connecté au serveur) ne serait, à l'inverse, soumis qu'aux permissions NTFS, et permettrait alors la modification.

**3.2** — Ce comportement s'explique par la différence entre un déplacement au sein d'un **même volume NTFS** et un déplacement (ou une copie) **entre deux volumes différents**. Au sein d'un même volume, un déplacement de dossier est en réalité une simple opération de renommage/relocalisation dans la structure du système de fichiers, qui **conserve** les permissions NTFS existantes du dossier déplacé. Entre deux volumes différents (comme ici, de `D:` vers `E:`), l'opération est en réalité une véritable **copie** suivie d'une suppression de l'original, et un nouveau fichier/dossier créé sur un volume NTFS hérite par défaut des permissions de son dossier parent de destination (comportement standard de création d'un nouvel objet NTFS), plutôt que de conserver les permissions personnalisées d'origine, qui ne sont pas préservées dans ce type d'opération inter-volumes.

**4.1** — Démarche de diagnostic complète, au-delà de la vérification superficielle initiale déjà effectuée sans résultat concluant :

1. **Vérifier les permissions effectives complètes**, pas seulement une lecture rapide de l'ACL du dossier lui-même — utiliser l'onglet "Permissions effectives" (ou un audit `icacls` remontant explicitement toute la hiérarchie de dossiers parents) pour l'utilisateur précis concerné, en tenant compte de tous les groupes auxquels il appartient réellement (pas seulement GG_Production, un utilisateur pouvant appartenir à d'autres groupes portant potentiellement un refus explicite non lié directement à GG_Production)

2. **Vérifier l'existence d'un refus explicite à un niveau parent** dans la hiérarchie de dossiers, hérité jusqu'au dossier consulté mais potentiellement moins visible qu'un refus directement présent sur le dossier lui-même — un audit superficiel limité au dossier immédiat pourrait manquer un refus hérité d'un ancêtre plus haut dans l'arborescence

3. **Vérifier séparément les permissions de partage** (`Get-SmbShareAccess`) du partage réseau utilisé pour cet accès, indépendamment des permissions NTFS déjà examinées — rappel direct de la section 7.5 et du piège déjà identifié dans l'étude de cas du cours, la combinaison la plus restrictive l'emportant

4. **Vérifier l'appartenance réelle et à jour de l'utilisateur au groupe GG_Production** au niveau Active Directory lui-même (rappel UE5-M28) — en tenant compte d'un possible délai de réplication si l'ajout au groupe est très récent, ou d'une confusion entre plusieurs groupes au nommage proche

Cette démarche en quatre étapes, plus exhaustive que la simple vérification initiale du dossier lui-même, permet d'identifier la cause réelle parmi plusieurs hypothèses distinctes plutôt que de conclure à tort à une absence de cause identifiable après un examen superficiel insuffisant.

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur système junior pour configurer et auditer des permissions NTFS complexes, incluant un refus explicite, une combinaison partage/NTFS, et pour diagnostiquer le scénario complet de l'étude de cas du cours.

## Architecture

Une VM Windows Server (membre du domaine des TP précédents si possible, pour disposer de groupes AD réels), avec un volume NTFS pour les tests.

## Objectifs
- Configurer une structure de permissions NTFS avec héritage
- Observer concrètement la règle de priorité du refus explicite
- Configurer et observer la combinaison permissions de partage/NTFS
- Diagnostiquer le scénario complet à plusieurs causes possibles

## Préparation

```powershell
New-Item -Path "C:\Data\Lab" -ItemType Directory -Force
```

## Étapes

### Étape 1 — Structure de base et héritage

```powershell
icacls "C:\Data\Lab" /inheritance:r   # désactive l'héritage entrant, repart d'une base propre pour le TP
icacls "C:\Data\Lab" /grant "NOMIA\GG_Production:(OI)(CI)M"
icacls "C:\Data\Lab" /grant "BUILTIN\Administrators:(OI)(CI)F"

New-Item -Path "C:\Data\Lab\SousDossier" -ItemType Directory
icacls "C:\Data\Lab\SousDossier"   # observer l'héritage automatique depuis le parent
```
*Question :* les permissions accordées sur `C:\Data\Lab` apparaissent-elles automatiquement sur `SousDossier` ? Sont-elles marquées comme héritées ou explicites dans la sortie `icacls` ?

### Étape 2 — Refus explicite prioritaire

```powershell
# Créer un utilisateur de test membre de GG_Production (adapter selon votre structure AD des TP précédents)
icacls "C:\Data\Lab" /deny "NOMIA:jdupont:(OI)(CI)W"

icacls "C:\Data\Lab"
```
*Simuler l'accès (ou documenter le résultat attendu si un compte de test n'est pas directement testable dans votre environnement) :* l'utilisateur "jdupont", bien que membre du groupe GG_Production autorisé en Modification, devrait se voir refuser spécifiquement l'écriture, conformément à la règle de priorité vue en cours.

*Question :* si vous retiriez maintenant "jdupont" du groupe GG_Production, le refus explicite aurait-il encore un effet visible ? Pourquoi ou pourquoi non ?

### Étape 3 — Combinaison partage/NTFS

```powershell
New-SmbShare -Name "LabPartage" -Path "C:\Data\Lab" -FullAccess "Everyone"
Get-SmbShareAccess -Name "LabPartage"

# Restreindre volontairement le partage plus que le NTFS
Grant-SmbShareAccess -Name "LabPartage" -AccountName "Everyone" -AccessRight Read -Force
Get-SmbShareAccess -Name "LabPartage"

icacls "C:\Data\Lab"   # rappel : permissions NTFS restées à Modification/Contrôle total pour les groupes concernés
```
*Question :* un utilisateur membre de GG_Production, accédant via `\\localhost\LabPartage`, pourrait-il désormais écrire dans ce dossier ? Justifiez avec la règle de combinaison vue en cours.

### Étape 4 — Diagnostic du scénario complet de l'étude de cas

*Scénario fourni :* reproduisez, à partir de la structure des étapes 1 à 3, une situation où un utilisateur membre de GG_Production ne parvient pas à écrire dans `LabPartage`, sans savoir a priori laquelle des causes (refus NTFS hérité, refus individuel, restriction de partage) en est responsable.

```powershell
# Appliquez la méthodologie de diagnostic en 4 étapes de l'exercice 4.1 du TD
icacls "C:\Data\Lab" /findsid "NOMIA\jdupont"   # ou une recherche équivalente pour identifier tous les refus concernant cet utilisateur
Get-SmbShareAccess -Name "LabPartage"
```
*À vous de jouer :* documentez votre démarche complète de diagnostic, en identifiant explicitement laquelle des causes possibles (ou combinaison de causes) explique le blocage dans votre configuration de laboratoire.

## Commandes de référence
`icacls`, `Get-SmbShareAccess`, `Grant-SmbShareAccess`, `New-SmbShare`

## Résultats attendus
- Confirmation de l'héritage automatique (étape 1)
- Preuve du refus explicite configuré, avec réflexion sur son maintien après retrait du groupe (étape 2)
- Analyse de la combinaison partage restrictif/NTFS permissif (étape 3)
- Démarche de diagnostic complète et conclusion pour le scénario de l'étape 4

## Questions

1. Dans l'étape 1, pourquoi avoir utilisé `/inheritance:r` avant de configurer les permissions du TP ? Que se serait-il passé si des permissions héritées d'un niveau supérieur (non maîtrisées dans le cadre de ce TP) étaient restées actives en plus de celles configurées ?
2. Dans l'étape 2, la règle du refus explicite s'applique-t-elle uniquement aux utilisateurs individuels, ou pourrait-elle également être appliquée à un groupe entier ? Quel serait l'effet si le refus avait ciblé le groupe GG_Production lui-même plutôt qu'un utilisateur individuel ?
3. Dans l'étape 3, quelle est la bonne pratique recommandée en cours pour éviter ce type de configuration source de confusion (partage plus restrictif que NTFS) ? Reformulez cette bonne pratique avec vos propres mots.

## Troubleshooting (panne à diagnostiquer)

**Situation fournie :** après avoir suivi les étapes 1 à 3, un étudiant retire le refus explicite de l'étape 2 (`icacls "C:\Data\Lab" /remove:d "NOMIA:jdupont"`), s'attendant à ce que l'utilisateur jdupont retrouve immédiatement un accès en écriture complet via le partage, mais constate que l'accès en écriture reste toujours limité.

**Démarche attendue :**
1. Symptôme : suppression du refus explicite NTFS, mais accès en écriture toujours non rétabli comme attendu
2. Hypothèse : l'étudiant a probablement oublié que la restriction de partage configurée à l'étape 3 (limitée à "Lecture seule" pour "Everyone") reste active indépendamment de la suppression du refus NTFS — les deux couches de restriction (NTFS et partage) sont indépendantes, et la suppression de l'une ne rétablit pas automatiquement un accès complet si l'autre reste elle-même restrictive
3. Vérification : `Get-SmbShareAccess -Name "LabPartage"` pour confirmer que la restriction de partage à "Lecture seule" est toujours active malgré la suppression du refus NTFS
4. Isolation : confirmer que les permissions NTFS, une fois le refus explicite retiré, autorisent bien désormais la Modification pour ce sujet (`icacls "C:\Data\Lab"`), isolant ainsi la permission de partage comme seule cause restante du blocage persistant
5. Correction : rétablir une permission de partage cohérente avec l'intention réelle (`Grant-SmbShareAccess -Name "LabPartage" -AccountName "Everyone" -AccessRight Full -Force`, ou toute permission de partage suffisamment large selon la bonne pratique déjà recommandée en cours de gérer la granularité au niveau NTFS uniquement)
6. Vérification : confirmer que l'accès en écriture est désormais effectivement rétabli via le partage
7. Documentation : noter cette expérience comme illustration concrète et personnellement vécue de la règle de combinaison de la section 7.5 — deux couches de permissions indépendantes doivent chacune être vérifiées et corrigées séparément, la correction d'une seule ne suffisant jamais si l'autre reste elle-même restrictive

## Validation

Le TP est validé si l'étudiant fournit :
- Les 4 résultats/analyses demandés
- Des réponses argumentées aux 3 questions
- Une identification correcte de la cause du scénario de troubleshooting (permission de partage restée restrictive), avec une correction cohérente

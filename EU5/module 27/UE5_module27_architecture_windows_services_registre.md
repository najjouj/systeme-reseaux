# UE5 — Module 27 : Architecture Windows, Services et Registre

*Premier module de l'UE5 — mobilise systématiquement les comparaisons avec l'architecture Linux déjà étudiée en UE1 (kernel/user space, systemd, journald).*

---

# PARTIE I — COURS

## 1. Titre du cours
Architecture Windows : noyau, services et registre — comparaison avec Linux

## 2. Niveau
Élève ingénieur — cycle ingénieur, entrée en UE5 administration Windows (S2)

## 3. Durée
CM : 4h (module d'ouverture UE5 — +3h TD, +5h TP en Parties II et III)

## 4. Prérequis
- UE1-Module 1 : architecture générale d'un OS, kernel space vs user space
- UE1-Module 6 : boot process, systemd, PID 1, journalctl (mobilisés en comparaison constante)
- UE3-Module 16 : gestion pratique des services sous Linux (comparaison directe)

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Décrire l'architecture Windows (mode noyau/mode utilisateur, HAL, sous-systèmes) et la comparer à l'architecture Linux déjà étudiée
- Expliquer le rôle du Service Control Manager (SCM) et le comparer à systemd
- Décrire la structure du registre Windows (ruches, clés, valeurs) et son rôle comparé à la configuration Linux distribuée en fichiers texte
- Utiliser l'Observateur d'événements (Event Viewer) pour diagnostiquer un problème, en le comparant à `journalctl`
- Gérer des services Windows (démarrage, arrêt, dépendances, type de démarrage) via l'interface graphique et PowerShell
- Diagnostiquer un service Windows qui ne démarre pas

## 6. Motivation

Vous maîtrisez déjà en profondeur l'architecture Linux : le noyau, systemd comme PID 1, la configuration distribuée en fichiers texte sous `/etc`, `journalctl` pour la journalisation. Windows Server répond aux mêmes besoins fondamentaux — démarrer des services, gérer une configuration système, journaliser les événements — mais avec des mécanismes et une philosophie sensiblement différents. Ce module ne réapprend pas ces concepts depuis zéro : il les **transpose**, en s'appuyant systématiquement sur vos acquis Linux comme point de référence, pour accélérer l'apprentissage et surtout développer la capacité à naviguer entre les deux écosystèmes — une compétence de plus en plus recherchée dans des infrastructures hybrides Linux/Windows, la norme dans la plupart des grandes organisations.

## 7. Concepts fondamentaux

### 7.1 Architecture Windows — vue d'ensemble

Comme Linux (rappel UE1-M1), Windows distingue un **mode noyau** (kernel mode) et un **mode utilisateur** (user mode), avec la même motivation fondamentale : isolation et sécurité, empêchant un programme utilisateur d'accéder directement au matériel ou à la mémoire d'autres processus.

```
┌─────────────────────────────────────────────────────┐
│  Mode utilisateur                                        │
│  Applications, services utilisateur, sous-systèmes         │
│  (Win32, WSL...)                                            │
├─────────────────────────────────────────────────────┤
│  Mode noyau                                              │
│  Executive (gestion mémoire, processus, E/S...)            │
│  Kernel (ordonnancement, synchronisation)                    │
│  HAL (Hardware Abstraction Layer)                             │
├─────────────────────────────────────────────────────┤
│  Matériel                                                │
└─────────────────────────────────────────────────────┘
```

**HAL (Hardware Abstraction Layer) — une différence architecturale notable :** Windows introduit une couche d'abstraction matérielle explicite et nommée entre le noyau et le matériel, facilitant la portabilité de Windows entre différentes architectures matérielles. Linux réalise une abstraction comparable, mais de façon plus diffuse à travers ses pilotes et son architecture modulaire, sans une couche unique aussi explicitement nommée et centralisée.

**Sous-systèmes** : Windows peut exécuter plusieurs "sous-systèmes" en mode utilisateur (historiquement POSIX, OS/2 ; aujourd'hui principalement Win32, et WSL — Windows Subsystem for Linux — qui permet d'exécuter un environnement Linux quasi natif directement sous Windows, un pont direct et de plus en plus utilisé entre les deux mondes étudiés dans ce programme).

### 7.2 Le Service Control Manager (SCM) — comparaison directe avec systemd

Le **SCM** (`services.exe`) est le gestionnaire central des services Windows — l'équivalent fonctionnel de systemd déjà étudié en profondeur (UE1-M6, UE3-M16), avec des différences de conception significatives :

| Aspect | systemd (Linux) | SCM (Windows) |
|---|---|---|
| Définition d'un service | Fichier `.service` texte (INI) | Entrée dans le registre (voir 7.3) |
| Démarrage parallèle | Oui, par graphe de dépendances | Historiquement plus séquentiel, amélioré avec le "démarrage différé" (Delayed Start) |
| Dépendances | `After=`/`Requires=`/`Wants=` | `DependOnService` (registre) |
| Journalisation | `journald` intégré nativement | Journal des événements (Event Log), mécanisme distinct |
| Outil de gestion | `systemctl` | `services.msc` (graphique) ou PowerShell (`Get-Service`, `Start-Service`...) |
| Redémarrage automatique | `Restart=`, avec garde-fous `StartLimitBurst` | Configuration "Recovery" par service, avec compteur de tentatives similaire en esprit |

**Point clé pédagogique :** malgré des mécanismes très différents, le SCM répond exactement au même besoin fondamental que systemd — démarrer, arrêter, superviser et redémarrer automatiquement des processus de fond, avec gestion de dépendances. Cette équivalence fonctionnelle, malgré une implémentation radicalement différente, est une leçon transversale importante : les problèmes d'ingénierie système se répètent d'un OS à l'autre, seules les solutions concrètes diffèrent.

### 7.3 Le registre Windows — comparaison avec la configuration Linux

Contrairement à Linux, où la configuration système est distribuée dans de nombreux fichiers texte sous `/etc` (rappel constant depuis l'UE1), Windows centralise l'essentiel de sa configuration système et applicative dans une base de données hiérarchique unique : le **registre**.

**Structure :** le registre est organisé en **ruches** (hives), chacune une racine logique :

| Ruche | Rôle |
|---|---|
| `HKEY_LOCAL_MACHINE` (HKLM) | Configuration matérielle et logicielle de la machine, incluant les services (équivalent en esprit à `/etc` pour la configuration système globale) |
| `HKEY_CURRENT_USER` (HKCU) | Configuration propre à l'utilisateur actuellement connecté |
| `HKEY_USERS` (HKU) | Configuration de tous les profils utilisateurs |
| `HKEY_CLASSES_ROOT` (HKCR) | Associations de types de fichiers, informations COM |
| `HKEY_CURRENT_CONFIG` (HKCC) | Profil matériel actif |

Chaque ruche contient des **clés** (organisées hiérarchiquement, comme des répertoires) contenant elles-mêmes des **valeurs** (nommées, typées — chaîne, nombre, binaire — comparables en esprit à une ligne `variable=valeur` dans un fichier de configuration Linux, mais structurée et typée nativement plutôt que du texte libre).

**Où trouver la définition d'un service dans le registre :** `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\<nom_du_service>`, contenant notamment le chemin de l'exécutable, le type de démarrage, et les dépendances — l'équivalent structurel du contenu d'un fichier `.service` systemd, mais stocké dans une base de registre plutôt que dans un fichier texte lisible directement.

**Avantages et inconvénients de cette centralisation, en miroir avec Linux :**
- **Avantage Windows** : un point d'accès unique et structuré, avec des outils de recherche et de sauvegarde centralisés
- **Avantage Linux** : des fichiers texte, éditables avec n'importe quel éditeur, versionnable facilement avec un outil comme Git, lisibles et modifiables sans outil spécialisé
- **Risque partagé** : dans les deux cas, une modification incorrecte de la configuration système (un fichier `/etc` corrompu, ou une clé de registre supprimée par erreur) peut empêcher le système de démarrer correctement — d'où l'importance, dans les deux mondes, de sauvegarder avant toute modification manuelle risquée

### 7.4 L'Observateur d'événements (Event Viewer) — comparaison avec journalctl

Windows centralise sa journalisation système dans le **Journal des événements** (Event Log), consultable via l'**Observateur d'événements** (`eventvwr.msc`) ou PowerShell (`Get-EventLog`/`Get-WinEvent`) — l'équivalent fonctionnel direct de `journalctl` déjà pratiqué intensivement en UE3-M16/M19.

**Journaux principaux :**
- **Application** : événements générés par les applications
- **Sécurité** : événements d'audit (connexions, accès à des ressources) — nécessite une politique d'audit activée
- **Système** : événements générés par le système d'exploitation lui-même et ses composants (comparable en esprit aux messages noyau visibles via `dmesg`/`journalctl -k` sous Linux)

**Niveaux de sévérité** (comparables aux niveaux syslog déjà étudiés en UE3-M19) : Information, Avertissement (Warning), Erreur (Error), Critique.

```powershell
Get-WinEvent -LogName System -MaxEvents 20
Get-WinEvent -FilterHashtable @{LogName='System'; Level=2} -MaxEvents 10   # Level 2 = Erreur
```

**Différence notable avec journald :** le Journal des événements Windows sépare structurellement les journaux par catégorie logique (Application/Sécurité/Système, et bien d'autres pour des rôles spécifiques comme Active Directory), alors que `journald` centralise tout dans un flux unique, filtrable après coup par unité ou service (`journalctl -u`) — deux philosophies différentes pour un même besoin de traçabilité système.

### 7.5 Gestion pratique des services Windows

**Via l'interface graphique** (`services.msc`) : liste des services avec leur état, type de démarrage (Automatique, Automatique différé, Manuel, Désactivé), possibilité de démarrer/arrêter/redémarrer, configurer les dépendances et le comportement de récupération en cas d'échec.

**Via PowerShell** (approfondi au module 32, utilisé ici en première prise en main) :
```powershell
Get-Service -Name "wuauserv"                      # état d'un service (Windows Update, exemple)
Start-Service -Name "wuauserv"
Stop-Service -Name "wuauserv"
Set-Service -Name "wuauserv" -StartupType Automatic
Get-Service | Where-Object {$_.Status -eq "Stopped" -and $_.StartType -eq "Automatic"}   # services censés être actifs mais arrêtés
```

**Types de démarrage — nuance importante :**
- **Automatique** : démarré au boot, immédiatement
- **Automatique (démarrage différé)** : démarré au boot, mais après les services automatiques standards, réduisant la contention au démarrage — un mécanisme dont l'intention rejoint celle des dépendances `After=` de systemd, sans en être un équivalent strict
- **Manuel** : démarré uniquement à la demande (par un utilisateur, ou par un autre service qui en dépend)
- **Désactivé** : ne peut pas être démarré, même manuellement, sans réactivation préalable

## 8. Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Registre (HKLM\SYSTEM\CurrentControlSet\Services\...)        │
│         │                                                      │
│         ▼                                                      │
│  Service Control Manager (SCM)                                   │
│  ── résout les dépendances (DependOnService)                       │
│  ── démarre/arrête les services                                     │
│         │                                                      │
│         ▼                                                      │
│  Processus de service (svchost.exe ou exécutable dédié)             │
│         │                                                      │
│         ▼                                                      │
│  Journal des événements (Application/Système/Sécurité)                │
│         │                                                      │
│         ▼                                                      │
│  Observateur d'événements / Get-WinEvent                             │
└─────────────────────────────────────────────────────────┘
```

## 9. Fonctionnement interne

De nombreux services Windows partagent en réalité un même processus hôte générique, `svchost.exe`, plutôt que chacun disposant de son propre exécutable dédié — une optimisation de ressources permettant de regrouper plusieurs services légers au sein d'un même processus, chacun chargé comme une DLL au sein de ce processus hôte partagé. Cette architecture, bien que différente du modèle Linux (où chaque service systemd correspond typiquement à son propre processus, éventuellement avec des enfants), répond à une préoccupation similaire de gestion des ressources — avec un compromis inverse en termes de diagnostic : sous Linux, `ps`/`systemctl status` identifie immédiatement le processus d'un service précis ; sous Windows, plusieurs services partageant un même `svchost.exe` nécessitent un outil dédié (`tasklist /svc`, ou l'onglet Services du Gestionnaire des tâches) pour identifier quel service correspond à quelle instance de `svchost.exe`.

## 10. Démonstration

```powershell
# Observer les services et leur état
Get-Service | Sort-Object Status -Descending | Select-Object -First 10

# Observer une clé de registre liée à un service
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Spooler"

# Consulter les événements système récents de niveau erreur
Get-WinEvent -FilterHashtable @{LogName='System'; Level=2} -MaxEvents 5
```

## 11. Commandes / outils

| Commande/Outil | Rôle | Exemple |
|---|---|---|
| `services.msc` | Interface graphique de gestion des services | — |
| `Get-Service`/`Start-Service`/`Stop-Service` | Gestion des services en PowerShell | `Get-Service wuauserv` |
| `regedit` | Éditeur de registre graphique | — |
| `Get-ItemProperty` (PowerShell) | Lire une valeur de registre en ligne de commande | `Get-ItemProperty -Path "HKLM:\..."` |
| `eventvwr.msc` | Observateur d'événements graphique | — |
| `Get-WinEvent` | Consultation des journaux d'événements en PowerShell | `Get-WinEvent -LogName System` |
| `tasklist /svc` | Associer les processus `svchost.exe` à leurs services | `tasklist /svc` |

**Erreur fréquente :** modifier directement une clé de registre sans sauvegarde préalable (`reg export` avant modification) — une erreur de registre peut, comme un fichier de configuration Linux corrompu, empêcher un service ou même le système entier de démarrer correctement, avec un processus de récupération potentiellement plus complexe qu'une simple restauration d'un fichier texte depuis une sauvegarde ou un contrôle de version.

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** une équipe migrant progressivement d'une infrastructure Windows vers un environnement hybride Linux/Windows peine à adapter ses réflexes de diagnostic : habituée à `journalctl -u <service>` pour consulter les logs d'un service Linux, elle cherche un équivalent direct sous Windows et se heurte à la séparation par catégorie du Journal des événements plutôt qu'un flux unifié par service.

**Analyse attendue :** ce cas illustre concrètement la différence de philosophie déjà soulignée en section 7.4 : `journalctl -u <service>` filtre un flux unique par l'unité systemd responsable, une opération directe et immédiate. Sous Windows, l'équivalent le plus proche nécessite de connaître dans quel journal (Application, Système, ou un journal dédié à un rôle spécifique comme Active Directory) le service concerné écrit ses événements, puis de filtrer ce journal par la source de l'événement (`Get-WinEvent -FilterHashtable @{LogName='Application'; ProviderName='NomDuService'}`) — une démarche en deux temps plutôt qu'un filtrage direct par nom de service. La bonne pratique pour une équipe dans cette situation est de documenter, pour chaque service critique de son infrastructure Windows, dans quel journal et sous quel nom de fournisseur (`ProviderName`) ses événements apparaissent, construisant ainsi progressivement l'équivalent d'une table de correspondance service/journal qui n'existe pas nativement de façon aussi unifiée que sous `journalctl -u`.

## 15. Sécurité

- Le registre, en tant que point de configuration centralisé et critique, doit être protégé par des permissions appropriées (rappel du principe du moindre privilège déjà établi en UE3-M14) — toute clé modifiable par un utilisateur non privilégié pourrait potentiellement être exploitée pour une élévation de privilèges ou une persistance malveillante (technique d'attaque bien documentée consistant à ajouter une entrée de démarrage automatique dans le registre)
- Le Journal des événements Sécurité, une fois une politique d'audit correctement configurée, est une source d'information essentielle pour la détection d'incidents de sécurité (tentatives de connexion échouées, modifications de permissions sensibles) — une discipline directement comparable à la surveillance des journaux d'authentification SSH déjà étudiée en UE3-M18
- Un service configuré pour s'exécuter avec un compte disposant de privilèges excessifs (par exemple SYSTEM alors qu'un compte de service dédié aux droits restreints suffirait) reproduit exactement le même risque déjà identifié pour les services Linux mal configurés (UE3-M16) — le principe du moindre privilège s'applique identiquement, indépendamment du système d'exploitation

## 16. Automatisation

Script PowerShell d'audit des services configurés en démarrage automatique mais actuellement arrêtés — l'équivalent direct, en esprit, du script de vérification de cohérence déjà écrit en Bash pour systemd (UE3-M16), ici transposé à l'écosystème Windows.

```powershell
# Audite les services censés démarrer automatiquement mais actuellement arrêtés
$servicesProblematiques = Get-Service | Where-Object {
    $_.StartType -eq "Automatic" -and $_.Status -eq "Stopped"
}

if ($servicesProblematiques) {
    Write-Warning "Services automatiques actuellement arrêtés :"
    $servicesProblematiques | Format-Table Name, DisplayName, Status
} else {
    Write-Output "OK : tous les services automatiques sont actifs"
}
```

*Note pédagogique :* ce script, volontairement simple, préfigure directement le module 32 (PowerShell fondamentaux) — sa syntaxe (`Where-Object`, pipeline d'objets) sera expliquée en détail dans ce module suivant ; il est présenté ici pour illustrer, dès ce premier module, la continuité de la démarche d'automatisation déjà pratiquée intensivement en Bash (UE4), transposée à un nouvel écosystème.

## 17. Troubleshooting

Scénario court : un service Windows configuré en démarrage automatique n'est pas actif après un redémarrage du serveur, sans message d'erreur immédiatement visible à l'utilisateur. Démarche : consulter l'état du service (`Get-Service` ou `services.msc`) pour confirmer son état actuel → consulter le Journal des événements Système, filtré autour de l'heure de démarrage du système, pour rechercher un événement lié à l'échec de démarrage de ce service (rappel direct de la méthodologie déjà appliquée à `journalctl -u <service> -b` sous Linux, UE3-M16/M22) → vérifier les dépendances du service (`Get-Service -Name <service> -RequiredServices`, équivalent PowerShell) pour s'assurer qu'aucune dépendance elle-même en échec n'empêche son démarrage → vérifier le compte utilisé par le service et ses permissions si l'échec semble lié à un problème d'accès à une ressource.

## 18. Synthèse

- Windows distingue mode noyau/mode utilisateur comme Linux, avec une HAL explicitement nommée et des sous-systèmes (dont WSL, pont direct vers Linux)
- Le SCM est l'équivalent fonctionnel de systemd, avec une gestion de dépendances et un mécanisme de récupération automatique comparables en esprit, malgré une implémentation différente
- Le registre centralise la configuration système (ruches, clés, valeurs), à l'inverse de la configuration Linux distribuée en fichiers texte sous `/etc` — chaque approche a ses avantages propres, avec un risque partagé de corruption en cas de modification incorrecte
- Le Journal des événements est l'équivalent de `journald`, mais organisé par catégories logiques (Application/Système/Sécurité) plutôt qu'un flux unique filtrable par service comme `journalctl -u`
- Les types de démarrage de service (Automatique, Automatique différé, Manuel, Désactivé) répondent à des besoins similaires aux dépendances et à l'activation systemd, sans être des équivalents stricts
- La méthodologie de diagnostic d'un service qui ne démarre pas transpose directement celle déjà maîtrisée sous Linux (état → logs → dépendances → permissions)

## 19. Questions d'évaluation

**QCM**
1. L'équivalent fonctionnel de systemd sous Windows est :
 a) le registre b) le Service Control Manager (SCM) c) l'Observateur d'événements d) PowerShell

2. La configuration des services Windows est stockée principalement dans :
 a) des fichiers texte sous `/etc` b) le registre, sous HKEY_LOCAL_MACHINE c) l'Observateur d'événements d) un fichier `.service`

**Questions ouvertes**
- Comparez la philosophie de configuration système entre Linux (fichiers texte distribués) et Windows (registre centralisé), avec un avantage et un inconvénient de chaque approche.
- Expliquez la différence entre `journalctl -u <service>` et l'équivalent le plus proche sous Windows pour consulter les logs d'un service précis.

**Exercice technique**
- Donnez la commande PowerShell permettant de lister tous les services actuellement arrêtés dont le type de démarrage est Automatique.

**Problème de diagnostic**
- Un service Windows configuré en démarrage automatique n'est pas actif après un redémarrage. Proposez une démarche de diagnostic en au moins 3 étapes, en établissant le parallèle avec la méthodologie déjà utilisée sous Linux.

## 20. Mini-projet

Développer un script PowerShell (en s'appuyant sur les éléments de syntaxe déjà entrevus dans ce module, approfondis au module 32) qui :
1. Audite l'ensemble des services configurés en démarrage automatique et vérifie leur état actuel (reprise et extension du script de la section 16)
2. Pour chaque service problématique détecté, consulte le Journal des événements Système à la recherche d'un événement d'erreur associé
3. Produit un rapport structuré, à la manière des scripts d'audit déjà développés en Bash au fil du programme (UE1 à UE4), démontrant la transférabilité de la démarche d'ingénierie au-delà du seul écosystème Linux
4. Documente les limites rencontrées dans cette première approche PowerShell, avant l'approfondissement syntaxique du module 32

---

# PARTIE II — TD

### Niveau 1 — Compréhension

**Exercice 1.1**
Décrivez l'architecture noyau/utilisateur de Windows et comparez-la à celle de Linux déjà étudiée en UE1.

**Exercice 1.2**
Expliquez le rôle du Service Control Manager, en le comparant explicitement à systemd.

**Exercice 1.3**
Décrivez la structure du registre Windows (ruches, clés, valeurs) et son rôle comparé à la configuration Linux sous `/etc`.

### Niveau 2 — Application

**Exercice 2.1**
Donnez la commande PowerShell permettant d'afficher l'état d'un service nommé "Spooler".

**Exercice 2.2**
Donnez le chemin de registre où se trouve la définition des services Windows.

**Exercice 2.3**
Donnez la commande PowerShell permettant de consulter les 10 derniers événements du journal Système.

### Niveau 3 — Analyse

**Exercice 3.1**
Un administrateur habitué à Linux cherche l'équivalent de `journalctl -u nginx -b` pour consulter les logs d'un service IIS (serveur web Windows) depuis le dernier démarrage. Expliquez pourquoi il n'existe pas d'équivalent aussi direct, et décrivez la démarche à suivre sous Windows pour obtenir une information comparable.

**Exercice 3.2**
Un service Windows est configuré en démarrage "Automatique (différé)" plutôt que "Automatique". Expliquez la différence pratique entre ces deux options, et proposez un cas d'usage où le démarrage différé serait préférable.

### Niveau 4 — Problème complexe

**Exercice 4.1**
Une équipe migre un service applicatif depuis un serveur Linux (où il était géré par systemd, avec `After=network-online.target` et `Requires=postgresql.service`) vers un serveur Windows. Elle souhaite reproduire un comportement de démarrage équivalent : attendre que le réseau soit disponible, et ne démarrer qu'après un service de base de données déjà migré séparément. En vous appuyant sur les mécanismes Windows vus en cours (types de démarrage, dépendances de service), proposez une configuration Windows reproduisant, autant que possible, l'intention de la configuration systemd d'origine, en identifiant explicitement les limites de cette transposition (ce qui ne peut pas être reproduit à l'identique).

---

## Correction du TD

**1.1** — Windows, comme Linux, distingue un mode noyau (accès privilégié au matériel et à la mémoire système) et un mode utilisateur (applications isolées, sans accès direct au matériel). Windows ajoute une couche d'abstraction matérielle explicitement nommée (HAL), facilitant la portabilité entre architectures matérielles ; Linux réalise une abstraction comparable de façon plus diffuse à travers son architecture de pilotes modulaire, sans couche unique aussi centralisée et nommée. Les deux systèmes partagent la même motivation fondamentale : isolation et sécurité entre code privilégié et code applicatif.

**1.2** — Le Service Control Manager (SCM, `services.exe`) est le gestionnaire central des services Windows, responsable de leur démarrage, arrêt, supervision et gestion des dépendances — un rôle fonctionnellement identique à celui de systemd sous Linux. Les différences résident dans l'implémentation : les services sont définis dans le registre plutôt que dans des fichiers `.service` texte, la journalisation passe par le Journal des événements plutôt que `journald` intégré nativement, et le démarrage est historiquement plus séquentiel que le modèle de graphe de dépendances parallèle de systemd, bien qu'amélioré par le mécanisme de démarrage différé.

**1.3** — Le registre est organisé en ruches (racines logiques comme HKEY_LOCAL_MACHINE ou HKEY_CURRENT_USER), chacune contenant des clés organisées hiérarchiquement (comparables à des répertoires), elles-mêmes contenant des valeurs nommées et typées. Il centralise l'essentiel de la configuration système et applicative de Windows, à l'inverse de Linux qui distribue sa configuration dans de nombreux fichiers texte sous `/etc` — le registre offre un point d'accès unique et structuré (avec recherche et sauvegarde centralisées), tandis que l'approche Linux offre une lisibilité directe, une éditabilité avec n'importe quel outil texte, et une compatibilité naturelle avec le contrôle de version.

**2.1** — `Get-Service -Name "Spooler"`.

**2.2** — `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\`.

**2.3** — `Get-WinEvent -LogName System -MaxEvents 10`.

**3.1** — Il n'existe pas d'équivalent aussi direct car le Journal des événements Windows organise ses journaux par **catégorie logique** (Application, Système, Sécurité, et des journaux dédiés à certains rôles) plutôt que par un flux unique filtrable directement par le nom du service responsable, comme le permet `journalctl -u` grâce à l'association native de chaque entrée de journal à son unité systemd d'origine. La démarche à suivre sous Windows consiste à identifier dans quel journal le service IIS écrit ses événements (probablement le journal Application, ou un journal dédié IIS selon la configuration), puis à filtrer ce journal par la source de l'événement (`ProviderName`) correspondant à IIS, avec `Get-WinEvent -FilterHashtable @{LogName='Application'; ProviderName='...'}` — une démarche en deux temps (identifier le journal, puis filtrer par source) plutôt qu'un filtrage direct et immédiat par nom de service.

**3.2** — Un service en démarrage "Automatique" démarre immédiatement au boot, dès que le système d'exploitation atteint le point où les services automatiques sont lancés. Un service en "Automatique (différé)" démarre également automatiquement, mais après un court délai, une fois que les services automatiques standards ont eu l'occasion de démarrer en priorité — réduisant la contention de ressources (CPU, disque) pendant la phase critique de démarrage du système. Cas d'usage où le démarrage différé serait préférable : un service non critique pour la disponibilité immédiate du système au démarrage (par exemple, un service de mise à jour ou de télémétrie), qui peut sans inconvénient démarrer quelques secondes ou minutes après les services essentiels, allégeant ainsi la charge au moment le plus sensible du démarrage.

**4.1** — Configuration Windows proposée : configurer le service applicatif avec un type de démarrage "Automatique (différé)" plutôt que "Automatique" immédiat, pour réduire la probabilité qu'il ne démarre avant que le réseau ne soit pleinement opérationnel — une reproduction **partielle et approximative** de l'intention de `After=network-online.target`, sans garantie stricte équivalente (le délai différé n'attend pas explicitement une confirmation de disponibilité réseau, contrairement à la cible systemd dédiée). Pour la dépendance vis-à-vis du service de base de données, configurer explicitement `DependOnService` dans la définition du service Windows (via le registre ou l'outil `sc.exe`/PowerShell `New-Service -DependsOn`), pointant vers le service de base de données Windows correspondant — une reproduction plus fidèle de l'intention de `Requires=postgresql.service`, le SCM garantissant que le service dépendant ne démarre qu'après le service de base de données. **Limites explicites de cette transposition :** contrairement à `After=network-online.target`, qui attend une confirmation active et spécifique de disponibilité réseau complète (pas seulement que l'interface soit techniquement "up"), Windows ne propose pas nativement d'équivalent aussi précis et garanti pour une dépendance réseau — le délai différé reste une approximation temporelle, pas une véritable vérification de disponibilité, ce qui constitue une limite réelle et à documenter explicitement pour l'équipe reprenant ce service migré, qui ne doit pas supposer une garantie équivalente à celle offerte par systemd sur ce point précis.

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur système junior pour prendre en main l'administration d'un serveur Windows Server de laboratoire, en explorant systématiquement les parallèles avec les outils Linux déjà maîtrisés, puis pour diagnostiquer un service en échec.

## Architecture

Une VM Windows Server (édition Evaluation disponible gratuitement auprès de Microsoft si nécessaire pour l'environnement de laboratoire), avec accès PowerShell en tant qu'administrateur.

## Objectifs
- Explorer les services Windows et leur définition dans le registre
- Consulter et filtrer le Journal des événements
- Provoquer et diagnostiquer un service en échec de démarrage
- Établir des parallèles explicites avec les outils Linux déjà maîtrisés

## Préparation

```powershell
Get-Host | Select-Object Version
whoami /priv | Select-String "SeDebugPrivilege"   # vérifier les privilèges administrateur, adapter si nécessaire
```

## Étapes

### Étape 1 — Explorer les services et leur définition registre

```powershell
Get-Service | Sort-Object Status, DisplayName | Format-Table -AutoSize | Select-Object -First 20

# Choisir un service actif et examiner sa définition dans le registre
$nomService = "Spooler"
Get-Service -Name $nomService
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\$nomService" | Select-Object ImagePath, Start, DependOnService
```
*Question :* la valeur `Start` de la clé de registre correspond à un code numérique (2=Automatique, 3=Manuel, 4=Désactivé). Quelle est la valeur pour le service choisi, et correspond-elle au type de démarrage affiché par `Get-Service`/`services.msc` ?

### Étape 2 — Consulter et filtrer le Journal des événements

```powershell
Get-WinEvent -LogName System -MaxEvents 10 | Format-Table TimeCreated, Id, LevelDisplayName, Message -AutoSize -Wrap

# Filtrer uniquement les erreurs des dernières 24h
$hier = (Get-Date).AddDays(-1)
Get-WinEvent -FilterHashtable @{LogName='System'; Level=2; StartTime=$hier}
```
*Question :* combien d'événements de niveau "Erreur" avez-vous trouvés dans les dernières 24h ? Comparez cette démarche à `journalctl -p err --since "24 hours ago"` déjà pratiqué sous Linux (UE3) — les deux commandes répondent-elles à la même intention ?

### Étape 3 — Provoquer et diagnostiquer un service en échec

```powershell
# Créer un service de démonstration pointant vers un exécutable inexistant (scénario volontaire d'échec)
New-Service -Name "LabServiceDefaillant" -BinaryPathName "C:\Chemin\Inexistant\app.exe" -StartupType Manual -DisplayName "Service de laboratoire défaillant"

Start-Service -Name "LabServiceDefaillant" -ErrorAction SilentlyContinue
Get-Service -Name "LabServiceDefaillant"

Get-WinEvent -FilterHashtable @{LogName='System'; Level=2} -MaxEvents 5 | Format-Table TimeCreated, Message -Wrap
```
*Résultat attendu :* le service reste à l'état "Stopped", et un événement d'erreur devrait apparaître dans le journal Système relatif à l'échec de démarrage.

Nettoyage :
```powershell
Remove-Service -Name "LabServiceDefaillant" -ErrorAction SilentlyContinue
# Si Remove-Service n'est pas disponible (versions plus anciennes) :
sc.exe delete "LabServiceDefaillant"
```

### Étape 4 — Script d'audit (reprise de la section 16 du cours)

```powershell
$servicesProblematiques = Get-Service | Where-Object {
    $_.StartType -eq "Automatic" -and $_.Status -eq "Stopped"
}

if ($servicesProblematiques) {
    Write-Warning "Services automatiques actuellement arrêtés :"
    $servicesProblematiques | Format-Table Name, DisplayName, Status
} else {
    Write-Output "OK : tous les services automatiques sont actifs"
}
```

## Commandes de référence
`Get-Service`, `Get-ItemProperty` (registre), `Get-WinEvent`, `New-Service`/`Remove-Service`

## Résultats attendus
- Correspondance entre la valeur `Start` du registre et le type de démarrage affiché (étape 1)
- Comparaison de la démarche de filtrage d'événements avec `journalctl` (étape 2)
- Preuve de l'échec de démarrage et de l'événement d'erreur associé (étape 3)
- Sortie du script d'audit (étape 4)

## Questions

1. Dans l'étape 1, pourquoi consulter directement le registre plutôt que de se contenter de `Get-Service` pour obtenir des informations sur un service ? Quelles informations le registre révèle-t-il que `Get-Service` seul ne montre pas directement ?
2. Dans l'étape 2, si vous deviez expliquer à un collègue habitué à Linux la structure du Journal des événements Windows en une phrase comparative avec `journald`, que diriez-vous ?
3. Dans l'étape 3, le service créé utilise un type de démarrage "Manual" plutôt que "Automatic" pour cette démonstration. Pourquoi ce choix est-il approprié pour un test en laboratoire, et quel risque y aurait-il eu à le configurer en "Automatic" sur cette VM de test ?

## Troubleshooting (panne à diagnostiquer)

**Situation fournie :** un étudiant, après avoir consulté `Get-Service -Name "LabServiceDefaillant"` à l'étape 3, ne trouve aucun événement d'erreur correspondant dans le Journal des événements Système malgré l'échec de démarrage manifeste du service, contrairement au résultat attendu.

**Démarche attendue :**
1. Symptôme : absence d'événement d'erreur visible dans le journal Système, alors que le service est manifestement resté à l'état "Stopped" après la tentative de démarrage
2. Hypothèse : l'événement d'erreur pourrait avoir été journalisé dans un journal différent de celui filtré (peut-être le journal Application plutôt que Système, selon la nature exacte de l'échec), ou le filtre temporel/de niveau utilisé pourrait être trop restrictif par rapport au moment exact et au niveau réel de l'événement généré
3. Vérification : élargir la recherche sans filtre de journal spécifique ni de niveau, sur une fenêtre temporelle plus large englobant précisément l'instant de la tentative de démarrage : `Get-WinEvent -FilterHashtable @{StartTime=(Get-Date).AddMinutes(-10)} | Where-Object {$_.Message -like "*LabServiceDefaillant*"}`
4. Isolation : confirmer dans quel journal et sous quel identifiant d'événement l'échec a réellement été consigné, une fois retrouvé par cette recherche élargie
5. Correction (de la méthode de recherche, pas d'un vrai problème système) : ajuster la commande de filtrage initiale pour cibler le bon journal et le bon niveau de sévérité une fois identifiés, pour de futures investigations similaires
6. Vérification : confirmer que la commande corrigée retrouve directement l'événement recherché
7. Documentation : noter cette expérience comme illustration concrète de la limite déjà soulignée en section 14 du cours — sans connaître à l'avance dans quel journal un service donné écrit ses événements, une recherche ciblée peut échouer à tort, d'où l'intérêt de construire progressivement, pour chaque service critique d'une infrastructure Windows, une documentation de correspondance service/journal/niveau habituel

## Validation

Le TP est validé si l'étudiant fournit :
- Les 4 résultats/comparaisons demandés
- Des réponses argumentées aux 3 questions
- Une démarche correcte pour le scénario de troubleshooting, aboutissant à retrouver l'événement recherché par une recherche élargie appropriée

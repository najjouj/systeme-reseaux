# UE3 — Module 18 : SSH et Durcissement d'Accès Distant

---

# PARTIE I — COURS

## 1. Titre du cours
SSH en profondeur : authentification par clé et durcissement de l'accès distant

## 2. Niveau
Élève ingénieur — cycle ingénieur, UE3 administration Linux (S2)

## 3. Durée
CM : 4h (module orienté sécurité opérationnelle — +3h TD, +5h TP en Parties II et III)

## 4. Prérequis
- UE3-Module 14 : utilisateurs et permissions (sudo, comptes de service)
- UE3-Module 17 : pare-feu (le port SSH déjà manipulé en pratique)
- UE2-Module 10 : TCP (SSH s'appuie sur une connexion TCP fiable)

## 5. Objectifs pédagogiques
À l'issue du cours, l'étudiant doit être capable de :
- Expliquer le principe de l'authentification par clé asymétrique et pourquoi elle est préférable à un mot de passe
- Générer, déployer et gérer des paires de clés SSH selon les bonnes pratiques
- Durcir la configuration du serveur SSH (`sshd_config`) selon un ensemble de mesures reconnues
- Utiliser un agent SSH pour éviter de ressaisir une phrase de passe à chaque connexion
- Configurer et comprendre le tunneling SSH (redirection de ports) pour un cas d'usage sécurisé
- Diagnostiquer un problème de connexion SSH (refus d'authentification, timeout, clé rejetée)

## 6. Motivation

SSH est le point d'entrée quasi universel pour l'administration distante d'un serveur Linux — et donc, logiquement, l'une des cibles les plus systématiquement attaquées sur tout serveur exposé sur Internet : des tentatives de connexion automatisées, testant des combinaisons de noms d'utilisateur et mots de passe courants, sont observées en continu sur n'importe quelle adresse IP publique exposant le port 22, souvent en quelques minutes après sa mise en ligne.

Ce module transforme SSH d'un simple outil de connexion en un dispositif d'accès rigoureusement sécurisé : authentification par clé plutôt que mot de passe, désactivation des connexions root directes, restriction fine des utilisateurs autorisés — l'ensemble des mesures qu'un ingénieur système doit appliquer systématiquement avant d'exposer un serveur en production, pas après un premier incident.

## 7. Concepts fondamentaux

### 7.1 Authentification par mot de passe vs par clé

**Authentification par mot de passe :** le client envoie un secret partagé (le mot de passe) que le serveur compare à sa version stockée (hachée). Vulnérable aux attaques par force brute (essais automatisés répétés) et à la réutilisation de mots de passe faibles ou compromis ailleurs.

**Authentification par clé asymétrique :** repose sur une paire de clés mathématiquement liées — une **clé privée** (gardée secrète, jamais transmise) et une **clé publique** (partagée sans risque, déployée sur les serveurs auxquels on souhaite se connecter). Le principe : le serveur peut vérifier qu'un client possède bien la clé privée correspondant à une clé publique déjà enregistrée, **sans que la clé privée ne transite jamais sur le réseau** — un défi cryptographique est résolu localement par le client à l'aide de sa clé privée, et le serveur valide la réponse à l'aide de la clé publique correspondante, sans jamais avoir besoin de connaître ou stocker le secret lui-même.

**Pourquoi c'est structurellement plus sûr :**
- Aucun secret transmis sur le réseau, contrairement à un mot de passe (même chiffré en transit par SSH lui-même, le principe de ne rien transmettre du tout est encore plus robuste)
- Une clé privée moderne (ex : Ed25519, ou RSA de taille suffisante) est pratiquement impossible à deviner par force brute, contrairement à de nombreux mots de passe humains
- La compromission d'un serveur ne révèle jamais la clé privée d'un client (seule la clé publique y est stockée) — contrairement à un mot de passe réutilisé sur plusieurs services, dont la fuite sur un service compromis met en péril tous les autres

### 7.2 Génération et déploiement d'une paire de clés

```bash
ssh-keygen -t ed25519 -C "commentaire-identifiant-la-cle"
# Génère ~/.ssh/id_ed25519 (privée) et ~/.ssh/id_ed25519.pub (publique)

ssh-copy-id -i ~/.ssh/id_ed25519.pub utilisateur@serveur
# Déploie la clé publique dans ~/.ssh/authorized_keys sur le serveur distant
```

**Ed25519 vs RSA :** Ed25519 (courbe elliptique moderne) offre une sécurité équivalente ou supérieure à RSA avec des clés bien plus courtes et des opérations cryptographiques plus rapides — recommandé par défaut pour toute nouvelle paire de clés, sauf contrainte de compatibilité avec un système très ancien ne le supportant pas encore.

**La phrase de passe (passphrase) de la clé privée :** protège la clé privée elle-même en cas de vol du fichier (ex : ordinateur portable dérobé) — sans phrase de passe, quiconque obtient le fichier de clé privée peut immédiatement s'en servir ; avec une phrase de passe, un attaquant devrait encore la déchiffrer, une couche de protection supplémentaire fortement recommandée, particulièrement pour une clé utilisée depuis un poste de travail personnel.

### 7.3 Permissions strictes — un prérequis absolu

SSH **refuse catégoriquement** de fonctionner avec des permissions trop permissives sur les fichiers de clés ou le répertoire `.ssh` — une mesure de sécurité intégrée au client/serveur SSH lui-même, pas une simple recommandation :

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519          # clé privée : lecture/écriture propriétaire uniquement
chmod 644 ~/.ssh/id_ed25519.pub       # clé publique : lisible par tous, sans risque
chmod 600 ~/.ssh/authorized_keys
```

**Rappel direct du module 14 :** cette exigence est une application concrète du principe du moindre privilège — une clé privée lisible par d'autres utilisateurs du système (permission `644` au lieu de `600`, par exemple) représenterait un risque de compromission local, indépendamment de toute attaque réseau.

### 7.4 Durcissement de `sshd_config`

Le fichier `/etc/ssh/sshd_config` centralise la configuration du serveur SSH. Mesures de durcissement recommandées :

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AllowUsers alice bob
Port 2222
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
```

- **`PermitRootLogin no`** : interdit toute connexion directe en tant que `root` — un administrateur doit se connecter avec son compte personnel puis élever ses privilèges via `sudo` (rappel direct du module 14 sur la traçabilité offerte par `sudo`), jamais se connecter directement en root
- **`PasswordAuthentication no`** : désactive complètement l'authentification par mot de passe, forçant l'authentification par clé pour toute connexion — élimine structurellement le risque d'attaque par force brute sur mot de passe
- **`AllowUsers`** : restreint explicitement la liste des comptes autorisés à se connecter via SSH, une approche liste blanche cohérente avec le principe déjà recommandé pour le pare-feu (module 17)
- **`Port 2222`** (changement du port par défaut) : mesure controversée et à nuancer (voir 7.5) — réduit le bruit des scans automatisés basiques, mais **n'est en aucun cas une mesure de sécurité robuste** en elle-même (sécurité par l'obscurité, insuffisante seule)
- **`MaxAuthTries`** : limite le nombre de tentatives d'authentification par connexion, ralentissant les tentatives automatisées
- **`ClientAliveInterval`/`ClientAliveCountMax`** : ferment automatiquement une session inactive après un délai défini, réduisant le risque d'une session oubliée ouverte et exploitable

**Après toute modification de `sshd_config` :** toujours valider la syntaxe avant de redémarrer le service (`sudo sshd -t`), et **ne jamais fermer sa session SSH active avant d'avoir confirmé qu'une nouvelle connexion fonctionne correctement** avec la configuration modifiée — rappel direct du principe de prudence déjà établi au module 17 pour les modifications de pare-feu à distance, applicable ici avec la même rigueur.

### 7.5 Sécurité par l'obscurité — nuance pédagogique importante

Changer le port par défaut de SSH (7.4) est une mesure fréquemment recommandée en pratique, mais elle doit être comprise pour ce qu'elle est réellement : elle réduit le **volume** de tentatives automatisées non ciblées (la majorité des scans de masse ciblent le port 22 par défaut), mais n'apporte **aucune protection réelle** contre un attaquant qui scannerait spécifiquement la machine (un scan de ports complet révèle immédiatement le nouveau port utilisé) ni contre une attaque ciblée. C'est une mesure de **réduction de bruit opérationnel** (moins de tentatives à filtrer dans les logs), pas une mesure de sécurité fondamentale — les véritables protections structurelles restent l'authentification par clé et la désactivation des mots de passe.

### 7.6 Agent SSH

Un **agent SSH** (`ssh-agent`) conserve en mémoire une clé privée déverrouillée (après une seule saisie de la phrase de passe), permettant des connexions SSH successives sans avoir à ressaisir cette phrase à chaque fois :

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519   # demande la phrase de passe une seule fois
ssh-add -l                     # lister les clés actuellement chargées dans l'agent
```

**Agent forwarding — usage et risque :** l'option `-A` de `ssh` (ou `ForwardAgent yes`) permet de faire suivre l'agent SSH local vers une machine distante, pratique pour "rebondir" via cette machine vers une troisième sans y déployer de clé privée. **Risque de sécurité important :** un administrateur root sur la machine intermédiaire (ou un attaquant l'ayant compromise) peut potentiellement utiliser l'agent transféré pour s'authentifier ailleurs à l'insu de son propriétaire, tant que la session reste active — une fonctionnalité à activer avec discernement, jamais par défaut sur une machine dont la confiance n'est pas totale.

### 7.7 Tunneling SSH (redirection de ports)

SSH permet de rediriger un port local ou distant à travers une connexion chiffrée, une fonctionnalité puissante pour accéder de façon sécurisée à un service normalement non exposé directement :

```bash
# Redirection locale : accéder à un service distant non exposé publiquement,
# via un tunnel chiffré, comme s'il était local
ssh -L 8080:localhost:80 utilisateur@serveur
# Le trafic vers localhost:8080 (côté client) est transporté de façon chiffrée
# jusqu'au serveur, puis relayé vers son propre localhost:80
```

Utile par exemple pour accéder de façon sécurisée à une interface d'administration web qui n'écoute volontairement que sur l'interface locale du serveur (`127.0.0.1`), sans jamais l'exposer directement sur le réseau — le tunnel SSH devient le seul chemin d'accès possible, combinant chiffrement et authentification forte.

## 8. Architecture

```
┌───────────────────────┐                    ┌───────────────────────┐
│  Client                  │                    │  Serveur                  │
│  ~/.ssh/id_ed25519         │  connexion chiffrée │  ~/.ssh/authorized_keys     │
│  (clé PRIVÉE, jamais       │ ◀────────────────▶ │  (clé PUBLIQUE de chaque    │
│   transmise)                │  défi cryptographique│   utilisateur autorisé)     │
└───────────────────────┘                    └───────────────────────┘
```

## 9. Fonctionnement interne

SSH établit d'abord une connexion TCP classique (rappel UE2-M10, three-way handshake), puis négocie une couche de chiffrement (échange de clés, algorithmes symétriques pour le chiffrement du flux, similaire en esprit à TLS bien que techniquement distinct), avant d'entamer la phase d'authentification proprement dite (par clé, ou par mot de passe si activé). Une fois authentifié, l'intégralité du trafic de la session (commandes, sorties, transferts de fichiers via `scp`/`sftp`) transite à travers ce canal chiffré unique — un tunnel SSH (7.7) n'est en réalité qu'un usage supplémentaire de ce même canal déjà établi et chiffré, multiplexé pour transporter un trafic applicatif additionnel en plus de la session interactive elle-même.

## 10. Démonstration

```bash
# Générer une paire de clés de démonstration
ssh-keygen -t ed25519 -f /tmp/demo_key -N ""

# Observer les permissions strictes exigées
ls -l /tmp/demo_key*

# Valider la syntaxe d'une configuration sshd avant redémarrage
sudo sshd -t

# Observer les tentatives de connexion échouées dans les logs
sudo journalctl -u sshd --since "10 minutes ago" | grep -i fail
```

## 11. Commandes / outils

| Commande | Rôle | Exemple |
|---|---|---|
| `ssh-keygen` | Générer une paire de clés | `ssh-keygen -t ed25519` |
| `ssh-copy-id` | Déployer une clé publique sur un serveur distant | `ssh-copy-id user@serveur` |
| `sshd -t` | Valider la syntaxe de `sshd_config` sans redémarrer | `sudo sshd -t` |
| `ssh-agent` / `ssh-add` | Gérer un agent de clés en mémoire | `ssh-add ~/.ssh/id_ed25519` |
| `ssh -L` / `-R` | Tunneling local/distant | `ssh -L 8080:localhost:80 user@serveur` |
| `journalctl -u sshd` | Consulter les logs du service SSH (rappel module 16) | `journalctl -u sshd -f` |

**Erreur fréquente :** redémarrer directement le service `sshd` après une modification de `sshd_config` sans avoir validé la syntaxe au préalable (`sshd -t`) — une erreur de syntaxe peut empêcher le service de redémarrer correctement, coupant potentiellement tout accès SSH futur au serveur si la session en cours venait également à se fermer avant correction.

## 12–13. TD et TP
Voir Parties II et III.

## 14. Étude de cas industrielle

**Contexte :** un audit de sécurité révèle que plusieurs serveurs d'une entreprise ont subi des tentatives de connexion SSH par force brute pendant plusieurs semaines, visibles dans les journaux (`journalctl -u sshd`), sans qu'aucune tentative n'ait réussi (authentification par mot de passe déjà désactivée sur ces serveurs). Cependant, le volume de ces tentatives dégrade légèrement les performances et génère un bruit considérable dans les journaux, compliquant la détection d'événements réellement significatifs.

**Analyse attendue :** bien que l'authentification par clé empêche déjà toute compromission via ces tentatives (mesure structurelle déjà en place et efficace), l'entreprise gagnerait à déployer un outil de **bannissement automatique** (comme `fail2ban`, qui analyse les journaux d'authentification et bloque temporairement, via le pare-feu, les adresses IP à l'origine de tentatives répétées) — une mesure complémentaire de réduction de bruit et de charge, distincte de la protection structurelle déjà assurée par la désactivation des mots de passe. Ce cas illustre une distinction importante : toutes les mesures de sécurité n'ont pas le même rôle — certaines empêchent structurellement une compromission (clé plutôt que mot de passe), d'autres réduisent le bruit opérationnel et la charge associée à des tentatives de toute façon vouées à l'échec (bannissement automatique, changement de port évoqué en 7.5) — les deux catégories sont utiles, mais ne doivent jamais être confondues en termes de garantie de sécurité réelle apportée.

## 15. Sécurité

- L'authentification par clé, combinée à `PasswordAuthentication no`, élimine structurellement le risque d'attaque par force brute sur mot de passe — la mesure la plus importante de ce module
- `PermitRootLogin no` force l'usage de `sudo` pour toute action privilégiée, avec la traçabilité individuelle déjà soulignée au module 14
- Un outil comme `fail2ban` (7.14) complète ces mesures structurelles en réduisant le bruit opérationnel des tentatives automatisées, sans s'y substituer
- La rotation et la révocation de clés doivent être anticipées : lorsqu'un collaborateur quitte une équipe, sa clé publique doit être retirée de tous les `authorized_keys` concernés — un processus organisationnel à documenter, pas seulement une opération technique ponctuelle
- Le tunneling SSH (7.7), bien qu'utile, peut aussi être détourné pour exfiltrer des données ou contourner un pare-feu périmétrique par un utilisateur malveillant disposant d'un accès SSH légitime — une considération à garder à l'esprit dans une politique de sécurité globale d'entreprise

## 16. Automatisation

Script Bash d'audit des accès SSH configurés, vérifiant la cohérence entre `AllowUsers` et les clés effectivement déployées, et signalant toute clé orpheline (déployée mais pour un utilisateur non listé, ou l'inverse).

```bash
#!/usr/bin/env bash
# Audite la cohérence entre AllowUsers (sshd_config) et les authorized_keys déployées
set -euo pipefail

CONFIG="/etc/ssh/sshd_config"
LOGFILE="/var/log/ssh_access_audit.log"

UTILISATEURS_AUTORISES=$(grep -oP '^AllowUsers\s+\K.*' "$CONFIG" 2>/dev/null || echo "")

if [[ -z "$UTILISATEURS_AUTORISES" ]]; then
    echo "$(date '+%F %T') - ALERTE: aucune directive AllowUsers définie, tout utilisateur système peut potentiellement se connecter" \
        | tee -a "$LOGFILE"
fi

for user_home in /home/*; do
    utilisateur=$(basename "$user_home")
    fichier_cles="$user_home/.ssh/authorized_keys"
    if [[ -f "$fichier_cles" ]]; then
        if [[ -n "$UTILISATEURS_AUTORISES" ]] && ! grep -qw "$utilisateur" <<< "$UTILISATEURS_AUTORISES"; then
            echo "$(date '+%F %T') - ALERTE: $utilisateur a des clés déployées mais n'est pas dans AllowUsers" \
                | tee -a "$LOGFILE"
        fi
    fi
done

echo "$(date '+%F %T') - Audit terminé" >> "$LOGFILE"
```

## 17. Troubleshooting

Scénario court : un utilisateur signale qu'il ne parvient plus à se connecter en SSH après avoir régénéré sa paire de clés sur un nouveau poste de travail, avec le message "Permission denied (publickey)". Démarche : vérifier que la **nouvelle** clé publique a bien été déployée dans `authorized_keys` du serveur (l'ancienne clé n'y donne plus accès si elle a été remplacée localement, pas mise à jour côté serveur — piège fréquent après un changement de poste de travail) → vérifier les permissions du répertoire `.ssh` et du fichier `authorized_keys` côté serveur (rappel section 7.3 — SSH refuse silencieusement l'authentification si les permissions sont trop permissives, sans toujours un message d'erreur explicite côté client) → utiliser `ssh -v` (verbeux) pour obtenir des informations de diagnostic détaillées côté client sur la raison précise de l'échec.

## 18. Synthèse

- L'authentification par clé asymétrique est structurellement supérieure à un mot de passe : aucun secret ne transite jamais sur le réseau
- SSH exige des permissions strictes sur les fichiers de clés (`600` pour la clé privée) — une mesure de sécurité intégrée, pas une simple recommandation
- Le durcissement de `sshd_config` (`PermitRootLogin no`, `PasswordAuthentication no`, `AllowUsers`) élimine structurellement plusieurs classes d'attaques courantes
- Changer le port SSH par défaut réduit le bruit opérationnel mais n'est pas une mesure de sécurité fondamentale (sécurité par l'obscurité)
- Toujours valider la syntaxe de `sshd_config` (`sshd -t`) et tester une nouvelle connexion avant de fermer la session active — même principe de prudence que pour le pare-feu (module 17)
- Un agent SSH évite de ressaisir une phrase de passe répétée, mais l'agent forwarding comporte un risque de sécurité à activer avec discernement
- Le tunneling SSH permet un accès sécurisé à des services non exposés directement, réutilisant le canal chiffré déjà établi de la session

## 19. Questions d'évaluation

**QCM**
1. Dans l'authentification par clé SSH, ce qui ne transite jamais sur le réseau est :
 a) la clé publique b) la clé privée c) le nom d'utilisateur d) l'adresse IP du serveur

2. `PermitRootLogin no` force :
 a) la désactivation totale de SSH b) l'usage d'un compte personnel puis de sudo pour toute action privilégiée c) l'authentification par mot de passe uniquement d) la fermeture automatique de toute session

**Questions ouvertes**
- Expliquez pourquoi le changement du port SSH par défaut n'est pas considéré comme une mesure de sécurité fondamentale, malgré son utilité opérationnelle.
- Expliquez le risque associé à l'agent forwarding SSH (`-A`), et dans quel contexte il devrait être évité.

**Exercice technique**
- Donnez la commande permettant de valider la syntaxe d'un fichier `sshd_config` modifié, sans redémarrer le service.

**Problème de diagnostic**
- Un utilisateur reçoit "Permission denied (publickey)" après avoir changé de poste de travail. Proposez la cause la plus probable et la vérification à effectuer en premier.

## 20. Mini-projet

Développer un script (Bash ou Python) qui :
1. Audite l'ensemble des configurations d'accès SSH d'un serveur (reprise et extension du script de la section 16) : cohérence `AllowUsers`/clés déployées, vérification des permissions strictes sur chaque `authorized_keys` du système
2. Détecte si `PasswordAuthentication` ou `PermitRootLogin` sont activés dans `sshd_config`, signalant ces écarts par rapport aux bonnes pratiques comme des anomalies de priorité critique
3. Analyse les journaux SSH récents pour quantifier le volume de tentatives d'authentification échouées, et alerte en cas de pic anormal (signature possible d'une attaque par force brute active)
4. Journalise chaque détection avec horodatage et niveau de gravité
5. Génère un rapport de synthèse destiné à un audit de sécurité périodique, distinguant clairement les mesures structurelles manquantes (critique) des mesures de réduction de bruit absentes (recommandé)

---

# PARTIE II — TD

### Niveau 1 — Compréhension

**Exercice 1.1**
Expliquez le principe de l'authentification par clé asymétrique SSH, en précisant ce qui reste toujours secret et ce qui peut être partagé sans risque.

**Exercice 1.2**
Citez trois directives de durcissement de `sshd_config` vues en cours et l'effet de chacune.

**Exercice 1.3**
Expliquez pourquoi le changement du port SSH par défaut n'est pas, à lui seul, une mesure de sécurité suffisante.

### Niveau 2 — Application

**Exercice 2.1**
Donnez les deux commandes permettant respectivement de générer une nouvelle paire de clés Ed25519, puis de déployer la clé publique correspondante sur un serveur distant.

**Exercice 2.2**
Donnez les permissions Unix correctes (notation octale) attendues pour le répertoire `~/.ssh` et pour le fichier de clé privée.

**Exercice 2.3**
Donnez la commande permettant de valider la syntaxe de `/etc/ssh/sshd_config` sans redémarrer le service SSH.

### Niveau 3 — Analyse

**Exercice 3.1**
Un administrateur modifie `sshd_config` pour ajouter `PasswordAuthentication no`, exécute `sudo systemctl restart sshd`, puis ferme immédiatement sa session SSH active pour tester une nouvelle connexion. La nouvelle connexion échoue car sa clé publique n'avait en réalité jamais été correctement déployée sur ce serveur. Quelle est la conséquence de cette succession d'actions, et quelle pratique de prudence (déjà rencontrée dans un module précédent pour un scénario analogue) aurait évité cette situation ?

**Exercice 3.2**
Un utilisateur active l'agent forwarding (`ssh -A`) pour se connecter à un serveur intermédiaire peu fiable, dans le but de rebondir ensuite vers un serveur final sensible. Expliquez le risque de sécurité concret que cela introduit, même si l'utilisateur ne se connecte jamais explicitement au serveur final depuis le serveur intermédiaire lui-même pendant que quelqu'un d'autre y aurait un accès.

### Niveau 4 — Problème complexe

**Exercice 4.1**
Une entreprise constate, via l'analyse de ses journaux SSH, un volume élevé et constant de tentatives de connexion échouées sur l'ensemble de ses serveurs exposés, mais aucune compromission réelle n'est détectée (authentification par clé strictement appliquée, mots de passe désactivés). Un ingénieur junior propose de désactiver complètement les journaux d'échecs d'authentification SSH pour "nettoyer" les logs et réduire le volume de stockage utilisé par la journalisation. Expliquez en quoi cette proposition serait une erreur, en identifiant ce qui serait perdu, et proposez une alternative permettant à la fois de réduire le bruit opérationnel réel (charge, taille des logs) et de conserver une capacité de détection en cas d'évolution du profil des tentatives (par exemple, une attaque qui deviendrait soudainement plus ciblée ou plus sophistiquée).

---

## Correction du TD

**1.1** — L'authentification par clé repose sur une paire de clés liées mathématiquement : une clé privée, gardée strictement secrète et jamais transmise sur le réseau, et une clé publique, qui peut être partagée et déployée sans risque sur tout serveur auquel on souhaite se connecter. Le serveur vérifie que le client possède bien la clé privée correspondante via un défi cryptographique résolu localement par le client, sans que la clé privée elle-même ne transite jamais sur le réseau à aucun moment de l'échange.

**1.2** — `PermitRootLogin no` : interdit toute connexion directe en tant que root, forçant l'usage d'un compte personnel puis de `sudo`. `PasswordAuthentication no` : désactive l'authentification par mot de passe, forçant l'authentification par clé et éliminant le risque de force brute sur mot de passe. `AllowUsers` : restreint explicitement la liste des comptes autorisés à se connecter via SSH, une approche liste blanche.

**1.3** — Changer le port SSH par défaut réduit uniquement le volume de tentatives automatisées non ciblées, qui scannent en masse le port 22 par défaut sur de larges plages d'adresses IP. Cette mesure n'apporte aucune protection réelle contre un attaquant qui effectuerait un scan de ports complet sur la machine ciblée spécifiquement (révélant immédiatement le nouveau port utilisé) ni contre une attaque véritablement ciblée — c'est une mesure de réduction de bruit opérationnel, pas une mesure de sécurité structurelle, un exemple de "sécurité par l'obscurité" insuffisante employée seule.

**2.1** — `ssh-keygen -t ed25519` puis `ssh-copy-id utilisateur@serveur` (ou `ssh-copy-id -i chemin_vers_la_cle.pub utilisateur@serveur` si la clé n'est pas celle par défaut).

**2.2** — `~/.ssh` : `700` (lecture/écriture/exécution pour le propriétaire uniquement). Clé privée : `600` (lecture/écriture pour le propriétaire uniquement, aucun droit pour le groupe ou les autres).

**2.3** — `sudo sshd -t`.

**3.1** — Conséquence : l'administrateur s'est retrouvé dans l'incapacité de se reconnecter au serveur via SSH, puisque `PasswordAuthentication no` est désormais actif et qu'aucune clé publique valide n'avait été correctement déployée au préalable — un scénario de coupure d'accès distant potentiellement sans solution simple (nécessitant un accès console physique/hors-bande selon l'infrastructure). La pratique de prudence qui aurait évité cette situation, déjà rencontrée dans un contexte analogue au module 17 (application d'une politique de pare-feu restrictive), consiste à **toujours tester une nouvelle connexion dans un second terminal, en parallèle, avant de fermer la session active** qui a permis d'appliquer la modification — si la nouvelle connexion avait été testée avant la fermeture de la session existante, l'échec aurait été immédiatement détecté et corrigeable (en revenant temporairement sur `PasswordAuthentication yes` depuis la session encore active) sans jamais perdre l'accès au serveur.

**3.2** — Le risque concret est que, pendant toute la durée où l'agent forwarding reste actif sur le serveur intermédiaire, **tout utilisateur disposant de privilèges suffisants sur ce serveur intermédiaire** (un administrateur système légitime de cette machine, ou un attaquant l'ayant compromise) peut potentiellement utiliser l'agent SSH transféré pour s'authentifier vers **n'importe quelle destination** pour laquelle la clé privée originale de l'utilisateur donne un accès légitime — y compris le serveur final sensible mentionné dans l'énoncé, sans que l'utilisateur d'origine n'ait lui-même explicitement initié cette connexion depuis le serveur intermédiaire. Le risque existe donc même en l'absence de toute action explicite de l'utilisateur vers le serveur final pendant que l'agent est transféré : c'est la simple présence de l'agent transféré, exploitable par un tiers ayant un accès suffisant à la machine intermédiaire, qui constitue le danger — d'où la recommandation de n'activer l'agent forwarding que vers des machines dont la confiance est totale et équivalente à celle accordée au poste d'origine.

**4.1** — La proposition serait une erreur car elle supprimerait la **capacité de détection** offerte par ces journaux, sans pour autant traiter la cause réelle du volume de tentatives (qui continueraient de se produire, seulement sans plus jamais être enregistrées). Ce qui serait perdu : la possibilité de détecter un **changement de profil** dans ces tentatives — par exemple, si les tentatives automatisées généralistes actuelles étaient un jour remplacées ou complétées par une tentative plus ciblée, plus sophistiquée, ou provenant d'une source inhabituelle (potentiellement le signe précurseur d'une attaque réellement dangereuse), l'absence totale de journalisation empêcherait toute détection de ce changement, qu'il s'agisse d'une analyse en temps réel ou d'une investigation a posteriori après un éventuel incident. Alternative recommandée : plutôt que de désactiver la journalisation, déployer un outil de bannissement automatique comme `fail2ban` (rappel étude de cas industrielle du cours), qui réduit concrètement le volume de tentatives répétées en bloquant temporairement les adresses IP responsables au niveau du pare-feu — réduisant ainsi à la fois la charge sur le service SSH et le volume de nouvelles entrées de journal générées par ces tentatives bloquées en amont, tout en conservant une journalisation complète et fonctionnelle pour toute analyse de sécurité future, y compris la détection d'un changement de profil d'attaque.

---

# PARTIE III — TP

## Contexte

Vous intervenez comme ingénieur système junior pour durcir l'accès SSH d'un serveur de laboratoire, en appliquant rigoureusement les précautions de prudence vues en cours, puis pour diagnostiquer un cas classique de clé mal déployée.

## Architecture

Une VM Linux avec un service SSH actif, accessible via au moins deux sessions/terminaux simultanés possibles depuis le poste de l'étudiant (indispensable pour appliquer la méthodologie de prudence du cours).

**Avertissement de sécurité pour ce TP :** comme au module 17, travaillez sur une VM de laboratoire jetable, jamais sur un accès de production que vous ne pouvez pas vous permettre de perdre.

## Objectifs
- Générer et déployer une paire de clés SSH selon les bonnes pratiques
- Durcir `sshd_config` en respectant la méthodologie de double-session de prudence
- Configurer un tunnel SSH local et vérifier son fonctionnement
- Diagnostiquer un cas de clé mal déployée

## Préparation

```bash
which ssh ssh-keygen ssh-copy-id sshd 2>/dev/null
```

## Étapes

### Étape 1 — Génération et déploiement de clé

```bash
ssh-keygen -t ed25519 -f ~/.ssh/lab_key -N ""   # sans phrase de passe pour simplifier le TP
ls -l ~/.ssh/lab_key ~/.ssh/lab_key.pub

# Déployer sur la VM cible (adapter utilisateur/adresse selon votre environnement)
ssh-copy-id -i ~/.ssh/lab_key.pub labuser@vm_cible

# Tester la connexion par clé
ssh -i ~/.ssh/lab_key labuser@vm_cible "echo Connexion par clé réussie"
```

### Étape 2 — Durcissement méthodique avec double session

**Terminal A (session de travail, à garder ouverte) :**
```bash
ssh labuser@vm_cible
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup
sudo tee -a /etc/ssh/sshd_config << 'EOF'
PermitRootLogin no
PasswordAuthentication no
AllowUsers labuser
EOF

sudo sshd -t   # valider la syntaxe AVANT tout redémarrage
sudo systemctl restart sshd
```

**Terminal B (nouvelle session de test, SANS fermer le Terminal A) :**
```bash
ssh -i ~/.ssh/lab_key labuser@vm_cible "echo Nouvelle connexion réussie avec la config durcie"
```
*Question :* si la commande du Terminal B avait échoué, quelle action auriez-vous entreprise **avant** de fermer le Terminal A ? Pourquoi cette méthodologie à deux terminaux est-elle la bonne pratique, cohérente avec l'exercice 3.1 du TD ?

*Seulement si le Terminal B confirme le succès :* vous pouvez fermer le Terminal A en toute sécurité.

### Étape 3 — Tunnel SSH local

```bash
# Sur la VM cible, lancer un service web minimal écoutant uniquement en local
ssh -i ~/.ssh/lab_key labuser@vm_cible "python3 -m http.server 8000 --bind 127.0.0.1 &"
sleep 1

# Depuis votre poste, créer le tunnel
ssh -i ~/.ssh/lab_key -L 8888:localhost:8000 labuser@vm_cible -N &
TUNNEL_PID=$!
sleep 1

curl -s http://localhost:8888/ | head -5

kill $TUNNEL_PID
```
*Question :* le service Python écoute exclusivement sur `127.0.0.1` de la VM cible (non exposé sur le réseau). Comment le tunnel SSH permet-il malgré tout d'y accéder depuis votre poste local, sans jamais exposer ce service directement sur le réseau de la VM ?

### Étape 4 — Diagnostiquer une clé mal déployée (scénario volontaire)

```bash
# Générer une seconde paire de clés SANS la déployer
ssh-keygen -t ed25519 -f ~/.ssh/lab_key_orpheline -N ""

# Tenter une connexion avec cette clé non déployée
ssh -i ~/.ssh/lab_key_orpheline -v labuser@vm_cible 2>&1 | tail -20
```
*Résultat attendu :* un échec "Permission denied (publickey)", avec des détails de diagnostic dans la sortie verbeuse (`-v`).

## Commandes de référence
`ssh-keygen`, `ssh-copy-id`, `sshd -t`, `ssh -L`, `ssh -v`

## Résultats attendus
- Preuve de connexion réussie par clé (étape 1)
- Confirmation de la méthodologie à double session appliquée avec succès (étape 2)
- Réponse HTTP obtenue via le tunnel (étape 3)
- Sortie verbeuse de l'échec de connexion avec la clé orpheline (étape 4)

## Questions

1. Dans l'étape 2, pourquoi est-il essentiel d'exécuter `sudo sshd -t` avant `systemctl restart sshd`, plutôt que de se fier uniquement au test de connexion du Terminal B pour détecter une éventuelle erreur ?
2. Si, à l'étape 3, vous aviez utilisé `-R` (redirection distante) plutôt que `-L` (redirection locale), le scénario d'usage aurait-il été le même ? Formulez une hypothèse sur la différence de cas d'usage entre les deux, même si elle n'a pas été détaillée en cours (indice : réfléchissez au sens du mot "distante" par rapport à votre position).
3. Dans l'étape 4, la sortie verbeuse (`-v`) confirme-t-elle avec certitude que la clé n'a jamais été déployée, ou existe-t-il une autre cause possible (rappel section 17 du cours) qui produirait un symptôme similaire ?

## Troubleshooting (panne à diagnostiquer)

**Situation fournie :** après avoir suivi l'étape 2 avec succès, un étudiant referme malencontreusement son Terminal B (celui qui avait confirmé la connexion réussie), puis, plusieurs minutes plus tard, tente une nouvelle connexion depuis un tout nouveau terminal, qui échoue avec "Permission denied (publickey)" — alors que la connexion avait fonctionné quelques minutes auparavant dans le Terminal B.

**Démarche attendue :**
1. Symptôme : une connexion qui fonctionnait précédemment échoue désormais depuis un nouveau terminal
2. Hypothèse : le nouveau terminal n'utilise probablement pas la même clé privée explicitement spécifiée dans les commandes précédentes (`-i ~/.ssh/lab_key`) — un simple `ssh labuser@vm_cible` sans l'option `-i` explicite pourrait tenter d'utiliser une clé par défaut différente (ex : `~/.ssh/id_ed25519` généré ailleurs, ou aucune clé du tout si aucune clé par défaut n'existe sur ce poste), plutôt que la clé `lab_key` spécifiquement générée et déployée au cours de ce TP
3. Vérification : comparer la commande exacte utilisée dans le nouveau terminal avec celle du Terminal B original — vérifier la présence ou l'absence de l'option `-i ~/.ssh/lab_key`
4. Isolation : confirmer, avec `ssh -v` sur le nouveau terminal, quelle(s) clé(s) le client tente réellement d'utiliser pour l'authentification
5. Correction : relancer la connexion en spécifiant explicitement `-i ~/.ssh/lab_key`, ou configurer un fichier `~/.ssh/config` associant automatiquement cette clé à cet hôte pour éviter d'avoir à le spécifier manuellement à chaque connexion future
6. Vérification : confirmer que la connexion réussit avec la clé correctement spécifiée
7. Documentation : noter ce piège comme rappel qu'une authentification par clé réussie dans un contexte précis (terminal, commande exacte) ne garantit pas automatiquement le même succès dans un contexte légèrement différent si la clé utilisée n'est pas explicitement la même — un fichier `~/.ssh/config` bien structuré est la solution professionnelle pérenne à ce type de confusion récurrente

## Validation

Le TP est validé si l'étudiant fournit :
- Les 4 résultats/captures demandés
- Des réponses argumentées aux 3 questions
- Une identification correcte de la cause du scénario de troubleshooting (clé non spécifiée explicitement dans le nouveau terminal), avec une correction cohérente incluant la piste `~/.ssh/config`

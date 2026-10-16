# UE3 — Projet Intégrateur de Clôture
## Déployer et sécuriser une infrastructure Linux

*Correspond au **Projet 1** du programme général : "Déployer et sécuriser une infrastructure Linux". Mobilise l'intégralité des modules 14 à 22 de l'UE3, ainsi que les fondations de l'UE1 et de l'UE2.*

---

## 1. Titre du projet
De l'installation à la résilience : déployer, sécuriser et superviser deux serveurs Linux de production

## 2. Niveau
Élève ingénieur — cycle ingénieur, clôture de l'UE3 administration Linux (S2)

## 3. Durée
2 à 3 semaines de travail en autonomie (binôme recommandé), avec 3 points de passage encadrés et une soutenance finale de 20 minutes par binôme.

## 4. Prérequis
L'intégralité de l'UE3 validée (modules 14 à 22), ainsi que l'UE1 (systèmes) et l'UE2 (réseaux).

## 5. Contexte du projet

**Continuité narrative avec le projet de clôture de l'UE2 :** l'entreprise fictive **Atelier Nomia**, dont l'infrastructure réseau (VLAN, DHCP, DNS, routage) a été conçue et déployée à la clôture de l'UE2, doit désormais y héberger ses premiers serveurs Linux réels. Deux serveurs sont à déployer sur le segment infrastructure déjà défini :

- **Serveur applicatif (`app-01`)** : héberge une application métier maison (fournie sous forme d'un script simple simulant un service réel, cf. section 9) et doit être accessible depuis le réseau interne uniquement
- **Serveur de sauvegarde (`backup-01`)** : reçoit les sauvegardes du serveur applicatif et de sa propre configuration, selon la règle 3-2-1 étudiée en cours (dans la limite de ce qu'un environnement de laboratoire permet de simuler pour la composante "hors site")

*Si le projet de clôture UE2 n'a pas été réalisé ou est indisponible, une infrastructure réseau minimale (un seul réseau plat avec passerelle et DNS basique) peut être substituée sans remettre en cause les objectifs de ce projet, à documenter comme hypothèse de départ.*

Vous intervenez comme équipe d'ingénieurs système pour déployer ces deux serveurs de bout en bout, les sécuriser selon les meilleures pratiques de l'UE3, mettre en place leur supervision et leurs sauvegardes, puis diagnostiquer un incident volontairement introduit en appliquant la méthodologie du module 22.

## 6. Objectifs pédagogiques

Ce projet doit démontrer la maîtrise intégrée de :
- L'installation et la configuration d'un système de stockage réfléchi (partitionnement, LVM) dès le départ (module 15)
- La mise en place d'une structure d'utilisateurs, de groupes et de permissions conforme au moindre privilège (module 14)
- Le déploiement d'une application maison comme service systemd robuste, avec garde-fous anti-boucle de crash (module 16)
- La sécurisation réseau de chaque serveur (pare-feu à politique liste blanche, déployé de façon prudente) (module 17)
- Le durcissement de l'accès SSH (authentification par clé, désactivation du mot de passe et de root) (module 18)
- La configuration d'une journalisation durable et de tâches planifiées fiables, avec heartbeat (module 19)
- Une stratégie de gestion des paquets et du noyau documentée (module 20)
- La mise en place d'une supervision de performance et d'une stratégie de sauvegarde **testée en restauration** (module 21)
- L'application rigoureuse de la méthodologie de troubleshooting en 9 étapes face à un incident réel (module 22)

## 7. Cahier des charges

### 7.1 Contraintes fonctionnelles — serveur applicatif (`app-01`)

- Stockage organisé via LVM, avec au minimum un volume logique dédié aux données applicatives, distinct de la racine, redimensionnable à chaud
- Utilisateurs : un compte administrateur nominatif par membre de l'équipe (accès `sudo` restreint aux commandes nécessaires à l'exploitation du service, pas un accès root complet) ; un compte de service dédié pour l'application, sans shell interactif
- L'application métier déployée comme unité systemd avec : redémarrage automatique en cas d'échec, garde-fous anti-boucle de crash (`StartLimitBurst`), utilisateur dédié, au moins une directive de durcissement (`NoNewPrivileges` ou équivalent)
- Pare-feu local avec politique par défaut `DROP` sur `INPUT`, autorisant uniquement SSH et le port applicatif nécessaire, déployé selon la méthodologie prudente du module 17 (ordre des règles, ou mécanisme de rollback)
- SSH durci : authentification par clé uniquement, `PermitRootLogin no`, `AllowUsers` restreint aux comptes légitimes
- Journalisation persistante (`journald`), configuration `logrotate` pour les logs applicatifs, tâche planifiée de nettoyage périodique avec mécanisme de heartbeat
- Politique de mise à jour documentée : quels paquets sont figés et pourquoi, stratégie vis-à-vis des mises à jour de sécurité du noyau

### 7.2 Contraintes fonctionnelles — serveur de sauvegarde (`backup-01`)

- Reçoit, via `rsync` sur SSH, une sauvegarde régulière planifiée des données applicatives de `app-01` et de sa configuration système critique (`/etc`)
- Stratégie de rotation définie et justifiée (complète/incrémentale/différentielle, cf. module 21) avec une politique de rétention explicite
- Accès SSH dédié à cette synchronisation, avec une clé spécifique à cet usage (pas de réutilisation d'une clé personnelle d'administrateur), idéalement restreinte en capacité si l'équipe souhaite approfondir (`command=` forcé dans `authorized_keys`, extension non obligatoire mais valorisée)
- Mêmes exigences de durcissement SSH et de pare-feu que `app-01`

### 7.3 Contraintes de validation

Comme pour le projet de clôture de l'UE2, chaque brique doit être **prouvée fonctionnelle**, pas seulement affirmée :
- Preuve qu'un volume logique peut être étendu à chaud sur `app-01` sans interruption du service (capture avant/après, `lvextend` + `resize2fs`/`xfs_growfs`)
- Preuve que le service applicatif redémarre automatiquement après un plantage provoqué volontairement, et se bloque proprement en `failed` après épuisement de `StartLimitBurst` si le plantage persiste
- Preuve de fonctionnement du pare-feu (connexion autorisée réussie, connexion interdite explicitement bloquée) sur les deux serveurs
- Preuve de connexion SSH par clé fonctionnelle et de refus d'une tentative par mot de passe
- **Preuve d'une restauration complète réellement testée** depuis `backup-01` vers un environnement isolé (jamais directement sur `app-01` en fonctionnement), avec mesure de la durée (RTO)

### 7.4 Contrainte d'incident volontaire (module 22)

Une fois l'infrastructure fonctionnelle et documentée, l'équipe doit **introduire volontairement un incident multi-causal**, non révélé dans le rapport avant la section de diagnostic, choisi parmi (ou combinant, pour un défi supplémentaire) :
- Une panne de service provoquée par un répertoire de logs manquant ou des permissions incorrectes (rappel direct M14/M16)
- Une saturation disque provoquée par un fichier supprimé mais toujours ouvert (rappel UE1-M4, repris explicitement au module 22)
- Un accès SSH cassé par une règle de pare-feu trop restrictive appliquée sans précaution (rappel M17/M18)
- Une tâche de sauvegarde silencieusement défaillante sans mécanisme de heartbeat pour la détecter (rappel M19/M21)

L'équipe doit produire un **rapport de diagnostic suivant explicitement les 9 étapes de la méthodologie du module 22**, comme si elle recevait ce ticket d'incident d'un tiers, en soutenance.

## 8. Architecture cible (schéma de référence)

```
                    Réseau infrastructure (VLAN 99, hérité du projet UE2)
                                    │
              ┌─────────────────────┴─────────────────────┐
              │                                               │
        ┌───────────┐                                   ┌───────────┐
        │  app-01     │◀──── rsync sur SSH (sauvegarde) ────│ backup-01  │
        │             │                                   │             │
        │ LVM:         │                                   │ LVM:         │
        │  lv_root      │                                   │  lv_root      │
        │  lv_data       │                                   │  lv_backups    │
        │             │                                   │             │
        │ Service:      │                                   │ Pas de service │
        │  app.service   │                                   │  applicatif,   │
        │  (systemd)     │                                   │  uniquement    │
        │             │                                   │  réception     │
        │ Pare-feu:      │                                   │             │
        │  DROP défaut,   │                                   │ Pare-feu:      │
        │  SSH + port app  │                                   │  DROP défaut,   │
        │             │                                   │  SSH uniquement  │
        │ SSH: clé only,   │                                   │             │
        │  root interdit    │                                   │ SSH: clé dédiée  │
        └───────────┘                                   └───────────┘
```

*Ce schéma est indicatif — l'équipe doit produire son propre schéma détaillé avec les choix réels effectués, en annexe du rapport.*

## 9. Environnement de laboratoire et application de démonstration

**Environnement recommandé :** deux VM Linux (plutôt que des conteneurs, pour une gestion réaliste de LVM, systemd et du pare-feu — cohérent avec les contraintes déjà identifiées aux modules 15 et 16), connectées sur le même réseau virtuel, avec un disque secondaire disponible sur `app-01` pour la démonstration LVM.

**Application de démonstration fournie ou à écrire par l'équipe** (un service HTTP minimal suffit, l'objectif du projet n'étant pas la complexité applicative mais la rigueur de son déploiement et de sa sécurisation) :

```python
# app.py - exemple minimal fourni comme point de départ
import http.server
import socketserver
import logging

logging.basicConfig(filename='/var/log/app-nomia/app.log', level=logging.INFO)

class Handler(http.server.SimpleHTTPRequestHandler):
    def do_GET(self):
        logging.info(f"Requête reçue: {self.path}")
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b"Atelier Nomia - Service applicatif actif\n")

with socketserver.TCPServer(("", 8080), Handler) as httpd:
    httpd.serve_forever()
```

## 10. Étapes et points de passage encadrés

### Phase 1 — Déploiement et durcissement de base (points de passage 1, fin de semaine 1)
- Partitionnement/LVM en place sur `app-01`
- Structure utilisateurs/groupes/sudo conforme au moindre privilège
- Service applicatif déployé en systemd avec garde-fous
- SSH durci sur les deux serveurs, pare-feu de base appliqué avec méthodologie prudente
- Validation par l'enseignant avant de poursuivre

### Phase 2 — Sauvegarde, supervision et validation fonctionnelle (points de passage 2, fin de semaine 2)
- Chaîne de sauvegarde `app-01` → `backup-01` fonctionnelle et planifiée
- Toutes les preuves de validation de la section 7.3 produites, y compris la **restauration réellement testée avec RTO mesuré**
- Point de passage : démonstration en direct de l'extension LVM à chaud et de la restauration

### Phase 3 — Incident et clôture (fin de semaine 2 ou 3 selon durée retenue)
- Introduction de l'incident choisi (section 7.4)
- Rédaction du rapport de diagnostic en 9 étapes
- Préparation de la soutenance

### Soutenance finale (20 minutes par binôme)
- 10 minutes : présentation de l'architecture, choix de sécurisation, démonstration de preuves clés (extension LVM à chaud, restauration)
- 5 minutes : présentation du diagnostic de l'incident (l'enseignant peut poser des questions comme si le binôme découvrait le ticket en direct, sans connaître l'incident à l'avance)
- 5 minutes de questions

## 11. Livrables attendus

- **Rapport de déploiement** : choix de partitionnement/LVM justifiés, structure utilisateurs/permissions, configuration complète des unités systemd, règles de pare-feu et configuration SSH (avec fichiers de configuration réels)
- **Preuves de validation fonctionnelle** : les 5 preuves de la section 7.3, chacune annotée
- **Documentation de la stratégie de sauvegarde** : type choisi (complète/incrémentale/différentielle), justification, politique de rétention, mesure du RTO obtenu lors du test de restauration
- **Rapport de diagnostic d'incident** : suivant explicitement les 9 étapes du module 22, incident révélé uniquement à ce stade du document
- **Support de soutenance**

## 12. Grille d'évaluation

| Critère | Pondération | Détail |
|---|---|---|
| Stockage et LVM (`app-01`) | 10% | Partitionnement réfléchi, extension à chaud démontrée |
| Utilisateurs et permissions | 10% | Moindre privilège appliqué, sudo restreint et justifié |
| Service applicatif systemd | 15% | Garde-fous anti-crash démontrés, durcissement appliqué |
| Pare-feu et SSH | 15% | Politique liste blanche fonctionnelle, durcissement SSH complet, méthodologie prudente respectée |
| Journalisation et tâches planifiées | 10% | Persistance, rotation, heartbeat fonctionnel |
| Sauvegarde et restauration | 15% | Chaîne fonctionnelle, **restauration réellement testée avec RTO mesuré** |
| Rapport de diagnostic d'incident (méthodologie 9 étapes) | 15% | Démarche méthodique visible, pas seulement la bonne réponse |
| Documentation et qualité de rédaction globale | 5% | Clarté, structure professionnelle |
| Soutenance | 5% | Clarté, capacité à répondre en diagnostic live |

**Note additionnelle, cohérente avec le module 22 :** un diagnostic correct de l'incident obtenu sans démarche méthodique visible (hypothèses non formulées, absence de collecte documentée, correction appliquée sans vérification) est pénalisé même si la conclusion finale est juste — comme pour le projet de clôture de l'UE2, l'évaluation porte sur la **démarche d'ingénieur**, pas seulement le résultat.

## 13. Contraintes importantes

- Toute règle de pare-feu ou modification `sshd_config` appliquée sans méthodologie prudente (rappel modules 17/18 : test avant fermeture de session, garde-fou SSH) et ayant conduit à une coupure d'accès non planifiée doit être documentée comme incident réel vécu, avec sa propre analyse en 9 étapes en complément (une occasion d'apprentissage, pas seulement une pénalité)
- Les preuves de validation doivent être réelles, produites sur le déploiement propre de l'équipe, jamais reconstituées après coup
- Le choix de l'incident volontaire (section 7.4) doit rester inconnu de l'enseignant jusqu'à la remise du rapport de diagnostic

## 14. Prolongement possible

Les huit outils développés au fil des modules 14 à 21 (Permission & Account Auditor, LVM Capacity Manager, Service Health Guardian, Safe Firewall Deployer, SSH Access Auditor & Hardener, Log & Schedule Guardian, Package & Kernel Compliance Auditor, Performance & Backup Health Guardian), ainsi que l'Assistant de diagnostic guidé du module 22, peuvent être déployés sur `app-01` et `backup-01` comme suite de supervision complémentaire — une intégration cohérente de ces outils dans le projet est valorisée en bonus, sans être une exigence du cahier des charges de base.

**Continuité vers l'UE4 :** ce projet prépare directement l'UE4 (Bash et automatisation Linux), où les scripts développés ici de façon ponctuelle pourront être généralisés, robustifiés et intégrés dans une véritable démarche d'automatisation d'infrastructure.

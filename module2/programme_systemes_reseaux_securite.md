# Programme de formation ingénieur
## Systèmes, Réseaux, Administration, Automatisation, Conteneurisation et Sécurité

**Public cible :** élèves ingénieurs (cycle ingénieur, niveau bac+3 à bac+5)
**Approche pédagogique :** Concept → Architecture → Fonctionnement interne → Outils → Démonstration → TD → TP → Troubleshooting → Sécurité → Automatisation → Étude de cas → Évaluation
**Durée totale estimée :** 4 semestres (S1 à S4), ~450–500h en présentiel encadré (CM+TD+TP), hors travail personnel et projets

---

## 1. Vue d'ensemble du parcours

| Semestre | Thématique dominante | UE | Volume horaire indicatif |
|---|---|---|---|
| S1 | Fondamentaux systèmes et réseaux | UE1, UE2 | ~110h |
| S2 | Administration Linux/Windows et scripting | UE3, UE4, UE5 | ~130h |
| S3 | Réseaux avancés, Python systèmes, sécurité | UE6, UE7, UE8 | ~130h |
| S4 | Conteneurisation, orchestration, projet intégrateur final | UE9, UE10, UE11 | ~100h |

Logique de progression : on ne peut pas sécuriser ou automatiser ce qu'on ne sait pas administrer ; on ne peut pas administrer ce qu'on ne comprend pas architecturalement. D'où l'ordre : **Fondamentaux → Administration → Automatisation/Réseaux avancés → Sécurité transverse → Conteneurisation/orchestration → Intégration**.

---

## 2. UE1 — Architecture des systèmes d'exploitation (S1)

**Prérequis :** notions de programmation (variables, structures de contrôle), aucune connaissance système préalable requise.
**Volume :** CM 12h / TD 8h / TP 10h = 30h

### Modules
1. **Introduction aux OS** — rôle, historique, familles (Unix/Linux, Windows), kernel vs user space
2. **Processus et threads** — cycle de vie, scheduling, contexte, PCB, ordonnancement (FIFO, round-robin, priorités)
3. **Mémoire** — mémoire virtuelle, pagination, segmentation, swap, gestion des adresses
4. **Système de fichiers** — inodes, arborescence, types de FS, montage, journaling
5. **I/O, interruptions, system calls, IPC** — pipes, signaux, sockets locaux, synchronisation, deadlocks
6. **Boot process** — BIOS/UEFI, bootloader, init/systemd

### Objectifs de compétence
- Expliquer le fonctionnement interne d'un OS moderne
- Diagnostiquer un problème lié aux processus, à la mémoire ou aux I/O
- Lire et interpréter l'état d'un système via ses structures internes (procfs, sysfs)

### Outils
`ps`, `top`, `htop`, `strace`, `lsof`, `/proc`, `vmstat`, `free`

### Projet du module
Aucun projet dédié — module fondateur, évalué par TD/TP/examen.

### Évaluation
20% contrôle continu / 20% TD / 30% TP / 30% examen

---

## 3. UE2 — Fondamentaux réseaux et TCP/IP (S1)

**Prérequis :** UE1 (notions I/O et sockets utiles)
**Volume :** CM 14h / TD 10h / TP 14h = 38h

### Modules
1. **Modèle OSI et TCP/IP** — encapsulation, couches, comparaison des modèles
2. **Ethernet, ARP, commutation** — trames, MAC, VLAN, trunking, spanning tree (intro)
3. **IPv4 / IPv6, subnetting, CIDR** — calcul de sous-réseaux, adressage, routage statique
4. **TCP en profondeur** — 3-way handshake, sequence/ACK numbers, sliding window, flow/congestion control, TIME_WAIT, machine à états TCP
5. **UDP, ICMP** — cas d'usage, différences avec TCP
6. **DNS, DHCP** — fonctionnement, résolution, baux DHCP
7. **Analyse de trafic** — capture et lecture Wireshark sur scénarios réels (handshake, perte de paquets, DNS lookup)

### Objectifs de compétence
- Calculer un plan d'adressage IPv4/IPv6 pour une infrastructure donnée
- Lire une capture Wireshark et identifier une anomalie réseau
- Expliquer précisément le comportement de TCP face à la perte de paquets

### Outils
`ip`, `ping`, `traceroute`, `dig`, `nslookup`, `ss`, Wireshark, `tcpdump`

### Projet intégrateur
**Projet 3** (partiel) : maquette réseau simple + première capture/analyse Wireshark

### Évaluation
20% CC / 20% TD / 30% TP / 30% examen

---

## 4. UE3 — Administration Linux (S2)

**Prérequis :** UE1, UE2
**Volume :** CM 12h / TD 10h / TP 20h = 42h

### Modules
1. Installation, boot, gestion des utilisateurs/groupes, permissions (rwx, ACL, sudo)
2. Filesystems, mount, LVM, gestion du stockage
3. Processus, services, systemd, journald
4. Réseau côté administration (interfaces, routes, firewall — iptables/nftables)
5. SSH, durcissement d'accès distant
6. Logs, cron, gestion des tâches planifiées
7. Package management (APT/DNF), mises à jour, gestion du noyau
8. Supervision, performance, sauvegarde (rsync, tar)
9. Troubleshooting méthodique (panne de service, disque plein, réseau down)

### Objectifs de compétence
- Administrer un serveur Linux de bout en bout de façon autonome
- Diagnostiquer une panne de service en suivant une méthodologie structurée
- Sécuriser un accès SSH et une configuration de base

### Outils
`ss`, `journalctl`, `systemctl`, `chmod`, `chown`, `find`, `grep`, `awk`, `sed`, `tar`, `rsync`, `crontab`, `ufw`/`nftables`

### Projet intégrateur
**Projet 1** : Déployer et sécuriser une infrastructure Linux

### Évaluation
15% CC / 20% TD / 35% TP / 30% projet

---

## 5. UE4 — Bash et automatisation Linux (S2)

**Prérequis :** UE3
**Volume :** CM 6h / TD 8h / TP 16h = 30h

### Modules
1. Syntaxe Bash : variables, conditions, boucles, fonctions, arguments, exit codes
2. Pipes, redirections, traitement de texte (`grep`, `sed`, `awk`, `cut`, `sort`, `uniq`, `xargs`)
3. Gestion des erreurs et bonnes pratiques de scripting
4. Automatisation : scripts de monitoring, vérification de services, analyse de logs, détection d'anomalies

### Objectifs de compétence
- Écrire des scripts Bash robustes avec gestion d'erreurs
- Automatiser des tâches d'administration récurrentes

### Projet intégrateur
**Projet 4** (partie Bash) : scripts de supervision et de sauvegarde automatisée

### Évaluation
20% TD / 40% TP / 40% mini-projet

---

## 6. UE5 — Administration Windows Server et PowerShell (S2)

**Prérequis :** UE1 (notions OS transférables)
**Volume :** CM 12h / TD 10h / TP 20h = 42h

### Modules
1. Architecture Windows, services, registre, journal des événements
2. Active Directory : domaines, contrôleurs de domaine, OU, objets
3. DNS, DHCP côté Windows Server
4. Group Policy (GPO), Kerberos, LDAP — authentification/autorisation
5. Permissions NTFS, ACL
6. PowerShell : cmdlets, pipeline d'objets, variables, fonctions, modules, remoting
7. Comparaison Bash (texte) vs PowerShell (objets) — implications pratiques
8. Durcissement Windows Server, sauvegarde, troubleshooting

### Objectifs de compétence
- Construire et administrer un environnement Windows Server réaliste avec AD/DNS/DHCP/GPO
- Écrire des scripts PowerShell d'administration et d'automatisation
- Expliquer les différences philosophiques Bash vs PowerShell avec des exemples concrets

### Projet intégrateur
**Projet 2** : Infrastructure Windows Server avec AD, DNS, DHCP et GPO
**Projet 5** : Automatisation PowerShell

### Évaluation
15% CC / 20% TD / 35% TP / 30% projet

---

## 7. UE6 — Réseaux avancés et services (S3)

**Prérequis :** UE2, UE3
**Volume :** CM 10h / TD 8h / TP 16h = 34h

### Modules
1. Routage dynamique (notions), NAT avancé
2. Firewalls, proxy, load balancing
3. VPN, TLS/SSL, certificats (introduction à la PKI)
4. Troubleshooting réseau avancé (scénarios multi-causes)
5. Wireshark avancé : analyse d'incidents réseau réels

### Objectifs de compétence
- Concevoir une architecture réseau incluant NAT, firewall et VPN
- Diagnostiquer des pannes réseau complexes par une démarche d'hypothèses

### Projet intégrateur
**Projet 3** (finalisation) : infrastructure réseau complète + analyse de trafic

### Évaluation
20% TD / 40% TP / 40% étude de cas

---

## 8. UE7 — Python pour systèmes, réseaux et sécurité (S3)

**Prérequis :** UE3, UE4, bases de programmation Python
**Volume :** CM 8h / TD 8h / TP 20h = 36h

### Modules
1. Rappels ciblés : fonctions, exceptions, fichiers, JSON/YAML/CSV, regex
2. `subprocess`, `os`, `pathlib`, `logging`, `argparse`
3. Réseau en Python : `requests`, `sockets`, interaction avec APIs REST, SSH (paramiko/fabric)
4. Programmes d'ingénierie : analyseur de logs, vérificateur de disponibilité, collecteur d'infos système, générateur de rapports
5. Scanner de services **en environnement de laboratoire autorisé uniquement**

### Objectifs de compétence
- Utiliser Python comme outil d'automatisation système/réseau (pas comme cours généraliste)
- Développer un outil de monitoring ou de collecte d'informations fonctionnel

### Projet intégrateur
**Projet 4** (partie Python) : automatisation combinée Bash + Python

### Évaluation
15% TD / 45% TP / 40% mini-projet

---

## 9. UE8 — Sécurité des systèmes et réseaux (transverse, S3)

**Prérequis :** UE3, UE5, UE6
**Volume :** CM 12h / TD 8h / TP 14h = 34h

### Modules
1. CIA Triad, AAA (Authentication, Authorization, Accounting), least privilege, defense in depth
2. Cryptographie fondamentale, TLS, certificats, PKI
3. Password security, MFA, hardening Linux/Windows
4. Firewalls, IDS/IPS, notions de SIEM, logging/monitoring de sécurité
5. Gestion des vulnérabilités, patch management
6. Réponse à incident — méthodologie (détection, containment, éradication, retour d'expérience)

### Objectifs de compétence
- Appliquer les principes de sécurité à toute technologie étudiée précédemment ("comment la sécuriser ?")
- Mener une réponse à incident structurée sur un scénario de laboratoire

### Contrainte
Tous les exercices pratiques sont réalisés en environnement de laboratoire isolé et autorisé (VM, réseau de test).

### Évaluation
20% CC / 20% TD / 30% TP / 30% étude de cas

---

## 10. UE9 — Conteneurisation avec Docker (S4)

**Prérequis :** UE3
**Volume :** CM 8h / TD 6h / TP 16h = 30h

### Modules
1. Virtualisation vs conteneurisation, namespaces, cgroups, isolation
2. Images, layers, registry, Dockerfile
3. Volumes, réseaux Docker, variables d'environnement, secrets
4. Docker Compose, cycle de vie des conteneurs, logs, limites de ressources
5. Sécurité des conteneurs (bonnes pratiques d'images, moindre privilège)

### Objectifs de compétence
- Conteneuriser une application multi-services avec Docker Compose
- Appliquer les bonnes pratiques de sécurité aux images et conteneurs

### Projet intégrateur
**Projet 6** : Conteneurisation d'une application avec Docker et Docker Compose

### Évaluation
20% TD / 40% TP / 40% mini-projet

---

## 11. UE10 — Orchestration Kubernetes (S4)

**Prérequis :** UE9
**Volume :** CM 6h / TD 4h / TP 12h = 22h

### Modules
1. Architecture Kubernetes : control plane, nodes, kubelet
2. Pods, Deployments, Services, ConfigMaps, Secrets
3. Namespaces, volumes, Ingress
4. Scaling, health checks

### Objectifs de compétence
- Déployer une application conteneurisée sur un cluster Kubernetes de laboratoire
- Configurer scaling et health checks de base

### Projet intégrateur
**Projet 7** : Déploiement d'une architecture conteneurisée avec Kubernetes

### Évaluation
20% TD / 40% TP / 40% mini-projet

---

## 12. UE11 — Projet intégrateur final (S4)

**Prérequis :** toutes les UE précédentes
**Volume :** encadrement 20h / travail projet ~60h

### Contenu
Conception d'une infrastructure complète intégrant : réseau, Linux, Windows/AD, services réseau (DNS/DHCP), Docker, automatisation Python/Bash/PowerShell, supervision, sécurité, sauvegarde, documentation.

### Livrables attendus
- Cahier des charges
- Architecture documentée (schémas réseau et systèmes)
- Scripts d'automatisation
- Politique de sécurité et de sauvegarde
- Documentation technique complète
- Soutenance

### Grille d'évaluation
| Critère | Pondération |
|---|---|
| Compréhension et justification des choix techniques | 15% |
| Maîtrise technique (Linux/Windows/réseau) | 20% |
| Qualité du code/des scripts | 15% |
| Sécurité de l'architecture | 15% |
| Qualité de l'architecture globale | 15% |
| Documentation | 10% |
| Autonomie et soutenance | 10% |

---

## 13. Synthèse des compétences visées en fin de parcours

À l'issue des 4 semestres, l'étudiant est capable de :

- Concevoir et justifier une architecture réseau et systèmes hétérogène (Linux/Windows)
- Administrer de façon autonome des serveurs Linux et Windows Server (AD, DNS, DHCP, GPO)
- Diagnostiquer méthodiquement des pannes systèmes, réseau et applicatives
- Automatiser des tâches d'administration en Bash, PowerShell et Python
- Conteneuriser et orchestrer des applications (Docker, Kubernetes)
- Appliquer les principes de sécurité (CIA, defense in depth, hardening, PKI) à toute infrastructure conçue
- Analyser du trafic réseau avec Wireshark et diagnostiquer des anomalies
- Documenter et présenter un projet d'infrastructure comme un ingénieur système/réseau/sécurité

---

## 14. Prochaine étape (mode interactif)

Conformément au mode de fonctionnement demandé : pour chaque UE ci-dessus, je peux maintenant, module par module :
1. Détailler prérequis, objectifs précis et progression pédagogique
2. Attendre votre validation
3. Puis développer le contenu complet (COURS / TD / TP / ÉVALUATION / PROJET / SLIDES / LAB / CORRECTION selon votre commande)

Dites-moi par quelle UE ou quel module vous voulez commencer.

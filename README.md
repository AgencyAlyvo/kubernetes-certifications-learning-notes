# Kubernetes Certifications — Notes & Synthèses - Documentation faite sur Kubernetes v1.37

Ce dépôt regroupe mes notes, synthèses techniques, commandes, exercices pratiques et révisions pour les certifications Kubernetes et Cloud Native de la CNCF.

L'objectif est de comprendre Kubernetes en profondeur, de maîtriser l'administration et le dépannage de clusters, le déploiement d'applications, la sécurité des workloads et l'écosystème Cloud Native, puis de développer des compétences directement applicables en **DevOps, SRE et Platform Engineering**, jusqu'aux environnements de production.

La préparation privilégie la pratique : manipulation de clusters, exercices en ligne de commande, résolution d'incidents, configuration de workloads et mise en œuvre de bonnes pratiques opérationnelles.

## Certifications

### 1. Certified Kubernetes Administrator (CKA)

Certification professionnelle validant les compétences nécessaires pour installer, configurer, administrer et dépanner des clusters Kubernetes.

#### Détails de l'examen

| Information | Détail |
|---|---|
| Organisme | CNCF / Linux Foundation |
| Code de certification | CKA |
| Type d'évaluation | Examen pratique en environnement Kubernetes |
| Format | Examen surveillé en ligne (*Online Proctored*) |
| Durée | 2 heures |
| Langue | Anglais |
| Validité | 2 ans |
| Prix indicatif | À vérifier sur le site officiel |
| Prérequis formels | Aucun |

#### Objectifs de préparation

- [ ] **1. Architecture du cluster**
  - [ ] Comprendre l'architecture Kubernetes et ses composants.
  - [ ] Distinguer le control plane et les nœuds workers.
  - [ ] Comprendre le rôle de l'API Server, d'etcd, du Scheduler et du Controller Manager.
  - [ ] Comprendre le fonctionnement de kubelet et du runtime de conteneurs.
  - [ ] Identifier les composants et leurs interactions lors d'une requête Kubernetes.

- [ ] **2. Installation, configuration et gestion du cluster**
  - [ ] Comprendre le bootstrap d'un cluster avec kubeadm.
  - [ ] Initialiser un cluster et joindre des nœuds.
  - [ ] Gérer les certificats et les composants du control plane.
  - [ ] Sauvegarder et restaurer etcd.
  - [ ] Effectuer les opérations de maintenance et de mise à niveau du cluster.

- [ ] **3. Workloads et planification**
  - [ ] Créer et gérer Pods, Deployments, ReplicaSets et DaemonSets.
  - [ ] Comprendre les stratégies de déploiement et les mises à jour.
  - [ ] Configurer les probes de démarrage, de disponibilité et de vivacité.
  - [ ] Utiliser les requests et limits CPU/mémoire.
  - [ ] Comprendre les taints, tolerations, labels, selectors et affinity.
  - [ ] Gérer les workloads en échec et les problèmes de scheduling.

- [ ] **4. Réseau et services**
  - [ ] Comprendre le modèle réseau Kubernetes.
  - [ ] Configurer Services ClusterIP, NodePort et LoadBalancer.
  - [ ] Comprendre le DNS interne et la découverte de services.
  - [ ] Configurer les Ingress ou les mécanismes de routage couverts par le programme officiel en vigueur.
  - [ ] Comprendre les CNI et le fonctionnement de la communication entre Pods.
  - [ ] Diagnostiquer les problèmes de connectivité et de résolution DNS.

- [ ] **5. Stockage**
  - [ ] Comprendre les volumes et leur cycle de vie.
  - [ ] Créer et utiliser PersistentVolumes et PersistentVolumeClaims.
  - [ ] Comprendre StorageClasses et le provisionnement dynamique.
  - [ ] Diagnostiquer les problèmes de montage et d'accès au stockage.

- [ ] **6. Dépannage**
  - [ ] Analyser les événements et les logs des Pods.
  - [ ] Utiliser `kubectl describe`, `kubectl logs` et `kubectl get`.
  - [ ] Diagnostiquer les problèmes de scheduling, de réseau et de stockage.
  - [ ] Identifier les dysfonctionnements des composants du cluster.
  - [ ] Utiliser les commandes Linux utiles au diagnostic.

- [ ] **7. Configuration et sécurité**
  - [ ] Gérer ConfigMaps et Secrets.
  - [ ] Comprendre les ServiceAccounts et les autorisations RBAC.
  - [ ] Configurer les contextes et namespaces.
  - [ ] Appliquer les bonnes pratiques de configuration des workloads.

#### Renouvellement de la certification

La validité et les modalités de renouvellement doivent être vérifiées auprès de la CNCF / Linux Foundation avant l'inscription à l'examen.

---

### 2. Certified Kubernetes Application Developer (CKAD)

Certification validant les compétences nécessaires pour concevoir, configurer et déployer des applications conteneurisées sur Kubernetes.

L'accent est mis sur les objets Kubernetes utilisés par les développeurs, la configuration des applications, le réseau, le stockage et le diagnostic des workloads.

#### Détails de l'examen

| Information | Détail |
|---|---|
| Organisme | CNCF / Linux Foundation |
| Code de certification | CKAD |
| Type d'évaluation | Examen pratique en environnement Kubernetes |
| Format | Examen surveillé en ligne (*Online Proctored*) |
| Durée | 2 heures |
| Langue | Anglais |
| Validité | 2 ans |
| Prix indicatif | À vérifier sur le site officiel |
| Prérequis formels | Aucun |

#### Objectifs de préparation

- [ ] **1. Conception et architecture des applications**
  - [ ] Comprendre les principaux objets Kubernetes utilisés pour exécuter des applications.
  - [ ] Créer des Pods, Deployments et Jobs.
  - [ ] Configurer les stratégies de déploiement et les mises à jour.
  - [ ] Utiliser les labels, selectors et namespaces.
  - [ ] Comprendre les workloads adaptés aux applications stateless et aux traitements ponctuels.

- [ ] **2. Configuration des applications**
  - [ ] Configurer les arguments et variables d'environnement des conteneurs.
  - [ ] Utiliser ConfigMaps et Secrets.
  - [ ] Configurer les commandes et arguments des conteneurs.
  - [ ] Définir les requests et limits de ressources.
  - [ ] Configurer les probes de démarrage, de vivacité et de disponibilité.
  - [ ] Utiliser les comptes de service lorsque nécessaire.

- [ ] **3. Déploiement des applications**
  - [ ] Créer et modifier des Deployments.
  - [ ] Effectuer des mises à jour progressives et des rollbacks.
  - [ ] Comprendre les stratégies de redémarrage et de gestion des échecs.
  - [ ] Utiliser les Jobs et CronJobs.
  - [ ] Comprendre les contraintes de planification des workloads.

- [ ] **4. Réseau et exposition des applications**
  - [ ] Exposer une application avec un Service.
  - [ ] Comprendre les types de Services et leurs usages.
  - [ ] Configurer la découverte de services et le DNS.
  - [ ] Comprendre les mécanismes d'exposition HTTP et les Ingress selon le programme en vigueur.
  - [ ] Diagnostiquer les problèmes de connectivité entre applications.

- [ ] **5. Persistance des données**
  - [ ] Utiliser les volumes dans les Pods.
  - [ ] Configurer les PersistentVolumeClaims.
  - [ ] Comprendre les StorageClasses.
  - [ ] Choisir une stratégie de stockage adaptée à l'application.

- [ ] **6. Dépannage des applications**
  - [ ] Lire les logs et événements Kubernetes.
  - [ ] Diagnostiquer les erreurs de démarrage des conteneurs.
  - [ ] Résoudre les problèmes de configuration et de scheduling.
  - [ ] Vérifier les probes, les ressources et les dépendances réseau.
  - [ ] Utiliser efficacement les commandes impératives et déclaratives de `kubectl`.

#### Renouvellement de la certification

La validité et les modalités de renouvellement doivent être vérifiées auprès de la CNCF / Linux Foundation avant l'inscription à l'examen.

---

### 3. Certified Kubernetes Security Specialist (CKS)

Certification avancée validant les compétences nécessaires pour sécuriser un cluster Kubernetes et ses workloads tout au long de leur cycle de vie.

Elle couvre notamment la configuration sécurisée des clusters, la gestion des identités et des accès, la sécurisation des conteneurs, la détection des menaces et les mécanismes de défense à l'exécution.

#### Détails de l'examen

| Information | Détail |
|---|---|
| Organisme | CNCF / Linux Foundation |
| Code de certification | CKS |
| Type d'évaluation | Examen pratique en environnement Kubernetes |
| Format | Examen surveillé en ligne (*Online Proctored*) |
| Durée | 2 heures |
| Langue | Anglais |
| Validité | 2 ans |
| Prix indicatif | À vérifier sur le site officiel |
| Prérequis | Certification CKA valide exigée |

#### Objectifs de préparation

- [ ] **1. Configuration sécurisée du cluster**
  - [ ] Comprendre les composants de sécurité de Kubernetes.
  - [ ] Sécuriser les paramètres du control plane.
  - [ ] Gérer les certificats et les communications entre composants.
  - [ ] Appliquer les bonnes pratiques de durcissement des nœuds.
  - [ ] Comprendre les recommandations du benchmark CIS Kubernetes.

- [ ] **2. Gestion des identités et des accès**
  - [ ] Configurer RBAC avec le principe du moindre privilège.
  - [ ] Gérer les ServiceAccounts.
  - [ ] Restreindre les permissions des utilisateurs et des workloads.
  - [ ] Comprendre les mécanismes d'authentification et d'autorisation Kubernetes.

- [ ] **3. Sécurité des workloads**
  - [ ] Appliquer les SecurityContext adaptés.
  - [ ] Comprendre les Pod Security Standards.
  - [ ] Limiter les privilèges des conteneurs.
  - [ ] Configurer les politiques réseau.
  - [ ] Comprendre les mécanismes d'isolation et les risques liés aux conteneurs privilégiés.

- [ ] **4. Chaîne d'approvisionnement logicielle**
  - [ ] Comprendre les risques liés aux images de conteneurs.
  - [ ] Analyser et vérifier les images.
  - [ ] Utiliser les outils de détection de vulnérabilités prévus par le programme.
  - [ ] Appliquer les bonnes pratiques de provenance et de confiance des artefacts.

- [ ] **5. Détection des menaces et sécurité à l'exécution**
  - [ ] Analyser les événements et les comportements suspects.
  - [ ] Comprendre les outils de détection à l'exécution couverts par le programme.
  - [ ] Examiner les logs et les traces d'activité.
  - [ ] Identifier les comportements anormaux des workloads.

- [ ] **6. Sécurité des applications et des données**
  - [ ] Limiter l'exposition des Secrets.
  - [ ] Comprendre les risques liés aux permissions excessives.
  - [ ] Appliquer des règles de segmentation réseau.
  - [ ] Comprendre le chiffrement des données et les mécanismes de protection du cluster.

- [ ] **7. Audit et réponse aux incidents**
  - [ ] Comprendre l'audit Kubernetes.
  - [ ] Analyser les événements de sécurité.
  - [ ] Identifier les configurations dangereuses.
  - [ ] Appliquer des mesures de confinement et de remédiation adaptées.

#### Renouvellement de la certification

La validité et les modalités de renouvellement doivent être vérifiées auprès de la CNCF / Linux Foundation avant l'inscription à l'examen.

---

### 4. Kubernetes and Cloud Native Associate (KCNA)

Certification d'entrée de gamme validant les connaissances fondamentales de Kubernetes et de l'écosystème Cloud Native.

Elle permet de consolider le vocabulaire, les concepts d'architecture et la compréhension des principaux composants de l'écosystème.

#### Objectifs de préparation

- [ ] **1. Architecture Kubernetes**
  - [ ] Comprendre les composants principaux d'un cluster.
  - [ ] Comprendre le rôle du control plane et des workers.
  - [ ] Identifier les principaux objets Kubernetes.

- [ ] **2. Concepts Kubernetes**
  - [ ] Comprendre Pods, Deployments, Services et namespaces.
  - [ ] Comprendre les principes de scheduling et de gestion des ressources.
  - [ ] Identifier les mécanismes de stockage et de réseau.

- [ ] **3. Cloud Native et orchestration**
  - [ ] Comprendre les principes des applications cloud natives.
  - [ ] Comprendre les conteneurs et l'orchestration.
  - [ ] Comprendre les architectures distribuées et les microservices.

- [ ] **4. Écosystème Cloud Native**
  - [ ] Identifier les rôles des outils de conteneurisation, de réseau et de stockage.
  - [ ] Comprendre les principes de CI/CD et de GitOps.
  - [ ] Comprendre les outils de supervision et d'observabilité.

- [ ] **5. Sécurité et conformité**
  - [ ] Comprendre les bases de la sécurité des workloads.
  - [ ] Identifier les risques courants liés aux configurations.
  - [ ] Comprendre les principes du moindre privilège.

#### Détails de l'examen

| Information | Détail |
|---|---|
| Organisme | CNCF / Linux Foundation |
| Code de certification | KCNA |
| Type d'évaluation | Questions à choix multiples |
| Format | À vérifier sur la page officielle |
| Durée | À vérifier sur la page officielle |
| Langue | À vérifier sur la page officielle |
| Validité | À vérifier sur la page officielle |
| Prix indicatif | À vérifier sur le site officiel |

---

### 5. Kubernetes and Cloud Native Security Associate (KCSA)

Certification d'entrée de gamme centrée sur les fondamentaux de la sécurité Kubernetes et Cloud Native.

Elle complète la compréhension générale de Kubernetes en abordant les principes de sécurité, les menaces, les mécanismes de protection et les bonnes pratiques.

#### Objectifs de préparation

- [ ] **1. Principes fondamentaux de la sécurité**
  - [ ] Comprendre les objectifs de confidentialité, d'intégrité et de disponibilité.
  - [ ] Comprendre le modèle de responsabilité partagée.
  - [ ] Identifier les menaces courantes dans les environnements conteneurisés.

- [ ] **2. Sécurité Kubernetes**
  - [ ] Comprendre l'authentification et l'autorisation.
  - [ ] Comprendre RBAC et le principe du moindre privilège.
  - [ ] Identifier les risques liés aux ServiceAccounts, aux Secrets et aux permissions.
  - [ ] Comprendre les politiques de sécurité des Pods.

- [ ] **3. Sécurité des conteneurs**
  - [ ] Comprendre les risques liés aux images.
  - [ ] Identifier les vulnérabilités des dépendances et des artefacts.
  - [ ] Comprendre l'isolation et les privilèges des conteneurs.

- [ ] **4. Sécurité réseau**
  - [ ] Comprendre la segmentation réseau.
  - [ ] Comprendre les NetworkPolicies.
  - [ ] Identifier les risques liés aux communications et à l'exposition des services.

- [ ] **5. Surveillance et conformité**
  - [ ] Comprendre l'audit et la journalisation.
  - [ ] Identifier les outils de détection et de contrôle.
  - [ ] Comprendre les principes de remédiation et d'amélioration continue.

#### Détails de l'examen

| Information | Détail |
|---|---|
| Organisme | CNCF / Linux Foundation |
| Code de certification | KCSA |
| Type d'évaluation | Questions à choix multiples |
| Format | À vérifier sur la page officielle |
| Durée | À vérifier sur la page officielle |
| Langue | À vérifier sur la page officielle |
| Validité | À vérifier sur la page officielle |
| Prix indicatif | À vérifier sur le site officiel |

---

## Organisation du dépôt

```text
kubernetes-certifications-learning-notes/
├── README.md
├── docs/
│   ├── 01-fundamentals/
├── labs/
│   ├── 01-kubernetes-basics/
├── .gitignore
```

### Rôle des répertoires

- `docs/` : cours synthétiques, concepts, fiches de révision et documentation.

# Architecture d'un cluster Kubernetes

Un cluster Kubernetes comprend un **Control Plane**, qui gère le cluster, et un ou plusieurs **Nodes**, qui exécutent les workloads. Pour assurer la haute disponibilité, les composants du Control Plane peuvent être déployés sur plusieurs machines.

- **Control Plane** : expose l'API du cluster, orchestre les workloads et veille à ce que l'état réel du cluster converge vers l'état désiré.
- **Nodes** : fournissent les ressources de calcul nécessaires à l'exécution des `Pods` et de leurs containers.

![Architecture d'un cluster Kubernetes](images/kubernetes-cluster-architecture.svg)

<br /><br />

---

<br /><br />

## Composants du Control Plane

- **`kube-apiserver`** : expose l'API HTTP Kubernetes et constitue le point d'entrée du Control Plane. Il reçoit les requêtes des utilisateurs et des composants du cluster, contrôle les opérations sur les ressources et sert d'interface d'accès à `etcd`.
  1. Reçoit les requêtes via l'API Kubernetes et applique les contrôles pertinents, notamment l'authentification, l'autorisation et, pour les écritures concernées, les contrôles d'admission et de validation.
  2. Lit les données du cluster ou, pour les écritures acceptées, enregistre les modifications dans `etcd`.
  3. Renvoie les résultats aux clients et expose des mécanismes d'observation, notamment les `watch`, permettant aux composants autorisés de suivre les changements des ressources.

- **`etcd`** : base de données clé-valeur distribuée qui stocke les données persistantes du cluster, notamment la configuration et l'état des ressources Kubernetes.
  1. Reçoit les opérations de lecture et d'écriture du `kube-apiserver`.
  2. Persiste les données et, dans un cluster `etcd` distribué, réplique les écritures entre ses membres selon le mécanisme de consensus.
  3. Renvoie les données ou le résultat des opérations au `kube-apiserver`.

- **`kube-scheduler`** : attribue les `Pods` qui n'ont pas encore de `Node` assigné à des nœuds appropriés.
  1. Observe via le `kube-apiserver` les `Pods` en attente d'affectation.
  2. Identifie les `Nodes` compatibles avec les besoins en ressources, les contraintes de placement et les règles de planification, puis évalue les candidats pour sélectionner un `Node`.
  3. Enregistre l'affectation du `Pod` au `Node` sélectionné via l'API du `kube-apiserver`.
  4. Le `kube-apiserver` traite la requête et persiste l'affectation dans `etcd`.
  5. Le `kubelet` du `Node` sélectionné observe l'affectation via l'API et prend en charge l'exécution du `Pod`.

- **`kube-controller-manager`** : exécute les `Controllers` Kubernetes, qui rapprochent l'état réel du cluster de l'état désiré.
  1. Les contrôleurs observent les ressources et leur état via le `kube-apiserver`, notamment au moyen de mécanismes `watch`.
  2. Ils comparent l'état observé à l'état désiré défini par les ressources Kubernetes.
  3. Lorsqu'un écart nécessite une action, ils effectuent les opérations relevant de leur responsabilité, par exemple en créant, modifiant ou supprimant des ressources via l'API du `kube-apiserver`.
  4. Le `kube-apiserver` traite les requêtes et persiste les modifications acceptées dans `etcd`.
  5. Les contrôleurs poursuivent leur boucle de réconciliation en tenant compte des changements observés, jusqu'à ce que l'état désiré soit atteint ou qu'une intervention supplémentaire soit nécessaire.

  Exemples : `Node Controller`, `Job Controller`, `EndpointSlice Controller`, `ServiceAccount Controller`.

- **`cloud-controller-manager` (optionnel)** : exécute les `Controllers` spécifiques au fournisseur Cloud, qui font le lien entre Kubernetes et les services d'infrastructure du fournisseur.
  1. Observe les ressources Kubernetes via le `kube-apiserver` et détermine les opérations nécessaires sur l'infrastructure Cloud.
  2. Interagit avec l'API du fournisseur Cloud pour récupérer des informations ou effectuer les opérations nécessaires sur l'infrastructure.
  3. Met à jour les ressources Kubernetes via le `kube-apiserver` lorsque nécessaire, après les contrôles applicables.
  4. Répète ce processus pour maintenir la cohérence entre les ressources Kubernetes et l'infrastructure du fournisseur.

  Exemples : gestion des `Nodes`, des routes réseau et des `Load Balancers`.

<br />

## Composants des Nodes

- **`kubelet`** : composant chargé de créer et de surveiller les `Pods` qui lui sont assignés. Il cherche à faire converger leur état réel vers l'état souhaité décrit dans leur `PodSpec`.
  1. **Observe les `Pods`** : récupère via le `kube-apiserver` les informations sur les `Pods` assignés à son `Node`. Il prend également en charge les `Static Pods`, dont les spécifications sont définies localement.
  2. **Synchronise leur état** : sa boucle de synchronisation (*Sync Loop*) traite les travaux associés aux `Pods`. Les *Pod workers* exécutent la logique nécessaire pour rapprocher leur état réel de l'état souhaité.
  3. **Prépare leur exécution** : coordonne les opérations nécessaires, notamment le montage des volumes et la mise à disposition des configurations et des secrets requis par les containers.
  4. **Pilote le `Container runtime`** : utilise la `Container Runtime Interface` (`CRI`) pour demander au runtime de créer le *`Pod sandbox`*, puis de créer et de démarrer les containers définis dans le `PodSpec`. Il lui demande également d'arrêter ou de supprimer les containers lorsque nécessaire.
  5. **Suit l'état des containers** : le `PLEG` (*Pod Lifecycle Event Generator*) interroge périodiquement le runtime pour détecter les changements d'état des containers et génère des événements utilisés par le `kubelet` pour déclencher des traitements de synchronisation. D'autres événements, notamment les changements observés via l'API, peuvent également déclencher ces traitements.
  6. **Exécute les probes configurées** : exécute les `startupProbe`, `readinessProbe` et `livenessProbe` définies dans les spécifications des containers. Leurs résultats permettent notamment de déterminer si un conteneur a terminé son démarrage, s'il est prêt à recevoir du trafic ou s'il doit être redémarré.
  7. **Actualise l'état du cluster** : met à jour l'état des `Pods` et les informations relatives à son `Node`, puis publie les mises à jour nécessaires via le `kube-apiserver`.

- **`kube-proxy` (optionnel)** : composant réseau qui implémente une partie du comportement des `Services`.
  1. Observe via le `kube-apiserver` les ressources nécessaires au fonctionnement des `Services`, notamment les `Services` et leurs points de terminaison.
  2. Configure ou met à jour les règles réseau locales pour diriger le trafic vers les bons `Pods`.
  3. Maintient ces règles en fonction des changements des `Services` et de leurs points de terminaison.
  
  Si une implémentation réseau fournit un comportement équivalent pour les `Services`, `kube-proxy` n'est pas nécessaire.

- **`Container runtime`** : composant chargé de gérer l'environnement d'exécution des `Pods` et le cycle de vie de leurs containers, par exemple `containerd` ou `CRI-O`.
  1. Reçoit les demandes du `kubelet` via la `Container Runtime Interface` (`CRI`).
  2. **Gère l'environnement d'exécution** : crée le *`Pod sandbox`*, puis crée, démarre, arrête et supprime les containers du `Pod` selon les demandes du `kubelet` via la `CRI`.
  3. Fournit au `kubelet` les informations sur l'état des containers et du *`Pod sandbox`*, que celui-ci utilise, avec d'autres informations, pour suivre et actualiser l'état des `Pods`.

<br /><br />

---

<br /><br />

## Variations architecturales

### Options de déploiement du **Control Plane**

Les composants du **Control Plane** peuvent être déployés de plusieurs manières :

- **Déploiement traditionnel** : Les composants du **Control Plane** s'exécutent directement sur des machines dédiées ou des machines virtuelles, souvent gérées en tant que services `systemd`.

- **`Static Pods`** : Les composants du **Control Plane** sont déployés sous forme de `pods static`, gérés par le `kubelet` sur des **Nodes** spécifiques. Il s'agit d'une approche courante utilisée par des outils comme `kubeadm`.

- **Déploiement self-hosted** : Le **Control Plane** s'exécute sous forme de `Pods` au sein du cluster Kubernetes lui-même, gérés par des `Deployments` et des `StatefulSets` ou d'autres primitives Kubernetes.

- **Kubernetes managé** : le fournisseur Cloud prend en charge le déploiement et la gestion du **Control Plane**, par exemple avec Amazon EKS, Azure AKS ou Google GKE.

<br />

### Outils de gestion de cluster

- **Outils de déploiement** : `kubeadm`, `kops` et `Kubespray` offrent différentes approches pour le déploiement et la gestion de clusters, chacun avec sa propre méthode d'agencement et de gestion des composants.

<br />

### Personnalisation et extensibilité

- **`kube-scheduler` personnalisé** : un scheduler personnalisé peut fonctionner en parallèle du `kube-scheduler` par défaut ou le remplacer entièrement.
- **Extension de l'API Kubernetes** : les `CustomResourceDefinitions` (CRD) permettent d'ajouter de nouveaux types de ressources à l'API Kubernetes, tandis que l'API Aggregation permet d'y intégrer des API supplémentaires.
- **Intégration Cloud** : les fournisseurs Cloud peuvent intégrer leurs services à Kubernetes via le `cloud-controller-manager`, notamment pour gérer les `Nodes`, les routes réseau et les `Load Balancers`.
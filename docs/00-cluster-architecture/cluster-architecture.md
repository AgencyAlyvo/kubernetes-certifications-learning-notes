# Architecture d'un cluster Kubernetes

Un cluster Kubernetes comprend un **Control Plane**, qui gère le cluster, et un ou plusieurs **Nodes**, qui exécutent les workloads. Pour assurer la haute disponibilité, les composants du Control Plane peuvent être déployés sur plusieurs machines.

- **Control Plane** : expose l'API du cluster, orchestre les workloads et veille à ce que l'état réel du cluster converge vers l'état désiré.
- **Nodes** : fournissent les ressources de calcul nécessaires à l'exécution des `Pods` et de leurs containers.

![Architecture d'un cluster Kubernetes](images/kubernetes-cluster-architecture.svg)

<br /><br />

---

<br /><br />

## Composants du Control Plane

- **`kube-apiserver`** : expose l'API HTTP Kubernetes et constitue le point d'entrée du Control Plane. Il reçoit les requêtes des utilisateurs et des composants du cluster, valide les opérations sur les ressources et sert d'interface d'accès à `etcd`.
  1. Reçoit les requêtes via l'API Kubernetes et valide les opérations selon les mécanismes applicables.
  2. Lit ou modifie les données persistantes du cluster dans `etcd`.
  3. Renvoie les résultats aux clients et permet aux composants autorisés d'observer les changements via l'API.

- **`etcd`** : base de données clé-valeur distribuée qui stocke les données persistantes du cluster, notamment la configuration et l'état des ressources Kubernetes.
  1. Reçoit les opérations de lecture et d'écriture du `kube-apiserver`.
  2. Stocke les données de manière persistante et les restitue à la demande du `kube-apiserver`.

- **`kube-scheduler`** : attribue les `Pods` qui n'ont pas encore de `Node` assigné à des nœuds appropriés.
  1. Observe via le `kube-apiserver` les `Pods` en attente d'affectation.
  2. Sélectionne un `Node` approprié en tenant compte des besoins en ressources, des contraintes de placement et des règles de planification.
  3. Enregistre l'affectation du `Pod` au `Node` via l'API du `kube-apiserver`.
  4. Le `kube-apiserver` persiste cette affectation dans `etcd`.
  5. Le `kubelet` du `Node` sélectionné observe l'affectation via l'API et prend en charge l'exécution du `Pod`.

- **`kube-controller-manager`** : exécute les `Controllers` Kubernetes, qui rapprochent l'état réel du cluster de l'état désiré.
  1. Les contrôleurs observent les ressources et leur état via le `kube-apiserver`.
  2. Ils comparent l'état observé à l'état désiré défini par les ressources Kubernetes.
  3. Lorsqu'une action est nécessaire, ils créent, modifient ou suppriment des ressources via l'API du `kube-apiserver`.
  4. Le `kube-apiserver` persiste les modifications dans `etcd`.
  5. Les contrôleurs continuent de surveiller les ressources pour vérifier que l'état souhaité est atteint.
  
  Exemples : `Node Controller`, `Job Controller`, `EndpointSlice Controller`, `ServiceAccount Controller`.

- **`cloud-controller-manager` (optionnel)** : exécute les `Controllers` spécifiques au fournisseur Cloud, qui font le lien entre Kubernetes et les services d'infrastructure du fournisseur.
  1. Observe les ressources Kubernetes via le `kube-apiserver`.
  2. Interagit avec l'API du fournisseur Cloud pour effectuer les opérations nécessaires sur l'infrastructure.
  3. Met à jour les ressources Kubernetes via le `kube-apiserver` lorsque nécessaire.
  
  Exemples : gestion des `Nodes`, des routes réseau et des `Load Balancers`.

<br />

## Composants des Nodes

- **`kubelet`** : Chargé de veiller à ce que les `Pods` qui lui sont assignés fonctionnent conformément à leur spécification.
  1. Observe via le `kube-apiserver` les `Pods` qui lui sont assignés. Il peut aussi gérer des `Static Pods` définis localement.
  2. Compare l'état observé à l'état attendu décrit par le `PodSpec`.
  3. Demande au `Container runtime`, via la `Container Runtime Interface` (`CRI`), de créer, démarrer, arrêter ou redémarrer les conteneurs nécessaires.
  4. Surveille l'état des conteneurs et vérifie leur santé, notamment au moyen des probes configurées.
  5. Transmet l'état des `Pods` et du `Node` au `kube-apiserver`.

- **`kube-proxy` (optionnel)** : composant réseau qui implémente une partie du comportement des `Services`.
  1. Observe via le `kube-apiserver` les ressources nécessaires au fonctionnement des `Services`.
  2. Configure ou met à jour les règles réseau locales pour diriger le trafic vers les bons `Pods`.
  3. Maintient ces règles en fonction des changements des `Services` et de leurs points de terminaison.
  
  Si une implémentation réseau fournit un comportement équivalent pour les `Services`, `kube-proxy` n'est pas nécessaire.

- **`Container runtime`** : composant chargé de l'exécution et du cycle de vie des conteneurs, par exemple `containerd` ou `CRI-O`.
  1. Reçoit les demandes du `kubelet` via la `CRI`.
  2. Effectue les opérations demandées sur les conteneurs.
  3. Renvoie au `kubelet` les informations sur leur état.

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
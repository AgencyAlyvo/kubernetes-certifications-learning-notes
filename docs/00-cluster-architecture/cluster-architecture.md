# Architecture d'un cluster Kubernetes

Un cluster Kubernetes comprend un **Control Plane**, qui gère le cluster, et un ou plusieurs **Nodes**, qui exécutent les workloads. Pour assurer la haute disponibilité, les composants du Control Plane peuvent être déployés sur plusieurs machines.

- **Control Plane** : expose l'API du cluster, orchestre les workloads et veille à ce que l'état réel du cluster converge vers l'état désiré.
- **Nodes** : fournissent les ressources de calcul nécessaires à l'exécution des `Pods` et de leurs containers.

![Architecture d'un cluster Kubernetes](images/kubernetes-cluster-architecture.svg)

<br /><br />

---

<br /><br />

## Composants du Control Plane

- **`kube-apiserver`** : expose l'API HTTP Kubernetes et constitue le point d'entrée du Control Plane. Il reçoit les requêtes des utilisateurs et des composants du cluster, valide les opérations sur les ressources et lit ou enregistre les données persistantes dans `etcd`.
- **`etcd`** : base de données clé-valeur distribuée qui stocke les données persistantes du cluster, notamment l'état des ressources Kubernetes. Le `kube-apiserver` y accède pour lire ou modifier ces données.
- **`kube-scheduler`** : surveille les nouveaux `Pods` qui n'ont pas encore de `Node` assigné, sélectionne un `Node` approprié en tenant compte des besoins en ressources et des contraintes, puis enregistre son affectation via l'API Kubernetes, exposée par le `kube-apiserver`. Le `kubelet` du `Node` sélectionné prend ensuite en charge l'exécution du `Pod`.
- **`kube-controller-manager`** : exécute les `Controllers` Kubernetes, qui surveillent les ressources via le `kube-apiserver` et effectuent les modifications nécessaires via son API pour rapprocher l'état réel de l'état désiré (`Node Controller`, `Job Controller`, `EndpointSlice Controller`, `ServiceAccount Controller`, etc.).
- **`cloud-controller-manager` (optionnel)** : exécute les `Controllers` spécifiques au fournisseur Cloud. Ils interagissent avec l'API Kubernetes et celle du fournisseur Cloud pour gérer certaines ressources de l'infrastructure, notamment les `Nodes`, les routes réseau et les `Load Balancers`.

<br />

## Composants des Nodes

- **`kubelet`** : Il récupère les `PodSpecs` qui lui sont destinés, principalement via le `kube-apiserver`, et veille à ce que les conteneurs décrits soient en cours d'exécution et en bonne santé. Pour cela, il demande au `Container runtime` de créer, démarrer, redémarrer ou arrêter les conteneurs, puis surveille leur état. Il transmet ensuite l'état des `Pods` et du `Node` au `kube-apiserver`. Il peut également gérer des `Static Pods` définis localement, mais ne gère pas les conteneurs qui ne sont pas créés par Kubernetes.
- **`kube-proxy` (optionnel)** : composant réseau exécuté sur chaque `Node`. Il observe les ressources Kubernetes nécessaires au fonctionnement des `Services` et adapte les règles réseau locales pour permettre les communications avec les `Pods` depuis l'intérieur ou l'extérieur du cluster. Si un plugin réseau fournit un mécanisme équivalent pour les `Services`, `kube-proxy` n'est pas nécessaire.
- **`Container runtime`** : composant responsable de l'exécution et du cycle de vie des conteneurs. Il reçoit les demandes du `kubelet` pour créer, démarrer ou arrêter les conteneurs et lui fournit leur état. Kubernetes utilise notamment des runtimes compatibles avec la `Container Runtime Interface` (`CRI`), comme `containerd` et `CRI-O`.

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
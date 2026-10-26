# Architecture d'un cluster Kubernetes

Un cluster Kubernetes comprend un **Control Plane**, qui gère le cluster, et un ou plusieurs **Nodes**, qui exécutent les workloads. Pour assurer la haute disponibilité, les composants du Control Plane peuvent être déployés sur plusieurs machines.

- **Control Plane** : expose l'API du cluster, orchestre les workloads et veille à ce que l'état réel du cluster converge vers l'état désiré.
- **Nodes** : fournissent les ressources de calcul nécessaires à l'exécution des `Pods` et de leurs containers.

![Architecture d'un cluster Kubernetes](images/kubernetes-cluster-architecture.svg)

## Composants du Control Plane

- **`kube-apiserver`** : expose l'API HTTP Kubernetes et constitue le point d'entrée du Control Plane.
- **`etcd`** : base de données clé-valeur distribuée qui stocke les données de configuration et l'état persistant du cluster.
- **`kube-scheduler`** : surveille les nouveaux `Pods` qui n'ont pas encore de `Node` assigné et sélectionne un `Node` sur lequel les exécuter, en tenant compte des besoins en ressources, des contraintes matérielles, logicielles et de politique, des affinités, de la localisation des données et d'autres facteurs.
- **`kube-controller-manager`** : exécute les `Controllers` Kubernetes, qui surveillent les ressources du cluster et agissent pour rapprocher l'état réel de l'état désiré (`Node Controller`, `Job Controller`, `EndpointSlice Controller`, `ServiceAccount Controller`, etc.).
- **`cloud-controller-manager` (optionnel)** : exécute les `Controllers` spécifiques au fournisseur Cloud, qui interagissent avec son API pour gérer certaines ressources de l'infrastructure, notamment les `Nodes`, les routes réseau et les `Load Balancers`.

## Composants des Nodes

- **`kubelet`** : s'assure que les containers décrits dans les `PodSpecs` qui lui sont fournis sont en cours d'exécution et en bonne santé.
- **`kube-proxy` (optionnel)** : maintient les règles réseau sur les `Nodes` afin de permettre les communications réseau avec les `Pods` depuis l'intérieur ou l'extérieur du cluster. Si un plugin réseau fournit un comportement équivalent pour les `Services`, `kube-proxy` n'est pas nécessaire.
- `Container runtime` : le Container runtime est responsable de l'exécution et du cycle de vie des containers.

## Fonctionnement général

1. **`kube-apiserver` ↔ `etcd`** : le `kube-apiserver` lit et enregistre les données du cluster dans `etcd`, qui assure leur stockage persistant.
2. **`kube-scheduler` → `kube-apiserver`** : le `kube-scheduler` détecte les `Pods` sans `Node` assigné, sélectionne un `Node` approprié et enregistre l'affectation via l'API Kubernetes.
3. **`kubelet` ↔ `kube-apiserver`** : le `kubelet` récupère les informations des `Pods` qui lui sont assignés et transmet leur état au `kube-apiserver`.
4. **`kubelet` ↔ `Container runtime`** : le `kubelet` demande au runtime de démarrer ou d'arrêter les containers et récupère leur état.
5. **`Controllers` ↔ `kube-apiserver`** : les `Controllers` observent les ressources du cluster et les modifient via l'API Kubernetes pour rapprocher l'état réel de l'état désiré.
6. **`kube-proxy` → `kube-apiserver`** : le `kube-proxy` observe les ressources nécessaires au fonctionnement des `Services` et adapte les règles réseau sur son `Node`, lorsqu'il est utilisé.
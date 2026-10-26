# Architecture d'un cluster Kubernetes

Un cluster Kubernetes comprend un **Control Plane**, qui gère le cluster, et un ou plusieurs **Nodes**, qui exécutent les workloads. Pour assurer la haute disponibilité, les composants du Control Plane peuvent être déployés sur plusieurs machines.

- **Control Plane** : expose l'API du cluster, orchestre les workloads et veille à ce que l'état réel du cluster converge vers l'état désiré.
- **Nodes** : fournissent les ressources de calcul nécessaires à l'exécution des `Pods` et de leurs containers.

![Architecture d'un cluster Kubernetes](images/kubernetes-cluster-architecture.svg)

<br /><br />

---

<br /><br />

## Composants du Control Plane

- **`kube-apiserver`** : expose l'API HTTP Kubernetes et constitue le point d'entrée du Control Plane. Il reçoit les requêtes des composants du cluster et lit ou enregistre les données persistantes dans `etcd`.
- **`etcd`** : base de données clé-valeur distribuée qui stocke les données de configuration et l'état persistant du cluster. Le `kube-apiserver` accède à ces données pour les consulter ou les modifier.
- **`kube-scheduler`** : surveille les nouveaux `Pods` qui n'ont pas encore de `Node` assigné, sélectionne un `Node` approprié en tenant compte des besoins en ressources et des contraintes, puis transmet l'affectation au `kube-apiserver`, qui l'enregistre via l'API Kubernetes.
- **`kube-controller-manager`** : exécute les `Controllers` Kubernetes, qui surveillent les ressources via le `kube-apiserver` et effectuent les modifications nécessaires via son API pour rapprocher l'état réel de l'état désiré (`Node Controller`, `ServiceAccount Controller`, etc.).
- **`cloud-controller-manager` (optionnel)** : exécute les `Controllers` spécifiques au fournisseur Cloud, qui interagissent avec son API pour gérer certaines ressources de l'infrastructure, notamment les `Nodes`, les routes réseau et les `Load Balancers`.

<br />

## Composants des Nodes

- **`kubelet`** : s'assure que les containers décrits dans les `PodSpecs` qui lui sont fournis sont en cours d'exécution et en bonne santé. Il communique avec le `kube-apiserver` pour récupérer les informations des `Pods` qui lui sont assignés et lui transmettre leur état. Il utilise le `Container runtime` pour démarrer, arrêter et surveiller les containers.
- **`kube-proxy` (optionnel)** : observe les ressources Kubernetes nécessaires au fonctionnement des `Services`, notamment via le `kube-apiserver`, et adapte les règles réseau sur son `Node` afin de permettre les communications avec les `Pods` depuis l'intérieur ou l'extérieur du cluster. Si un plugin réseau fournit un comportement équivalent pour les `Services`, `kube-proxy` n'est pas nécessaire.
- **`Container runtime`** : est responsable de l'exécution et du cycle de vie des containers. Il reçoit les demandes du `kubelet` pour démarrer ou arrêter les containers et lui fournit leur état.
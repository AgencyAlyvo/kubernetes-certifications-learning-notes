## Architecture d'un cluster Kubernetes

Un cluster Kubernetes se compose de deux parties :

- **Control Plane** : Gère le cluster et maintient son état désiré.
- **Node** : Exécutent les **Pods** qui hébergent les applications conteneurisées.

![Architecture d'un cluster Kubernetes](images/kubernetes-cluster-architecture.svg)

### Composants du **Control Plane**

- **kube-apiserver** : Le composant qui expose **l'API HTTP de Kubernetes**.
- **etcd** : Le composant de **Base de donnée clé-valeur** qui **stocke l'état du cluster Kubernetes**.
- **kube-scheduler** : Composant qui recherche les **Pods** qui ne sont pas encore affectés à un **Node** et attribue chacun d'eux à un **Node** approprié.
- **kube-controller-manager** : Composant qui exécute les **Controllers** qui **veillent** à ce que **l'état réel du cluster corresponde à l'état souhaité**.
- **cloud-controller-manager (OPTIONNEL)** : Le composant qui assure l'intégration avec le ou les fournisseurs de **Services Cloud** sous-jacents.

### Composants du **Node**

- **kubelet** : S'assure que les **Pods** sont en **cours d'exécution**, y compris leurs **conteneurs**.
- **kube-proxy (OPTIONNEL)** : Maintient les règles réseau sur les **Nodes** afin d'assurer le fonctionnement des **Services**.
- **Container runtime** : Fournit **l'environnement d'exécution des conteneurs**.
# Controllers

## C'est quoi un controller ?

- Un `Controller` est une **boucle de contrôle** (control loop) : une boucle sans fin qui observe l'état du cluster et fait ou demande des changements pour rapprocher l'**état réel** de l'**état désiré**.
- Les `Controllers` intégrés à Kubernetes (built-in) tournent tous dans un seul composant du `Control Plane` : le **`kube-controller-manager`**.
- Les `Controllers` custom (ingress controllers, cert-manager, operators, ou les tiens) tournent **hors** du `Control Plane`, en général comme des `Pods` dans le cluster.

```text
état désiré (spec)  ──▶  Controller  ──▶  actions  ──▶  état réel
        ▲                                                  │
        └────────────────── observe (watch) ───────────────┘
```

Exemples de certains `Controllers` built-in :

| Controller | Observe | Fait en sorte que… |
|---|---|---|
| `Deployment` controller | les `Deployments` | un `ReplicaSet` existe pour la version voulue du `Pod` template (rolling update) |
| `ReplicaSet` controller | les `ReplicaSets` | le nombre de `Pods` demandé (`replicas`) tourne en permanence |
| `Job` controller | les `Jobs` | des `Pods` s'exécutent jusqu'au nombre de `completions` voulu |
| `Node` controller | les `Nodes` | un `Node` qui ne répond plus passe en `Unknown`, puis ses `Pods` sont évincés |
| `EndpointSlice` controller | les `Services` et les `Pods` | la liste des `Pods` derrière chaque `Service` est à jour |
| `ServiceAccount` controller | les `Namespaces` | chaque nouveau `namespace` a un `ServiceAccount` `default` |

---

## Comment un controller agit-il ?

Un `Controller` ne fait rien lui-même sur les `Nodes` : il **compare** l'état désiré à l'état réel, **décide** ce qu'il faut créer, modifier ou supprimer, et **envoie ces requêtes** à l'API server (`kube-apiserver`). L'API server (`kube-apiserver`) enregistre les objets dans `etcd`, et les autres composants (`kube-scheduler`, `kubelet`…) font le reste.

Exemple avec le `ReplicaSet` controller :

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web
spec:
  replicas: 3            # état désiré : 3 Pods
  selector:
    matchLabels:
      app: web
  template: ...
status:
  replicas: 2            # état réel : 2 Pods tournent (écrit par le controller)
```

Ce que fait le `ReplicaSet` controller, en boucle :

| Étape | Qui | Quoi |
|---|---|---|
| 1. Observe | `ReplicaSet` controller | `watch` les `ReplicaSets` et les `Pods` via l'API server (`kube-apiserver`) |
| 2. Compare | `ReplicaSet` controller | désiré = 3 `Pods` (`spec.replicas`) / réel = 2 `Pods` avec le label `app=web` |
| 3. Agit | `ReplicaSet` controller | envoie à l'API server la création d'1 `Pod` (comme un `kubectl create`). Il ne lance rien lui-même |
| 4. Exécute | `kube-scheduler`, `kubelet` | le scheduler choisit un `Node`, le `kubelet` de ce `Node` démarre le `Pod` |
| 5. Met à jour | `ReplicaSet` controller | écrit `status.replicas: 3` dans le `ReplicaSet` |

---

## Où tournent les controllers ?

| Type | Où | Exemples |
|---|---|---|
| **Built-in** | dans le `kube-controller-manager` (un seul processus, dans le `Control Plane`) | `Deployment`, `ReplicaSet`, `Job`, `Node`, `EndpointSlice`, `ServiceAccount`..etc |
| **Externes / Custom** | hors du `Control Plane`, en général comme des `Pods` dans le cluster | ingress controllers, cert-manager, operators, tes propres `Controllers`..etc |

→ voir [custom-controllers](../18-extending-kubernetes/custom-controllers.md)
# Owners and Dependents

- Certains objets sont **owners** d'autres objets, appelés **dependents** (exemple : un `ReplicaSet` est owner de ses `Pods`).
- Le lien est stocké dans `metadata.ownerReferences` du dependent.
- Kubernetes le remplit automatiquement pour les objets créés par `Deployment`, `ReplicaSet`, `DaemonSet`, `Job`, `CronJob`, `ReplicationController`.
- Différent des `labels` : les `labels` servent à sélectionner des groupes d'objets, les `ownerReferences` disent qui appartient à qui (utilisé pour la suppression).

---

## Exemple

Chaîne : `Deployment` → `ReplicaSet` → `Pods`.

```yaml
# kubectl get pod myapp-7d9f8b6c5d-x2k9p -o yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-7d9f8b6c5d-x2k9p
  namespace: default
  ownerReferences:                                 # owner du Pod
  - apiVersion: apps/v1
    kind: ReplicaSet
    name: myapp-7d9f8b6c5d                         # name + uid de l'owner
    uid: a1b2c3d4-5e6f-4a7b-8c9d-0e1f2a3b4c5d
    controller: true                               # cet owner est le controller du Pod
    blockOwnerDeletion: true                       # le Pod peut bloquer la suppression de son owner (foreground)
```

- `blockOwnerDeletion: true` est posé automatiquement quand un `controller` crée le dependent.
- ⚠️ Pas d'`ownerReferences` entre `namespaces` : un owner namespacé doit être dans le même `namespace` que son dependent.
- Un dependent à l'échelle du cluster ne peut avoir qu'un owner à l'échelle du cluster.

---

## Suppression en cascade

| Mode | Commande | Effet |
|---|---|---|
| background (défaut) | `kubectl delete deploy myapp` | L'owner est supprimé tout de suite, les dependents ensuite par le garbage collector |
| foreground | `kubectl delete deploy myapp --cascade=foreground` | Finalizer `foregroundDeletion` sur l'owner : les dependents avec `blockOwnerDeletion: true` sont supprimés d'abord, puis l'owner |
| orphan | `kubectl delete rs myapp-7d9f8b6c5d --cascade=orphan` | Finalizer `orphan` sur l'owner : l'owner est supprimé, les dependents sont conservés |

→ voir [finalizers](finalizers.md)

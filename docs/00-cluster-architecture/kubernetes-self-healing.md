# Kubernetes Self-Healing

## C'est quoi le self-healing ?

- Kubernetes répare tout seul les pannes courantes : il redémarre les containers qui échouent, replanifie les `Pods` quand un `Node` tombe, et maintient en permanence l'**état désiré**.
- Il n'y a pas de composant « self-healing » : ce sont les `Controllers` du `kube-controller-manager` et le `kubelet` de chaque `Node` qui, chacun dans leur domaine, comparent en boucle l'état réel à l'état désiré et corrigent l'écart.

---

## Qui répare quoi ?

| Panne | Qui répare | Comment |
|---|---|---|
| Un container crash | `kubelet` | le redémarre selon la `restartPolicy` du `Pod` |
| Un `Pod` d'un `Deployment` meurt | `ReplicaSet` controller (`kube-controller-manager`) | en recrée un pour garder le nombre de `replicas` du `ReplicaSet` créé par le `Deployment` |
| Un `Pod` d'un `StatefulSet` meurt | `StatefulSet` controller (`kube-controller-manager`) | en recrée un avec le même nom et le même volume |
| Un `Pod` d'un `DaemonSet` meurt | `DaemonSet` controller (`kube-controller-manager`) | en recrée un **sur le même `Node`** |
| Un `Node` tombe | node controller (`kube-controller-manager`), puis le controller de chaque `Pod` | le node controller évince les `Pods` du `Node` après 5 min ; les controllers (`ReplicaSet`, `StatefulSet`…) les recréent sur un autre `Node` → voir [nodes](nodes.md) |
| Le `Node` d'un `Pod` avec volume tombe | `PersistentVolume` controller (`kube-controller-manager`) | détache le volume et le rattache au nouveau `Pod` sur un autre `Node` |
| Un `Pod` derrière un `Service` n'est plus prêt | `EndpointSlice` controller (`kube-controller-manager`) | le retire des endpoints du `Service` : le trafic ne va qu'aux `Pods` sains |

---

## Ce que Kubernetes ne répare pas

- **Un bug applicatif** : le `kubelet` redémarre le container en boucle (`CrashLoopBackOff`), mais c'est à toi de corriger l'application.
- **Un volume devenu indisponible** : il faut intervenir sur le stockage.
- **Un `Pod` créé seul**, sans `Deployment` ni autre controller : s'il meurt ou si son `Node` tombe, il n'est jamais recréé.
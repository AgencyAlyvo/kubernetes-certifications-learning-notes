# Leases

## C'est quoi une Lease ?

- Une `Lease` est un objet Kubernetes (API group `coordination.k8s.io`) qui sert de **verrou avec une date d'expiration** : celui qui la détient (`holderIdentity`) doit la renouveler régulièrement (`renewTime`), sinon elle expire et quelqu'un d'autre peut la prendre.
- Ce renouvellement régulier est un **heartbeat** : « je suis toujours là ». Si la date n'est plus mise à jour, Kubernetes en déduit que le composant est tombé.
- Kubernetes s'en sert pour le heartbeat des `Nodes`, et pour l'élection du leader de `kube-controller-manager` et `kube-scheduler` en HA.

<br /><br />

---

<br /><br />

## Heartbeat des Nodes

Comment le `Control Plane` sait-il qu'un `Node` est vivant ?

1. À la création d'un `Node`, Kubernetes crée une `Lease` du **même `name`** dans le `namespace` : `kube-node-lease`.
2. Toutes les 10 s, le `kubelet` du `Node` met à jour `spec.renewTime` de sa `Lease` : c'est son **heartbeat**.
3. Le node controller (dans `kube-controller-manager`) surveille cette date. Si elle n'est plus mise à jour, le `Node` est considéré injoignable → condition `Ready` à `Unknown` → ses `Pods` sont évincés après 5 min.

```yaml
# kubectl get lease node01 -n kube-node-lease -o yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: node01                                # même name que le Node
  namespace: kube-node-lease                  # namespace réservé aux Leases des Nodes
spec:
  holderIdentity: node01                      # détenue par le kubelet de node01
  leaseDurationSeconds: 40                    # sans renouvellement pendant 40 s, la Lease est expirée
  renewTime: "2026-10-28T13:45:12.000000Z"    # dernier heartbeat du kubelet
```

```bash
# Lister les Leases des Nodes : une par Node, avec le même name
kubectl get leases -n kube-node-lease
# NAME     HOLDER   AGE
# node01   node01   10d        # HOLDER = le kubelet de node01, qui renouvelle cette Lease
# node02   node02   10d

# Voir la date du dernier heartbeat d'un Node
kubectl get lease node01 -n kube-node-lease -o jsonpath='{.spec.renewTime}'
```

<br /><br />

---

<br /><br />

## Heartbeat de kube-controller-manager et kube-scheduler en HA (élection du leader)

En HA, le `Control Plane` tourne sur plusieurs machines, donc il y a plusieurs `kube-controller-manager` et plusieurs `kube-scheduler`.

1. Chaque composant (`kube-controller-manager` et `kube-scheduler`) a une `Lease` à son nom dans le namespace `kube-system`.
2. L'instance qui réussit à prendre la `Lease` devient **leader** : elle seule agit. Les autres instances tournent mais attendent.
3. Le leader renouvelle la `Lease` en continu (son heartbeat). S'il tombe, elle expire au bout de quelques secondes, une instance en attente la prend et devient le nouveau leader.

| Composant | Ce que fait le leader | Ce que font les autres instances |
|---|---|---|
| `kube-controller-manager` | exécute tous les `Controllers` | rien, en attente |
| `kube-scheduler` | choisit un `Node` pour chaque `Pod` qui n'en a pas | rien, en attente |
| `kube-apiserver` | **pas d'élection** : toutes les instances servent en même temps, derrière un load balancer | — |

```bash
# Lister les Leases du namespace kube-system
kubectl get leases -n kube-system
# NAME                      HOLDER                                   AGE
# kube-controller-manager   cp-1_5f3a9c2e-...                        10d    # HOLDER = l'instance leader (ici sur cp-1)
# kube-scheduler            cp-2_8b1d4e7f-...                        10d    # le leader du scheduler est sur cp-2
# apiserver-07a5ea9b...     apiserver-07a5ea9b..._0c8914f7-...       10d    # identité d'un kube-apiserver, juste informatif

# Savoir quelle instance est leader pour un composant
kubectl get lease kube-scheduler -n kube-system -o jsonpath='{.spec.holderIdentity}'
```

<br /><br />

---

<br /><br />

## Résumé

- `Lease` = verrou à renouveler (`renewTime`) ; le renouvellement est un heartbeat.
- `kube-node-lease` : une `Lease` par `Node`, renouvelée par son `kubelet`. Plus de heartbeat → `Node` en `Unknown`.
- `kube-system` : une `Lease` par composant (`kube-controller-manager`, `kube-scheduler`) = qui est leader en HA ; une seule instance agit, les autres attendent. Pas d'élection pour `kube-apiserver`.

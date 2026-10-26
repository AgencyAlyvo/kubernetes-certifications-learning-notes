# Namespaces

- Isolent des groupes de ressources dans un même cluster.
- Uniquement pour les ressources namespacées (`Deployment`, `Service`…), et non pour les objets à l'échelle du cluster (`Node`, `PersistentVolume`, `StorageClass`…).
- Un `namespace` ne peut pas contenir un autre `namespace`.
- Une ressource appartient à un seul `namespace`.

**Quand les utiliser :**

- Quand plusieurs équipes ou projets partagent un cluster.
- Pour répartir les ressources du cluster entre utilisateurs via `ResourceQuota`.
- Pour séparer des versions d'une même app → `labels`, pas besoin de `namespaces`.
- En prod : éviter le `namespace` `default`.

---

## Créer un `namespace`

Le `name` doit être un **RFC 1123 DNS label**. Ne pas utiliser le préfixe `kube-` (réservé au système).

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: shop
  # label ajouté automatiquement par Kubernetes (immuable) :
  # labels:
  #   kubernetes.io/metadata.name: shop
```

Ce `label` automatique permet de cibler un `namespace` par son `name`, par ex. dans une `NetworkPolicy` :

```yaml
namespaceSelector:
  matchLabels:
    kubernetes.io/metadata.name: shop
```

---

## Mettre une ressource dans un `namespace`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: shop    # sans ce champ → namespace du contexte courant (default par défaut)
```

---

## `Namespaces` initiaux

| `Namespace` | Rôle |
|---|---|
| `default` | Pour démarrer sans créer de `namespace` |
| `kube-system` | Objets créés par le système Kubernetes |
| `kube-public` | Lisible par tous, même non authentifiés (convention, pas obligation) |
| `kube-node-lease` | Objets `Lease` des `nodes` → heartbeats du `kubelet` pour détecter une `node` en panne |

---

## Commandes utiles

```bash
kubectl create namespace shop                            # créer
kubectl get namespace                                    # lister (alias : ns)
kubectl get pods -n shop                                 # -n = --namespace
kubectl get pods -A                                      # tous les namespaces
kubectl config set-context --current --namespace=shop    # namespace par défaut du contexte
kubectl config view --minify | grep namespace:           # vérifier
kubectl api-resources --namespaced=true                  # ressources namespacées
kubectl api-resources --namespaced=false                 # ressources cluster-wide
```

---

## `Namespaces` et DNS

Chaque `Service` a une entrée DNS : `<service>.<namespace>.svc.cluster.local`.

- Même `namespace` → `<service>` suffit (la même config marche en dev / staging / prod).
- Autre `namespace` → il faut le **FQDN** (Fully Qualified Domain Name = nom DNS complet) : `<service>.<namespace>.svc.cluster.local`.

```yaml
env:
- name: DB_HOST
  value: db                               # Service "db" du même namespace
- name: CACHE_HOST
  value: redis.cache.svc.cluster.local    # FQDN : Service "redis" du namespace "cache"
```
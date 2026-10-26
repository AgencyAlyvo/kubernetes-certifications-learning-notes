# Object Names and IDs

- `Name` : unique par type de ressource (et par `namespace` pour les ressources namespacées).
- `UID` : unique dans tout le cluster, sur toute sa durée de vie.

> Un seul `Pod` `myapp-1234` par `namespace`, mais un `Pod` **et** un `Deployment` peuvent tous les deux s'appeler `myapp-1234`.

Attributs non uniques → `labels` et `annotations`.

---

## `Name`

Défini dans `metadata.name`. Deux objets peuvent avoir le même `name` s'ils sont de **types différents** ou dans des **`namespaces` différents**. Après suppression, le `name` est réutilisable.

```yaml
# ✅ Classique : une app = Deployment + Service + ConfigMap avec le même name (spec omis)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: shop
---
apiVersion: v1
kind: Service
metadata:
  name: myapp            # ✅ type différent
  namespace: shop
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp            # ✅ type différent
  namespace: shop
```

```yaml
# ✅ La même app dans 2 namespaces (ex. staging / prod)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: staging
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp            # ✅ namespace différent
  namespace: prod
```

```yaml
# Ressource cluster-scoped (pas de namespace) → name unique dans tout le cluster
apiVersion: v1
kind: Namespace
metadata:
  name: prod             # un seul Namespace "prod" dans le cluster
```

❌ Même type + même `name` + même `namespace` → `Error from server (AlreadyExists)`.

Ce qui identifie un objet :

```yaml
apiVersion: apps/v1      # groupe = apps (la version v1 ne compte pas)
kind: Deployment         # type
metadata:
  namespace: default     # namespace
  name: myapp            # name
```

---

## generateName

À la place de `name` : tu donnes un **préfixe**, Kubernetes ajoute un suffixe aléatoire. Marche avec `kubectl create`, **pas** avec `kubectl apply`.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: batch-   # → batch-x7k2p, batch-m4q9z… à chaque kubectl create
```

---

## Règles de nommage

Minuscules, chiffres et `-` ; commence par une lettre ; finit par une lettre ou un chiffre ; 63 caractères max.
Le `.` (et jusqu'à 253 caractères) est accepté pour la plupart des objets, mais **pas** pour les `Namespaces` et les `Services`.

```yaml
name: my-app-v2          # ✅
name: my.app             # ✅ Deployment, ConfigMap… / ❌ Service, Namespace
name: MyApp              # ❌ majuscules
name: my_app             # ❌ underscore
name: -myapp             # ❌ commence par un tiret
name: 1app               # ❌ commence par un chiffre
```

---

## `UID`

Généré par Kubernetes. Un objet supprimé puis recréé avec le même `name` a un **nouvel `UID`**. Les liens entre objets (`ownerReferences`) pointent vers l'`UID`.

Exemple d'un `Pod` créé par un `Deployment` (`kubectl get pod <pod> -o yaml`) :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-7d9f8b6c5d-x2k9p
  generateName: myapp-7d9f8b6c5d-              # nommé par son ReplicaSet via generateName
  uid: 3f2b8c1e-9a4d-4e6f-b1c2-7d8e9f0a1b2c    # généré par Kubernetes
  ownerReferences:
  - apiVersion: apps/v1
    kind: ReplicaSet
    name: myapp-7d9f8b6c5d
    uid: a1b2c3d4-5e6f-4a7b-8c9d-0e1f2a3b4c5d  # lien vers l'UID du ReplicaSet
```

→ voir [owner-references-and-dependents](../object-lifecycle/owner-references-and-dependents.md)
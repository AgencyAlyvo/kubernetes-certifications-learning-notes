# Object Names and UIDs

## C'est quoi un name ?

- Le nom d'un objet, choisi par l'utilisateur dans `metadata.name`.
- Unique par type de ressource (et par `namespace` pour les ressources namespacées).
- Réutilisable après la suppression de l'objet.
- Pour les attributs non uniques → `labels` et `annotations`.

```yaml
metadata:
  name: myapp    # choisi par l'utilisateur
```

---

## C'est quoi un UID ?

- Un identifiant généré par Kubernetes (format UUID).
- Unique dans tout le cluster, sur toute sa durée de vie : **jamais réutilisé**.
- Un objet supprimé puis recréé avec le même `name` a un **nouvel `UID`**.

```yaml
metadata:
  name: myapp
  uid: 3f2b8c1e-9a4d-4e6f-b1c2-7d8e9f0a1b2c    # généré par Kubernetes
```

---

## Quand deux objets peuvent-ils avoir le même name ?

- ✅ Types différents (ex. un `Pod` **et** un `Deployment` qui s'appellent `myapp`).
- ✅ `Namespaces` différents.
- ❌ Même type + même `name` + même `namespace` → `Error from server (AlreadyExists)`.

---

## Comment Kubernetes identifie un objet ?

Avec 4 attributs :

```yaml
apiVersion: apps/v1      # groupe = apps (la version v1 ne compte pas)
kind: Deployment         # type
metadata:
  namespace: default     # namespace
  name: myapp            # name
```

---

## Comment générer un name automatiquement ?

Avec `generateName` à la place de `name` : tu donnes un **préfixe**, Kubernetes ajoute un suffixe aléatoire.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: batch-   # → batch-x7k2p, batch-m4q9z…
```

- Marche avec `kubectl create`, **pas** avec `kubectl apply`.
- C'est comme ça que les `Pods` d'un `ReplicaSet` sont nommés (ex. `myapp-7d9f8b6c5d-x2k9p`).

---

## Quelles sont les règles de nommage ?

- Minuscules, chiffres et `-`.
- Commence par une lettre, finit par une lettre ou un chiffre.
- 63 caractères max.
- Le `.` (et jusqu'à 253 caractères) est accepté pour la plupart des objets, mais **pas** pour les `Namespaces` et les `Services`.

```yaml
name: my-app-v2          # ✅
name: my.app             # ✅ Deployment, ConfigMap… / ❌ Service, Namespace
name: MyApp              # ❌ majuscules
name: my_app             # ❌ underscore
name: -myapp             # ❌ commence par un tiret
name: 1app               # ❌ commence par un chiffre
```

---

## À quoi sert l'UID ?

Les liens entre objets (`ownerReferences`) pointent vers l'`UID` : Kubernetes sait ainsi exactement quel objet est l'owner, même si un autre objet a été recréé avec le même `name`.

Exemple d'un `Pod` créé par un `Deployment` (`kubectl get pod <pod> -o yaml`) :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-7d9f8b6c5d-x2k9p
  generateName: myapp-7d9f8b6c5d-              # nommé par son ReplicaSet via generateName
  uid: 3f2b8c1e-9a4d-4e6f-b1c2-7d8e9f0a1b2c    # UID du Pod, généré par Kubernetes
  ownerReferences:
  - apiVersion: apps/v1
    kind: ReplicaSet
    name: myapp-7d9f8b6c5d
    uid: a1b2c3d4-5e6f-4a7b-8c9d-0e1f2a3b4c5d  # lien vers l'UID du ReplicaSet
```

→ voir [finalizers-owner-and-dependents](../object-lifecycle/finalizers-owner-and-dependents.md)

---

## Résumé

- `name` : choisi par l'utilisateur, unique par type + `namespace`, réutilisable après suppression.
- `UID` : généré par Kubernetes, unique dans tout le cluster, jamais réutilisé.
- Même type + même `name` + même `namespace` → `AlreadyExists`.
- Identification d'un objet : groupe + type + `namespace` + `name`.
- `generateName` : préfixe + suffixe aléatoire, avec `kubectl create` uniquement.
- Règles : minuscules, chiffres, `-`, commence par une lettre, 63 caractères max (`.` et 253 max pour la plupart des objets, sauf `Namespace` et `Service`).

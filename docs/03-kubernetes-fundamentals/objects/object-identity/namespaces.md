# Namespaces

## C'est quoi un namespace ?

- Un `namespace` isole des groupes de ressources dans un même cluster.
- Il ne concerne que les ressources namespacées (`Deployment`, `Service`…), et non les objets à l'échelle du cluster (`Node`, `PersistentVolume`, `StorageClass`…).
- Un `namespace` ne peut pas contenir un autre `namespace`.
- Une ressource appartient à un seul `namespace`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: shop    # ce Deployment appartient au namespace "shop"
```

---

## Quand utiliser des namespaces ?

- Quand plusieurs équipes ou projets partagent un cluster.
- Pour répartir les ressources du cluster entre utilisateurs via `ResourceQuota`.
- Pour séparer des versions d'une même app → `labels`, pas besoin de `namespaces`.
- En prod : éviter le `namespace` `default`.

---

## Comment créer un namespace ?

1. Choisir un `name` valide : **RFC 1123 DNS label**, sans le préfixe `kube-` (réservé au système).
2. Créer le `namespace` :

   ```bash
   kubectl create namespace shop
   ```

3. Kubernetes ajoute automatiquement le `label` `kubernetes.io/metadata.name` :

   ```yaml
   # kubectl get namespace shop -o yaml
   apiVersion: v1
   kind: Namespace
   metadata:
     name: shop
     labels:
       kubernetes.io/metadata.name: shop    # ajouté automatiquement par Kubernetes (immuable)
   ```

---

## Comment mettre une ressource dans un namespace ?

1. Avec le champ `metadata.namespace` dans le YAML :

   ```yaml
   metadata:
     name: myapp
     namespace: shop
   ```

2. Ou avec `-n` dans la commande :

   ```bash
   kubectl apply -f myapp.yaml -n shop
   ```

3. Sans les deux → la ressource va dans le `namespace` du contexte courant (`default` par défaut).

⚠️ Si le YAML et `-n` indiquent deux `namespaces` différents → erreur.

---

## Comment cibler un namespace par son name ?

Avec le `label` automatique `kubernetes.io/metadata.name`, par exemple dans une `NetworkPolicy` :

```yaml
namespaceSelector:                       # sélectionne des namespaces via leurs labels
  matchLabels:
    kubernetes.io/metadata.name: shop    # → le namespace "shop"
```

---

## Quels sont les namespaces créés par défaut ?

Kubernetes crée automatiquement ces 4 `namespaces` dans **tout cluster**, dès son installation. Des outils installés ensuite (CNI, ingress controller…) peuvent en ajouter d'autres (ex. `calico-system`, `ingress-nginx`).

| `Namespace` | Rôle |
|---|---|
| `default` | Pour démarrer sans créer de `namespace` |
| `kube-system` | Objets créés par le système Kubernetes |
| `kube-public` | Lisible par tous, même non authentifiés (convention, pas obligation) |
| `kube-node-lease` | Objets `Lease` des `nodes` → heartbeats du `kubelet` pour détecter une `node` en panne |

---

## Comment joindre un Service d'un autre namespace ?

- Même `namespace` → `<service>`
- Autre `namespace` → 2 possibilités :
  - `<service>.<namespace>.svc.cluster.local` : le **FQDN** (nom DNS complet).
  - `<service>.<namespace>` : forme courte, complétée automatiquement en FQDN par le `Pod`.

```yaml
# Pod du namespace "shop"
env:
- name: DB_HOST
  value: db                               # Service "db" du namespace shop (le même)
- name: CACHE_HOST
  value: redis.cache.svc.cluster.local    # Service "redis" du namespace cache (un autre)
```

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
kubectl api-resources --namespaced=false                 # ressources à l'échelle du cluster
```

---

## Résumé

- `namespace` = isolation de ressources namespacées dans un cluster. Pas d'imbrication, une ressource = un seul `namespace`.
- Créer : `kubectl create namespace shop`. `name` en RFC 1123 DNS label, pas de préfixe `kube-`.
- Sans `metadata.namespace` ni `-n` → `namespace` du contexte courant (`default` par défaut).
- 4 `namespaces` créés par défaut dans tout cluster : `default`, `kube-system`, `kube-public`, `kube-node-lease`.
- DNS : `<service>` dans le même `namespace`, `<service>.<namespace>.svc.cluster.local` sinon.
- `label` automatique `kubernetes.io/metadata.name` → cibler un `namespace` par son `name`.

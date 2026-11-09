# Kubernetes API

## C'est quoi l'API Kubernetes ?

- Une **API HTTP** exposée par le `kube-apiserver`, le cœur du control plane.
- Elle sert à lire et modifier l'état des objets du cluster (`Pods`, `Namespaces`, `ConfigMaps`…), stocké dans `etcd`.
- **Tout** passe par elle : `kubectl`, les composants du cluster (`kubelet`, `kube-scheduler`, `controllers`) et les clients externes.

```text
kubectl get pods  →  GET /api/v1/namespaces/default/pods  →  kube-apiserver  →  etcd
```

---

## C'est quoi une ressource et un objet ?

| Terme | Exemple | Où on le voit |
|---|---|---|
| **Ressource** : un type d'objet exposé par l'API | `pods` | dans l'URL `/api/v1/namespaces/default/pods` et dans `kubectl get pods` |
| **Objet** : une instance d'une ressource, stockée dans `etcd` | le `Pod` `nginx` | dans l'URL `/api/v1/namespaces/default/pods/nginx` et dans `kubectl get pod nginx` |

```bash
kubectl api-resources                         # liste toutes les ressources du cluster
# NAME          SHORTNAMES   APIVERSION   NAMESPACED   KIND
# pods          po           v1           true         Pod
# deployments   deploy       apps/v1      true         Deployment
# nodes         no           v1           false        Node
```

---

## Comment l'API est-elle organisée ? (groupes et versions)

Chaque ressource appartient à un **API group**, servi dans une **version** :

| Groupe | `apiVersion` | Chemin | Exemples |
|---|---|---|---|
| **core** (sans nom) | `v1` | `/api/v1` | `Pod`, `Service`, `ConfigMap`, `Secret`, `Namespace`, `Node` |
| **nommé** | `<group>/<version>` | `/apis/<group>/<version>` | `apps/v1` → `Deployment`, `batch/v1` → `Job`, `networking.k8s.io/v1` → `Ingress` |

```yaml
apiVersion: v1          # groupe core → /api/v1/namespaces/default/pods
kind: Pod
---
apiVersion: apps/v1     # groupe apps, version v1 → /apis/apps/v1/namespaces/default/deployments
kind: Deployment
```

---

## Comment fonctionnent les versions d'API ?

- Les versions d'une ressource (`v1beta1`, `v1`…) sont des **représentations des mêmes données** : l'API server (`kube-apiserver`) convertit automatiquement. Un objet créé en `v1beta1` peut être lu ou modifié en `v1`, jusqu'au retrait de `v1beta1`.

| Niveau | `apiVersion` | Stabilité |
|---|---|---|
| alpha | `v1alpha1` | Expérimental, peut casser à un upgrade |
| beta | `v1beta1` | Testé, mais il faudra migrer quand l'API passe stable |
| stable (GA) | `v1` | Compatibilité garantie |

---

## Comment étendre l'API ?

- `CustomResourceDefinition` : ajouter un nouveau type de ressource, servi par le `kube-apiserver`.
- Aggregation layer : brancher un API server supplémentaire (exemple : `metrics-server` pour `metrics.k8s.io`).

→ voir [CustomResourceDefinition](../../18-extending-kubernetes/custom-resource-definitions-crd.md)

---

## Comment explorer l'API ?

```bash
kubectl api-resources                           # ressources (name, shortnames, apiVersion, namespaced, kind)
kubectl api-versions                            # groupes / versions servis par le cluster
kubectl explain pod.spec.containers             # champs d'une ressource (schéma OpenAPI)
kubectl explain deployment --recursive          # tous les champs
kubectl apply -f pod.yaml --dry-run=server      # validation complète par l'API server (kube-apiserver), sans créer l'objet
```

---

## Résumé

- API HTTP du `kube-apiserver`, tout passe par elle, état stocké dans `etcd`.
- Ressource = type (`pods`), objet = instance (`Pod` `nginx`), kind = nom dans le YAML (`Pod`).
- Groupe core → `apiVersion: v1`. Groupes nommés → `apiVersion: <group>/<version>` (ex. `apps/v1`).
- Versions `v1alpha1` → `v1beta1` → `v1` (GA, compatibilité garantie), toutes converties par l'API server.
- Explorer : `kubectl api-resources`, `api-versions`, `explain`. Valider : `--dry-run=server`.

# Finalizers

## C'est quoi un finalizer ?

- Une clé dans `metadata.finalizers` qui **bloque la suppression** d'un objet tant qu'un `controller` n'a pas fait son nettoyage.
- Sert à nettoyer des ressources liées avant la suppression, ou à empêcher une suppression accidentelle.
- Ajouté automatiquement par Kubernetes, ou défini par soi-même.
- Format : `<prefix>/<name>` (ex. `example.com/cleanup-bucket`), obligatoire pour un `finalizer` personnalisé.

```yaml
metadata:
  finalizers:
  - kubernetes.io/pv-protection    # nom avec un prefix, comme une key de label
```

---

## Comment fonctionnent les finalizers ?

1. `kubectl delete` → l'API server remplit `metadata.deletionTimestamp` et renvoie `202` (Accepted).
2. L'objet passe en `Terminating` : il n'est pas supprimé tant que `metadata.finalizers` n'est pas vide.
3. Le `controller` détecte le `deletionTimestamp` et fait son nettoyage.
4. Le `controller` retire sa clé de `metadata.finalizers`.
5. `finalizers` vide → l'objet est supprimé.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data
  deletionTimestamp: "2026-10-27T10:00:00Z"    # étape 1 : rempli par l'API server → Terminating
  finalizers:
  - kubernetes.io/pv-protection                # étapes 3-4 : retiré par le controller quand c'est OK
```

---

## Exemple : comment Kubernetes protège les volumes ?

Le mécanisme est le même pour **n'importe quel objet** qui a un `finalizer` (`Namespace`, ressource custom gérée par un operator…). Ici, avec un `PersistentVolume` :

1. Un `Pod` utilise un `PersistentVolume` → Kubernetes ajoute le `finalizer` `kubernetes.io/pv-protection`.
2. On supprime le `PersistentVolume` → il passe en `Terminating`.
3. Le `Pod` n'utilise plus le volume → Kubernetes retire le `finalizer`.
4. Le `PersistentVolume` est supprimé.

Même principe avec `kubernetes.io/pvc-protection` sur les `PersistentVolumeClaims`.

---

## Comment ça marche avec un owner reference et ses dependents ?

- **Owner Reference** : l'objet parent (ex. un `Deployment`).
- **Dependents** : les objets qui lui appartiennent (ex. son `ReplicaSet` et ses `Pods`). Chacun pointe vers son owner via le champ `metadata.ownerReferences`.

Exemple : `kubectl delete deploy myapp --cascade=foreground`

1. Kubernetes ajoute automatiquement le `finalizer` `foregroundDeletion` sur le `Deployment` → `Terminating`.
2. Kubernetes supprime les dependents (`ReplicaSet` puis `Pods`), trouvés grâce à leurs `ownerReferences`.
3. Les dependents sont supprimés → Kubernetes retire `foregroundDeletion`.
4. Le `Deployment` est supprimé.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  deletionTimestamp: "2026-10-27T10:00:00Z"    # suppression demandée → Terminating
  finalizers:
  - foregroundDeletion                         # ajouté auto : attend la suppression des dependents
```

⚠️ Si un dependent garde un `finalizer`, l'étape 2 ne se termine jamais → l'owner reste en `Terminating`.

→ voir [owner-references-and-dependents](owner-references-and-dependents.md)

---

## Comment débloquer un objet en `Terminating` ?

1. Voir ses `finalizers` :

   ```bash
   kubectl get pvc data -o jsonpath='{.metadata.finalizers}'
   ```

2. Trouver ce qui bloque, selon le cas :

   - **Volume** (`PersistentVolumeClaim`, `PersistentVolume`) → un `Pod` utilise encore le volume :

     ```bash
     kubectl describe pvc data    # ligne "Used By" : Pods qui utilisent encore le PVC
     ```

   - **Owner** (ex. `Deployment` supprimé en foreground) → un de ses dependents (les objets qui lui appartiennent : `ReplicaSets`, `Pods`) n'arrive pas à être supprimé :

     ```bash
     kubectl get pods -l app=myapp                                   # des Pods du Deployment sont-ils encore là ?
     kubectl get pod <pod> -o jsonpath='{.metadata.finalizers}'      # ont-ils un finalizer qui bloque ?
     ```

3. Régler la cause → le `controller` retire le `finalizer` tout seul.
4. Dernier recours, retirer les `finalizers` à la main :

   ```bash
   kubectl patch pvc data --type=merge -p '{"metadata":{"finalizers":null}}'
   ```

---

## Finalizers courants

| Finalizer | Rôle |
|---|---|
| `kubernetes.io/pvc-protection` | Le `PersistentVolumeClaim` n'est pas supprimé tant qu'un `Pod` l'utilise |
| `kubernetes.io/pv-protection` | Le `PersistentVolume` n'est pas supprimé tant qu'il est utilisé |
| `foregroundDeletion` | L'owner attend la suppression de ses dependents |
| `orphan` | L'owner est supprimé, ses dependents sont conservés |

---

## Résumé

- `finalizers` = clés dans `metadata.finalizers` qui bloquent la suppression tant qu'un `controller` n'a pas fait son nettoyage.
- Cycle : `kubectl delete` → `deletionTimestamp` + `202` → `Terminating` → le `controller` retire sa clé → objet supprimé.
- Après le `DELETE` : on peut retirer des `finalizers`, pas en ajouter. Le `deletionTimestamp` n'est plus modifiable. La suppression est irréversible.
- `finalizer` personnalisé : prefix obligatoire (ex. `example.com/finalizer-name`), sinon l'API server rejette l'écriture.
- Objet bloqué en `Terminating` → vérifier `finalizers` et `ownerReferences`.

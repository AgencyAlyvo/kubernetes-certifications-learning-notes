# Finalizers, Owners et Dependents

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

## Finalizers courants

| Finalizer | Rôle |
|---|---|
| `kubernetes.io/pvc-protection` | Le `PersistentVolumeClaim` n'est pas supprimé tant qu'un `Pod` l'utilise |
| `kubernetes.io/pv-protection` | Le `PersistentVolume` n'est pas supprimé tant qu'il est utilisé |
| `foregroundDeletion` | L'owner attend la suppression de ses dependents |
| `orphan` | L'owner est supprimé, ses dependents sont conservés |

---

## Exemple : comment Kubernetes protège les volumes ?

Le mécanisme est le même pour **n'importe quel objet** qui a un `finalizer` (`Namespace`, ressource custom gérée par un operator…). Ici, avec un `PersistentVolume` :

1. Un `Pod` utilise un `PersistentVolume` → Kubernetes ajoute le `finalizer` `kubernetes.io/pv-protection`.
2. On supprime le `PersistentVolume` → il passe en `Terminating`.
3. Le `Pod` n'utilise plus le volume → Kubernetes retire le `finalizer`.
4. Le `PersistentVolume` est supprimé.

Même principe avec `kubernetes.io/pvc-protection` sur les `PersistentVolumeClaims`.

<br /><br />

---

<br /><br />

## C'est quoi un owner et un dependent ?

- **Owner** : un objet qui possède d'autres objets (exemple : un `Deployment` possède son `ReplicaSet`, un `ReplicaSet` possède ses `Pods`).
- **Dependent** : un objet qui appartient à un owner (exemple : les `Pods` d'un `ReplicaSet`).
- Quand on supprime un owner, ses dependents sont supprimés aussi (par défaut).

Le lien est posé **sur le dependent**, dans `metadata.ownerReferences` :

```yaml
# Pod créé par un ReplicaSet (kubectl get pod <pod> -o yaml)
apiVersion: v1
kind: Pod
metadata:
  name: myapp-7d9f8b6c5d-x2k9p
  ownerReferences:                                 # posé automatiquement par le controller du ReplicaSet
  - apiVersion: apps/v1
    kind: ReplicaSet
    name: myapp-7d9f8b6c5d                         # name de l'owner
    uid: a1b2c3d4-5e6f-4a7b-8c9d-0e1f2a3b4c5d      # UID de l'owner
    controller: true                               # cet owner est le controller du Pod
    blockOwnerDeletion: true                       # ce Pod peut bloquer la suppression de son owner (foreground)
```

**`ownerReferences`**

- Pointe vers l'owner avec son `name` et son `UID`.
- Rempli **automatiquement** pour les objets créés par les `ReplicaSets`, `DaemonSets`, `Deployments`, `Jobs`, et `CronJobs`.
- Modifiable à la main, mais en pratique on n'en a pas besoin.

**`blockOwnerDeletion`**

- Booléen sur le dependent : `true` → ce dependent peut bloquer la suppression de son owner, en suppression foreground.
- Mis **automatiquement** à `true` quand c'est un `controller` qui pose l'`ownerReference` (exemple : le `controller` de `Deployment`).

---

## Comment ça marche avec un owner et ses dependents ?

Exemple : `kubectl delete deploy myapp --cascade=foreground`

1. Kubernetes ajoute automatiquement le `finalizer` `foregroundDeletion` sur le `Deployment` → `Terminating`.
2. Kubernetes supprime les dependents (`ReplicaSet` puis `Pods`), trouvés grâce à leurs `ownerReferences`, et attend ceux qui ont `blockOwnerDeletion: true`.
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

## Résumé

- `finalizers` = clés dans `metadata.finalizers` qui bloquent la suppression tant qu'un `controller` n'a pas fait son nettoyage.
- Cycle : `kubectl delete` → `deletionTimestamp` + `202` → `Terminating` → le `controller` retire sa clé → objet supprimé.
- Après le `DELETE` : on peut retirer des `finalizers`, pas en ajouter. Le `deletionTimestamp` n'est plus modifiable. La suppression est irréversible.
- `finalizer` personnalisé : prefix obligatoire (ex. `example.com/finalizer-name`), sinon l'API server rejette l'écriture.
- Owner = objet qui possède d'autres objets, dependent = objet qui lui appartient (lien via `metadata.ownerReferences` sur le dependent, posé automatiquement).
- Suppression d'un owner → ses dependents aussi. `--cascade=foreground` → `finalizer` `foregroundDeletion` (dependents d'abord), `--cascade=orphan` → dependents conservés.
- Objet bloqué en `Terminating` → vérifier `finalizers` et `ownerReferences`.

# Label selectors

## C'est quoi un label selector ?

- Un `label selector` identifie un **ensemble d'objets** via leurs `labels`.
- Contrairement aux `names` et aux `UIDs`, les `labels` ne sont pas uniques : plusieurs objets portent les mêmes `labels`, et le `selector` sert à les regrouper. C'est le mécanisme de regroupement de base de Kubernetes.
- 2 types : **equality-based** et **set-based**.
- Pour combiner plusieurs conditions, on les sépare par `,` : un objet est sélectionné seulement s'il remplit **toutes** les conditions (ET logique).
- ⚠️ Pas d'opérateur **OU logique**, sauf sur les `values` avec `in`.

| Type | Opérateurs | Exemple |
|---|---|---|
| equality-based | `=`, `==`, `!=` | `environment=production,tier!=frontend` |
| set-based | `in`, `notin`, `exists` (la `key` seule) | `environment in (production, qa)` |

---

## Comment fonctionne un selector equality-based ?

```text
environment = production                 # a la key environment avec la value production
tier != frontend                         # value ≠ frontend, OU n'a pas la key tier
environment=production,tier!=frontend    # production ET pas frontend
```

- `=` et `==` sont synonymes.
- ⚠️ `!=` sélectionne aussi les objets qui **n'ont pas** la `key`.
- Un objet qui a des `labels` en plus est quand même sélectionné.

---

## Comment fonctionne un selector set-based ?

```text
environment in (production, qa)     # value = production OU qa
tier notin (frontend, backend)      # value ≠ frontend ET ≠ backend, OU n'a pas la key tier
partition                           # a la key partition (value ignorée)
!partition                          # n'a pas la key partition
partition,environment notin (qa)    # a la key partition ET environment ≠ qa
```

- Plus expressif que equality-based : `environment=production` ≡ `environment in (production)`, et `!=` ≡ `notin`.
- Mixable avec equality-based : `partition in (customerA, customerB),environment!=qa`.

---

## Comment sélectionner avec kubectl ?

Option `-l` (= `--selector`), avec les deux types :

```bash
kubectl get pods -l environment=production,tier=frontend           # equality-based
kubectl get pods -l 'environment in (production, qa)'              # set-based → entre quotes (parenthèses)
kubectl get pods -l 'environment,environment notin (frontend)'     # a la key ET value ≠ frontend
kubectl get pods -l app=guestbook,role=replica -L tier             # -L affiche un label en colonne
kubectl label pods -l app=nginx tier=fe                            # -l marche aussi avec label, delete, logs…
kubectl delete pods -l app=nginx
```

Côté API, le `selector` passe en query parameter : `?labelSelector=environment%3Dproduction,tier%3Dfrontend`.

---

## Comment un objet sélectionne-t-il d'autres objets ?

Certains objets ont un champ `selector` dans leur YAML pour désigner les objets qu'ils gèrent (ex. les `Pods` d'un `Service` ou d'un `Deployment`).

### `Service` → equality-based uniquement

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  selector:              # Pods ciblés par le Service (map simple)
    component: redis     # ≡ component=redis
  ports:
  - port: 6379
```

### `Deployment`, `ReplicaSet`, `Job`, `DaemonSet` → set-based aussi

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  selector:
    matchLabels:                # map {key: value} ≡ { key: component, operator: In, values: [redis] }
      component: redis
    matchExpressions:           # liste de conditions
    - { key: tier, operator: In, values: [cache] }
    - { key: environment, operator: NotIn, values: [dev] }
  template:
    metadata:
      labels:                   # doivent satisfaire le selector
        component: redis
        tier: cache
        environment: production
    spec:
      containers:
      - name: redis
        image: redis
```

- Opérateurs de `matchExpressions` : `In`, `NotIn`, `Exists`, `DoesNotExist`.
- ⚠️ En YAML : `In` / `NotIn` avec majuscule. Avec `kubectl -l` : `in` / `notin` en minuscules.
- `values` obligatoire et non vide pour `In` et `NotIn`.
- Un `Pod` est sélectionné seulement s'il remplit **toutes** les conditions de `matchLabels` **et** de `matchExpressions` (ET logique).
- ⚠️ Les `labels` du `template` doivent satisfaire le `selector`, sinon erreur à la création (`selector does not match template labels`).
- ⚠️ Deux `ReplicaSets` d'un même `namespace` ne doivent pas avoir des `selectors` qui se chevauchent, sinon le `controller` ne sait plus combien de replicas garder.

### `nodeSelector` → choisir les `nodes` d'un `Pod`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cuda-test
spec:
  nodeSelector:                       # equality-based : le Pod va sur une node qui a ce label
    accelerator: nvidia-tesla-p100
  containers:
  - name: cuda-test
    image: "registry.k8s.io/cuda-vector-add:v0.1"
```

→ voir [node-selector](../../../05-scheduling/nodes/node-selector.md)

---

## Label selector ou field selector ?

| | `label selector` (`-l`) | `field selector` (`--field-selector`) |
|---|---|---|
| Sélectionne sur | les `labels` (`metadata.labels`) | les champs de l'objet (`metadata.name`, `spec.*`, `status.*`) |
| Opérateurs | `=`, `!=`, `in`, `notin`, `exists` | `=`, `!=` |
| Champs possibles | n'importe quel `label` | liste limitée, selon le type |
| Dans le YAML (`selector`) | ✅ | ❌ (`kubectl` / API seulement) |

→ voir [field-selectors](field-selectors.md)

---

## Résumé

- `label selector` = sélectionne un ensemble d'objets via leurs `labels`. `,` = ET, pas de OU.
- equality-based : `=`, `==`, `!=`. set-based : `in`, `notin`, `key`, `!key` (plus expressif).
- `!=` et `notin` sélectionnent aussi les objets sans la `key`.
- `kubectl get pods -l key=value`, set-based entre quotes : `-l 'key in (a, b)'`.
- `Service` → `selector` map simple (equality). `Deployment`, `ReplicaSet`, `Job`, `DaemonSet` → `matchLabels` + `matchExpressions` (`In`, `NotIn`, `Exists`, `DoesNotExist`).
- Les `labels` du `template` doivent satisfaire le `selector`.

# Label selectors

## C'est quoi un label selector ?

- Un `label selector` sélectionne des objets selon leurs `labels`.
- Les `labels` ne sont pas uniques : plusieurs objets portent les mêmes, et le `selector` sert à les regrouper.
- Utilisable avec `kubectl` (`-l`) **et** dans le YAML des objets (`selector`), par exemple un `Service` qui cible ses `Pods`.

```bash
kubectl get pods -l app=nginx    # Pods qui ont le label app=nginx
```

---

## Quels opérateurs ?

- equality-based : `=`, `==` (synonymes) et `!=`.
- set-based : `in`, `notin`, et la `key` seule (`partition`, `!partition`).
- Pour combiner plusieurs conditions, on les sépare par `,` : un objet est sélectionné seulement s'il remplit **toutes** les conditions (ET logique). Les deux types se mélangent.
- ⚠️ Pas de OU logique entre conditions. Le seul OU possible est sur les `values`, avec `in`.
- ⚠️ `!=` et `notin` sélectionnent aussi les objets qui **n'ont pas** la `key`.

```bash
kubectl get pods -l environment=production                                  # key environment avec la value production
kubectl get pods -l tier!=frontend                                          # tier ≠ frontend, OU pas de key tier
kubectl get pods -l environment=production,tier!=frontend                   # production ET pas frontend
kubectl get pods -l 'environment in (production, qa)'                       # environment = production OU qa (set-based → entre quotes)
kubectl get pods -l 'tier notin (frontend, backend)'                        # tier ≠ frontend et ≠ backend, OU pas de key tier
kubectl get pods -l partition                                               # a la key partition (value ignorée)
kubectl get pods -l '!partition'                                            # n'a pas la key partition
kubectl get pods -l 'partition in (customerA, customerB),environment!=qa'   # set-based + equality-based
```

---


## Comment un objet sélectionne-t-il ses Pods ?

Certains objets ont un champ `selector` dans leur YAML pour désigner les `Pods` qu'ils gèrent.

| Objet | Champ | Opérateurs |
|---|---|---|
| `Service` | `spec.selector` (map simple) | equality-based uniquement |
| `Deployment`, `ReplicaSet`, `DaemonSet`, `Job` | `spec.selector.matchLabels` + `matchExpressions` | set-based aussi : `In`, `NotIn`, `Exists`, `DoesNotExist` |
| `Pod` | `spec.nodeSelector` (map simple) | equality-based, pour choisir une `node` → voir [node-selector](../../../05-scheduling/nodes/node-selector.md) |

```yaml
# Service → map simple
spec:
  selector:
    component: redis          # ≡ component=redis
```

```yaml
# Deployment → matchLabels + matchExpressions
spec:
  selector:
    matchLabels:
      component: redis        # ≡ { key: component, operator: In, values: [redis] }
    matchExpressions:
    - { key: tier, operator: In, values: [cache] }
    - { key: environment, operator: NotIn, values: [dev] }
  template:
    metadata:
      labels:                 # doivent satisfaire le selector
        component: redis
        tier: cache
        environment: production
```

- Un `Pod` est sélectionné seulement s'il remplit **toutes** les conditions de `matchLabels` **et** de `matchExpressions` (ET logique).
- ⚠️ En YAML : `In` / `NotIn` avec majuscule. Avec `kubectl -l` : `in` / `notin` en minuscules.
- ⚠️ Les `labels` du `template` doivent satisfaire le `selector`, sinon erreur à la création (`selector does not match template labels`).
- `values` obligatoire et non vide pour `In` et `NotIn`.

---

## Exemples utiles pour l'examen

```bash
kubectl get pods -l app=guestbook -L tier                   # -L affiche un label en colonne
kubectl get pods -A -l app=nginx                            # dans tous les namespaces
kubectl delete pods -l app=nginx                            # -l marche avec get, delete, label, logs…
kubectl label pods -l app=nginx tier=fe                     # ajouter tier=fe à tous les Pods app=nginx
kubectl logs -l app=nginx                                   # logs de tous les Pods app=nginx
kubectl get pods -l app=nginx --field-selector status.phase=Running   # combinable avec --field-selector
```

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

- `label selector` = sélectionne des objets via leurs `labels`. `,` = ET, pas de OU entre conditions.
- equality-based : `=`, `!=`. set-based : `in`, `notin`, `key`, `!key`. `!=` et `notin` prennent aussi les objets sans la `key`.
- `kubectl get pods -l key=value`, set-based entre quotes : `-l 'key in (a, b)'`.
- `Service` → `selector` map simple. `Deployment`, `ReplicaSet`, `DaemonSet`, `Job` → `matchLabels` + `matchExpressions` (`In`, `NotIn`, `Exists`, `DoesNotExist`).
- Les `labels` du `template` doivent satisfaire le `selector`.

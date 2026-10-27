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

| Type | Opérateur | Exemple | Sélectionne les objets qui… |
|---|---|---|---|
| equality-based | `=` (ou `==`) | `environment=production` | ont la `key` `environment` avec la `value` `production` |
| equality-based | `!=` | `tier!=frontend` | ont `tier` ≠ `frontend`, **ou n'ont pas** la `key` `tier` |
| set-based | `in` | `environment in (production, qa)` | ont `environment` = `production` **ou** `qa` |
| set-based | `notin` | `tier notin (frontend, backend)` | ont `tier` ≠ `frontend` et ≠ `backend`, **ou n'ont pas** la `key` `tier` |
| set-based | `key` | `partition` | ont la `key` `partition` (`value` ignorée) |
| set-based | `!key` | `!partition` | n'ont pas la `key` `partition` |

- Pour combiner plusieurs conditions, on les sépare par `,` : un objet est sélectionné seulement s'il remplit **toutes** les conditions (ET logique). Les deux types se mélangent : `partition in (customerA, customerB),environment!=qa`.
- ⚠️ Pas de OU logique entre conditions. Le seul OU possible est sur les `values`, avec `in`.
- ⚠️ `!=` et `notin` sélectionnent aussi les objets qui **n'ont pas** la `key`.

---

## Comment sélectionner avec kubectl ?

```bash
kubectl get pods -l environment=production,tier=frontend    # equality-based, 2 conditions
kubectl get pods -l 'environment in (production, qa)'       # set-based → entre quotes (parenthèses)
kubectl get pods -l '!partition'                            # sans la key partition
kubectl get pods -l app=guestbook -L tier                   # -L affiche un label en colonne
kubectl delete pods -l app=nginx                            # -l marche avec get, delete, label, logs…
kubectl label pods -l app=nginx tier=fe                     # ajouter tier=fe à tous les Pods app=nginx
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

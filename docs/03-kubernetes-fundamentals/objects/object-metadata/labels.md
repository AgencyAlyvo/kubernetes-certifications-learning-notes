# Labels

## C'est quoi un label ?

- Un `label` est une paire `key`/`value` attachée à un objet (exemple : `Pod`).
- Les `labels` servent à **organiser** et à **sélectionner** des objets.
- Ils peuvent être définis à la création de l'objet, puis ajoutés ou modifiés à tout moment.
- Chaque `key` doit être unique pour un même objet.
- Les `labels` ne sont pas uniques : plusieurs objets peuvent porter les mêmes `labels`.
- Pour les informations non identifiantes → `annotations`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: label-demo
  labels:
    environment: production    # key: environment / value: production
    app: nginx                 # key: app / value: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
```

---

## Comment est composée une key ?

`<prefix>/<name>` : le `prefix` est optionnel, le `name` est obligatoire.

```yaml
labels:
  app: nginx                    # name seul (pas de prefix) → key privée à l'utilisateur
  example.com/team: payments    # prefix example.com + name team
```

- `name` : 63 caractères max, commence et finit par `[a-z0-9A-Z]`, `-` `_` `.` autorisés entre.
- `value` : mêmes règles que `name`, peut être vide.
- `prefix` : DNS subdomain (ex. `example.com`), 253 caractères max.
- Les prefix `kubernetes.io/` et `k8s.io/` sont réservés aux composants Kubernetes.

---

## Quels labels utiliser ? (recommended labels)

- Kubernetes recommande un jeu de `labels` communs, avec le prefix `app.kubernetes.io/`, pour que tous les outils (Helm, dashboards…) décrivent les applications de la même façon.
- Recommandés, pas obligatoires. À appliquer sur **chaque objet** d'une application.

| `Label` | Description | Exemple |
|---|---|---|
| `app.kubernetes.io/name` | Nom de l'application | `mysql` |
| `app.kubernetes.io/instance` | Nom unique de l'instance de l'application | `mysql-abcxyz` |
| `app.kubernetes.io/version` | Version de l'application (SemVer, hash…) | `5.7.21` |
| `app.kubernetes.io/component` | Composant dans l'architecture | `database` |
| `app.kubernetes.io/part-of` | Application de plus haut niveau dont celle-ci fait partie | `wordpress` |
| `app.kubernetes.io/managed-by` | Outil qui gère l'application | `Helm` |

Exemple : WordPress (web) + MySQL (base de données), installés avec Helm :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: wordpress
    app.kubernetes.io/instance: wordpress-abcxyz
    app.kubernetes.io/version: "4.9.4"
    app.kubernetes.io/component: server
    app.kubernetes.io/part-of: wordpress
    app.kubernetes.io/managed-by: Helm
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app.kubernetes.io/name: mysql
    app.kubernetes.io/instance: mysql-abcxyz
    app.kubernetes.io/version: "5.7.21"
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: wordpress      # MySQL fait partie de l'application WordPress
    app.kubernetes.io/managed-by: Helm
```

**`name` vs `instance`** : une application peut être installée plusieurs fois dans un cluster (ex. 2 sites WordPress). Toutes les installations ont le même `name` (`wordpress`), mais chaque instance a un `instance` unique (`wordpress-abcxyz`, `wordpress-defuvw`).

---

## Commandes utiles

```bash
kubectl get pods --show-labels                 # afficher les labels
kubectl get pods -L app -L tier                # afficher des labels en colonnes (-L = --label-columns)
kubectl label pod mypod tier=fe                # ajouter un label
kubectl label pod mypod tier=be --overwrite    # modifier un label existant (sinon erreur)
kubectl label pod mypod tier-                  # supprimer un label
```

Sélectionner des objets par leurs `labels` (`-l`, `selector`) → voir [label-selectors](../object-selection/label-selectors.md).

---

## Résumé

- `label` = paire `key`/`value` sur un objet, pour organiser et sélectionner. Pas unique entre objets.
- `key` = `<prefix>/<name>`, prefix optionnel. `kubernetes.io/` et `k8s.io/` réservés.
- Recommended labels : prefix `app.kubernetes.io/` (`name`, `instance`, `version`, `component`, `part-of`, `managed-by`), pour que les outils se comprennent.
- `kubectl label pod mypod key=value` / `--overwrite` / `key-`.

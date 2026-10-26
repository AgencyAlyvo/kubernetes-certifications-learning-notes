# Labels

- Un `label` est une paire `key`/`value` attachée à un objet (exemple : `Pod`).
- Les `labels` servent à **organiser** et à **sélectionner** des sous-ensembles d'objets.
- Ils peuvent être définis à la création de l'objet, puis ajoutés ou modifiés à tout moment.
- Chaque `key` doit être unique pour un même objet.
- Les `labels` ne sont pas uniques : plusieurs objets peuvent porter les mêmes `labels`.
- Pour les informations non identifiantes → `annotations`.

---

## Exemple

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: label-demo
  labels:                      # labels du Pod
    environment: production    # key: environment / value: production
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
```

---

## Syntaxe

Une `key` est composée de 2 parties séparées par `/` :

- `prefix` : optionnel.
- `name` : obligatoire.

```yaml
labels:
  app: nginx
  # key   = app        → name seul, pas de prefix
  # value = nginx

  example.com/team: payments
  # key   = example.com/team
  #   prefix = example.com
  #   name   = team
  # value = payments
```

| Partie | Obligatoire | Règles |
|---|---|---|
| `prefix` | Non | DNS subdomain, 253 caractères max, suivi d'un `/` |
| `name` | Oui | 63 caractères max, commence et finit par un alphanumérique `[a-z0-9A-Z]`, `-` `_` `.` autorisés entre |
| `value` | Non (peut être vide) | 63 caractères max, si non vide : commence et finit par un alphanumérique `[a-z0-9A-Z]`, `-` `_` `.` autorisés entre |

- Majuscules et `_` autorisés, contrairement aux `names` d'objets.
- Sans `prefix`, la `key` est considérée comme privée à l'utilisateur.
- Les composants automatisés (`kube-scheduler`, `kube-controller-manager`, `kube-apiserver`, `kubectl`, outils tiers) qui ajoutent des `labels` doivent utiliser un `prefix`.
- Les prefix `kubernetes.io/` et `k8s.io/` sont réservés aux composants core de Kubernetes.

---

## Utiliser plusieurs labels

Un seul `label` ne suffit souvent pas : on en combine plusieurs pour distinguer les ensembles de ressources.

```yaml
# frontend
labels:
  app: guestbook
  app.kubernetes.io/name: guestbook    # recommended label → voir recommended-labels
  tier: frontend
---
# Redis master
labels:
  app: guestbook
  app.kubernetes.io/name: guestbook
  tier: backend
  role: master
---
# Redis replica
labels:
  app: guestbook
  app.kubernetes.io/name: guestbook
  tier: backend
  role: replica
```

- `app` → pratique pour les requêtes manuelles et la CLI.
- `app.kubernetes.io/name` → convention recommandée, adaptée à l'outillage et à l'automatisation ([recommended-labels](recommended-labels.md)).

---

## Commandes utiles

```bash
kubectl get pods --show-labels                 # afficher tous les labels
kubectl get pods -L app -L tier                # afficher des labels en colonnes (-L = --label-columns)
kubectl label pods -l app=nginx tier=fe        # ajouter tier=fe à tous les Pods app=nginx
kubectl label pod mypod tier=be --overwrite    # modifier un label existant
kubectl label pod mypod tier-                  # supprimer le label tier
```

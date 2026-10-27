# Annotations

## C'est quoi une annotation ?

- Une `annotation` est une paire `key`/`value` attachée à un objet, pour stocker des métadonnées **non identifiantes**.
- Contrairement aux `labels`, les `annotations` ne servent **pas** à identifier ni à sélectionner des objets.
- La `value` peut être petite ou grande, structurée ou non, et contenir des caractères interdits dans les `labels`.
- Un même objet peut avoir des `labels` **et** des `annotations`.
- Les `keys` et les `values` sont forcément des strings (pas de nombre, booléen, liste…).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: annotations-demo
  annotations:
    imageregistry: "https://hub.docker.com/"    # key: imageregistry / value: une URL
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
```

---

## Labels ou annotations ?

| | `labels` | `annotations` |
|---|---|---|
| Sert à | identifier et **sélectionner** des objets | stocker des **informations** sur un objet |
| Utilisable dans un `selector` / `-l` | ✅ | ❌ |
| `value` | 63 caractères max, caractères limités | n'importe quelle string : espaces, JSON, YAML… |
| Taille | — | 256 KiB max pour toutes les `annotations` d'un objet |

---

## Quoi mettre dans une annotation ?

- Infos de build / release / image : timestamp, release ID, branche git, numéro de PR, hash d'image, registry.
- Pointeurs vers le logging, le monitoring, l'audit.
- Infos sur l'outil client qui a créé l'objet (nom, version), pour le debug.
- Provenance : URLs d'objets liés dans d'autres composants.
- Contact des responsables : téléphone, astreinte, site de l'équipe.
- Directives pour modifier le comportement d'un composant (ex. un ingress controller).
- Champs gérés par une couche de configuration déclarative (ex. `kubectl apply`).

```yaml
metadata:
  annotations:
    example.com/git-commit: "a1b2c3d"
    example.com/oncall: "https://wiki.example.com/team-shop"
    example.com/config: '{"retries": 3, "timeout": "30s"}'    # du JSON, interdit dans un label
```

---

## Comment est composée une key ?

Même format que les `labels` : `<prefix>/<name>`, le `prefix` est optionnel, le `name` est obligatoire.

- `name` : 63 caractères max, commence et finit par `[a-z0-9A-Z]`, `-` `_` `.` autorisés entre.
- `prefix` : DNS subdomain (ex. `example.com`), 253 caractères max.
- Les prefix `kubernetes.io/` et `k8s.io/` sont réservés aux composants Kubernetes.
- `value` : aucune restriction de caractères. Données binaires → encoder en base64.

---

## Commandes utiles

```bash
kubectl annotate pod mypod example.com/owner=team-shop                 # ajouter une annotation
kubectl annotate pod mypod example.com/owner=team-pay --overwrite      # modifier (sinon erreur)
kubectl annotate pod mypod example.com/owner-                          # supprimer
kubectl describe pod mypod                                             # section "Annotations:"
kubectl get pod mypod -o jsonpath='{.metadata.annotations}'            # toutes les annotations
kubectl annotate deploy myapp kubernetes.io/change-cause="image v2"    # visible dans kubectl rollout history
```

---

## Résumé

- `annotation` = paire `key`/`value` pour stocker des infos **non identifiantes** sur un objet. Pas de sélection possible.
- `labels` → sélectionner ; `annotations` → informer (build, contact, config, directives pour un outil).
- `key` = `<prefix>/<name>` comme les `labels`. `value` = n'importe quelle string, 256 KiB max par objet.
- `kubectl annotate pod mypod key=value` / `--overwrite` / `key-`.

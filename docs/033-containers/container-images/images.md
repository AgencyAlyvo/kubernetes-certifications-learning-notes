# Images

## C'est quoi une image ?

- Une image de container contient une application et toutes ses dépendances, prête à s'exécuter.
- On la construit, on la pousse dans un **registry**, puis on la référence dans un `Pod` : c'est le `Container runtime` qui la télécharge (pull) sur demande du `kubelet`.

```yaml
spec:
  containers:
  - name: web
    image: registry.example.com:5000/shop/web:1.4.2    # registry[:port]/nom:tag
```

<br /><br />

---

<br /><br />

## Comment est composé le nom d'une image ?

`[registry[:port]/]nom[:tag][@digest]`

| Partie | Rôle | Si absent |
|---|---|---|
| registry | où est stockée l'image (`registry.k8s.io`, `ghcr.io`…) | `docker.io` (Docker Hub) |
| nom | le nom de l'image, avec éventuellement un chemin (`library/busybox`, `shop/web`) | — |
| tag | une version, **modifiable** (le registry peut la faire pointer vers une autre image) | `latest` |
| digest | hash du contenu (`sha256:…`), **immuable** : identifie exactement une image | — |

```text
busybox                                   → docker.io/library/busybox:latest
busybox:1.32.0                            → docker.io/library/busybox:1.32.0
registry.k8s.io/pause:3.5                 → registry custom + tag
registry.k8s.io/pause@sha256:1ff6c18f…    → par digest : toujours exactement la même image
registry.k8s.io/pause:3.5@sha256:1ff6…    → tag + digest : seul le digest compte pour le pull
```

- ⚠️ En prod, pas de `:latest` : impossible de savoir quelle version tourne, et impossible de rollback proprement. Utiliser un tag précis (`v1.42.0`) ou un digest.
- CKS : référencer les images **par digest** garantit que le code qui tourne est exactement celui attendu, même si le registry change ce que pointe un tag.

<br /><br />

---

<br /><br />

## C'est quoi l'imagePullPolicy ?

Le champ `imagePullPolicy` d'un container dit au `kubelet` **quand demander au `Container runtime`, via la `CRI`, de télécharger (pull) l'image** : le `kubelet` décide, le `Container runtime` télécharge.

| `imagePullPolicy` | Comportement |
|---|---|
| `IfNotPresent` | pull seulement si l'image n'est pas déjà sur le `Node` |
| `Always` | à chaque démarrage du container, le `kubelet` demande au `Container runtime` de pull. Le runtime contacte le registry et ne retélécharge que les layers manquants (donc peu coûteux si l'image est déjà là) |
| `Never` | jamais de pull : si l'image n'est pas sur le `Node`, le container ne démarre pas |

```yaml
spec:
  containers:
  - name: web
    image: shop/web:1.4.2
    imagePullPolicy: Always    # pull à chaque démarrage du container
```

<br /><br />

---

<br /><br />

## Quelle valeur par défaut si l'imagePullPolicy n'est pas spécifiée ?

Si le champ est absent, Kubernetes le remplit **à la création** de l'objet, selon l'image :

| Image | `imagePullPolicy` par défaut |
|---|---|
| tag `:latest` ou pas de tag | `Always` |
| tag autre que `:latest` | `IfNotPresent` |
| digest | `IfNotPresent` |

- ⚠️ Cette valeur n'est pas recalculée si on change l'image plus tard : passer un `Deployment` de `web:1.0` à `web:latest` ne change pas la policy, il faut la modifier soi-même.

<br /><br />

---

<br /><br />

## Pourquoi un Pod est-il en ImagePullBackOff ?

- `ImagePullBackOff` = le container ne démarre pas parce que l'image n'a pas pu être téléchargée.
- Causes classiques : nom ou tag faux, registry injoignable, image privée sans `imagePullSecrets`.

```bash
kubectl get pods
# NAME   READY   STATUS             RESTARTS   AGE
# web    0/1     ImagePullBackOff   0          2m

kubectl describe pod web # section Events : "Failed to pull image ... : not found" / "unauthorized"
```

<br /><br />

---

<br /><br />

## Comment utiliser un registry privé ?

Méthode recommandée : un `Secret` de type `kubernetes.io/dockerconfigjson`, référencé par le `Pod` dans `imagePullSecrets`.

```bash
# 1. Créer le Secret avec les identifiants du registry (une fois par namespace)
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=corentin \
  --docker-password='...' \
  --docker-email=contact@example.com
```

Ce que cette commande crée (`kubectl get secret regcred -o yaml`) :

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: regcred
type: kubernetes.io/dockerconfigjson     # type réservé aux identifiants de registry
data:
  .dockerconfigjson: eyJhdXRocyI6ey...   # un fichier config.json (format Docker) encodé en base64
```

Le contenu décodé de `.dockerconfigjson` est le même que `~/.docker/config.json` après un `docker login` :

```json
{
  "auths": {
    "registry.example.com": {
      "username": "corentin",
      "password": "...",
      "email": "contact@example.com",
      "auth": "Y29yZW50aW46Li4u"    # base64 de "username:password"
    }
  }
}
```

```yaml
# 2. Le référencer dans le Pod (même namespace que le Secret)
spec:
  containers:
  - name: web
    image: registry.example.com/shop/web:1.4.2
  imagePullSecrets:
  - name: regcred
```

```yaml
# Ou une fois pour tous les Pods d'un ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
imagePullSecrets:
- name: regcred
```
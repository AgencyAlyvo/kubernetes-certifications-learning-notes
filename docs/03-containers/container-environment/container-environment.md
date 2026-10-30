# Container Environment

## Qu'est-ce qu'un container reçoit de Kubernetes ?

Kubernetes fournit 3 choses à chaque container :

| Ressource | Contenu |
|---|---|
| Un **système de fichiers** | l'image du container + les volumes montés dans le `Pod` |
| Des **informations sur lui-même** | hostname, variables d'environnement, infos sur son `Pod` |
| Des **informations sur le cluster** | les `Services` du namespace, en variables d'environnement et via le DNS |

<br /><br />

---

<br /><br />

## Informations sur le container

- Le **hostname** du container = le `name` du `Pod`. Accessible avec la commande `hostname`, ou depuis le code de l'application avec l'appel `gethostname()` de la libc.
- Le `name` et le `namespace` du `Pod` sont disponibles en variables d'environnement via la [Downward API](https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/).
- Les variables d'environnement définies dans le `Pod` (`env`, `envFrom`) et celles de l'image (`ENV` du Dockerfile) sont aussi disponibles.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
spec:
  containers:
  - name: app
    image: shop/web:1.4.2
    env:
    - name: APP_MODE                  # variable définie par toi
      value: production
    - name: POD_NAME                  # Downward API : name du Pod
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_NAMESPACE             # Downward API : namespace du Pod
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
```

```bash
kubectl exec web -- hostname    # web  (= name du Pod)
kubectl exec web -- env         # APP_MODE, POD_NAME, POD_NAMESPACE, + les variables des Services
```

<br /><br />

---

<br /><br />

## Informations sur le cluster

- À la création du container, Kubernetes injecte en variables d'environnement **tous les `Services` qui existaient à ce moment** dans le même namespace (plus ceux du `Control Plane`).
- Pour un `Service` `foo` : `FOO_SERVICE_HOST` (son IP) et `FOO_SERVICE_PORT` (son port).

```bash
kubectl exec web -- env | grep SERVICE
# KUBERNETES_SERVICE_HOST=10.96.0.1        # Service kubernetes (l'API server), namespace default
# KUBERNETES_SERVICE_PORT=443
# DB_SERVICE_HOST=10.96.45.12              # Service db du même namespace
# DB_SERVICE_PORT=5432
```

- ⚠️ Un `Service` créé **après** le container n'a pas de variable : il faut recréer le `Pod`. C'est pour ça qu'on utilise plutôt le **DNS** (`db.shop.svc.cluster.local`), qui fonctionne pour tous les `Services`, à tout moment.
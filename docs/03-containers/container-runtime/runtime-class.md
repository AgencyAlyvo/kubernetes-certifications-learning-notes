# Runtime Class

## C'est quoi une RuntimeClass ?

- Une `RuntimeClass` permet de choisir, **par `Pod`**, la configuration du `Container runtime` qui exécute ses containers.
- Usage principal : faire tourner certains `Pods` dans un runtime plus isolé (sandbox), au prix d'un peu de performance. Exemples : `gVisor` (`runsc`, un noyau en espace utilisateur) ou `Kata Containers` (une micro-VM par `Pod`).
- Sans `runtimeClassName`, le `Pod` utilise le runtime par défaut (`runc`).

```yaml
spec:
  template:
    spec:
      runtimeClassName: gvisor    # les Pods de ce Deployment tournent dans gVisor au lieu de runc
      containers:
      - name: app
        image: shop/web:1.4.2
```

<br /><br />

---

<br /><br />

## Quels runtimes peut-on choisir ?

| Runtime | Handler | Comment ça isole | Performance | Quand |
|---|---|---|---|---|
| **`runc`** (défaut) | `runc` | namespaces + cgroups Linux : les containers partagent le **noyau du `Node`** | native | tout ce qui est fiable |
| **`gVisor`** | `runsc` | un noyau en espace utilisateur (`runsc`) intercepte les appels système : le container ne touche pas au vrai noyau | un peu plus lent (surcoût sur les syscalls) | `Pods` non fiables : code tiers, multi-tenant |
| **`Kata Containers`** | `kata` | chaque `Pod` tourne dans une **micro-VM** avec son propre noyau | plus lent au démarrage, plus de mémoire | isolation maximale, workloads sensibles |
| **`crun`** | `crun` | comme `runc`, écrit en C | plus rapide et plus léger que `runc` | alternative à `runc`, par défaut sur CRI-O |

- Sécurité ↔ performance : plus l'isolation est forte, plus le surcoût est grand. `runc` pour le commun, un sandbox (`gVisor`, `Kata`) pour ce qu'on ne fait pas confiance.
- Tous passent par le même `Container runtime` (`containerd` ou `CRI-O`) : c'est lui qui appelle le bon runtime bas niveau selon le handler.

<br /><br />

---

<br /><br />

## Comment ça marche ?

| Étape | Où | Quoi |
|---|---|---|
| 1. Configurer le runtime sur les `Nodes` | `/etc/containerd/config.toml` | déclarer un **handler** : un nom qui pointe vers un runtime (ex. `runsc` pour gVisor) |
| 2. Créer la `RuntimeClass` | objet Kubernetes, à l'échelle du cluster | `handler` = le nom déclaré à l'étape 1 |
| 3. L'utiliser | `spec.runtimeClassName` du `Pod` | le `kubelet` demande au runtime de lancer le `Pod` avec ce handler |

```toml
# 1. /etc/containerd/config.toml, sur chaque Node concerné
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runsc]    # handler "runsc"
  runtime_type = "io.containerd.runsc.v1"                          # gVisor
```

```yaml
# 2. RuntimeClass (API group node.k8s.io, non namespacée)
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor       # name utilisé dans les Pods (runtimeClassName)
handler: runsc       # handler configuré dans containerd
```

```yaml
# 3. Deployment : ses Pods référencent la RuntimeClass par son name
apiVersion: apps/v1
kind: Deployment
metadata:
  name: untrusted
spec:
  replicas: 3
  selector:
    matchLabels:
      app: untrusted
  template:                         # spec des Pods créés par le Deployment
    metadata:
      labels:
        app: untrusted
    spec:
      runtimeClassName: gvisor      # → RuntimeClass "gvisor" → handler "runsc" → gVisor sur le Node
      containers:
      - name: app
        image: shop/web:1.4.2
```

- Les étapes 1 et 2 préparent le cluster, une seule fois (admin). L'étape 3 se fait workload par workload : `runtimeClassName` va dans `spec.template.spec` d'un `Deployment`, `StatefulSet`, `DaemonSet` ou `Job` (ou dans `spec` d'un `Pod` seul), et tous les `Pods` créés l'auront.
- Sans `runtimeClassName`, les `Pods` tournent avec le runtime par défaut (`runc`), même si la `RuntimeClass` existe.

- ⚠️ Si la `RuntimeClass` n'existe pas, ou si le runtime ne connaît pas le handler, le `Pod` passe en phase `Failed` → voir l'erreur dans `kubectl describe pod` (Events).
- Par défaut, tous les `Nodes` sont supposés avoir la même configuration de runtime.

<br /><br />

---

<br /><br />

## Et si les Nodes n'ont pas tous les mêmes runtimes ?

Par défaut, Kubernetes suppose que tous les `Nodes` sont configurés pareil. Mais en pratique, les `Nodes` peuvent avoir des runtimes différents : certains avec gVisor, d'autres avec Kata, la plupart avec `runc` seulement. Un `Pod` `runtimeClassName: gvisor` planifié sur un `Node` sans gVisor passera en `Failed`.

Solution : le champ `scheduling` de la `RuntimeClass`, qui ajoute automatiquement un `nodeSelector` (et des `tolerations`) à tous les `Pods` qui l'utilisent. Plus besoin de le répéter dans chaque `Deployment`.

1. Labelliser les `Nodes` qui ont gVisor : `kubectl label node node02 runtime=gvisor`.
2. (Optionnel) Les tainter pour que les autres `Pods` n'y aillent pas : `kubectl taint node node02 runtime=gvisor:NoSchedule`.
3. Déclarer ces contraintes dans la `RuntimeClass` :

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
scheduling:
  nodeSelector:
    runtime: gvisor          # ajouté à chaque Pod gvisor → il ne va que sur les Nodes labellisés
  tolerations:
  - key: runtime
    value: gvisor
    effect: NoSchedule       # ajouté à chaque Pod gvisor → il peut aller sur les Nodes taintés
```

<br /><br />

---

<br /><br />

## Comment compter les ressources du sandbox ? (overhead)

Un runtime sandboxé consomme des ressources **en plus** de celles des containers : la micro-VM de Kata, le noyau en espace utilisateur de gVisor. Par défaut, le scheduler ne les voit pas et peut surcharger un `Node`.

Le champ `overhead` de la `RuntimeClass` déclare ce surcoût. Il est ajouté aux `requests` des containers pour le scheduling, les quotas et l'éviction.

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata
handler: kata
overhead:
  podFixed:
    cpu: 250m        # chaque Pod kata réserve 250m CPU en plus des requests de ses containers
    memory: 120Mi    # et 120Mi de mémoire
```

Exemple : un `Pod` `kata` dont le container demande `cpu: 500m` compte pour `750m` dans le scheduling.

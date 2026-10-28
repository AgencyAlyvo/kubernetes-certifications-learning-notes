# Nodes

## C'est quoi un Node ?

- Un `Node` est une machine, virtuelle ou physique, sur laquelle Kubernetes exécute les `Pods`.
- Un `Node` est géré par le `Control Plane` et un `Node` contient les composants nécessaires pour faire tourner des `Pods` : `kubelet`, `Container runtime` et `kube-proxy (optionnel)`.

```bash
kubectl get nodes
# NAME     STATUS   ROLES           AGE   VERSION
# cp-1     Ready    control-plane   10d   v1.35.0
# node01   Ready    <none>          10d   v1.35.0
```

---

## Comment un Node rejoint-il le cluster ?

Pour que Kubernetes connaisse une machine, il faut un objet `Node` dans l'API server (`kube-apiserver`). Il est créé de 2 façons :

- **Par le `kubelet` lui-même**, au démarrage (`--register-node=true`, par défaut). C'est ce que fait `kubeadm join`.
- **À la main**, avec `kubectl` (`kubelet` lancé avec `--register-node=false`). Rare.

Un `Node` reçoit des `Pods` seulement s'il est **sain** : son `kubelet` est enregistré sous le même `name` que l'objet `Node`, et ses services tournent. Sinon, le `Node` est ignoré jusqu'à ce qu'il redevienne sain.

```yaml
apiVersion: v1
kind: Node
metadata:
  name: node01                       # doit correspondre au kubelet enregistré
  labels:
    kubernetes.io/hostname: node01
```

Options du `kubelet` pour l'auto-enregistrement :

| Option | Rôle | Exemple |
|---|---|---|
| `--kubeconfig` | Chemin du fichier kubeconfig du `kubelet` : adresse de l'API server (`kube-apiserver`) + identifiants pour s'y authentifier | `/etc/kubernetes/kubelet.conf` |
| `--register-node` | Le `kubelet` crée lui-même son objet `Node` au démarrage | `true` (défaut) |
| `--register-with-taints` | Taints posés dès l'enregistrement, pour réserver le `Node` aux `Pods` qui ont la toleration | `gpu=true:NoSchedule` |
| `--node-labels` | `Labels` posés dès l'enregistrement, pour un `nodeSelector` (ignorés si changés après) | `disktype=ssd` |
| `--node-ip` | IP déclarée dans `status.addresses`, utilisée par l'API server pour joindre le `Node` | `10.0.0.11` |
| `--node-status-update-frequency` | Intervalle d'envoi du statut à l'API server (heartbeat) | `10s` (défaut) |

- Par sécurité, le `kubelet` d'un `Node` ne peut modifier **que son propre** objet `Node`, pas ceux des autres.

---

## Comment administrer un Node ?

```bash
# Donner un rôle au Node : le label node-role.kubernetes.io/<role> remplit la colonne ROLES de kubectl get nodes
kubectl label node node01 node-role.kubernetes.io/worker=worker

# Poser un label sur le Node, pour y placer des Pods avec un nodeSelector (ex. nodeSelector: {disktype: ssd})
kubectl label node node01 disktype=ssd

# Rendre le Node unschedulable : le scheduler n'y place plus de nouveaux Pods, ceux qui tournent déjà restent
# (à faire avant un reboot ou une maintenance)
kubectl cordon node01

# Annuler le cordon : le Node accepte de nouveau des Pods
kubectl uncordon node01

# Vider le Node : cordon + éviction de tous ses Pods, recréés ailleurs par leurs controllers
# --ignore-daemonsets : obligatoire si des DaemonSets tournent dessus (leurs Pods ne peuvent pas être déplacés)
kubectl drain node01 --ignore-daemonsets

# Voir le statut du Node : Conditions (Ready, MemoryPressure, DiskPressure…), Capacity / Allocatable, Pods qui tournent dessus, Events
kubectl describe node node01
```

---

## Résumé

- `Node` = machine (VM ou physique) qui exécute les `Pods`, avec `kubelet` + `Container runtime` + `kube-proxy`.
- L'objet `Node` est créé par le `kubelet` lui-même (`--register-node`, défaut) ; `NodeRestriction` l'empêche de toucher aux autres `Nodes`.
- `kubectl cordon` → unschedulable (existants intacts), `drain` → cordon + éviction, `uncordon` → retour à la normale.
- `kubectl describe node` → Conditions (`Ready`, `MemoryPressure`, `DiskPressure`, `PIDPressure`), Capacity / Allocatable.
- `Node` injoignable → `Ready: Unknown` → taint `NoExecute` → éviction des `Pods` après 5 min.

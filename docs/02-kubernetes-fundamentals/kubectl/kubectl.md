# kubectl

## C'est quoi kubectl ?

- `kubectl` est l'outil en ligne de commande principal pour communiquer avec un cluster Kubernetes.
- Il sert à créer, inspecter, mettre à jour et supprimer des objets Kubernetes.
- Chaque commande est traduite en une ou plusieurs requêtes HTTP vers l'API Kubernetes (le `kube-apiserver`), qui valide la requête, l'applique dans `etcd` et renvoie le résultat.
- Les autres clients (bibliothèques, dashboards comme Headlamp) passent par la même API.

```text
kubectl get pods  →  requête HTTP  →  kube-apiserver  →  etcd  →  résultat
```

---

## Comment kubectl trouve-t-il le cluster ? (KUBECONFIG)

`kubectl` lit un fichier **KUBECONFIG** qui contient l'adresse du cluster et les identifiants pour s'y connecter :

1. Par défaut : `$HOME/.kube/config`.
2. Sinon, la variable d'environnement `KUBECONFIG`.
3. Sinon, l'option `--kubeconfig` sur la commande.

Un kubeconfig définit 3 types d'entrées :

| Entrée | Contient |
|---|---|
| `clusters` | l'adresse du `kube-apiserver` et son certificat CA |
| `users` | les identifiants (certificat client, token…) |
| `contexts` | un `cluster` + un `user` (+ un `namespace` par défaut) |

```yaml
# $HOME/.kube/config
apiVersion: v1
kind: Config
clusters:
- name: dev
  cluster:
    server: https://dev.example.com:6443
    certificate-authority-data: ...
- name: prod
  cluster:
    server: https://prod.example.com:6443
    certificate-authority-data: ...
users:
- name: corentin
  user:
    client-certificate-data: ...
    client-key-data: ...
contexts:
- name: dev-corentin                # un contexte = un cluster + un user
  context:
    cluster: dev
    user: corentin
    namespace: shop                 # namespace par défaut de ce contexte
- name: prod-corentin
  context:
    cluster: prod
    user: corentin
current-context: dev-corentin       # contexte actif : utilisé par toutes les commandes kubectl
```

---

## Comment changer de cluster ? (use-context)

Un kubeconfig peut définir plusieurs `clusters`, `users` et `contexts` : on bascule de l'un à l'autre sans reconfigurer son environnement.

```bash
kubectl config get-contexts                              # lister les contextes (* = actif)
kubectl config current-context                           # afficher le contexte actif
kubectl config use-context prod-corentin                 # changer de contexte actif
kubectl config set-context --current --namespace=shop    # changer le namespace par défaut du contexte actif
kubectl config view --minify                             # afficher la config du contexte actif
kubectl --context=dev-corentin get pods                  # une seule commande sur un autre contexte
kubectl --kubeconfig=/tmp/other-config get pods          # une seule commande avec un autre fichier
```

⚠️ À l'examen, chaque question commence par `kubectl config use-context <nom>` : ne pas l'oublier, sinon on travaille sur le mauvais cluster.

---

## Que peut-on faire avec kubectl ?

| Catégorie | Exemples |
|---|---|
| Gérer les ressources | `kubectl apply -f`, `kubectl create`, `kubectl delete`, `kubectl edit`, `kubectl scale` |
| Inspecter l'état du cluster | `kubectl get`, `kubectl describe`, `kubectl get events`, `kubectl top` |
| Déboguer | `kubectl logs`, `kubectl exec`, `kubectl port-forward`, `kubectl debug` |
| Opérations sur le cluster | `kubectl drain`, `kubectl cordon` / `uncordon`, `kubectl taint` |
| Scripter et automatiser | `-o json`, `-o yaml`, `-o jsonpath`, `-o custom-columns` |

---

## Déclaratif ou impératif ?

| | Déclaratif | Impératif |
|---|---|---|
| Commande | `kubectl apply -f fichier.yaml` | `kubectl create`, `kubectl run`, `kubectl expose`… |
| Pour | la prod : fichiers versionnés, traçables, GitOps | le dev, les tests, et **l'examen** (plus rapide) |

```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml    # impératif → génère un YAML à modifier puis kubectl apply
```

---

## Compatibilité des versions

- `kubectl` tolère 1 version mineure par rapport au control plane : `kubectl` 1.32 marche avec un cluster 1.31, 1.32 ou 1.33.

```bash
kubectl version    # version de kubectl et du kube-apiserver
```
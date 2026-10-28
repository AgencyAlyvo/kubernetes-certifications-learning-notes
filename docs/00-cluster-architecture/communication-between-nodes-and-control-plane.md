# Communication entre les Nodes et le Control Plane

## Comment ça communique ?

Kubernetes suit un modèle **hub-and-spoke (en étoile)** : tout passe par l'API server (`kube-apiserver`), aucun autre composant du `Control Plane` n'expose de service aux `Nodes`.

| Sens | Qui parle à qui | Protocole | Pour quoi |
|---|---|---|---|
| `Node` → `Control Plane` | `kubelet` et `Pods` → `kube-apiserver` | HTTPS, port 6443 (ou 443) | toutes les requêtes à l'API Kubernetes |
| `Control Plane` → `Node` | `kube-apiserver` → `kubelet` | HTTPS, port 10250 | `logs`, `exec`, `port-forward` |

---

## Node → Control Plane (sécurisé par défaut)

- L'API server (`kube-apiserver`) écoute en **HTTPS** avec une authentification client obligatoire.
- Le `kubelet` s'authentifie avec un **certificat client**, délivré automatiquement par le **TLS bootstrapping** au `kubeadm join`.
- Un `Pod` peut aussi parler à l'API server (`kube-apiserver`). Pour ça, Kubernetes lui donne **automatiquement** 3 choses :
  1. Un **token** lié à son `ServiceAccount`, pour s'authentifier.
  2. Le **certificat racine** du cluster, pour vérifier qu'il parle bien au vrai API server (`kube-apiserver`).
  3. Une **adresse** : le `Service` `kubernetes` du namespace `default` (`https://kubernetes.default.svc`).

  Le token et le certificat sont montés dans le `Pod`, dans `/var/run/secrets/kubernetes.io/serviceaccount/`.

- ⚠️ CKS : tout `Pod` a donc ce token, y compris un `Pod` compromis par un attaquant. Si l'application n'a pas besoin de l'API Kubernetes, ne pas le monter :
  ```yaml
  spec:
    automountServiceAccountToken: false    # pas de token dans le Pod
  ```

---

## Control Plane → Node (à sécuriser)

### API server (`kube-apiserver`) → kubelet

Le `kube-apiserver` appelle l'API du `kubelet` (port `10250`) pour :

| Commande | Ce que fait l'API server (`kube-apiserver`) |
|---|---|
| `kubectl logs` | demande les logs d'un container au `kubelet` du `Node` |
| `kubectl exec` / `attach` | ouvre une session dans un container via le `kubelet` |
| `kubectl port-forward` | fait transiter un port local vers un `Pod` via le `kubelet` |

⚠️ **Problème par défaut** : l'API server (`kube-apiserver`) ne vérifie pas le certificat du `kubelet` → un attaquant sur le réseau peut se faire passer pour un `kubelet` (man-in-the-middle).

**Comment sécuriser :**

| Côté | Option | Effet |
|---|---|---|
| `kube-apiserver` | `--kubelet-certificate-authority=/etc/kubernetes/pki/ca.crt` | vérifie le certificat du `kubelet` avec le CA du cluster |
| `kubelet` | `--anonymous-auth=false` | refuse les requêtes sans authentification sur son API |
| `kubelet` | `--authorization-mode=Webhook` | demande à l'API server (`kube-apiserver`) si l'appelant a le droit (RBAC) |

**Où mettre ces options (cluster `kubeadm`) :**

1. Côté `kube-apiserver` : dans le manifest du static pod. Le `kubelet` redémarre le `Pod` tout seul à la sauvegarde.

   ```yaml
   # /etc/kubernetes/manifests/kube-apiserver.yaml
   spec:
     containers:
     - command:
       - kube-apiserver
       - --kubelet-certificate-authority=/etc/kubernetes/pki/ca.crt    # à ajouter
       ...
   ```

2. Côté `kubelet` : dans son fichier de configuration (pas en flags), puis redémarrer le service.

   ```yaml
   # /var/lib/kubelet/config.yaml
   authentication:
     anonymous:
       enabled: false                               # = --anonymous-auth=false
     webhook:
       enabled: true                                # authentifie via l'API server
     x509:
       clientCAFile: /etc/kubernetes/pki/ca.crt     # accepte aussi les certificats signés par le CA
   authorization:
     mode: Webhook                                  # = --authorization-mode=Webhook
   ```

   ```bash
   systemctl restart kubelet
   ```

3. Vérifier que l'API du `kubelet` refuse les requêtes anonymes :

   ```bash
   curl -sk https://<ip-du-node>:10250/pods    # doit renvoyer "Unauthorized"
   ```

---

## Résumé

- Hub-and-spoke : tout passe par l'API server (`kube-apiserver`).
- `Node` → API server (`kube-apiserver`) : HTTPS + certificat client du `kubelet` (TLS bootstrapping). `Pod` → API server : `ServiceAccount` token + `Service` `kubernetes`. Sûr par défaut.
- API server → `kubelet` (`logs`, `exec`, `port-forward`) : certificat du `kubelet` **non vérifié** par défaut → `--kubelet-certificate-authority` + auth / authz du `kubelet` (CKS).
- SSH tunnels dépréciés, remplacés par Konnectivity.

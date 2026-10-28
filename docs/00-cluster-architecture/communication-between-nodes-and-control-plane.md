# Communication entre les Nodes et le Control Plane

## Comment ça communique ?

Kubernetes suit un modèle **hub-and-spoke (en étoile)** : tout passe par l'API server, aucun autre composant du `Control Plane` n'expose de service aux `Nodes`.

| Sens | Qui parle à qui | Protocole | Pour quoi |
|---|---|---|---|
| `Node` → `Control Plane` | `kubelet` et `Pods` → `kube-apiserver` | HTTPS, port 6443 (ou 443) | toutes les requêtes à l'API Kubernetes |
| `Control Plane` → `Node` | `kube-apiserver` → `kubelet` | HTTPS, port 10250 | `logs`, `exec`, `port-forward` |

---

## Node → Control Plane (sécurisé par défaut)

- L'API server écoute en **HTTPS** avec une authentification client obligatoire.
- Le `kubelet` s'authentifie avec un **certificat client**, délivré automatiquement par le **TLS bootstrapping** au `kubeadm join`.
- Un `Pod` qui veut parler à l'API server utilise son `ServiceAccount` : Kubernetes injecte dans le `Pod` le certificat racine du cluster et un token, et le `Service` `kubernetes` (namespace `default`) pointe vers l'API server.

```bash
# Depuis un Pod : CA + token montés automatiquement
ls /var/run/secrets/kubernetes.io/serviceaccount/    # ca.crt  namespace  token
curl --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
     -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
     https://kubernetes.default.svc/api
```

→ Ce sens est sûr par défaut, même sur un réseau public.

---

## Control Plane → Node (à sécuriser, CKS)

### API server → kubelet

- Utilisé pour `kubectl logs`, `kubectl exec` / `attach` et `kubectl port-forward`.
- ⚠️ Par défaut, l'API server **ne vérifie pas le certificat du `kubelet`** → man-in-the-middle possible sur un réseau non fiable.
- Pour sécuriser :
  1. `--kubelet-certificate-authority=<ca.crt>` sur le `kube-apiserver` → vérifie le certificat du `kubelet`.
  2. Activer l'authentification et l'autorisation de l'API du `kubelet` (`--anonymous-auth=false`, `--authorization-mode=Webhook`).

### API server → nodes, pods, services (proxy)

- Connexions en **HTTP** par défaut : ni authentifiées, ni chiffrées. Pas sûr sur un réseau non fiable.

### Tunnels

- **SSH tunnels** : dépréciés.
- **Konnectivity** : le remplaçant. Un serveur dans le réseau du `Control Plane`, des agents sur les `Nodes` ; les agents ouvrent la connexion vers le serveur, et tout le trafic `Control Plane` → `Nodes` passe dedans. Utilisé par les clusters managés.

---

## Résumé

- Hub-and-spoke : tout passe par l'API server.
- `Node` → API server : HTTPS + certificat client du `kubelet` (TLS bootstrapping). `Pod` → API server : `ServiceAccount` token + `Service` `kubernetes`. Sûr par défaut.
- API server → `kubelet` (`logs`, `exec`, `port-forward`) : certificat du `kubelet` **non vérifié** par défaut → `--kubelet-certificate-authority` + auth / authz du `kubelet` (CKS).
- SSH tunnels dépréciés, remplacés par Konnectivity.

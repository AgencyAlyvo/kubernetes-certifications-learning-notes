# Cluster Addons

## C'est quoi un addon ?

- Un addon est une fonctionnalité du cluster qui n'est **pas un composant du `Control Plane` ou du `Node`**, mais qui tourne dans le cluster sous forme de ressources Kubernetes classiques (`Deployment`, `DaemonSet`…).
- Comme ils rendent un service à tout le cluster, leurs ressources namespacées vivent dans le namespace **`kube-system`**.

```bash
kubectl get pods -n kube-system
# NAME                              READY   STATUS    AGE
# coredns-5dd5756b68-abcde          1/1     Running   10d    # addon DNS (Deployment)
# calico-node-x7k2p                 1/1     Running   10d    # addon réseau CNI (DaemonSet, un Pod par Node)
# metrics-server-7f8b9c6d5-fghij    1/1     Running   10d    # addon monitoring (Deployment)
# kube-apiserver-cp-1               1/1     Running   10d    # pas un addon : composant du Control Plane (Static Pod)
```

<br /><br />

---

<br /><br />

## Quels sont les addons principaux ?

| Addon | Rôle | Implémentation courante | Obligatoire ? |
|---|---|---|---|
| **DNS** | serveur DNS interne : résout les noms des `Services` et des `Pods` (`<service>.<namespace>.svc.cluster.local`). Les containers l'utilisent automatiquement | `CoreDNS` | Oui en pratique : tout repose dessus |
| **Network plugin CNI** | attribue une IP à chaque `Pod` et permet aux `Pods` de communiquer entre `Nodes`. Implémente la spec **CNI** (Container Network Interface) | `Calico`, `Cilium`, `Flannel` | Oui : sans CNI, les `Nodes` restent `NotReady` |
| **Container resource monitoring** | collecte les métriques CPU / mémoire des `Nodes` et des `Pods` | `metrics-server` (pour `kubectl top` et l'`HPA`), `Prometheus` | Non, mais nécessaire pour `kubectl top` et l'autoscaling |
| **Cluster-level logging** | centralise les logs des containers dans un stockage avec recherche | `Fluentd` / `Fluent Bit` + `Elasticsearch` ou `Loki` | Non |
| **Web UI** | interface web pour gérer et dépanner le cluster | `Dashboard`, `Headlamp` | Non |

- Avec `kubeadm init`, seuls `CoreDNS` et `kube-proxy` sont installés : le **Network plugin CNI est à installer soi-même** juste après.

→ Liste complète des addons disponibles : https://kubernetes.io/docs/concepts/cluster-administration/addons/

# Kubernetes Certifications — Notes & Synthèses

*Documentation réalisée sur Kubernetes v1.37.*

Ce dépôt regroupe mes notes, synthèses techniques, commandes, exercices pratiques et révisions pour les certifications Kubernetes de la CNCF : **CKA**, **CKAD**, et **CKS**.

L'objectif est de comprendre Kubernetes en profondeur, de maîtriser l'administration et le dépannage de clusters, le déploiement d'applications et la sécurité des workloads, puis de développer des compétences directement applicables en **DevOps, SRE et Platform Engineering**, jusqu'aux environnements de production.

## Organisation du dépôt

```text
kubernetes-certifications-learning-notes/
├── README.md
├── docs/
│   ├── 00-cluster-architecture/
│   ├── 01-cluster-addons/
│   ├── 02-kubernetes-fundamentals/
│   ├── 03-containers/
│   ├── 04-workloads/
│   ├── 05-deployment-strategies/
│   ├── 06-autoscaling/
│   ├── 07-scheduling/
│   ├── 08-resource-management/
│   ├── 09-networking/
│   ├── 10-storage/
│   ├── 11-configuration/
│   ├── 12-security/
│   ├── 13-cluster-installation/
│   ├── 14-cluster-maintenance/
│   ├── 15-cluster-upgrades/
│   ├── 16-troubleshooting/
│   ├── 17-application-packaging/
│   └── 18-extending-kubernetes/
├── labs/
└── .gitignore
```

### Documentation — `docs/`

Chaque fiche suit la même structure : des sections sous forme de questions (« C'est quoi… ? », « Comment… ? »), des exemples YAML et des commandes commentés, et un résumé en fin de fiche.

| Répertoire | Contenu |
|---|---|
| `00-cluster-architecture/` | Composants du control plane et des nodes et leur fonctionnement ensemble, Nodes, controllers, Leases, self-healing, communication entre les nodes et le control plane. |
| `01-cluster-addons/` | Addons du cluster : DNS (CoreDNS), plugins réseau (CNI), monitoring, logging, dashboard. |
| `02-kubernetes-fundamentals/` | API Kubernetes, `kubectl` et kubeconfig, names et UIDs, namespaces, labels, annotations, label et field selectors, finalizers, owners et dependents. |
| `03-containers/` | Images et registries, environnement des containers, lifecycle hooks, CRI et `crictl`, cgroups, RuntimeClass. |
| `04-workloads/` | Pods, types de containers (applicatif, init, sidecar, ephemeral), probes, Deployments, ReplicaSets, StatefulSets, DaemonSets, Jobs et CronJobs. |
| `05-deployment-strategies/` | Rolling updates, rollbacks, statut et historique des rollouts. |
| `06-autoscaling/` | HPA, VPA, autoscaling événementiel, autoscaling proportionnel au cluster, autoscaling des nodes. |
| `07-scheduling/` | nodeSelector, node affinity, taints et tolerations, pod affinity et anti-affinity, topology spread constraints, priorité et préemption. |
| `08-resource-management/` | Requests et limits, QoS classes, ResourceQuota, LimitRange, pression sur les nodes et éviction. |
| `09-networking/` | Modèle réseau, CNI, Services, EndpointSlices, DNS, Ingress, Gateway API, NetworkPolicies, dual-stack, topology-aware routing. |
| `10-storage/` | Volumes, PV, PVC, StorageClasses, provisionnement dynamique, CSI, snapshots, clonage, stockage éphémère local. |
| `11-configuration/` | ConfigMaps, Secrets, kubeconfig. |
| `12-security/` | Authentification, certificats et CSR, autorisation et RBAC, ServiceAccounts, admission controllers, Pod Security Standards et Admission, SecurityContext, user namespaces. |
| `13-cluster-installation/` | Installation avec `kubeadm`, control plane en haute disponibilité, clusters multi-zones, validation des nodes. |
| `14-cluster-maintenance/` | Maintenance du control plane et des nodes, sauvegarde et restauration d'etcd. |
| `15-cluster-upgrades/` | Upgrade du control plane, des nodes et de l'OS. |
| `16-troubleshooting/` | Logs, events et debug ; dépannage des Pods, des nodes, du control plane, du réseau et du stockage. |
| `17-application-packaging/` | Helm et Kustomize. |
| `18-extending-kubernetes/` | CustomResourceDefinitions, controllers personnalisés, Operators. |

### Exercices pratiques — `labs/`

Exercices, manipulations de clusters et mises en situation pour mettre en pratique les notions de la documentation.
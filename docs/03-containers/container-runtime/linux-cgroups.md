# Linux cgroups

## C'est quoi un cgroup ?

- Un **cgroup** (control group) est un mécanisme du kernel Linux qui limite et comptabilise les ressources (CPU, mémoire…) d'un groupe de processus.
- C'est avec les cgroups que le `kubelet` et le `Container runtime` appliquent les `requests` et `limits` CPU / mémoire des `Pods` et de leurs containers.
- Il existe 2 versions : **cgroup v1** et **cgroup v2**. cgroup v2 est la version actuelle (stable dans Kubernetes depuis 1.25).

---

## Quels sont les prérequis pour cgroup v2 ?

| Prérequis | Valeur |
|---|---|
| Kernel Linux | ≥ 5.8 |
| `Container runtime` | `containerd` ≥ 1.4 ou `CRI-O` ≥ 1.20 |
| Distribution | cgroup v2 activé par défaut : Ubuntu ≥ 22.04, Debian ≥ 11, RHEL ≥ 9… |
| cgroup driver | `systemd`, configuré sur le `kubelet` **et** sur le `Container runtime` (voir plus bas) |

- ⚠️ cgroup v1 est **déprécié** depuis Kubernetes 1.35 : le `kubelet` refuse de démarrer sur un `Node` en v1, sauf `failCgroupV1: false` dans sa configuration.

---

## Comment vérifier la version sur un Node ?

```bash
stat -fc %T /sys/fs/cgroup/
# cgroup2fs  → cgroup v2
# tmpfs      → cgroup v1
```

---

## C'est quoi le cgroup driver ?

Le `kubelet` et le `Container runtime` créent **tous les deux** des cgroups : le `kubelet` pour chaque `Pod`, le runtime pour chaque container. Le cgroup driver, c'est la **façon dont ils les créent**. 

Il y en a 2 :

| Driver | Comment le `kubelet` et le `Container runtime` créent les cgroups |
|---|---|
| `cgroupfs` | directement, en créant des répertoires dans `/sys/fs/cgroup` (le système de fichiers des cgroups) |
| `systemd` | en passant par `systemd`, le premier processus lancé par Linux au démarrage (PID 1), qui gère déjà les composants du `Node` (`kubelet`, `containerd`…) et leurs cgroups |

---

## Comment configurer le cgroup driver ?

| Version | Qui décide du driver | Ce qu'il faut configurer |
|---|---|---|
| Kubernetes ≥ 1.31 avec `containerd` ≥ 2.0 ou `CRI-O` ≥ 1.28 | le `Container runtime` : le `kubelet` lui demande son driver via la `CRI` et ignore son propre `cgroupDriver` | le `Container runtime` seulement |
| Plus ancien (`containerd` 1.x) | chacun le sien | le `Container runtime` **et** le `kubelet`, avec la même valeur |

### 1. Côté `containerd`

`SystemdCgroup` est à `false` dans la config générée par `containerd config default` : le passer à `true`.

```toml
# /etc/containerd/config.toml

# containerd 2.x (version = 3)
[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc.options]
  SystemdCgroup = true

# containerd 1.x (version = 2)
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```

### 2. Côté `kubelet` (seulement avec `containerd` 1.x)

```yaml
# /var/lib/kubelet/config.yaml
cgroupDriver: systemd      # valeur mise par défaut par kubeadm
```

### 3. Redémarrer

```bash
systemctl restart containerd
systemctl restart kubelet
```

### 4. Vérifier

```bash
# Côté Container runtime : afficher la config CRI de containerd, et y lire le driver de runc
crictl info | grep -i systemdcgroup
#   "SystemdCgroup": true     → driver systemd
#   "SystemdCgroup": false    → driver cgroupfs

# Côté kubelet : chercher dans ses logs le driver qu'il a retenu, ou une erreur cgroup au démarrage
journalctl -u kubelet | grep -i "cgroup driver"
#   "Using cgroup driver" driver="systemd"    → celui que le kubelet utilise (reçu du runtime depuis 1.31)
```

---

## Résumé

- cgroups = mécanisme Linux qui applique les `requests` / `limits` des containers.
- cgroup v2 : kernel ≥ 5.8, `containerd` ≥ 1.4 ou `CRI-O` ≥ 1.20, driver `systemd` des deux côtés. v1 déprécié depuis 1.35.
- Vérifier : `stat -fc %T /sys/fs/cgroup/` → `cgroup2fs`.
- Driver `systemd` : `SystemdCgroup = true` dans `containerd`. Depuis Kubernetes 1.31, le `kubelet` récupère le driver du runtime via la `CRI` ; avec `containerd` 1.x, aligner aussi `cgroupDriver: systemd` dans le `kubelet`.

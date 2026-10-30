# Container Runtime Interface (CRI)

## C'est quoi la CRI ?

- La `CRI` est l'**interface** entre le `kubelet` et le `Container runtime`. Grâce à elle, le `kubelet` peut piloter n'importe quel runtime qui l'implémente (`containerd`, `CRI-O`…) sans être recompilé.
- Chaque `Node` doit avoir un runtime qui parle la `CRI`, sinon le `kubelet` ne peut pas lancer de `Pods`.
- Techniquement : une API **gRPC**. Le `kubelet` est le **client**, le runtime est le **serveur**, et ils communiquent via un socket Unix sur le `Node`.
- Par cette interface, le `kubelet` demande au runtime deux choses : gérer les **containers** (créer le `Pod sandbox`, démarrer, arrêter) et gérer les **images** (télécharger, lister, supprimer).

```text
kubelet  ──gRPC (CRI)──▶  socket Unix  ──▶  containerd / CRI-O (Container runtime)
                                              │
                                              ▼
                                   runc / crun / runsc (gVisor) / kata    ← runtime bas niveau, selon la RuntimeClass
```

<br /><br />

---

<br /><br />

## Comment le kubelet trouve-t-il le runtime ?

Le `kubelet` ne devine pas où est le runtime : on lui donne le **chemin du socket** dans sa configuration, avec le champ `containerRuntimeEndpoint` (ou le flag `--container-runtime-endpoint`).

```yaml
# /var/lib/kubelet/config.yaml
containerRuntimeEndpoint: unix:///run/containerd/containerd.sock    # ici, containerd
```

| Runtime installé sur le `Node` | Valeur à mettre |
|---|---|
| `containerd` | `unix:///run/containerd/containerd.sock` |
| `CRI-O` | `unix:///var/run/crio/crio.sock` |

<br /><br />

---

<br /><br />

## Comment parler au runtime directement ? (crictl)

`crictl` est le client en ligne de commande de la `CRI` : il parle au runtime comme le `kubelet` le fait. Indispensable pour déboguer un `Node` quand `kubectl` ne répond plus.

Comme le `kubelet`, `crictl` ne devine pas où est le runtime : on lui donne le **chemin du socket** dans son propre fichier de configuration, `/etc/crictl.yaml` (ou le flag `--runtime-endpoint`). C'est une configuration **séparée** de celle du `kubelet`, mais elle doit pointer vers le même socket.

```yaml
# /etc/crictl.yaml (créé par kubeadm)
runtime-endpoint: unix:///run/containerd/containerd.sock    # ici, containerd
```

| Runtime installé sur le `Node` | Valeur à mettre |
|---|---|
| `containerd` | `unix:///run/containerd/containerd.sock` |
| `CRI-O` | `unix:///var/run/crio/crio.sock` |

- Si `/etc/crictl.yaml` manque, `crictl` affiche un avertissement et teste les sockets connus un par un. On peut aussi passer le socket à la main : `crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps`.

```bash
crictl ps                      # containers en cours sur ce Node
crictl ps -a                   # y compris les containers arrêtés
crictl pods                    # Pod sandboxes
crictl images                  # images présentes sur le Node
crictl logs <container-id>     # logs d'un container
crictl inspect <container-id>  # détails d'un container
crictl info                    # config et état du runtime
```

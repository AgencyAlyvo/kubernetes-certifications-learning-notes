# Standard (applicatif) Containers

## C'est quoi un container applicatif ?

- Le container applicatif est le container **principal** d'un `Pod` : celui qui fait tourner l'application (un serveur web, une API, une base de données…).
- Déclaré dans `spec.containers`. Un `Pod` en a au moins un, parfois plusieurs s'ils doivent vivre et mourir ensemble (sans contrainte d'ordre entre eux : pour ça, voir les sidecars).
- Il vit aussi longtemps que le `Pod` : s'il se termine, le `kubelet` le redémarre selon le `restartPolicy` du `Pod` (`Always` par défaut).
- C'est le seul type de container dont les ports peuvent être exposés par un `Service`, et dont les probes conditionnent le `Ready` du `Pod`.

<br /><br />

---

<br /><br />

## Les champs qu'on remplit tout le temps

```yaml
spec:
  template:
    spec:
      containers:
      - name: web                                   # obligatoire, unique dans le Pod (utilisé par kubectl logs -c, exec -c…)
        image: shop/web:1.4.2                       # obligatoire → voir images
        imagePullPolicy: IfNotPresent               # quand pull l'image → voir images
        command: ["python"]                         # remplace l'ENTRYPOINT de l'image (optionnel)
        args: ["app.py", "--port=8080"]             # remplace le CMD de l'image (optionnel)
        ports:
        - containerPort: 8080                       # informatif : documente le port, ne l'ouvre pas (un Service cible n'importe quel port)
          name: http
        env:                                        # variables d'environnement → voir container-environment, configmaps, secrets
        - name: APP_MODE
          value: production
        resources:                                  # → voir pod-requests-and-limits
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            memory: 256Mi
        volumeMounts:                               # monte un volume déclaré dans spec.volumes → voir volumes
        - name: data
          mountPath: /data
        livenessProbe:                              # → voir probes
          httpGet:
            path: /healthz
            port: http
        readinessProbe:
          httpGet:
            path: /ready
            port: http
        securityContext:                            # → voir security-contexts
          runAsNonRoot: true
          allowPrivilegeEscalation: false
      volumes:
      - name: data
        emptyDir: {}
```

| Champ | Rôle | Obligatoire |
|---|---|---|
| `name` | identifie le container dans le `Pod` ; unique, c'est lui qu'on passe à `-c` | ✅ |
| `image` | l'image à exécuter | ✅ |
| `command` / `args` | écrasent l'`ENTRYPOINT` / le `CMD` de l'image | ❌ |
| `ports` | documente les ports écoutés ; purement informatif, sauf pour nommer un port (`name`) réutilisé par les probes ou un `Service` | ❌ |
| `env` / `envFrom` | variables d'environnement, en dur ou depuis un `ConfigMap` / `Secret` | ❌ |
| `resources` | `requests` (pour le scheduling) et `limits` (plafond) CPU / mémoire | ❌, mais à mettre en prod |
| `volumeMounts` | où monter les volumes du `Pod` dans le container | ❌ |
| `livenessProbe` / `readinessProbe` / `startupProbe` | santé du container | ❌, mais à mettre en prod |
| `securityContext` | utilisateur, privilèges, capabilities… | ❌, mais attendu en CKS |
| `lifecycle` | hooks `postStart` / `preStop` | ❌ |

<br /><br />

---

<br /><br />

## command et args : le piège classique

`command` et `args` ne s'ajoutent pas à l'image, ils **remplacent** ce qu'elle définit :

| Dans le YAML | Ce qui s'exécute |
|---|---|
| ni `command` ni `args` | `ENTRYPOINT` + `CMD` de l'image |
| `args` seul | `ENTRYPOINT` de l'image + `args` |
| `command` seul | `command` seul (le `CMD` de l'image est ignoré) |
| `command` + `args` | `command` + `args` |

```bash
# À l'examen : créer un Pod avec une commande, en impératif
kubectl run busybox --image=busybox:1.36 --restart=Never -- sh -c 'echo hello; sleep 3600'
#                                                         ↑ tout ce qui suit -- devient args (ENTRYPOINT de l'image conservé)
kubectl run busybox --image=busybox:1.36 --restart=Never --command -- sh -c 'echo hello'
#                                                         ↑ avec --command, ça devient command (ENTRYPOINT remplacé)
```

<br /><br />

---

<br /><br />

## Commandes utiles

```bash
kubectl logs web -c app                    # logs d'un container (-c obligatoire si plusieurs containers)
kubectl logs web -c app --previous         # logs de l'instance précédente, après un crash
kubectl exec -it web -c app -- sh          # shell dans le container
kubectl describe pod web                   # section "Containers" : image, state, last state, restart count, events
kubectl get pod web -o jsonpath='{.status.containerStatuses[*].restartCount}'   # nombre de redémarrages
```
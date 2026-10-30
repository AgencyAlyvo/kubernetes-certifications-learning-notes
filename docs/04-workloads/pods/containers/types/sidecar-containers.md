# Sidecar Containers

## C'est quoi un sidecar ?

- Un sidecar est un container **secondaire** qui tourne à côté du container applicatif, dans le même `Pod`, pour lui rendre un service : envoi des logs, proxy réseau, métriques, synchronisation de données… sans toucher au code de l'application.
- Le container sidecar partage avec le container applicatif le **réseau** du `Pod` (même IP, ils se joignent via `localhost`) et, si on le veut, des **volumes**.
- Depuis Kubernetes 1.33 (stable), un sidecar se déclare comme un **init container avec `restartPolicy: Always`** : il démarre avant le container applicatif, comme un init container, mais **continue de tourner** à côté de lui, et ne s'arrête **qu'après** lui, à la fin du `Pod`.

```yaml
spec:
  template:
    spec:
      initContainers:
      - name: logshipper                        # sidecar : démarre AVANT app, et reste en vie
        image: alpine:3.20
        restartPolicy: Always                   # ← sans ce champ, ce serait un init container classique (il se terminerait avant app) ; avec, c'est un sidecar (il reste en vie)
        command: ['sh', '-c', 'tail -F /opt/logs.txt']
        volumeMounts:
        - name: data
          mountPath: /opt
      containers:
      - name: app                               # container applicatif
        image: alpine:3.20
        command: ['sh', '-c', 'while true; do echo "logging" >> /opt/logs.txt; sleep 1; done']
        volumeMounts:
        - name: data
          mountPath: /opt
      volumes:
      - name: data                              # volume partagé : app écrit, logshipper lit
        emptyDir: {}
```

<br />

Un `Pod` peut avoir **plusieurs sidecars**, et les mélanger avec des **init containers** classiques. Tout ce qui est dans `initContainers` démarre **dans l'ordre de déclaration, un par un** : un container sidecar est considéré prêt dès que son processus tourne (ou que sa `startupProbe` réussit), et le suivant démarre.

```yaml
spec:
  template:
    spec:
      initContainers:
      - name: migrate                           # 1. init container : s'exécute, se termine (exit 0)
        image: shop/migrate:1.4.2
      - name: proxy                             # 2. sidecar container : démarre dès que migrate est fini, reste en vie
        image: envoy:1.31
        restartPolicy: Always
      - name: logshipper                        # 3. sidecar container : démarre dès que proxy tourne, reste en vie
        image: fluent-bit:3.1
        restartPolicy: Always
      containers:
      - name: app                               # 4. démarre quand les 3 initContainers sont passés
        image: shop/web:1.4.2
```

À l'arrêt, les sidecars sont stoppés **après** `app`, dans l'ordre inverse : `logshipper` puis `proxy`.

<br /><br />

---

<br /><br />

## Comment ça marche ?

1. Le `kubelet` attend que le réseau et le stockage du `Pod` soient prêts.
2. Il lance les `initContainers` **un par un, dans l'ordre** du YAML. Un init container classique doit se terminer (`exit 0`) ; un sidecar est considéré prêt dès que son **processus tourne** (ou que sa `startupProbe` réussit), et il **reste en vie**.
3. Quand tous les `initContainers` sont passés, les containers applicatifs démarrent, **tous en parallèle**, avec les sidecars déjà en place.
4. Pendant la vie du `Pod`, si un sidecar tombe, le `kubelet` le **redémarre** seul, sans toucher aux containers applicatifs. Sa `readinessProbe`, s'il en a une, compte dans le `Ready` du `Pod`.
5. À l'arrêt du `Pod`, les containers applicatifs reçoivent `SIGTERM` en premier. Les sidecars ne sont arrêtés **qu'une fois les containers applicatifs stoppés**, dans l'ordre inverse de déclaration.

Pendant le démarrage, le `Pod` affiche le même statut `Init:x/y` que pour des init containers :

```bash
kubectl get pods
# NAME    READY   STATUS     RESTARTS   AGE
# web     0/2     Init:0/1   0          5s      # le sidecar logshipper démarre
# web     2/2     Running    0          12s     # READY 2/2 : app + sidecar sont comptés comme des containers du Pod

kubectl logs web -c logshipper      # logs du sidecar (-c = son name)
kubectl describe pod web            # section "Init Containers" : le sidecar y apparaît, State: Running
```
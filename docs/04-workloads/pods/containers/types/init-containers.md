# Init Containers

## C'est quoi un init container ?

- Un init container est un container qui s'exécute **AVANT** les containers applicatifs d'un `Pod`, et qui **se termine** une fois son travail fait.
- Il sert à préparer le terrain : attendre qu'un `Service` soit disponible, générer un fichier de config, cloner un dépôt, lancer une migration…
- Il utilise sa **propre image**, différente de celle de l'application. Les outils nécessaires à l'initialisation (`nslookup`, `curl`, `git`…) sont dans l'image de l'init container, pas dans celle de l'application : l'image de l'application reste minimale, donc avec moins d'outils qu'un attaquant pourrait détourner.
- Déclaré dans `spec.initContainers`, à côté de `spec.containers`.

```yaml
spec:
  template:
    spec:
      initContainers:
      - name: wait-for-db                    # 1. s'exécute en premier : boucle tant que le Service db ne répond pas
        image: busybox:1.36
        command: ['sh', '-c', 'until nslookup db; do echo waiting for db; sleep 2; done']
      containers:
      - name: app                            # 2. ne démarre QUE quand wait-for-db s'est terminé avec succès
        image: shop/web:1.4.2
```

Tant que `wait-for-db` n'a pas fini, `app` n'est pas lancé : l'application ne démarre jamais sans sa base de données disponible.

<br /><br />

---

<br /><br />

## Comment ça marche ?

1. Le `kubelet` attend que le réseau et le stockage du `Pod` soient prêts.
2. Il lance les init containers **un par un, dans l'ordre** du YAML. Chacun doit se terminer avec succès (`exit 0`) avant que le suivant démarre.
3. Si un init container échoue, le `kubelet` le **relance** jusqu'à ce qu'il réussisse (sauf `restartPolicy: Never` → le `Pod` passe en `Failed`).
4. Quand tous ont réussi, les containers applicatifs démarrent, **tous en parallèle**.

Pendant l'initialisation, le `Pod` est en `Pending` et sa condition `Initialized` est `False` :

```bash
kubectl get pods
# NAME    READY   STATUS     RESTARTS   AGE
# web     0/1     Init:0/2   0          6m      # 0 init container terminé sur 2
# web     0/1     Init:1/2   0          7m      # le 1er a réussi, le 2e tourne
# web     1/1     Running    0          9m      # tous terminés, l'app tourne

kubectl logs web -c wait-for-db    # logs d'un init container (-c = son name)
kubectl describe pod web           # section "Init Containers" : State de chacun
```

- Si le `Pod` redémarre, **tous** les init containers sont rejoués → leur code doit être idempotent (pouvoir tourner deux fois sans casser quelque chose, ex. ne pas planter si un fichier existe déjà).
- Un `Pod` bloqué en `Init:0/1` ou `Init:Error` / `Init:CrashLoopBackOff` → regarder `kubectl logs <pod> -c <init-container>`.
# Container Lifecycle Hooks

## C'est quoi un lifecycle hook ?

- Un hook permet d'exécuter du code à un moment précis de la vie d'un container : juste après sa création (`postStart`) ou juste avant son arrêt (`preStop`).
- C'est le `kubelet` qui déclenche le hook, et un **handler** qui dit quoi faire : une commande dans le container (`exec`), une requête HTTP (`httpGet`) ou une pause (`sleep`).
- Usage typique : préparer quelque chose au démarrage, ou laisser l'application finir proprement avant l'arrêt (vider un cache, se retirer d'un load balancer, sauvegarder un état).

```yaml
spec:
  template:
    spec:
      containers:
      - name: app
        image: shop/web:1.4.2
        lifecycle:
          postStart:
            exec:
              command: ["/bin/sh", "-c", "echo started > /tmp/ready"]    # juste après la création du container
          preStop:
            exec:
              command: ["/bin/sh", "-c", "/app/drain.sh"]                # juste avant l'arrêt du container
```

<br /><br />

---

<br /><br />

## Quels hooks existent ?

### `postStart` : juste après la création du container

- Déclenché dès que le container est créé, **en même temps** que son `ENTRYPOINT` (le processus principal). Le hook peut donc s'exécuter avant, pendant ou après le démarrage de l'application : aucun ordre n'est garanti.
- Tant que le hook n'est pas terminé, le container ne passe pas en `Running`.

### `preStop` : juste avant l'arrêt du container

- Déclenché quand le container doit s'arrêter : `kubectl delete`, rolling update, probe en échec, éviction…
- Le signal `TERM` n'est envoyé au container **qu'une fois le hook terminé**. C'est ce qui permet de finir proprement (voir plus bas).
- Si le container est déjà arrêté, le hook échoue.

### `stopSignal` : quel signal pour arrêter le container

- Pas un hook : un réglage qui choisit le signal envoyé au container à l'arrêt (par défaut `SIGTERM`, ou le `STOPSIGNAL` de l'image). Utile pour une application qui s'arrête proprement sur un autre signal (ex. `SIGQUIT` pour nginx).

```yaml
lifecycle:
  stopSignal: SIGQUIT
```

- ⚠️ Si `postStart` ou `preStop` **échoue**, le container est tué.
- ⚠️ `postStart` en `httpGet` n'a pas de sens : rien ne garantit que l'application écoute déjà quand le hook part.

<br /><br />

---

<br /><br />

## Quels handlers ?

| Handler | Quoi | Exécuté par |
|---|---|---|
| `exec` | une commande, dans les namespaces et cgroups du container (ses ressources comptent pour le container) | le `Container runtime` |
| `httpGet` | une requête HTTP vers un endpoint du container | le `kubelet` |
| `sleep` | une pause d'une durée donnée | le `kubelet` |

```yaml
# exec : lance une commande dans le container
spec:
  template:
    spec:
      containers:
      - name: app
        image: shop/web:1.4.2
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "/app/drain.sh"]    # ex. finir les requêtes en cours, vider un cache
```

```yaml
# httpGet : le kubelet appelle un endpoint HTTP du container
spec:
  template:
    spec:
      containers:
      - name: app
        image: shop/web:1.4.2
        ports:
        - containerPort: 8080
        lifecycle:
          preStop:
            httpGet:
              path: /shutdown    # endpoint exposé par l'application pour s'arrêter proprement
              port: 8080
```

```yaml
# sleep : le kubelet attend avant d'envoyer le TERM
spec:
  template:
    spec:
      containers:
      - name: app
        image: shop/web:1.4.2
        lifecycle:
          preStop:
            sleep:
              seconds: 5    # laisse le temps aux Services de retirer le Pod de leurs endpoints avant le TERM
```

<br /><br />

---

<br /><br />

## Comment preStop et terminationGracePeriodSeconds s'articulent-ils ?

Le délai de grâce couvre **le hook + l'arrêt normal du container** :

```text
kubectl delete pod
   │
   ▼  terminationGracePeriodSeconds démarre (30 s par défaut)
preStop s'exécute ──────────────────────▶ fini
                                           │
                                           ▼
                                  signal TERM envoyé au container l'application s'arrête
                                           │  
                                           ▼
       si le total dépasse le délai de grâce → signal KILL
```

- Exemple : `terminationGracePeriodSeconds: 60`, le hook prend 55 s, l'application met 10 s à s'arrêter → 65 s > 60 s → le container est tué de force avant d'avoir fini proprement.
- Un `preStop` qui bloque laisse le `Pod` en `Terminating` jusqu'à la fin du délai de grâce.
- Garder les hooks **légers** ; un hook long n'a de sens que pour sauvegarder un état avant l'arrêt.

<br /><br />

---

<br /><br />

## Comment déboguer un hook ?

- Les logs d'un hook ne sont **pas** dans `kubectl logs`.
- En cas d'échec, le `kubelet` émet un Event : `FailedPostStartHook` ou `FailedPreStopHook`.
- En général, un hook est appelé une seule fois. Mais Kubernetes ne garantit pas l'unicité : dans de rares cas (ex. le `kubelet` redémarre pendant l'exécution du hook), il peut être rejoué. Le script du hook doit donc pouvoir tourner deux fois sans casser quelque chose.

```bash
kubectl describe pod web
# Events:
#   Warning  FailedPostStartHook  kubelet  Exec lifecycle hook ([badcommand]) ... failed - error: command 'badcommand' exited with 126
#   Normal   Killing              kubelet  FailedPostStartHook                 ← le container est tué
#   Warning  BackOff              kubelet  Back-off restarting failed container
```
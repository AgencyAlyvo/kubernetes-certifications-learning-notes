# Ephemeral Containers

## C'est quoi un ephemeral container ?

- Un ephemeral container est un container **temporaire** qu'on ajoute à un `Pod` **déjà en cours d'exécution**, pour l'inspecter ou le déboguer. Il ne sert jamais à faire tourner une application.
- Pourquoi ça existe : un `Pod` est immuable, on ne peut pas lui ajouter un container après sa création. Pour inspecter un `Pod` qui tourne sans le recréer, l'ephemeral container est la seule option.
- Quand `kubectl exec` ne suffit pas :
  - le container applicatif a **crashé** (plus de processus dans lequel entrer) ;
  - l'image est **distroless** (pas de shell, pas de `ps`, `curl`, `netstat`…), ce qui est justement recommandé pour réduire la surface d'attaque (CKS).
- Il se crée avec `kubectl debug`, jamais dans un YAML.

```bash
# Ajouter un container busybox au Pod "web", attaché au container applicatif "app", et ouvrir un shell dedans
kubectl debug -it web --image=busybox:1.36 --target=app
```

<br /><br />

---

<br /><br />

## Comment ça marche ?

1. `kubectl debug` appelle un endpoint spécial de l'API (`ephemeralcontainers`), pas `pod.spec` : impossible de l'ajouter avec `kubectl edit` ou `kubectl apply`.
2. Le `kubelet` démarre le ephemeral container dans le `Pod`, à côté des autres.
3. Avec `--target=<container-applicatif>`, l'ephemeral container partage le **namespace de processus** du container applicatif visé : depuis l'ephemeral container, `ps` montre les processus du container applicatif, et `/proc/<pid>/root/` donne accès à son système de fichiers.
4. Quand on quitte le shell, le container se termine. Il reste visible dans le `Pod` (état `Terminated`) : on ne peut ni le modifier ni le supprimer, il disparaît avec le `Pod`.

```bash
kubectl debug -it web --image=busybox:1.36 --target=app
# / # ps aux                          → les processus de "app" sont visibles grâce à --target
# / # ls /proc/1/root/etc/nginx/      → le système de fichiers de "app", via son PID

kubectl describe pod web              # section "Ephemeral Containers" : le container et son état
kubectl get pod web -o jsonpath='{.spec.ephemeralContainers[*].name}'
```

<br /><br />

---

<br /><br />

## Autres usages de kubectl debug

```bash
kubectl debug -it web --image=busybox:1.36 --target=app         # ephemeral container dans le Pod (ce qu'on vient de voir)
kubectl debug web -it --copy-to=web-debug --container=app -- sh # copie du Pod avec une commande modifiée (pas un ephemeral container)
kubectl debug node/node01 -it --image=busybox:1.36              # Pod de debug sur un Node, avec son filesystem monté dans /host
```
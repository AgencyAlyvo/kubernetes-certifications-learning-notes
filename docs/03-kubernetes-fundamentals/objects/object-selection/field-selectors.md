# Field selectors

## C'est quoi un field selector ?

- Un `field selector` sélectionne des objets selon la `value` d'un ou plusieurs **champs** de la ressource (`metadata.name`, `status.phase`…), et non selon leurs `labels`.
- Utilisable avec `kubectl` (`--field-selector`) et l'API, mais **pas** dans le YAML d'un objet (contrairement au `label selector`).

```bash
kubectl get pods --field-selector status.phase=Running    # Pods dont le champ status.phase vaut Running
```

---

## Quels opérateurs ?

- `=`, `==` et `!=` uniquement.
- ⚠️ Pas de set-based (`in`, `notin`, `exists`).
- Pour combiner plusieurs conditions, on les sépare par `,` : un objet est sélectionné seulement s'il remplit **toutes** les conditions (ET logique).

```bash
kubectl get services -A --field-selector metadata.namespace!=default                # Services hors du namespace default

kubectl get pods --field-selector=status.phase!=Running,spec.restartPolicy=Always   # pas Running ET restartPolicy Always
```

---

## Quels champs sont supportés ?

- Ça dépend du type de ressource. Tous les types supportent `metadata.name` et `metadata.namespace`.
- Un champ non supporté → `Error from server (BadRequest): ... "foo.bar" is not a known field selector`.

| Kind | Champs |
|---|---|
| `Pod` | `spec.nodeName`, `spec.restartPolicy`, `spec.schedulerName`, `spec.serviceAccountName`, `spec.hostNetwork`, `status.phase`, `status.podIP`, `status.podIPs`, `status.nominatedNodeName` |
| `Event` | `involvedObject.kind`, `involvedObject.namespace`, `involvedObject.name`, `involvedObject.uid`, `involvedObject.apiVersion`, `involvedObject.resourceVersion`, `involvedObject.fieldPath`, `reason`, `reportingComponent`, `source`, `type` |
| `Secret` | `type` |
| `Service` | `spec.clusterIP`, `spec.type` |
| `Namespace` | `status.phase` |
| `ReplicaSet` | `status.replicas` |
| `Job` | `status.successful` |
| `Node` | `spec.unschedulable` |
| `CertificateSigningRequest` | `spec.signerName` |

- Custom resources : `metadata.name` et `metadata.namespace`, plus les champs déclarés dans `spec.versions[*].selectableFields` de la `CustomResourceDefinition`.

---

## Exemples utiles pour l'examen

```bash
kubectl get pods -A --field-selector spec.nodeName=node01                     # Pods qui tournent sur un node
kubectl get pods -A --field-selector status.phase!=Running                    # Pods qui ne sont pas Running
kubectl get events --field-selector type=Warning                              # Events Warning seulement
kubectl get events --field-selector involvedObject.name=mypod                 # Events d'un objet précis
kubectl get secrets -A --field-selector type=kubernetes.io/tls                # Secrets TLS
kubectl get svc -A --field-selector spec.type=LoadBalancer                    # Services LoadBalancer
kubectl get pods -l app=nginx --field-selector status.phase=Running           # combinable avec -l
```

---

## Label selector ou field selector ?

| | `label selector` (`-l`) | `field selector` (`--field-selector`) |
|---|---|---|
| Sélectionne sur | les `labels` (`metadata.labels`) | les champs de l'objet (`metadata.name`, `spec.*`, `status.*`) |
| Opérateurs | `=`, `!=`, `in`, `notin`, `exists` | `=`, `!=` |
| Champs possibles | n'importe quel `label` | liste limitée, selon le type |
| Dans le YAML (`selector`) | ✅ | ❌ (`kubectl` / API seulement) |

---

## Résumé

- `field selector` = filtre sur la `value` d'un champ de l'objet (`status.phase=Running`, `metadata.namespace!=default`…).
- Opérateurs : `=`, `==`, `!=` seulement. `,` = ET. Pas de `in` / `notin`.
- Champs supportés limités selon le type ; `metadata.name` et `metadata.namespace` partout.
- `kubectl get pods --field-selector spec.nodeName=node01`, combinable avec `-l`.

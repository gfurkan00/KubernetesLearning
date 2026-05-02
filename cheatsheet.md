# Kubectl Cheatsheet completo

## 🌐 Flag globali (funzionano con quasi tutti i comandi)

| Flag | Cosa fa |
|------|---------|
| `-n <namespace>` | Specifica il namespace |
| `-A` / `--all-namespaces` | Tutti i namespace |
| `-o wide` | Output esteso (IP, nodo, ecc.) |
| `-o yaml` / `-o json` | Output completo in YAML/JSON |
| `-o name` | Solo i nomi (utile in script) |
| `-o jsonpath='{...}'` | Estrae campi specifici |
| `-o custom-columns=NAME:.metadata.name,...` | Colonne personalizzate |
| `-l <key>=<val>` | Filtra per label |
| `--field-selector status.phase=Running` | Filtra per campo |
| `-w` / `--watch` | Aggiorna in tempo reale |
| `--show-labels` | Mostra le label |
| `--context <ctx>` | Cambia contesto cluster |
| `-v=<level>` | Verbose (1-9), utile per debug API |
| `--dry-run=client -o yaml` | Genera YAML senza applicare |

---

## 📋 GET — leggere risorse

```bash
kubectl get pods                          # lista pod nel namespace corrente
kubectl get pods -o wide                  # + IP, nodo, NOMINATED NODE
kubectl get pods -A                       # tutti i namespace
kubectl get pods -n kube-system           # namespace specifico
kubectl get pods -w                       # watch (streaming)
kubectl get pods --show-labels            # mostra le label
kubectl get pods -l app=nginx             # filtra per label
kubectl get pods -l 'env in (prod,stg)'   # set-based selector
kubectl get pods --field-selector status.phase=Running
kubectl get pods --sort-by=.metadata.creationTimestamp
kubectl get pod nginx -o yaml             # YAML completo
kubectl get pod nginx -o jsonpath='{.status.podIP}'
kubectl get all                           # pod + svc + deploy + rs (no tutto)
kubectl get all -A                        # in tutti i ns
kubectl api-resources                     # tutti i tipi di risorse
kubectl api-resources --namespaced=true   # solo namespaced
kubectl explain pod.spec.containers       # documentazione campo
```

---

## 🔍 DESCRIBE — dettagli + eventi

```bash
kubectl describe pod nginx                # tutto: stato, events, volumi
kubectl describe node worker-1            # capacità, pod, condizioni
kubectl describe svc my-service
```

> `describe` è la prima cosa da guardare quando qualcosa non funziona — la sezione `Events` in fondo è oro.

---

## 🚀 CREATE / APPLY — creare risorse

```bash
kubectl apply -f manifest.yaml            # idempotente (preferito)
kubectl apply -f ./dir/                   # tutta la directory
kubectl apply -k ./overlay/               # Kustomize
kubectl create -f manifest.yaml           # fallisce se esiste
kubectl create deployment nginx --image=nginx
kubectl create namespace dev
kubectl create configmap my-cm --from-file=config.txt
kubectl create secret generic my-sec --from-literal=pass=1234
kubectl run debug --image=busybox -it --rm -- sh   # pod usa-e-getta
```

### Generare YAML al volo (utilissimo!)
```bash
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deploy.yaml
kubectl run nginx --image=nginx --dry-run=client -o yaml
kubectl create svc clusterip my-svc --tcp=80:8080 --dry-run=client -o yaml
```

---

## ✏️ EDIT / PATCH / SET — modificare

```bash
kubectl edit deployment nginx             # apre $EDITOR
kubectl set image deployment/nginx nginx=nginx:1.25  # cambia immagine
kubectl set env deployment/nginx KEY=value
kubectl set resources deployment nginx --limits=cpu=200m,memory=512Mi
kubectl scale deployment nginx --replicas=5
kubectl autoscale deployment nginx --min=2 --max=10 --cpu-percent=80
kubectl patch deployment nginx -p '{"spec":{"replicas":3}}'
kubectl label pod nginx env=prod          # aggiungi label
kubectl label pod nginx env-              # rimuovi label
kubectl annotate pod nginx note="ciao"
```

---

## 🗑️ DELETE

```bash
kubectl delete pod nginx
kubectl delete -f manifest.yaml
kubectl delete pods --all -n dev
kubectl delete pod nginx --grace-period=0 --force   # forza (usa con cautela)
kubectl delete pods -l app=nginx                    # per label
```

---

## 📜 LOGS — leggere log

```bash
kubectl logs nginx                        # log del pod
kubectl logs nginx -c sidecar             # container specifico
kubectl logs -f nginx                     # follow (tail -f)
kubectl logs --tail=100 nginx             # ultime N righe
kubectl logs --since=1h nginx             # ultima ora
kubectl logs --previous nginx             # log del container crashato
kubectl logs -l app=nginx --all-containers --max-log-requests=10
kubectl logs deployment/nginx             # log di un pod del deploy
```

---

## 💻 EXEC — eseguire comandi nel container

```bash
kubectl exec nginx -- ls /etc             # comando singolo
kubectl exec -it nginx -- sh              # shell interattiva
kubectl exec -it nginx -c sidecar -- bash # container specifico
kubectl exec nginx -- env                 # vedi le env vars
```

---

## 🌐 PORT-FORWARD / PROXY / CP

```bash
kubectl port-forward pod/nginx 8080:80              # locale:pod
kubectl port-forward svc/my-svc 8080:80
kubectl port-forward deployment/nginx 8080:80
kubectl proxy --port=8001                           # proxy verso API server
kubectl cp ./file.txt nginx:/tmp/file.txt           # copia in pod
kubectl cp nginx:/tmp/file.txt ./file.txt           # copia da pod
kubectl cp -c sidecar ns/pod:/path ./local
```

---

## 🔄 ROLLOUT — gestire deployment

```bash
kubectl rollout status deployment/nginx
kubectl rollout history deployment/nginx
kubectl rollout history deployment/nginx --revision=3
kubectl rollout undo deployment/nginx                # rollback
kubectl rollout undo deployment/nginx --to-revision=2
kubectl rollout restart deployment/nginx             # restart pod
kubectl rollout pause deployment/nginx
kubectl rollout resume deployment/nginx
```

---

## 🖥️ NODE — gestione nodi

```bash
kubectl get nodes -o wide
kubectl describe node worker-1
kubectl top node                          # CPU/RAM (richiede metrics-server)
kubectl top pod -A
kubectl cordon worker-1                   # marca non-schedulabile
kubectl uncordon worker-1
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data
kubectl taint nodes worker-1 dedicated=gpu:NoSchedule
```

---

## ⚙️ CONFIG — contesti e cluster

```bash
kubectl config get-contexts
kubectl config current-context
kubectl config use-context prod
kubectl config set-context --current --namespace=dev   # cambia ns default
kubectl config view
kubectl config view --minify                           # solo contesto attivo
```

---

## 🔐 AUTH / RBAC

```bash
kubectl auth can-i create pods                         # io posso?
kubectl auth can-i delete pods --as=user@example.com   # un altro?
kubectl auth can-i '*' '*' --all-namespaces            # admin?
kubectl auth whoami
```

---

## 📊 DEBUG / TROUBLESHOOTING

```bash
kubectl get events --sort-by=.lastTimestamp -A
kubectl get events --field-selector type=Warning
kubectl describe pod <name>                            # sezione Events
kubectl debug pod/nginx -it --image=busybox            # ephemeral container
kubectl debug node/worker-1 -it --image=ubuntu         # debug nodo
kubectl top pod --containers
kubectl get pod nginx -o yaml | grep -A 5 status
```

### Pattern utile: pod usa-e-getta per testare rete
```bash
kubectl run tmp --rm -it --image=nicolaka/netshoot -- bash
kubectl run curl --rm -it --image=curlimages/curl -- sh
```

---

## 🎯 Output personalizzato (jsonpath)

```bash
# Solo i nomi
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Pod + nodo dove gira
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}'

# Tutte le immagini in uso
kubectl get pods -A -o jsonpath='{.items[*].spec.containers[*].image}' | tr ' ' '\n' | sort -u

# Custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName
```

---

## ⚡ Alias e scorciatoie

I tipi di risorsa hanno alias brevi:

| Lungo | Corto |
|-------|-------|
| `pods` | `po` |
| `services` | `svc` |
| `deployments` | `deploy` |
| `replicasets` | `rs` |
| `namespaces` | `ns` |
| `configmaps` | `cm` |
| `persistentvolumes` | `pv` |
| `persistentvolumeclaims` | `pvc` |
| `ingresses` | `ing` |
| `statefulsets` | `sts` |
| `daemonsets` | `ds` |
| `nodes` | `no` |
| `serviceaccounts` | `sa` |

```bash
kubectl get po,svc,deploy -n dev          # combina più tipi
```

---

## 🔥 Comandi che userai 100 volte al giorno

```bash
kubectl get pods -o wide
kubectl describe pod <nome>
kubectl logs -f <pod>
kubectl exec -it <pod> -- sh
kubectl apply -f .
kubectl delete -f .
kubectl get events --sort-by=.lastTimestamp
kubectl rollout restart deployment/<nome>
kubectl config set-context --current --namespace=<ns>
```

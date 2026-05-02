# Adopting Sidecar Containers

Link: https://kubernetes.io/docs/tutorials/configuration/pod-sidecar-containers/

Sidecar container nativi (k8s ≥ 1.29, stable in 1.33): container ausiliari nel Pod gestiti da k8s con avvio sequenziale, spegnimento ordinato e compatibilità con i Job.

In una riga: **sidecar nativo = container in `spec.initContainers` con `restartPolicy: Always`**.

## Cose da capire

### Cos'è un Pod
Wrapper attorno a 1+ container che condividono network (stesso IP) e volumi. I container nello stesso Pod si parlano via `localhost`.

### Cos'è un sidecar
Container ausiliario nello stesso Pod del main container. Casi d'uso tipici:
- spedire log a un sistema esterno
- esporre metriche
- proxy per service mesh (Envoy/Istio)
- refresh certificati / sync file

Idea generale: una responsabilità per container. Il main fa il suo lavoro, i sidecar fanno il contorno.

### Vecchio stile vs nativo

| | Vecchio (pre-1.29) | Nativo (1.29+) |
|---|---|---|
| Dove | `spec.containers` | `spec.initContainers` con `restartPolicy: Always` |
| Avvio | parallelo al main → race condition | sequenziale, prima del main |
| Spegnimento | parallelo al main → log persi | dopo il main (LIFO) → graceful |
| Job | bloccava il completamento | non blocca il Job |

### Init container regolare vs sidecar (la chiave)

Stessa sezione del YAML (`spec.initContainers`), comportamento diverso a seconda del `restartPolicy`:

- **Init regolare** (senza `restartPolicy`): parte → fa il suo lavoro → termina con exit 0 → solo allora parte il prossimo. Per setup una-tantum.
- **Sidecar** (con `restartPolicy: Always`): parte → diventa "ready" → parte il prossimo (k8s non aspetta che termini, perché non terminerà). Resta running per tutta la vita del Pod.

### Ordine di avvio (top-down nel YAML)
```
1. Init container regolari    → uno alla volta, ognuno aspetta che il precedente termini (exit 0)
2. Sidecar                    → uno alla volta, ognuno aspetta che il precedente sia "ready"
3. Main container             → partono in parallelo, dopo che tutti gli init+sidecar sono pronti
```

### Ordine di spegnimento (LIFO)
```
1. Main container             → ricevono SIGTERM, si fermano
2. Sidecar                    → SOLO ORA SIGTERM, in ordine inverso
3. Pod terminato
```

In pratica: se il main scrive log fino all'ultimo, il sidecar logger è ancora vivo per spedirli. Se il main parla con un proxy, il proxy resta su finché il main non ha chiuso le connessioni.

### Perché funzionano nei Job

Un Job "completa" quando tutti i container in `spec.containers` escono con exit 0. I container in `spec.initContainers` (anche un sidecar in loop infinito) non contano. Quando il main termina, k8s ferma automaticamente i sidecar e il Pod va in `Completed`.

Esempio concreto: un Job di backup DB che usa Istio (sidecar Envoy per mTLS):
- Vecchio modo: backup finisce → Envoy continua il loop → Job mai completo → Pod fantasma fino a `kubectl delete` manuale.
- Nativo: backup finisce → k8s manda SIGTERM a Envoy → Job `Complete` → Pod `Completed`. Pulito.

## I 3 pattern visti nel tutorial

### 1. Sidecar minimo (1 main + 1 sidecar)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-test
spec:
  containers:
  - name: app
    image: nginx:1.27
  initContainers:
  - name: my-sidecar
    image: busybox:1.36
    restartPolicy: Always
    command: ['sh', '-c', 'while true; do echo "Sidecar alive"; sleep 5; done']
```

`kubectl describe pod sidecar-test` deve mostrare `Restart Policy: Always` nell'init container. Se non c'è, un mutating webhook l'ha strippato.

### 2. Pod completo: init regolare + sidecar multipli + main

```yaml
spec:
  containers:
  - name: web-app
    image: nginx:1.27

  initContainers:
  # Init regolare: parte, fa la sua cosa, termina
  - name: setup
    image: busybox:1.36
    command: ['sh', '-c', 'echo "Setup done"; sleep 2']

  # Sidecar 1
  - name: log-aggregator
    image: busybox:1.36
    restartPolicy: Always
    command: ['sh', '-c', 'while true; do echo "log-aggregator running"; sleep 5; done']

  # Sidecar 2
  - name: metrics-exporter
    image: busybox:1.36
    restartPolicy: Always
    command: ['sh', '-c', 'while true; do echo "processes=$(ps aux | wc -l)"; sleep 10; done']
```

Per vedere l'ordine di avvio:
```bash
kubectl get pod app-with-sidecars -w   # in un terminale separato, prima dell'apply
kubectl apply -f app-with-sidecars.yaml
# READY: 0/3 → 1/3 → 2/3 → 3/3
```

Sequenza interna: `setup` (Completed) → `log-aggregator` (Ready) → `metrics-exporter` (Ready) → `web-app` (Ready).

### 3. Sidecar in un Job (il caso killer)

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-with-sidecar
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: worker
        image: busybox:1.36
        command: ['sh', '-c', 'echo "Job iniziato"; sleep 5; echo "Job finito"; exit 0']
      initContainers:
      - name: helper-sidecar
        image: busybox:1.36
        restartPolicy: Always               # sidecar in loop infinito
        command: ['sh', '-c', 'while true; do echo "sidecar still here"; sleep 2; done']
```

```bash
kubectl apply -f job-with-sidecar.yaml
# dopo ~7s:
kubectl get job job-with-sidecar               # STATUS: Complete, 1/1
kubectl get pod -l job-name=job-with-sidecar   # STATUS: Completed
kubectl logs -l job-name=job-with-sidecar -c helper-sidecar
```

Senza `restartPolicy: Always` (cioè spostando `helper-sidecar` in `spec.containers`), lo stesso Job rimarrebbe in `Running` per sempre.

### Cleanup

```bash
kubectl delete pod sidecar-test app-with-sidecars 2>/dev/null
kubectl delete job job-with-sidecar 2>/dev/null
rm sidecar-test.yaml app-with-sidecars.yaml job-with-sidecar.yaml
```

## Gotcha

- **Posto giusto**: `spec.initContainers` + `restartPolicy: Always`. Non `spec.containers`.
- **`restartPolicy` ha due significati diversi**:
  - sul Pod (`spec.restartPolicy`) → policy per i main container
  - sull'init container → marker che lo fa diventare un sidecar
- **Cluster < 1.29**: il `restartPolicy` su un init container viene ignorato → l'init si comporta da init normale (con un loop infinito blocca tutto in `Init:0/N`).
- **Mutating webhook**: alcune versioni vecchie di tool (es. service mesh) fanno full update del Pod e perdono il `restartPolicy`. Su Minikube vanilla non capita.
- **`kubectl apply` non aggiorna un Pod esistente con lo stesso nome**: se modifichi un Pod nudo, devi fare `kubectl delete pod NAME && kubectl apply -f`. Per i Deployment basta `rollout restart`.
- **Errore frequente nei `command`**: in YAML inline list, `'sh','c','...'` (manca il `-`) fa partire `sh` che cerca un file chiamato `c` → `Exit Code 2` → `CrashLoopBackOff`. Sintassi corretta: `'sh', '-c', '...'`. Brutta esperienza vista dal vivo.
- **Debug rapido di un sidecar che crasha**: `kubectl describe pod NAME` (cerca `Last State`, `Exit Code`) + `kubectl logs NAME -c SIDECAR --previous`.

## Cheat

```bash
# Inspect
kubectl get pod NAME
kubectl get pod NAME -w                                  # watch in tempo reale
kubectl describe pod NAME                                # cerca "Restart Policy: Always"
kubectl logs NAME -c CONTAINER
kubectl logs NAME -c CONTAINER --previous                # log dell'istanza precedente
kubectl logs NAME -c CONTAINER --tail=N

# Job
kubectl get job NAME
kubectl get pod -l job-name=NAME
kubectl logs -l job-name=NAME -c CONTAINER

# Refresh di un Pod nudo
kubectl delete pod NAME && kubectl apply -f file.yaml

# Verifica feature gate (per cluster custom)
kubectl get --raw /metrics | grep kubernetes_feature_enabled | grep SidecarContainers
```

# Configuring Redis using a ConfigMap

Link: https://kubernetes.io/docs/tutorials/configuration/configure-redis-using-configmap/

Configurare Redis con un ConfigMap che contiene `redis.conf` montato come file dentro il Pod. Il punto centrale: Redis legge il file di config solo all'avvio, quindi modifiche al ConfigMap richiedono di ricreare il Pod.

## Concetti

- Pod singolo (no Deployment) → "restart" significa `kubectl delete` + `kubectl apply`
- ConfigMap come volume con `items` → mappa una chiave specifica del ConfigMap a un path specifico nel container
- `emptyDir` → volume temporaneo, vive quanto il Pod
- `exec -it` → esegue un comando interattivo dentro un container in esecuzione
- Volume sempre aggiornato, app no → kubelet aggiorna il file nel volume entro ~1 min, ma se l'app non rilegge la config attiva non cambia

## Flow

### 1. ConfigMap iniziale + Pod Redis

`example-redis-config.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-redis-config
data:
  redis-config: ""
```

```bash
kubectl apply -f example-redis-config.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
```

### 2. Manifest del Pod Redis

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
  - name: redis
    image: redis:8.0.2
    command: ["redis-server", "/redis-master/redis.conf"]   # legge il file SOLO all'avvio
    ports:
    - containerPort: 6379
    volumeMounts:
    - mountPath: /redis-master-data
      name: data                                            # storage temporaneo
    - mountPath: /redis-master
      name: config                                          # qui finisce redis.conf
  volumes:
    - name: data
      emptyDir: {}
    - name: config
      configMap:
        name: example-redis-config
        items:
        - key: redis-config                                 # chiave del ConfigMap
          path: redis.conf                                  # diventa /redis-master/redis.conf
```

### 3. Ispezione iniziale

```bash
kubectl get pod/redis configmap/example-redis-config
kubectl describe configmap/example-redis-config

# Entra in Redis CLI
kubectl exec -it pod/redis -- redis-cli
> CONFIG GET maxmemory             # "0"           (default: nessun limite)
> CONFIG GET maxmemory-policy      # "noeviction"  (errore alla scrittura quando piena)
> exit
```

### 4. Aggiornare il ConfigMap

`example-redis-config.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-redis-config
data:
  redis-config: |                  # block scalar: preserva i newline
    maxmemory 2mb
    maxmemory-policy allkeys-lru
```

```bash
kubectl apply -f example-redis-config.yaml
kubectl describe configmap/example-redis-config   # ora contiene i 2 valori
```

### 5. Verificare che Redis NON ha ancora preso la nuova config

```bash
kubectl exec -it pod/redis -- redis-cli
> CONFIG GET maxmemory             # ancora "0" — il file nel volume è aggiornato, ma Redis no
> exit

# Per verificare che il file SIA aggiornato:
kubectl exec pod/redis -- cat /redis-master/redis.conf
```

### 6. Ricreare il Pod e verificare

```bash
kubectl delete pod redis
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml

kubectl exec -it pod/redis -- redis-cli
> CONFIG GET maxmemory             # "2097152"     (2 MB in byte)
> CONFIG GET maxmemory-policy      # "allkeys-lru"
> exit
```

### Cleanup

```bash
kubectl delete pod/redis configmap/example-redis-config
rm example-redis-config.yaml
```

## Note

- Pod vs Deployment: con un Deployment basterebbe `kubectl rollout restart deployment/NAME`. Con un Pod nudo serve delete + apply.
- `items` nel volume ConfigMap: utile per esporre solo alcune chiavi con path specifici. Senza `items`, ogni chiave diventa un file omonimo nel `mountPath`.
- Block scalar YAML (`|`): preserva i newline. Alternativa `>` collassa in spazi. Per file di config: `|`.
- Il `kubectl exec ... -- cat /redis-master/redis.conf` mostra il file aggiornato anche se Redis ha la vecchia config attiva — è la dimostrazione concreta del punto chiave del tutorial.
- `CONFIG SET` di Redis cambia la config a runtime ma non riscrive il file. Best practice: `redis.conf` per i default, `CONFIG SET` per tweak temporanei.

## Cheat

```bash
# ConfigMap da YAML
kubectl apply -f example-redis-config.yaml
kubectl describe configmap/NAME

# Pod
kubectl get pod/redis
kubectl delete pod redis && kubectl apply -f <url>     # "restart" di un Pod nudo

# Inspect runtime
kubectl exec -it pod/redis -- redis-cli                # interactive
kubectl exec pod/redis -- redis-cli CONFIG GET maxmemory   # one-shot
kubectl exec pod/redis -- cat /redis-master/redis.conf     # leggi file montato
```

# Configure a Pod to Use a PersistentVolume for Storage

Link: https://kubernetes.io/docs/tutorials/configuration/configure-persistent-volume-storage/

Storage persistente in Kubernetes: i container sono effimeri, ma i dati possono sopravvivere al loro ciclo di vita grazie a PersistentVolume (PV) e PersistentVolumeClaim (PVC).

## Idea centrale (3 livelli, 3 risorse separate)

```
[Admin del cluster]              [Sviluppatore]                  [Sviluppatore]
        ↓                              ↓                              ↓
 PersistentVolume (PV)  ←─bind─  PersistentVolumeClaim (PVC) ←─usa─ Pod
  (storage fisico)              (richiesta di storage)         (lo monta)
```

L'admin fornisce lo storage (PV), lo sviluppatore lo richiede via PVC, k8s fa il binding automatico, il Pod monta il PVC senza sapere niente del PV sottostante. Lifecycle indipendenti → i dati vivono finché vive il PV, non il Pod.

## Glossario

| Termine | Cosa è |
|---|---|
| PersistentVolume (PV) | risorsa cluster-wide che rappresenta uno spazio di storage fisico (hostPath, NFS, EBS, ...) |
| PersistentVolumeClaim (PVC) | richiesta namespaced di storage. Si bind-a a un PV compatibile |
| StorageClass | "etichetta" per categorizzare lo storage. PV e PVC devono usare la stessa per il binding (qui: `manual`) |
| `hostPath` | tipo di PV che usa una directory del filesystem del nodo. Solo per dev/test |
| Capacity | quanto spazio offre il PV / chiede il PVC. PV ≥ PVC |
| Access Modes | come il volume può essere montato. Devono essere compatibili tra PV e PVC |
| Reclaim Policy | cosa succede al PV quando si cancella il PVC: `Retain` (dati restano), `Delete` (cancellati) |

### Access Modes

- **ReadWriteOnce (RWO)** — un solo nodo monta in r/w. Il caso più comune.
- **ReadOnlyMany (ROX)** — tanti nodi montano in sola lettura.
- **ReadWriteMany (RWX)** — tanti nodi montano in r/w (raro, richiede storage che lo supporti es. NFS).
- **ReadWriteOncePod** — un singolo Pod (non solo nodo) monta in r/w. Più sicuro.

### Binding: come k8s associa PVC ↔ PV

Quando creo un PVC, k8s cerca un PV con:
1. stessa `storageClassName`
2. capacity ≥ a quella richiesta dal PVC
3. accessModes compatibili

Match → entrambi vanno in `Bound`. Una volta bound il legame è permanente fino alla cancellazione del PVC.

### Reclaim Policy

- **Retain** (default per PV manuali): cancello il PVC → il PV resta, i dati restano. L'admin pulisce a mano.
- **Delete**: cancello il PVC → PV cancellato e dati persi. Comodo ma pericoloso in produzione.

## Flow

### 1. Setup nodo Minikube (creo un file fisicamente)

```bash
minikube ssh
sudo mkdir -p /mnt/data
sudo sh -c "echo 'Hello from Kubernetes storage' > /mnt/data/index.html"
exit
```

### 2. PersistentVolume — chi fornisce lo storage

`pv-volume.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: task-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual            # chiave per il match col PVC
  capacity:
    storage: 10Gi
  accessModes:                        # camelCase! e attenzione allo spazio dopo "-"
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
```

```bash
kubectl apply -f pv-volume.yaml
kubectl get pv task-pv-volume        # STATUS: Available, CLAIM vuoto
```

### 3. PersistentVolumeClaim — chi richiede lo storage

`pv-claim.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pv-claim
spec:
  storageClassName: manual            # stessa del PV → match
  accessModes:
    - ReadWriteOnce                   # stessi modes → match
  resources:
    requests:
      storage: 3Gi                    # PV ne offre 10 → match
```

```bash
kubectl apply -f pv-claim.yaml
kubectl get pv,pvc                    # entrambi in STATUS: Bound
```

Cosa interessante: il PVC vede l'intera capacità del PV bound (10Gi), non i 3Gi che ha chiesto. Con dynamic provisioning invece k8s creerebbe un PV nuovo della taglia esatta richiesta.

### 4. Pod — chi monta il PVC

`pv-pod.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  volumes:
    - name: task-pv-storage           # nome interno (qualunque)
      persistentVolumeClaim:
        claimName: task-pv-claim      # punta al PVC, non al PV
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage       # deve corrispondere a volumes[].name
```

```bash
kubectl apply -f pv-pod.yaml
kubectl get pod task-pv-pod           # 1/1 Running
```

### 5. Verificare che nginx serve il file

```bash
# Modo "completo" (testa la pipeline HTTP):
kubectl exec -it task-pv-pod -- /bin/bash
apt update && apt install -y curl
curl http://localhost/
# → "Hello from Kubernetes storage"

# Modo rapido (one-shot, senza installare niente):
kubectl exec task-pv-pod -- cat /usr/share/nginx/html/index.html
```

### 6. Dimostrazione della persistenza (il vero senso del tutorial)

```bash
kubectl delete pod task-pv-pod                                 # Pod morto
kubectl get pv,pvc                                             # PV e PVC ancora Bound
kubectl apply -f pv-pod.yaml                                   # Pod nuovo (container nginx fresh)
kubectl exec task-pv-pod -- cat /usr/share/nginx/html/index.html
# → "Hello from Kubernetes storage" (il file è ancora lì!)
```

I dati vivono nel PV (fisicamente in `/mnt/data` del nodo), il Pod li monta solo. Cancellando/ricreando il Pod, le modifiche persistono.

### Cleanup

```bash
# Ordine importante: prima il Pod, poi il PVC, poi il PV
kubectl delete pod task-pv-pod
kubectl delete pvc task-pv-claim
kubectl delete pv task-pv-volume

# Pulizia dati fisici sul nodo
minikube ssh
sudo rm -rf /mnt/data
exit

rm pv-volume.yaml pv-claim.yaml pv-pod.yaml
```

## Da ricordare

- `hostPath` non è per produzione: i dati vivono sul filesystem di un solo nodo. Pod rischedulato altrove → dati non trovati. In produzione: NFS, EBS, GCE PD, ecc.
- Binding una volta sola: il legame PVC↔PV è permanente fino alla cancellazione del PVC.
- Capacity ≠ uso reale: il PVC vede l'intera capacità del PV bound, non quella richiesta.
- `storageClassName`:
  - vuoto/omesso → usa lo StorageClass di default del cluster (di solito dynamic provisioning).
  - specificato (`manual`) → forza binding manuale, niente provisioning automatico.
- PVC `Pending` per sempre = nessun PV combacia. `kubectl describe pvc NAME` (sezione Events) per capire perché.
- Reclaim policy `Retain`: dopo aver cancellato il PVC, il PV resta in stato `Released` (non `Available`) finché un admin non fa cleanup. Per riusarlo va cancellato e ricreato.

## Errori YAML che ho visto

- Tutti i campi k8s sono in **camelCase** (`accessModes`, non `AccessModes`). Maiuscole iniziali → `unknown field` dall'API server.
- Il trattino di una list deve avere uno **spazio dopo**: `- ReadWriteOnce`, non `-ReadWriteOnce`. Senza spazio è una stringa, non un elemento di lista.

Per validare un YAML senza applicarlo:
```bash
kubectl apply --dry-run=client -f file.yaml     # lato client
kubectl apply --dry-run=server -f file.yaml     # lato API server (più rigorosa)
kubectl explain pv.spec.accessModes             # docs inline su un campo
```

## Cheat

```bash
# Inspect
kubectl get pv                                  # cluster-wide
kubectl get pvc                                 # nel namespace corrente
kubectl get pv,pvc
kubectl describe pvc NAME                       # eventi di binding (utile se Pending)

# Apply / delete
kubectl apply -f pv-volume.yaml
kubectl delete pvc NAME
kubectl delete pv NAME

# Test dentro il Pod
kubectl exec -it POD -- /bin/bash
kubectl exec POD -- cat /path/file

# Minikube node access
minikube ssh
```

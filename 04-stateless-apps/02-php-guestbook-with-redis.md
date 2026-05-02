# PHP Guestbook with Redis

Link: https://kubernetes.io/docs/tutorials/stateless-application/guestbook/

App multi-tier classica: frontend PHP (3 Pod) che scrive su Redis leader (1 Pod) e legge dai Redis follower (2 Pod). 6 risorse k8s in totale: 3 Deployment + 3 Service. È il primo tutorial dove si vede chiaramente il **service discovery via DNS interno** del cluster.

## Architettura

```
Browser → frontend Service (LoadBalancer) → frontend Pods (3 repliche, PHP)
                                              │
                              writes  ┌───────┴──────┐  reads
                                      ▼              ▼
                          redis-leader Service   redis-follower Service
                            (ClusterIP)            (ClusterIP)
                                │                      │
                                ▼                      ▼
                         redis-leader (1 Pod)    redis-follower (2 Pod)
```

Il frontend usa il **nome del Service** (`redis-leader`, `redis-follower`) per connettersi. Il DNS interno del cluster risolve i nomi in cluster-IP automaticamente.

## Concetti

- **Service discovery via DNS**: ogni Service ha un FQDN del tipo `NAME.NAMESPACE.svc.cluster.local`. Da un Pod nello stesso namespace basta `NAME` (es. `redis-leader`).
- **ClusterIP vs LoadBalancer**: backend (Redis) → ClusterIP, accessibile solo da dentro il cluster. Frontend → LoadBalancer, esposto fuori.
- **Label per separare ruoli**: stessa app `redis` ma con `role: leader` o `role: follower`. I Service usano selector specifici per distinguere quale gruppo di Pod targetizzare.
- **Replicas asimmetriche**: 1 leader (write), 2 follower (read replicas), 3 frontend.

## Flow

### 1. Redis Leader

```bash
kubectl apply -f https://k8s.io/examples/application/guestbook/redis-leader-deployment.yaml
kubectl apply -f https://k8s.io/examples/application/guestbook/redis-leader-service.yaml
```

Manifest del Deployment (parti chiave):
```yaml
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: redis
        role: leader
        tier: backend
    spec:
      containers:
      - name: leader
        image: registry.k8s.io/redis@sha256:cb111d1bd870a6a471385a4a69ad17469d326e9dd91e0e455350cacf36e1b3ee
        ports:
        - containerPort: 6379
```

Service ClusterIP (default `type` se omesso):
```yaml
spec:
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    app: redis
    role: leader            # solo Pod leader, non i follower
    tier: backend
```

### 2. Redis Follower (2 repliche)

```bash
kubectl apply -f https://k8s.io/examples/application/guestbook/redis-follower-deployment.yaml
kubectl apply -f https://k8s.io/examples/application/guestbook/redis-follower-service.yaml
```

Stesso pattern del leader, ma con `replicas: 2` e `role: follower`. L'image è preconfigurata per replicare il leader scoperto via DNS.

### 3. Frontend PHP (3 repliche, LoadBalancer)

```bash
kubectl apply -f https://k8s.io/examples/application/guestbook/frontend-deployment.yaml
kubectl apply -f https://k8s.io/examples/application/guestbook/frontend-service.yaml
```

Manifest del Deployment (parte chiave):
```yaml
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google-samples/gb-frontend:v4
        env:
        - name: GET_HOSTS_FROM
          value: dns                    # dice al PHP di scoprire Redis via DNS
        ports:
        - containerPort: 80
```

Service tipo LoadBalancer:
```yaml
spec:
  type: LoadBalancer
  ports:
  - port: 80
  selector:
    app: guestbook
    tier: frontend
```

Su Minikube `EXTERNAL-IP` resta `<pending>` — serve `minikube tunnel` o `minikube service` per accedere.

### 4. Accedere al frontend

3 modi su Minikube:

```bash
# Modo 1: port-forward (semplice, niente sudo)
kubectl port-forward svc/frontend 8080:80
# poi browser su http://localhost:8080

# Modo 2: minikube tunnel (richiede sudo, in altro terminale)
minikube tunnel
EXTERNAL_IP=$(kubectl get svc frontend -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "http://$EXTERNAL_IP"

# Modo 3: minikube service (apre browser)
minikube service frontend
```

Sul browser: scrivi un messaggio nel campo "Messages" e premi Submit. Aggiorna → il messaggio è ancora lì.

### 5. Scaling

```bash
kubectl scale deployment frontend --replicas=6
kubectl get deployment frontend         # 6/6
kubectl get pods -l app=guestbook,tier=frontend     # 6 Pod

# Riduco a 3
kubectl scale deployment frontend --replicas=3
```

Lo scaling del frontend non tocca i Pod di Redis. Le richieste vengono bilanciate sui nuovi frontend dal Service automaticamente.

### Cleanup

```bash
# Tutto in 2 comandi sfruttando le label
kubectl delete deployment,service -l app=guestbook
kubectl delete deployment,service -l app=redis

kubectl get all                         # solo il Service "kubernetes" di sistema
```

## Come funziona il service discovery

Il codice PHP del Guestbook (`/var/www/html/guestbook.php` dentro il Pod frontend) ha questa logica:

```php
$host = 'redis-leader';                                    // stringa, NON un IP
if (getenv('GET_HOSTS_FROM') == 'env') {
  $host = getenv('REDIS_LEADER_SERVICE_HOST');             // modalità legacy
}
$client = new Predis\Client(['host' => $host, 'port' => 6379]);
$client->set('guestbook', $_GET['value']);
```

Quando Predis apre la connessione TCP, dietro le quinte succede:

```
1. Predis → fsockopen('redis-leader', 6379)
2. libc → getaddrinfo('redis-leader')
3. libc legge /etc/resolv.conf:
     nameserver 10.96.0.10           ← cluster-IP del Service kube-dns
     search default.svc.cluster.local svc.cluster.local cluster.local
4. libc manda query DNS a 10.96.0.10:53
5. Pod CoreDNS riceve la query, cerca nella sua tabella interna
   (sincronizzata con i Service del cluster), risponde 10.103.247.62
6. Predis apre TCP a 10.103.247.62:6379 (cluster-IP del Service redis-leader)
7. kube-proxy NAT-ta verso il Pod redis-leader (10.244.x.x:6379)
```

L'app PHP **non chiama k8s**. Fa solo una chiamata DNS standard, identica a quella che farebbe in qualunque altro contesto. La differenza è che il nameserver punta a CoreDNS (un Pod gestito da k8s) invece che a un DNS pubblico/aziendale.

Verifica:
```bash
FRONTEND_POD=$(kubectl get pods -l app=guestbook,tier=frontend -o jsonpath='{.items[0].metadata.name}')
kubectl exec $FRONTEND_POD -- getent hosts redis-leader
# 10.103.247.62  redis-leader.default.svc.cluster.local

kubectl exec $FRONTEND_POD -- cat /etc/resolv.conf
# nameserver 10.96.0.10
# search default.svc.cluster.local svc.cluster.local cluster.local
```

`getent` è un tool generico Linux, non sa niente di PHP né di k8s. Ottiene lo stesso risultato di Predis perché il DNS lookup è una **operazione di sistema**, non dell'app.

## Modalità DNS vs ENV

Il codice del Guestbook supporta due modalità per scoprire Redis:

| | `GET_HOSTS_FROM=dns` (questo tutorial) | `GET_HOSTS_FROM=env` (legacy) |
|---|---|---|
| Cosa contiene `$host` | stringa `"redis-leader"` | IP `"10.103.247.62"` (env var iniettata da k8s al boot) |
| Quando viene risolto | a ogni connessione (lookup DNS) | una volta sola, al boot del Pod |
| Cosa succede se Service ricreato | continua a funzionare (DNS dinamico) | l'app si rompe (env fissa al boot) |
| Ordine di deploy importante | no | sì (Service prima del Pod) |

Le env var `REDIS_LEADER_SERVICE_HOST`, `REDIS_LEADER_SERVICE_PORT` ecc. vengono **comunque** iniettate da k8s in ogni Pod (per ogni Service esistente nel namespace al boot). Il PHP semplicemente le ignora se `GET_HOSTS_FROM=dns`.

## Note

- I dati del guestbook stanno in memoria (no PersistentVolume). Se i Pod Redis muoiono, i messaggi si perdono. In produzione si userebbe StatefulSet + PV.
- Il frontend non conosce gli IP dei Pod Redis: usa solo i nomi dei Service. Se Redis viene ricreato con un IP diverso, il PHP non si accorge di niente — il DNS lookup restituisce sempre il cluster-IP del Service (statico), e il Service inoltra al Pod corrente.
- `kubectl delete -l LABEL=VALUE` cancella gruppi di risorse in colpo solo. Comodo per app multi-componente che condividono una label.
- Il tutorial dice "minimum 2 nodes" ma su Minikube single-node funziona lo stesso (k8s schedula tutti i Pod sul singolo nodo). Il requisito multi-node è solo per simulare scenari di failover reali.

## Cheat

```bash
# Deploy completo
for f in redis-leader-deployment redis-leader-service \
         redis-follower-deployment redis-follower-service \
         frontend-deployment frontend-service; do
  kubectl apply -f https://k8s.io/examples/application/guestbook/$f.yaml
done

# Inspect
kubectl get all
kubectl get pods -l app=guestbook,tier=frontend
kubectl get pods -l app=redis,role=leader
kubectl get pods -l app=redis,role=follower

# Accesso
kubectl port-forward svc/frontend 8080:80
minikube service frontend                          # alternativa con browser

# Service discovery dal Pod
kubectl exec POD -- getent hosts redis-leader
kubectl exec POD -- cat /etc/resolv.conf

# Scaling
kubectl scale deployment frontend --replicas=N

# Cleanup
kubectl delete deployment,service -l app=guestbook
kubectl delete deployment,service -l app=redis
```

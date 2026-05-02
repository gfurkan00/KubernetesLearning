# Exposing an External IP Address

Link: https://kubernetes.io/docs/tutorials/stateless-application/expose-external-ip-address/

5 repliche di un'app Hello World, esposte tramite un Service di tipo `LoadBalancer`. Il tutorial originale assume un cloud provider, su Minikube serve `minikube tunnel` per ottenere un EXTERNAL-IP reale.

## Concetti

- **Service di tipo LoadBalancer**: chiede al cloud provider di creare un load balancer esterno con IP pubblico. Su AWS/GCP succede automatico, su Minikube serve `minikube tunnel`.
- **EXTERNAL-IP**: il campo del Service che mostra l'IP pubblico assegnato. Se è `<pending>` significa che il LB non è pronto (Minikube senza tunnel, oppure cloud lento).
- **LoadBalancer Ingress**: nel `describe`, l'IP esterno effettivo. Equivale all'EXTERNAL-IP.
- **Endpoints**: lista degli IP:porta dei Pod dietro al Service. Aggiornati in automatico quando i Pod cambiano.

## Flow

### 1. Deployment con 5 repliche

Manifest dell'esempio (vedi anche [`load-balancer-example.yaml`](./load-balancer-example.yaml)):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: load-balancer-example
  name: hello-world
spec:
  replicas: 5
  selector:
    matchLabels:
      app.kubernetes.io/name: load-balancer-example
  template:
    metadata:
      labels:
        app.kubernetes.io/name: load-balancer-example
    spec:
      containers:
      - image: gcr.io/google-samples/hello-app:2.0
        name: hello-world
        ports:
        - containerPort: 8080
```

```bash
kubectl apply -f https://k8s.io/examples/service/load-balancer-example.yaml

kubectl get deployments hello-world          # 5/5 ready
kubectl get rs                               # ReplicaSet creato dal Deployment
kubectl get pods -o wide                     # 5 Pod, ognuno con un suo IP
```

### 2. Service di tipo LoadBalancer

```bash
kubectl expose deployment hello-world --type=LoadBalancer --name=my-service

kubectl get services my-service
# EXTERNAL-IP: <pending>      ← su Minikube senza tunnel è normale
```

### 3. minikube tunnel per ottenere EXTERNAL-IP

In **un terminale separato** (richiede sudo, resta aperto):
```bash
minikube tunnel
```

Lascialo lì, poi nel terminale principale:
```bash
kubectl get services my-service              # ora EXTERNAL-IP ha un valore
```

### 4. Accedere all'app

Con `minikube tunnel` attivo:
```bash
EXTERNAL_IP=$(kubectl get svc my-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl http://$EXTERNAL_IP:8080
# Hello, world!
# Version: 2.0.0
# Hostname: hello-world-xxx-yyy
```

`Hostname` cambia a ogni richiesta → load balancing tra i 5 Pod.

Alternativa rapida senza tunnel:
```bash
minikube service my-service                  # apre nel browser
```

### 5. Verifica Endpoints

```bash
kubectl describe services my-service
# Cerca: LoadBalancer Ingress, Port, NodePort, Endpoints
```

Gli IP nella riga `Endpoints:` devono coincidere uno-a-uno con i Pod IP visti in `kubectl get pods -o wide`.

### Cleanup

```bash
kubectl delete service my-service
kubectl delete deployment hello-world
# Stoppa minikube tunnel con Ctrl+C nell'altro terminale
```

## Note

- Su cloud reale (GKE/AWS/Azure) `EXTERNAL-IP` viene assegnato automaticamente in 1-2 minuti.
- Su Minikube due strade:
  - `minikube tunnel` → IP esterno raggiungibile via curl come in cloud (più realistico)
  - `minikube service NAME` → apre nel browser, fa proxy temporaneo (più rapido)
- Il Service eredita il `selector` dalle label del Deployment (`app.kubernetes.io/name: load-balancer-example`). Se cambi le label o il selector senza match, il Service resta con `Endpoints: none` e nessuna richiesta arriva ai Pod.
- `kubectl expose` è imperativo. Per la versione dichiarativa basta aggiungere un `kind: Service` al manifest del Deployment (o un file separato).

## Cheat

```bash
kubectl expose deployment NAME --type=LoadBalancer --name=SVC_NAME
kubectl get svc SVC_NAME
kubectl describe svc SVC_NAME
kubectl get endpoints SVC_NAME

# Minikube
minikube tunnel                              # in un altro terminale, sudo
minikube service SVC_NAME                    # apri nel browser
minikube service SVC_NAME --url              # solo URL
```

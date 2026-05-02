# Expose Your App — Service

Link: https://kubernetes.io/docs/tutorials/kubernetes-basics/expose/expose-intro/

Cos'è un Service e perché ne abbiamo bisogno: i Pod nascono, muoiono, cambiano IP. Il Service è un endpoint stabile davanti ai Pod, fa load balancing, e li raggruppa via label.

## Perché serve

I Pod sono effimeri — quando muoiono e rinascono cambiano IP. Il client non può puntare a un IP che cambia. Il Service ha invece un IP stabile (cluster-IP) e fa da front-end ai Pod che hanno determinate label. Quando un Pod muore e rinasce, il Service aggiorna automaticamente la lista degli endpoint dietro le quinte.

## Label e Selector

- **Label**: coppie chiave-valore sui Pod, Deployment, Service, ecc. Es: `app=kubernetes-bootcamp`.
- **Selector**: il Service dice quali Pod vuole davanti includendo un selector di label.

Le label sono il meccanismo di "raggruppamento" universale in k8s — si usano per Service, Deployment, NetworkPolicy, ecc.

## I 4 tipi di Service

| Type | Espone su | Quando |
|---|---|---|
| ClusterIP (default) | IP interno al cluster | comunicazione tra microservizi nello stesso cluster |
| NodePort | porta su tutti i Node (30000-32767) | dev/test, accesso esterno semplice |
| LoadBalancer | IP esterno dal cloud (AWS ELB, GCP LB...) | produzione su cloud |
| ExternalName | alias DNS verso un nome esterno | mappare servizi esterni (es. DB SaaS) |

## Comandi

```bash
# Esporre il Deployment come Service NodePort
kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080
```

(versione dichiarativa: [`kubernetes-bootcamp-service.yaml`](./kubernetes-bootcamp-service.yaml))

```bash
# Verifica
kubectl get services
# NAME                  TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)          AGE
# kubernetes-bootcamp   NodePort    10.x.x.x     <none>        8080:31234/TCP   30s

kubectl describe service/kubernetes-bootcamp
# Nei dettagli: Selector (quali label cerca), Endpoints (IP:porta dei Pod che matchano)

# Accesso esterno
export NODE_PORT=$(kubectl get services/kubernetes-bootcamp \
  -o go-template='{{(index .spec.ports 0).nodePort}}')
curl http://$(minikube ip):$NODE_PORT

# Filtri per label
kubectl get pods -l app=kubernetes-bootcamp
kubectl get services -l app=kubernetes-bootcamp

# Aggiungere un label al volo a un Pod
kubectl label pod $POD_NAME version=v1
kubectl describe pod $POD_NAME

# Cancellare un Service (i Pod restano vivi)
kubectl delete service kubernetes-bootcamp
```

## Note

- `kubectl expose` legge il selector dal Deployment: se il Deployment ha `app=NAME`, il Service eredita `selector: app=NAME`. Per selector custom usare la versione dichiarativa.
- `port` vs `targetPort` vs `nodePort`:
  - `port`: porta del Service (cluster-IP)
  - `targetPort`: porta del Pod (default = port)
  - `nodePort`: porta esposta sul Node (per `type: NodePort`, default in 30000-32767)
- Su Minikube `type: LoadBalancer` non funziona davvero (manca il cloud LB). `EXTERNAL-IP` resta `<pending>`. Per accedere uso `minikube tunnel` oppure `minikube service NAME`.
- DNS interno: ogni Service è raggiungibile da qualsiasi Pod come `NAME.NAMESPACE.svc.cluster.local`. Per Service nello stesso namespace basta `NAME` (es. `http://kubernetes-bootcamp:8080`).

## Cheat rapida

```bash
kubectl expose deployment NAME --type=NodePort --port=8080
kubectl get svc                                        # alias per services
kubectl get endpoints NAME                             # vedi i Pod targetizzati

# Label
kubectl get pods -l KEY=VALUE
kubectl label pods POD KEY=VALUE
kubectl label pods POD KEY-                            # rimuovere

# Minikube
minikube service NAME --url
minikube ip
```

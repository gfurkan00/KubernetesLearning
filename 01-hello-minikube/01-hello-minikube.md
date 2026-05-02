# Hello Minikube

Link: https://kubernetes.io/docs/tutorials/hello-minikube/

Il "Hello World" di Kubernetes — avvio un cluster locale con Minikube, deploy di un'app demo (`agnhost`), Service per accederci dal browser.

## Cose da ricordare

- **Pod**: unità più piccola di k8s, contiene 1+ container che condividono rete e volumi.
- **Deployment**: gestisce i Pod (li ricrea se muoiono, scaling, rolling update).
- **Service**: endpoint stabile davanti ai Pod (i Pod cambiano IP, il Service no).
- **Addon**: estensioni di Minikube già pronte (dashboard, metrics-server, ingress, ecc.).

## Comandi

```bash
# Cluster
minikube start
minikube status
minikube dashboard            # apre la dashboard nel browser

# Crea il deployment con immagine demo
kubectl create deployment hello-node \
  --image=registry.k8s.io/e2e-test-images/agnhost:2.53 \
  -- /agnhost netexec --http-port=8080

# Verifiche
kubectl get deployments
kubectl get pods
kubectl logs <pod-name>

# Esporre come Service e accedere
kubectl expose deployment hello-node --type=LoadBalancer --port=8080
kubectl get services
minikube service hello-node   # apre direttamente nel browser

# Addon
minikube addons list
minikube addons enable metrics-server
```

Per la versione dichiarativa di Deployment + Service vedi [`hello-node-deployment.yaml`](./hello-node-deployment.yaml).

## Note

- `minikube stop` ferma ma mantiene lo stato. `minikube delete` cancella tutto, devi ripartire da zero.
- Su Minikube `type: LoadBalancer` non crea un vero load balancer (non c'è cloud sotto), quindi `EXTERNAL-IP` resta `<pending>`. Per accedere comunque uso `minikube service NAME`.
- Il `--` nel `kubectl create deployment` separa gli args di kubectl da quelli passati al container.

## Cleanup

```bash
kubectl delete service hello-node
kubectl delete deployment hello-node
minikube stop
```

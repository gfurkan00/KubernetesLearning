# Scale Your App

Link: https://kubernetes.io/docs/tutorials/kubernetes-basics/scale/scale-intro/

Scaling di un Deployment cambiando il numero di repliche. Con il Service che già c'è davanti, il traffico viene distribuito automaticamente tra i Pod.

## Idee fondamentali

- **Scale out** = più repliche dello stesso Pod
- **Scale in** = ridurre repliche
- **Scale to zero** = 0 Pod attivi (il Deployment esiste ma non c'è nessun Pod che gira)

Lo scaling cambia solo `spec.replicas` del Deployment. K8s si occupa del resto.

## ReplicaSet

Quando crei un Deployment, k8s crea automaticamente un ReplicaSet (RS) che a sua volta gestisce le repliche dei Pod:

```
Deployment (descrive)  →  ReplicaSet (gestisce repliche)  →  Pods
```

Tu non interagisci quasi mai col ReplicaSet — usi sempre il Deployment. Quando fai un rolling update (tutorial 06), il Deployment crea un nuovo RS per la nuova versione e scala giù il vecchio.

## Comandi

```bash
# Stato iniziale
kubectl get deployments
kubectl get rs                                # vedere il ReplicaSet sottostante

# Scalo a 4 repliche
kubectl scale deployments/kubernetes-bootcamp --replicas=4

kubectl get deployments                       # READY: 4/4
kubectl get pods -o wide                      # 4 Pod con IP diversi

# Test del load balancing
export NODE_PORT=$(kubectl get services/kubernetes-bootcamp \
  -o go-template='{{(index .spec.ports 0).nodePort}}')

for i in {1..8}; do curl http://$(minikube ip):$NODE_PORT; echo; done
# le risposte mostrano "Running on: ...AAAA", "...BBBB", "...CCCC", "...DDDD" alternati
# = round-robin del Service tra i 4 Pod

# Scalo giù a 2
kubectl scale deployments/kubernetes-bootcamp --replicas=2
kubectl get pods -o wide                      # 2 in Terminating, 2 restanti
```

## Note

- `kubectl scale` è imperativo. In produzione si modifica il YAML (campo `spec.replicas`) e si fa `kubectl apply`.
- Per scaling automatico c'è l'**HPA** (Horizontal Pod Autoscaler): scala in base a CPU/memoria/metriche custom. Es:
  ```bash
  kubectl autoscale deployment NAME --min=2 --max=10 --cpu-percent=70
  kubectl get hpa
  ```
  Richiede `metrics-server` installato sul cluster.
- Scale-to-zero non significa "stop": il Deployment c'è ancora, ma il Service ha 0 endpoint → richieste falliscono.
- Se i Node non hanno risorse (CPU/mem) per i nuovi Pod, restano in `Pending`. `kubectl describe pod POD` mostra `FailedScheduling` con il motivo.

## Algoritmo del load balancing

Il Service distribuisce il traffico tra tutti i Pod che matchano il suo selector. Su `kube-proxy` in modalità iptables (default) è quasi-random ma uniforme nel tempo — non è esattamente round-robin "ordinato" ma di fatto la distribuzione è equa.

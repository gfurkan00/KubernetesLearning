# Create a Cluster (con Minikube)

Link: https://kubernetes.io/docs/tutorials/kubernetes-basics/create-cluster/cluster-intro/

Cosa è un cluster k8s e come avvio il primo cluster locale.

## Cluster Kubernetes — cosa è

Un cluster è un insieme di macchine ("nodes") che fanno girare workload containerizzati in modo coordinato. K8s automatizza:

- scheduling dei container sui nodi
- self-healing (riavvia i Pod morti, ne crea di nuovi su altri nodi se un nodo cade)
- scaling orizzontale
- rollout di nuove versioni senza downtime

## Architettura: due tipi di componenti

```
Control Plane                    Worker Nodes
(il cervello)                    (eseguono i Pod)
- API server                     - kubelet (agente)
- Scheduler                      - container runtime
- Controller manager             - kube-proxy
- etcd
```

Il control plane prende le decisioni (cosa schedulare, dove, quando ri-creare un Pod morto). I worker nodes eseguono i container effettivi. In produzione: minimo 3 worker per ridondanza, e il control plane su 3+ macchine per HA.

## Minikube

Cluster k8s "tutto-in-uno" su una macchina sola — control plane e worker insieme, gira come VM o container Docker. Per dev/test/learning, **non per produzione**.

## Comandi

```bash
minikube start                 # primo avvio scarica le immagini, ci mette qualche minuto
kubectl version                # versione client + server
kubectl cluster-info           # endpoint del control plane
kubectl get nodes              # nodi del cluster (su Minikube: 1)
```

Su Minikube il singolo nodo fa sia da control plane sia da worker. Output di `get nodes`:

```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   1m    v1.35.1
```

## Note

- Driver di Minikube: di default usa `docker` se disponibile. Per forzare: `minikube start --driver=virtualbox` (o kvm2, hyperkit). Su Linux `docker` di solito è il più rapido.
- Multi-node per simulare un cluster vero: `minikube start --nodes=3`.
- Se `kubectl cluster-info` non risponde, il control plane non è raggiungibile — controllare `minikube status`.

## Quick reference

```bash
minikube start [--driver=docker] [--nodes=N]
minikube stop                  # ferma, mantiene lo stato
minikube delete                # cancella il cluster
minikube status

kubectl get nodes -o wide      # con IP, OS, versione kubelet
```

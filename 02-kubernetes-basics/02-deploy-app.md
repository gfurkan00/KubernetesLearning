# Deploy an App

Link: https://kubernetes.io/docs/tutorials/kubernetes-basics/deploy-app/deploy-intro/

Primo deploy con `kubectl create deployment`. Differenza Deployment vs Pod e accesso "a basso livello" via `kubectl proxy`.

## Deployment vs Pod

```
Deployment  →  ReplicaSet  →  Pod  →  Container
(astrazione    (gestisce      (unità    (l'app vera)
 alta)         le repliche)   minima)
```

- **Pod**: unità minima k8s. Contiene 1+ container, ha un suo IP, condivide network e volumi tra i container interni.
- **Deployment**: oggetto di alto livello che dichiara "voglio N repliche di questo Pod sempre running". Si occupa di crearli, sostituirli quando muoiono, rilanciarli su altri nodi se un nodo cade, aggiornarli senza downtime.

In produzione **non si crea mai un Pod a mano** — sempre Deployment (o StatefulSet/DaemonSet/Job a seconda del caso). Un Pod nudo non si riavvia da solo.

## Comandi

```bash
# Verifica cluster
kubectl version
kubectl get nodes

# Crea il Deployment (modo imperativo)
kubectl create deployment kubernetes-bootcamp \
  --image=gcr.io/google-samples/kubernetes-bootcamp:v1

kubectl get deployments
# NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
# kubernetes-bootcamp    1/1     1            1           30s
```

Per la versione dichiarativa: [`kubernetes-bootcamp-deployment.yaml`](./kubernetes-bootcamp-deployment.yaml).

## Accesso al Pod via proxy (debug)

I Pod stanno su rete privata interna al cluster, di default non sono raggiungibili da fuori. `kubectl proxy` apre un tunnel locale verso l'API server, utile per debug.

In un terminale a parte (resta aperto):
```bash
kubectl proxy                 # tunnel su localhost:8001
```

Nell'altro:
```bash
curl http://localhost:8001/version

# Salvo nome del Pod in variabile
export POD_NAME=$(kubectl get pods -o go-template \
  --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')

curl http://localhost:8001/api/v1/namespaces/default/pods/$POD_NAME:8080/proxy/
# → "Hello Kubernetes bootcamp! | Running on: ... | v=1"
```

In produzione `kubectl proxy` non si usa: per esporre l'app serve un Service (vedi tutorial 04).

## Note

- Imperativo (`kubectl create`) vs dichiarativo (`kubectl apply -f`): in produzione/CI sempre dichiarativo, perché versioni il YAML in git.
- Per ottenere un YAML partendo da un comando imperativo: `--dry-run=client -o yaml`:
  ```bash
  kubectl create deployment NAME --image=IMG --dry-run=client -o yaml > deployment.yaml
  ```
  Trick utilissimo, lo uso spesso per non scrivere il manifest da zero.

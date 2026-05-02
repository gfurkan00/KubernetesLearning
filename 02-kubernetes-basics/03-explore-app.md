# Explore Your App — Pods e Nodes

Link: https://kubernetes.io/docs/tutorials/kubernetes-basics/explore/explore-intro/

I 4 comandi di base per ispezionare e debuggare un Pod: `get`, `describe`, `logs`, `exec`. Sono quelli che si usano di più con kubectl in assoluto.

## Pod e Node, quick recap

**Pod**:
- gruppo di 1+ container che condividono IP, port space, volumi
- sempre co-locati e co-schedulati sullo stesso Node
- quando muore non torna su da solo (a meno che un Deployment lo gestisca)

**Node**:
- macchina worker (VM o fisica)
- runtime del container (containerd, CRI-O, ...) + kubelet (agente che parla col control plane) + kube-proxy (networking dei Service)
- ospita tanti Pod

## I 4 comandi

### `get` — vista d'insieme

```bash
kubectl get pods                              # default namespace
kubectl get pods -o wide                      # con IP e Node
kubectl get pods -A                           # tutti i namespace
kubectl get pods -l app=kubernetes-bootcamp   # filtro per label
```

Stati comuni: `Running`, `Pending`, `Succeeded`, `Failed`, `CrashLoopBackOff`, `ImagePullBackOff`.

### `describe` — dettagli + Events

```bash
kubectl describe pod POD_NAME
```

In fondo c'è la sezione `Events` — è il primo posto da guardare quando qualcosa non funziona. Mostra i tentativi di scheduling, pull immagine, start, errori, ecc.

### `logs` — log del container

```bash
kubectl logs POD_NAME                         # log default
kubectl logs POD_NAME -c CONTAINER            # multi-container
kubectl logs POD_NAME --previous              # log dell'istanza morta (per CrashLoopBackOff!)
kubectl logs POD_NAME -f                      # follow, come tail -f
kubectl logs POD_NAME --tail=50
kubectl logs -l app=NAME                      # tutti i Pod con un certo label
```

`--previous` salva la giornata quando un Pod è in crash loop e i log dell'istanza attuale non mostrano il vero errore.

### `exec` — eseguire comandi nel container

```bash
# One-shot
kubectl exec POD_NAME -- env
kubectl exec POD_NAME -- ls /app
kubectl exec POD_NAME -- cat /etc/config/file

# Shell interattiva
kubectl exec -it POD_NAME -- /bin/bash
kubectl exec -it POD_NAME -- sh           # alpine ha solo sh
```

`-i` = stdin aperto, `-t` = TTY. Servono entrambi per shell interattiva.

## Workflow tipico per debug

```bash
# 1. Nome del Pod in variabile
export POD_NAME=$(kubectl get pods -o go-template \
  --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')

# 2. proxy in un terminale separato
kubectl proxy

# 3. Verifica che l'app risponda
curl http://localhost:8001/api/v1/namespaces/default/pods/$POD_NAME:8080/proxy/

# 4. Entra dentro il container
kubectl exec -it $POD_NAME -- bash
# > cat server.js
# > curl http://localhost:8080
# > exit
```

## Gotcha

- Status `Running` non significa "app sana", solo che il container è acceso. Per la salute applicativa servono `livenessProbe` e `readinessProbe`.
- Container senza shell (image `distroless` o simili): `kubectl exec` fallisce. Workaround: `kubectl debug` per attaccare un container ephemeral.
- `kubectl logs --previous` non funziona se il Pod non è mai stato ricreato (cioè non c'è un'istanza precedente a cui guardare).

## Cheat

```bash
kubectl get pods
kubectl get pods -o wide
kubectl get all                               # tutto: deploy, svc, pod, ...

kubectl describe pod POD
kubectl describe deployment DEPLOY
kubectl describe node NODE

kubectl logs POD [-f] [--previous] [-c CONTAINER]

kubectl exec POD -- COMMAND
kubectl exec -it POD -- bash
```

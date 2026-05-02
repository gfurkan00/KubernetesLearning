# Rolling Update

Link: https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/

Aggiornare un'app senza downtime, e fare rollback se l'update va male. È la feature che rende k8s "production-ready" per CI/CD.

## Come funziona

Invece di sostituire tutti i Pod insieme (downtime garantito), k8s sostituisce gradualmente i vecchi con i nuovi:

```
Stato:    [v1] [v1] [v1] [v1]                4 Pod v1
          ↓
          [v1] [v1] [v1] [v2]                1 nuovo creato
          [v1] [v1] [v2]                     1 vecchio rimosso
          [v1] [v1] [v2] [v2]
          [v1] [v2] [v2]
          ...
          [v2] [v2] [v2] [v2]                update completo
```

Durante l'update il Service routa il traffico solo ai Pod ready → zero downtime.

## Parametri (default)

- `maxUnavailable: 1` → max 1 Pod in meno rispetto alle repliche desiderate durante l'update
- `maxSurge: 1` → max 1 Pod sopra le repliche desiderate (per consentire il "salto")

Configurabili in `spec.strategy.rollingUpdate` del Deployment.

Ogni rolling update genera una nuova **revision**, salvata da k8s. Da qui la possibilità di rollback istantaneo.

## Comandi

```bash
# Versione corrente
kubectl describe pods | grep -i image
# Image:  gcr.io/google-samples/kubernetes-bootcamp:v1

# Update a v2
kubectl set image deployments/kubernetes-bootcamp \
  kubernetes-bootcamp=docker.io/jocatalin/kubernetes-bootcamp:v2

# Stato del rollout
kubectl rollout status deployments/kubernetes-bootcamp

# Verifica
curl http://$(minikube ip):$NODE_PORT     # ora mostra "v=2"
kubectl describe pods | grep -i image     # Image: ...:v2

# Storico
kubectl rollout history deployment/kubernetes-bootcamp
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>
```

Sintassi `set image`: `kubectl set image deployment/NAME CONTAINER=NEW_IMAGE`.

## Update fallito + rollback

```bash
# Lancio un update con immagine inesistente
kubectl set image deployments/kubernetes-bootcamp \
  kubernetes-bootcamp=gcr.io/google-samples/kubernetes-bootcamp:v10

# Verifica del fallimento
kubectl get pods                     # nuovi Pod in ImagePullBackOff
kubectl rollout status deployments/kubernetes-bootcamp
# bloccato — k8s non sostituisce i vecchi finché i nuovi non sono ready

# Rollback alla revision precedente
kubectl rollout undo deployments/kubernetes-bootcamp

# Verifica
kubectl rollout status deployments/kubernetes-bootcamp
kubectl describe pods | grep -i image     # tornato a v2
```

Punto importante: anche durante un update fallito, i vecchi Pod restano vivi finché i nuovi non sono ready → zero downtime anche con un update sbagliato.

## Cleanup

```bash
kubectl delete deployments/kubernetes-bootcamp services/kubernetes-bootcamp
```

## Da ricordare

- `kubectl set image` è imperativo. Per CI/CD si aggiorna il YAML in git e si fa `kubectl apply`.
- `kubectl rollout undo` torna alla revision N-1. Per andare a una specifica: `--to-revision=N`.
- `spec.revisionHistoryLimit: 10` (default) tiene solo le ultime 10 revisions. Mettere a 0 perdi lo storico (sconsigliato).
- `change-cause`: per documentare il perché di un rollout, usare l'annotazione `kubernetes.io/change-cause`. Compare in `rollout history`.
- Strategia alternativa al rolling: `spec.strategy.type: Recreate` (kill all, poi ricrea, **con downtime**). Per workload che non possono coesistere in 2 versioni.
- `kubectl rollout restart deployment/NAME` forza la ricreazione dei Pod senza modificare l'image. Utile per riavviare i Pod (es. quando la config in env var è cambiata e serve restart per applicarla).

## Cheat

```bash
# Update
kubectl set image deployment/NAME CONTAINER=NEW_IMAGE
kubectl edit deployment/NAME              # apre il manifest live
kubectl apply -f deployment.yaml

# Rollout
kubectl rollout status deployment/NAME
kubectl rollout history deployment/NAME
kubectl rollout pause deployment/NAME
kubectl rollout resume deployment/NAME
kubectl rollout undo deployment/NAME [--to-revision=N]
kubectl rollout restart deployment/NAME
```

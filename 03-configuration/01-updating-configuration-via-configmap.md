# Updating Configuration via a ConfigMap

Link: https://kubernetes.io/docs/tutorials/configuration/updating-configuration-via-a-configmap/

Aggiornare la configurazione di un'app aggiornando il ConfigMap, e capire la differenza fondamentale tra:
- ConfigMap montato come **volume** → si aggiorna automaticamente nei Pod
- ConfigMap come **env var** → richiede restart dei Pod

## Concetti

| | |
|---|---|
| **ConfigMap** | dati di configurazione non confidenziali (key-value) |
| **Volume mount** | espone i dati del ConfigMap come **file** dentro il container |
| **Env var da ConfigMap** | espone i dati del ConfigMap come **variabili d'ambiente** |
| **Kubelet sync period** | intervallo entro cui kubelet propaga i cambi del ConfigMap nei volumi montati (~10-60s) |
| **Rollout restart** | ricrea i Pod di un Deployment con strategia rolling, senza downtime |
| **Immutable ConfigMap** | con `immutable: true` non è più modificabile, alleggerisce kubelet |

## Creare un ConfigMap (modo veloce)

```bash
kubectl create configmap NAME --from-literal=KEY=VALUE
```

Altri modi: `--from-file=path`, `--from-env-file=.env`, oppure direttamente da YAML.

## Esempio 1 — ConfigMap come Volume

```bash
kubectl create configmap sport --from-literal=sport=football
kubectl apply -f https://k8s.io/examples/deployments/deployment-with-configmap-as-volume.yaml
```

Il Deployment monta il ConfigMap in `/etc/config/`. Estratti chiave del manifest:

```yaml
spec:
  containers:
    - name: alpine
      command: ["/bin/sh", "-c", "while true; do echo \"$(date) My preferred sport is $(cat /etc/config/sport)\"; sleep 10; done;"]
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config       # qui il ConfigMap diventa file
  volumes:
    - name: config-volume
      configMap:
        name: sport
```

Verifica e logs:
```bash
kubectl get pods --selector=app.kubernetes.io/name=configmap-volume
kubectl logs deployments/configmap-volume
# ... My preferred sport is football
```

Cambio il valore e osservo l'aggiornamento automatico:
```bash
kubectl edit configmap sport          # cambia football → cricket
kubectl logs deployments/configmap-volume --follow
# ... My preferred sport is football
# ... My preferred sport is cricket   ← cambia da solo entro ~1 min
```

Punto chiave: il volume si aggiorna **senza riavviare i Pod**. L'app però deve rileggere il file (file watcher o polling).

## Esempio 2 — ConfigMap come Env Variable

```bash
kubectl create configmap fruits --from-literal=fruits=apples
kubectl apply -f https://k8s.io/examples/deployments/deployment-with-configmap-as-envvar.yaml
```

Estratti del manifest:

```yaml
spec:
  containers:
    - name: alpine
      env:
        - name: FRUITS
          valueFrom:
            configMapKeyRef:
              name: fruits
              key: fruits
      command: ["/bin/sh", "-c", "while true; do echo \"$(date) My preferred fruits are ${FRUITS}\"; sleep 10; done;"]
```

```bash
kubectl logs deployments/configmap-env-var
# ... My preferred fruits are apples

kubectl edit configmap fruits         # cambia apples → oranges
kubectl logs deployments/configmap-env-var --follow
# ... My preferred fruits are apples  ← NON cambia!
```

Le env var vengono iniettate al boot del processo. Modificare il ConfigMap non aggiorna i Pod già in esecuzione. Per applicare:

```bash
kubectl rollout restart deployment/configmap-env-var
kubectl logs deployments/configmap-env-var --follow
# ... My preferred fruits are oranges
```

## Esempio 3 — Immutable ConfigMap

```bash
kubectl create configmap immutable-config --from-literal=setting=initial
kubectl edit configmap immutable-config
```

Aggiungere a livello root:
```yaml
immutable: true
```

Da quel momento in poi ogni modifica fallisce:
```
The ConfigMap "immutable-config" is invalid: data: Invalid value: ...:
field is immutable when `immutable` is set to true
```

Use case: config statiche e diffuse → kubelet non monitora più il ConfigMap, meno carico sul cluster. Per cambiare valore: cancellare e ricreare ConfigMap **e** Pod che lo usano.

## Multi-container e Sidecar (cenni)

- **Multi-container Pod**: più container nello stesso Pod possono montare lo stesso ConfigMap-volume con `mountPath` diversi → tutti ricevono gli aggiornamenti contemporaneamente.
- **Sidecar pattern**: un container ausiliario monitora il file di config e segnala al main di ricaricarsi (es. `nginx -s reload`). Vedi tutorial dedicato sui sidecar.

## Cleanup

```bash
kubectl delete deployment configmap-volume configmap-env-var
kubectl delete configmap sport fruits immutable-config
```

## Volume vs Env Var (quick reference)

| | Volume Mount | Env Var |
|---|---|---|
| Aggiornamento auto | sì (~kubelet sync period) | no |
| Richiede restart Pod | no | sì (`kubectl rollout restart`) |
| App deve... | rileggere il file | essere riavviata |
| Caso d'uso tipico | config dinamica / hot-reload | config statica al boot |

## Note sparse

- `kubectl edit` apre vim di default. Cambia con `KUBE_EDITOR=nano` (o quello che preferisci).
- Vedere il contenuto di un ConfigMap velocemente: `kubectl get configmap NAME -o yaml` o `kubectl describe configmap NAME`.
- Cancellare un ConfigMap usato da un Pod non blocca i Pod in esecuzione, ma se vengono ricreati falliranno (`CreateContainerConfigError`) finché non lo ricrei.
- `kubectl rollout restart` patcha l'annotation `kubectl.kubernetes.io/restartedAt` nel template — è il modo "pulito" per forzare nuovi Pod senza modificare lo spec.

## Cheat

```bash
kubectl create configmap NAME --from-literal=KEY=VALUE
kubectl edit configmap NAME
kubectl delete configmap NAME

kubectl get configmap
kubectl get configmap NAME -o yaml
kubectl describe configmap NAME

kubectl logs deployments/NAME --follow
kubectl rollout restart deployment/NAME
```

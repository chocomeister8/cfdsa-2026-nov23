# Umami on Kubernetes: Summary (namespace `workshop03`)

Updated 25 Nov 2026. Cluster: Kind (`mycluster`, one control-plane and one worker node) with a Cilium Gateway. All commands use `C:\Course\Kubernates\kubectl.exe`, written below as `kubectl`.

## Architecture

```
Browser -> HTTPRoute (umami-172.25.0.4.nip.io) -> umamiapp-svc:3000 -> umamiapp-deploy (Umami)
                                                                              |
                                                                  umamidb-svc:5432 -> umamidb-deploy (Postgres + PVC)
```

| Resource | Name | Purpose | File |
|---|---|---|---|
| Namespace | `workshop03` | Holds everything | `config.yaml` |
| ConfigMap | `umami-cm` | `PGDATA` (non-sensitive setting) | `config.yaml` |
| Secret | `umamidb-secret` | `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` | `config.yaml` |
| Secret | `umamiapp-secret` | `APP_SECRET` | `config.yaml` |
| PVC | `umamidb-pvc` | 1Gi volume, mounted at `/var/lib/postgresql/data` | `umamidb.yaml` |
| Deployment | `umamidb-deploy` | 1 replica, `postgres:16-alpine`, `Recreate` strategy, port 5432 | `umamidb.yaml` |
| Service | `umamidb-svc` | ClusterIP, port 5432 (internal only) | `umamidb.yaml` |
| Deployment | `umamiapp-deploy` | 1 replica, `ghcr.io/umami-software/umami:latest`, port 3000 | `umamiapp.yaml` |
| Service | `umamiapp-svc` | ClusterIP, port 3000 -> targetPort 3000 | `umamiapp.yaml` |
| HTTPRoute | `umamiapp-httproute` | Host `umami-172.25.0.4.nip.io`, parent Gateway `default-gtw` (namespace `default`) | `umamiapp.yaml` |
| HPA | `umamiapp-hpa` | 1 to 4 replicas, scales above 80% CPU | `umamiapp.yaml` |

## How the objects relate

Objects are linked by names and labels. Kubernetes does not connect them automatically, so a name that does not match exactly breaks the link.

| Link | Written as | Must match |
|---|---|---|
| Deployment -> ConfigMap | `envFrom.configMapRef.name: umami-cm` | The ConfigMap's `metadata.name` |
| Deployment -> Secret | `envFrom.secretRef.name` or `secretKeyRef.name` | The Secret's `metadata.name` |
| Deployment -> PVC | `persistentVolumeClaim.claimName: umamidb-pvc` | The PVC's `metadata.name` |
| Volume -> mount | `volumeMounts[].name` | `volumes[].name` in the same pod spec |
| Service -> Pod | `selector: app: umamidb` | The pod template labels, not the Deployment's own labels |
| Service -> port | `targetPort: db-port` | The container's port name |
| App -> database | Host `umamidb-svc` | The Service name (a PVC has no DNS name) |
| HTTPRoute -> Gateway | `parentRefs` | Gateway name and namespace |
| HTTPRoute -> Service | `backendRefs: umamiapp-svc` | The Service name and port |

Everything must be in the same namespace, because a pod can only read a Secret, ConfigMap or PVC from its own namespace.

## ConfigMap vs Secret

| | ConfigMap | Secret |
|---|---|---|
| Holds | Non-sensitive settings | Sensitive data (passwords, API keys) |
| Used here for | `PGDATA` | Database credentials, `APP_SECRET` |
| Stored as | Plain text | Base64-encoded (not encryption) |
| `kubectl describe` | Shows the values | Hides the values |

Both can be loaded as environment variables with `envFrom`, or one key at a time with `configMapKeyRef` / `secretKeyRef`. Neither updates in a running pod when edited. Run `kubectl rollout restart deployment/<name> -n workshop03` after a change. Rule of thumb: if leaking the value would be a problem, use a Secret.

## Config file (`config.yaml`)

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: workshop03

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: umami-cm
  namespace: workshop03
data:
  PGDATA: /var/lib/postgresql/data/pgdata

---
apiVersion: v1
kind: Secret
metadata:
  name: umamidb-secret
  namespace: workshop03
type: Opaque
stringData:
  POSTGRES_USER: umami
  POSTGRES_PASSWORD: mydbpassword
  POSTGRES_DB: umami

---
apiVersion: v1
kind: Secret
metadata:
  name: umamiapp-secret
  namespace: workshop03
type: Opaque
stringData:
  APP_SECRET: replace-with-a-long-random-string
```

Do not push this file to GitHub with real credentials. Use a `.gitignore` entry, or commit a copy with placeholder values.

## Database file (`umamidb.yaml`)

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: umamidb-pvc
  namespace: workshop03
spec:
  storageClassName: standard
  accessModes:
  - ReadWriteOncePod
  resources:
    requests:
      storage: 1Gi

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: umamidb-deploy
  namespace: workshop03
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: umamidb
  template:
    metadata:
      labels:
        app: umamidb
    spec:
      volumes:
      - name: umamidb-vol
        persistentVolumeClaim:
          claimName: umamidb-pvc
      containers:
      - name: umamidb-container
        image: postgres:16-alpine
        imagePullPolicy: IfNotPresent
        envFrom:
        - configMapRef:
            name: umami-cm
        - secretRef:
            name: umamidb-secret
        ports:
        - name: db-port
          containerPort: 5432
        volumeMounts:
        - name: umamidb-vol
          mountPath: /var/lib/postgresql/data

---
apiVersion: v1
kind: Service
metadata:
  name: umamidb-svc
  namespace: workshop03
spec:
  type: ClusterIP
  selector:
    app: umamidb
  ports:
  - port: 5432
    targetPort: db-port
```

Notes:
- `PGDATA` now comes from the ConfigMap through `envFrom`, so it no longer appears under `env`. It points to a `pgdata` subdirectory because Postgres will not start if its data directory is the volume root.
- `Recreate` avoids a stuck rollout, since a `ReadWriteOncePod` volume can only be used by one pod at a time.

## App file (`umamiapp.yaml`)

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: umamiapp-deploy
  namespace: workshop03
spec:
  replicas: 1
  selector:
    matchLabels:
      app: umamiapp
  template:
    metadata:
      labels:
        app: umamiapp
    spec:
      containers:
      - name: umamiapp-container
        image: ghcr.io/umami-software/umami:latest
        imagePullPolicy: IfNotPresent
        envFrom:
        - secretRef:
            name: umamiapp-secret
        ports:
        - name: http-port
          containerPort: 3000
        env:
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: umamidb-secret
              key: POSTGRES_USER
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: umamidb-secret
              key: POSTGRES_PASSWORD
        - name: POSTGRES_DB
          valueFrom:
            secretKeyRef:
              name: umamidb-secret
              key: POSTGRES_DB
        - name: DATABASE_URL
          value: postgresql://$(POSTGRES_USER):$(POSTGRES_PASSWORD)@umamidb-svc.workshop03.svc.cluster.local:5432/$(POSTGRES_DB)
        resources:
          requests:
            memory: 256Mi
            cpu: 100m
          limits:
            memory: 512Mi
            cpu: 500m
        readinessProbe:
          httpGet:
            path: /api/heartbeat
            port: http-port
          initialDelaySeconds: 10
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /api/heartbeat
            port: http-port
          initialDelaySeconds: 30
          periodSeconds: 30

---
apiVersion: v1
kind: Service
metadata:
  name: umamiapp-svc
  namespace: workshop03
spec:
  type: ClusterIP
  selector:
    app: umamiapp
  ports:
  - port: 3000
    targetPort: 3000

---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: umamiapp-httproute
  namespace: workshop03
spec:
  hostnames:
  - umami-172.25.0.4.nip.io
  parentRefs:
  - name: default-gtw
    namespace: default
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: umamiapp-svc
      port: 3000

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: umamiapp-hpa
  namespace: workshop03
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: umamiapp-deploy
  minReplicas: 1
  maxReplicas: 4
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
```

Notes:
- `DATABASE_URL` is built from the `umamidb-secret` values with `$(VAR)` references, so the credentials never appear in plain text. Kubernetes only expands `$(VAR)` inside a container's `env`, so `DATABASE_URL` has to stay here, and the variables it uses must be defined earlier in the list.
- If the password contains characters like `@` or `/`, it needs URL-encoding.
- The hostname must use the Gateway's real address (see below).

## Deploy order

```
kubectl apply -f config.yaml       # creates the namespace, ConfigMap and Secrets
kubectl apply -f umamidb.yaml
kubectl wait --for=condition=Ready pod -l app=umamidb -n workshop03 --timeout=180s
kubectl apply -f umamiapp.yaml
kubectl get pods -n workshop03 -w  # Ctrl+C to stop watching
kubectl get all,pvc,httproute,hpa,cm,secret -n workshop03
```

Each object lives in exactly one file. Do not repeat the Namespace or a Secret in the other files, because a stale copy re-applied later would overwrite the real value.

## Resetting and cleanup

```
kubectl delete ns workshop03
kubectl get ns                     # wait until workshop03 is gone before re-applying
```

Deleting the namespace removes everything inside it, including the PVC and the database data. The Gateway lives in the `default` namespace and is not affected. After a reset, Umami's default login is `admin` / `umami`.

## Accessing the app

The pod must be `1/1 Running` first. `Running` with `0/1` means the readiness probe is still failing.

**Through the Gateway.** Check the address with `kubectl get gateway default-gtw -n default`. It is `172.25.0.4` (class `cilium`, `PROGRAMMED: True`), so the route hostname is `umami-172.25.0.4.nip.io`. Test from inside Docker's network:

```
docker exec mycluster-worker curl -s -H "Host: umami-172.25.0.4.nip.io" http://172.25.0.4/api/heartbeat
docker run --rm --network kind curlimages/curl -s -H "Host: umami-172.25.0.4.nip.io" http://172.25.0.4/api/heartbeat
```

Both returned `{"ok":true}`, which confirms the whole chain: Gateway, HTTPRoute, Service and app.

**From Windows.** `curl.exe http://umami-172.25.0.4.nip.io/api/heartbeat` timed out. Kind runs its nodes on a private Docker network, and Windows has no route to `172.25.0.4`. The Gateway itself is fine. To use the app in a Windows browser, forward the Service directly (this bypasses the Gateway):

```
kubectl port-forward svc/umamiapp-svc -n workshop03 3000:3000
```

Then open `http://localhost:3000` and sign in with `admin` / `umami`. Change the password afterwards.

## Metrics server (needed for the HPA)

`kubectl top nodes` returned `error: Metrics API not available`. Kind does not include metrics-server, and without it the HPA shows `<unknown>` for CPU and cannot scale. To install it:

```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl patch -n kube-system deployment metrics-server --type=json -p "[{\"op\":\"add\",\"path\":\"/spec/template/spec/containers/0/args/-\",\"value\":\"--kubelet-insecure-tls\"}]"
kubectl rollout status deployment/metrics-server -n kube-system
kubectl top nodes
kubectl get hpa -n workshop03
```

`--kubelet-insecure-tls` is needed because Kind's kubelets use self-signed certificates. It is fine for a local workshop, not for production. The HPA target should change from `<unknown>/80%` to a real percentage once readings arrive.

## Issues hit and how they were fixed

| Problem | Cause | Fix |
|---|---|---|
| Database YAML rejected | `apiVersion: v1` used for a Deployment; `volumes`/`containers` placed directly under `spec` | Use `apps/v1`, with `selector` and `template`, and put `volumes`/`containers` under `template.spec` |
| Pod would not start | Image typo `postgres:16-apline` | `postgres:16-alpine` |
| "Volume not found" | Volume names did not match between `volumes` and `volumeMounts` | Use identical names (`umamidb-vol`) |
| Secret not found | Secret was in `day03`, pod in `workshop03` | Put every object in `workshop03` |
| PVC mounted at the wrong path | Mounted at `/var/lib/mysql` (a MySQL path) | Mount at `/var/lib/postgresql/data`, with `PGDATA` set to the `pgdata` subdirectory |
| App file still said `api` | Copied from the earlier `api.yaml` template | New names, labels (`app: umamiapp`) and namespace |
| Wrong hostname | `umami.<ip>.nip.io` (dot) and the wrong IP (`172.25.39.200`) | `umami-<gateway-ip>.nip.io` with the Gateway's real address, `172.25.0.4` |
| `port-forward`: "pod is not running, status=Pending" | The large Umami image was still downloading (about 1m52s on first pull) | Wait for `1/1 Running`; later starts are fast because the image is cached |
| Readiness probe failed right after start | App was still running its database setup | Normal during startup; the pod turns `1/1` once `/api/heartbeat` answers |
| `unknown field ...-secretRef` on apply | Missing space after the dash (`-secretRef:`) | `- secretRef:` (always a space after the dash) |
| `unknown field` under `valueFrom` | `secretRef` used inside `valueFrom` | Inside `valueFrom` the field is `secretKeyRef`; only `envFrom` uses `secretRef` |
| Secret `umami-secret` not found | Wrong Secret name | The Secret is `umamiapp-secret` |
| App could not reach the database | Host `umamidb-pvc...` named the PVC, which has no DNS name | Use the Service: `umamidb-svc` or `umamidb-svc.workshop03.svc.cluster.local` |
| Edits seemed to have no effect | The file on disk was not the one that was edited, or was not saved | Check the file with `findstr`, then `apply --dry-run=server` before applying |
| App restarted once after a reset | App connected before Postgres finished initializing a fresh volume | Harmless; it recovered on retry. A `pg_isready` readiness probe on the database would prevent it |
| HPA missing from `get all` | Its block was commented out in `umamiapp.yaml` | Uncomment the whole block and apply again |

## Verification checklist

- [x] `umamidb-deploy` and `umamiapp-deploy` are `1/1 Running`, and `umamidb-pvc` is `Bound`
- [x] `umamidb-svc` and `umamiapp-svc` are `ClusterIP`
- [x] `umami-cm`, `umamidb-secret` and `umamiapp-secret` exist in `workshop03`
- [x] HTTPRoute `umamiapp-httproute` is accepted, with hostname `umami-172.25.0.4.nip.io`
- [x] The Gateway answers `{"ok":true}` on `/api/heartbeat` from inside Docker's network
- [ ] `get hpa -n workshop03` lists `umamiapp-hpa` (it was missing from the last output)
- [ ] metrics-server installed, and the HPA shows a real CPU percentage instead of `<unknown>`
- [ ] Signing in at `http://localhost:3000` (via port-forward) works, which proves the database connection

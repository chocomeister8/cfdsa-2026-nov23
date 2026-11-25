# Umami on Kubernetes: Summary (namespace `workshop03`)

Cluster: Kind (`mycluster`, one control-plane and one worker node). All commands use `C:\Course\Kubernates\kubectl.exe`.

## Architecture

```
Browser -> HTTPRoute (umami-<gateway-ip>.nip.io) -> umamiapp-svc:3000 -> umamiapp-deploy (Umami)
                                                                              |
                                                                  umamidb-svc:5432 -> umamidb-deploy (Postgres + PVC)
```

| Resource | Name | Purpose |
|---|---|---|
| Namespace | `workshop03` | Holds everything |
| Secret | `umamidb-secret` | `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` |
| PVC | `umamidb-pvc` | 1Gi volume, mounted at `/var/lib/postgresql/data` |
| Deployment | `umamidb-deploy` | 1 replica, `postgres:16-alpine`, `Recreate` strategy, port 5432 |
| Service | `umamidb-svc` | ClusterIP, port 5432 (internal only) |
| Secret | `umamiapp-secret` | `APP_SECRET` |
| Deployment | `umamiapp-deploy` | 1 replica, `ghcr.io/umami-software/umami:latest`, port 3000 |
| Service | `umamiapp-svc` | ClusterIP, port 3000 -> targetPort 3000 |
| HTTPRoute | `umamiapp-httproute` | Host `umami-<loadbalancer-ip>.nip.io`, parent Gateway `default-gtw` |
| HPA | `umamiapp-hpa` | 1 to 4 replicas, scales above 80% CPU |

## Database file (`umamidb.yaml`)

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: workshop03

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
        envFrom:
        - secretRef:
            name: umamidb-secret
        env:
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
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
- `PGDATA` points to a `pgdata` subdirectory because Postgres will not start if its data directory is the volume root.
- `Recreate` avoids a stuck rollout, since a `ReadWriteOncePod` volume can only be used by one pod at a time.

## App file (`umamiapp.yaml`)

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: umamiapp-secret
  namespace: workshop03
type: Opaque
stringData:
  APP_SECRET: replace-with-a-long-random-string

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
          value: postgresql://$(POSTGRES_USER):$(POSTGRES_PASSWORD)@umamidb-svc:5432/$(POSTGRES_DB)
        - name: APP_SECRET
          valueFrom:
            secretKeyRef:
              name: umamiapp-secret
              key: APP_SECRET
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
  - umami-172.25.39.200.nip.io
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
- `DATABASE_URL` is built from the `umamidb-secret` values with `$(VAR)` references, so the credentials never appear in plain text. The referenced variables must be defined earlier in the `env` list.
- The Postgres secret is in the same namespace as the app, which is required for a pod to read it.
- The hostname uses a dash, `umami-<ip>.nip.io`. Get the real IP from `kubectl get gateway default-gtw -n default`.

## Deploy order

```
kubectl apply -f umamidb.yaml      # creates the namespace, so it goes first
kubectl get pods -n workshop03 -w  # wait for umamidb-deploy 1/1 Running (Ctrl+C to stop)
kubectl apply -f umamiapp.yaml
kubectl get all,pvc,httproute,hpa -n workshop03
```

## Accessing the app

The pod must be `1/1 Running` first. `Running` with `0/1` means the readiness probe is still failing.

```
kubectl wait --for=condition=Ready pod -l app=umamiapp -n workshop03 --timeout=300s
kubectl port-forward deploy/umamiapp-deploy -n workshop03 3000:3000
```

Then open `http://localhost:3000`, or `http://umami-<gateway-ip>.nip.io` if the Gateway is reachable. Umami's default login is `admin` / `umami`, so change the password after signing in.

## Issues hit and how they were fixed

| Problem | Cause | Fix |
|---|---|---|
| Database YAML rejected | `apiVersion: v1` used for a Deployment; `volumes`/`containers` placed directly under `spec` | Use `apps/v1`, with `selector` and `template`, and put `volumes`/`containers` under `template.spec` |
| Pod would not start | Image typo `postgres:16-apline` | `postgres:16-alpine` |
| "Volume not found" | Volume names in `volumes` did not match `volumeMounts` | Use identical names (`umamidb-vol`) |
| Secret not found | Secret was in `day03`, pod in `workshop03` | Put every object in `workshop03` |
| PVC mounted at the wrong path | Mounted at `/var/lib/mysql` (a MySQL path) | Mount at `/var/lib/postgresql/data`, with `PGDATA` set to the `pgdata` subdirectory |
| App file still said `api` | Copied from the earlier `api.yaml` template | New names, labels (`app: umamiapp`) and namespace; removed the unused metrics port and extra backend |
| Wrong hostname | `umami.<ip>.nip.io` (dot) | `umami-<ip>.nip.io` (dash) |
| `port-forward`: "pod is not running, status=Pending" | Large Umami image was still downloading (about 1m52s on first pull) | Wait for `1/1 Running`; later starts are fast because the image is cached |
| Readiness probe failed right after start | App was still running its database setup | Normal during startup; the pod turns `1/1` once `/api/heartbeat` answers |

## Verification checklist

- [ ] `umamidb-deploy` is `1/1 Running`, and `umamidb-pvc` is `Bound`
- [ ] `umamiapp-deploy` is `1/1 Running`, and its logs show no database connection errors
- [ ] `umamidb-svc` and `umamiapp-svc` are both `ClusterIP`
- [ ] `describe httproute umamiapp-httproute -n workshop03` shows `Accepted: True` and `ResolvedRefs: True`
- [ ] `get hpa -n workshop03` lists `umamiapp-hpa`. Its CPU target shows `<unknown>` until metrics-server is installed (Kind does not include it, and it usually needs `--kubelet-insecure-tls`)

## Cleanup

```
kubectl delete ns workshop03
```

This removes everything in the namespace, including the PVC and its data.

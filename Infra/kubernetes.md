# Kubernetes Cheatsheet

## Mental Model

Kubernetes (K8s) is a **container orchestration system**: you declare the desired state of your workloads and K8s continuously works to make reality match that declaration. You never say "run this container on that machine" — you say "I want 3 replicas of this pod running at all times" and K8s handles scheduling, restarts, and scaling. Everything is a resource with a YAML manifest. The control loop is: observe → diff → act.

**Key hierarchy:** Cluster → Nodes → Pods → Containers

---

## Install & Minimal Setup

```bash
# kubectl — the CLI to talk to any cluster
# Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# macOS
brew install kubectl

# Local cluster for development
brew install minikube     # or: kind (Kubernetes in Docker)
minikube start
minikube status

# Verify
kubectl version --client
kubectl cluster-info
kubectl get nodes
```

```bash
# kubeconfig — tells kubectl which cluster to talk to
~/.kube/config            # default location

# Switch contexts (clusters)
kubectl config get-contexts
kubectl config use-context my-cluster
kubectl config current-context
```

---

## Core Concepts

### 1. Pod
The smallest deployable unit — one or more containers that share network and storage.

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: my-app
spec:
  containers:
    - name: app
      image: python:3.11-slim
      ports:
        - containerPort: 8000
      env:
        - name: ENV
          value: "production"
      resources:
        requests:
          memory: "128Mi"
          cpu: "100m"
        limits:
          memory: "256Mi"
          cpu: "500m"
```

```bash
kubectl apply -f pod.yaml
kubectl get pods
kubectl describe pod my-pod
kubectl logs my-pod
kubectl exec -it my-pod -- /bin/bash   # shell into container
kubectl delete pod my-pod
```

### 2. Deployment
Manages a ReplicaSet to ensure N replicas of a pod are always running. **Use this instead of bare Pods.**

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app          # must match pod template labels
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: my-registry/my-app:v1.2.0
          ports:
            - containerPort: 8000
          envFrom:
            - secretRef:
                name: my-secrets
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 10
```

```bash
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl rollout status deployment/my-app
kubectl rollout history deployment/my-app
kubectl rollout undo deployment/my-app          # rollback
kubectl scale deployment my-app --replicas=5
kubectl set image deployment/my-app app=my-registry/my-app:v1.3.0  # update image
```

### 3. Service
Stable network endpoint for a set of pods. Pods come and go; a Service provides a consistent IP/DNS name.

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-svc
spec:
  selector:
    app: my-app             # routes to pods with this label
  ports:
    - port: 80              # service port (what clients use)
      targetPort: 8000      # container port (where app listens)
  type: ClusterIP           # ClusterIP | NodePort | LoadBalancer
```

| Type | Use case |
|---|---|
| `ClusterIP` | Internal cluster access only (default) |
| `NodePort` | Exposes on each node's IP + static port (dev/testing) |
| `LoadBalancer` | Cloud provider creates an external LB (production) |

### 4. Ingress
HTTP(S) routing from external traffic into services. Needs an Ingress Controller (nginx, traefik) deployed first.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app-svc
                port:
                  number: 80
  tls:
    - hosts:
        - api.myapp.com
      secretName: my-tls-secret
```

### 5. ConfigMap & Secret

```yaml
# ConfigMap — non-sensitive config
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "INFO"
  MAX_WORKERS: "4"

# Secret — sensitive data (base64 encoded)
apiVersion: v1
kind: Secret
metadata:
  name: my-secrets
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQxMjM=    # echo -n "password123" | base64
  API_KEY: c2VjcmV0a2V5           # echo -n "secretkey" | base64
```

```bash
# Create imperatively (avoids storing secrets in YAML files)
kubectl create secret generic my-secrets \
  --from-literal=DB_PASSWORD=password123 \
  --from-literal=API_KEY=secretkey
```

### 6. Namespace
Logical isolation within a cluster. Separate environments, teams, or apps.

```bash
kubectl create namespace staging
kubectl get all -n staging
kubectl apply -f deployment.yaml -n staging
kubectl config set-context --current --namespace=staging  # set default namespace
```

### 7. Persistent Volumes

```yaml
# PersistentVolumeClaim — request storage without knowing the backend
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard

# Mount in a pod
volumes:
  - name: data-vol
    persistentVolumeClaim:
      claimName: data-pvc
containers:
  - volumeMounts:
      - mountPath: /data
        name: data-vol
```

---

## Most-Used kubectl Commands

```bash
# Get resources
kubectl get pods                          # all pods in current namespace
kubectl get pods -A                       # all namespaces
kubectl get pods -o wide                  # with node and IP info
kubectl get all                           # pods, services, deployments, etc.

# Describe & debug
kubectl describe pod <name>               # events, conditions, resource usage
kubectl logs <pod>                        # stdout
kubectl logs <pod> -f                     # follow (tail)
kubectl logs <pod> --previous             # logs from crashed container
kubectl exec -it <pod> -- bash            # interactive shell
kubectl port-forward pod/<name> 8080:8000 # forward local port to pod

# Apply / delete
kubectl apply -f manifest.yaml
kubectl delete -f manifest.yaml
kubectl delete pod <name> --grace-period=0  # force delete

# Resource editing
kubectl edit deployment my-app           # open in $EDITOR
kubectl patch deployment my-app -p '{"spec":{"replicas":2}}'

# Node info
kubectl get nodes -o wide
kubectl top nodes                         # CPU/mem usage (metrics-server required)
kubectl top pods
```

---

## Most-Used Patterns

### Health Probes

```yaml
# Readiness — pod receives traffic only when this passes
readinessProbe:
  httpGet:
    path: /ready
    port: 8000
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3

# Liveness — pod is restarted when this fails
livenessProbe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 30
  periodSeconds: 10
```

### HorizontalPodAutoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: data-cleanup
spec:
  schedule: "0 2 * * *"   # every day at 02:00
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: cleanup
              image: my-app:latest
              command: ["python", "scripts/cleanup.py"]
```

---

## Gotchas

- **Pods are ephemeral** — never store state in a pod's filesystem. Use PVCs or external storage.
- **CrashLoopBackOff** — the container keeps failing on start. Check `kubectl logs <pod> --previous` and `kubectl describe pod`.
- **ImagePullBackOff** — K8s can't pull the image. Check the image name/tag, registry credentials (`imagePullSecrets`), and network access.
- **Pending pods** — usually insufficient cluster resources or node selector / taint mismatches. Check `kubectl describe pod`.
- **Secrets are base64 encoded, not encrypted** — base64 is not security. Use Sealed Secrets, Vault, or AWS Secrets Manager for real encryption at rest.
- **`kubectl apply` vs `kubectl create`** — `apply` is declarative and idempotent (can be run multiple times). `create` fails if the resource already exists. Always use `apply` in scripts.
- **Resource requests vs limits** — `requests` is used for scheduling (guaranteed). `limits` is the hard cap (container is killed if exceeded). Set both.

---

## Quick Links

- [Kubernetes Docs](https://kubernetes.io/docs)
- [kubectl Cheat Sheet (official)](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [k9s](https://k9scli.io) — terminal UI for K8s, highly recommended
- [Helm](https://helm.sh) — package manager for K8s manifests
- [Lens](https://k8slens.dev) — GUI desktop client
- [Kustomize](https://kustomize.io) — overlay-based config management (built into kubectl)

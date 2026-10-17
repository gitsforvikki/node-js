# Lesson 218 — Kubernetes Fundamentals for Node.js

## 1. Why Kubernetes?
Docker runs a container; **Kubernetes** manages many containers and machines, ensuring desired replicas, networking and rolling updates.

```text
Client -> Ingress / LB -> Service -> Pod 1 (Express)
                                 -> Pod 2 (Express)
                                 -> Pod 3 (Express)
                                            |
                                         MongoDB
```

## 2. Core objects
| Object | Role |
| --- | --- |
| Node | Machine running workloads |
| Pod | Smallest deployable unit, usually one app container |
| Deployment | Manages replicated Pods and rollouts |
| Service | Stable route to matching Pods |
| Ingress | HTTP ingress rules; requires controller |
| ConfigMap | Non-secret configuration |
| Secret | Sensitive configuration; secure with RBAC/encryption |
| HPA | Autoscaler driven by resource/custom metrics |

Pod IPs and container filesystem are ephemeral. Store user data in external durable storage.

## 3. Minimal deployment + service
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-api
spec:
  replicas: 2
  selector:
    matchLabels: { app: node-api }
  template:
    metadata:
      labels: { app: node-api }
    spec:
      containers:
        - name: api
          image: registry.example/node-api:1.0.0
          ports:
            - containerPort: 3000
          env:
            - name: PORT
              value: "3000"
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
            periodSeconds: 10
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              memory: "512Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: node-api
spec:
  selector:
    app: node-api
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP
```

Use a real image and implement health routes. `ClusterIP` is internal, not a public endpoint.

## 4. Run and scale
```bash
kubectl apply -f deployment.yaml
kubectl get pods
kubectl get services
kubectl scale deployment node-api --replicas=3
kubectl rollout status deployment/node-api
```

The Deployment creates the desired Pods; a Service selects them using labels.

## 5. Readiness vs liveness
**Readiness** indicates whether new traffic should go to this Pod; if database is down, it may report unready. **Liveness** detects an unhealthy app process; don't restart every pod just because a shared DB briefly fails.

## 6. Security and reliability
Keep credentials in protected Secret mechanisms, not image layers or ConfigMaps. Kubernetes Secret base64 data is encoding, not encryption by itself. Implement SIGTERM shutdown, resource limits, centralized logs, image scanning and rollout monitoring. Multiple API replicas multiply DB pool usage.

## 7. Stateful services
Running MongoDB/Redis reliably inside Kubernetes introduces storage, replication, backup and upgrade work. A managed database is often simpler for initial production projects.

## 8. Common mistakes
Deploying `latest` without traceable version; keeping uploaded files only in Pods; no readiness probes; exposing secrets in YAML; liveness coupled to external dependencies; overscaling API without considering DB capacity.

## Practice
Deploy a sample API to Kind/Minikube, scale from 2 to 3 replicas, change image version and monitor rollout. Explain Pod vs Deployment vs Service.

**Interview answer:** Kubernetes orchestrates containers with Pods, Deployments and Services. Node.js apps should externalize state, expose appropriate health probes, enforce resource budgets and shut down gracefully.

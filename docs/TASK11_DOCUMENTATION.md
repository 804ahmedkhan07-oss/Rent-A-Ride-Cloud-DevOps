# Task 11 — Kubernetes Resource Management, Health Probes & Horizontal Pod Autoscaling

## Objective

The objective of this task was to advance the existing Kubernetes deployment by implementing resource management, container health checks, Metrics Server, Horizontal Pod Autoscaling (HPA), and load testing using a Kind cluster.

The application consists of:

* Frontend
* Backend
* MongoDB

All application resources are deployed in the `rent-a-ride` namespace.

---

## 1. Kubernetes Cluster

A Kubernetes cluster was created using Kind with:

* 1 Control Plane node
* 2 Worker nodes

Cluster verification:

```bash
kubectl get nodes
```

All three nodes were confirmed to be in the `Ready` state.

Example:

```text
rent-a-ride-control-plane   Ready    control-plane
rent-a-ride-worker          Ready    <none>
rent-a-ride-worker2         Ready    <none>
```

---

## 2. Application Namespace

A dedicated namespace was used for the application:

```text
rent-a-ride
```

Frontend, backend, MongoDB, and HPA resources were deployed inside this namespace.

---

## 3. Resource Management

CPU and memory requests and limits were configured for the backend.

### Backend Resources

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

This means:

* Kubernetes reserves at least `100m` CPU for the backend.
* Kubernetes reserves at least `128Mi` memory.
* The backend can use up to `500m` CPU.
* The backend can use up to `512Mi` memory.

The configuration was verified using:

```bash
kubectl describe pod <backend-pod> -n rent-a-ride
```

Verified output:

```text
Limits:
  cpu:     500m
  memory:  512Mi

Requests:
  cpu:     100m
  memory:  128Mi
```

---

## 4. Backend Health Endpoint

A dedicated health endpoint was added to the backend:

```javascript
App.get("/health", (req, res) => {
  res.status(200).json({ status: "ok" });
});
```

This endpoint provides Kubernetes with a lightweight way to determine whether the backend application is responding correctly.

---

## 5. Liveness Probe

A liveness probe was configured for the backend:

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 10
  periodSeconds: 10
```

The purpose of the liveness probe is to determine whether the backend container is still functioning.

If the application becomes unhealthy and repeatedly fails the probe, Kubernetes can restart the container.

The configuration was verified using:

```bash
kubectl describe pod -n rent-a-ride -l app=backend
```

Verified:

```text
Liveness: http-get http://:3000/health
delay=10s
period=10s
```

---

## 6. Readiness Probe

A readiness probe was also configured:

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 5
```

The readiness probe determines whether the backend is ready to receive traffic.

Verified output:

```text
Readiness: http-get http://:3000/health
delay=5s
period=5s
```

The backend pod was confirmed to be:

```text
Ready: True
```

This ensures that Kubernetes only sends traffic to a backend pod that has successfully passed its readiness check.

---

## 7. Metrics Server

Initially, the Kind cluster did not have Metrics Server available.

The following command initially returned:

```text
error: Metrics API not available
```

Metrics Server was then installed:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

The Kind environment initially produced TLS certificate validation errors when Metrics Server attempted to scrape kubelet metrics.

The error was:

```text
tls: failed to verify certificate
```

The Metrics Server deployment was therefore configured with:

```text
--kubelet-insecure-tls
```

After the configuration change, Metrics Server became healthy:

```text
metrics-server   1/1   Running
```

Metrics collection was then verified with:

```bash
kubectl top pods -n rent-a-ride
```

---

## 8. Horizontal Pod Autoscaler

An HPA was configured for the backend Deployment.

File:

```text
k8s/backend-hpa.yaml
```

Configuration:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler

metadata:
  name: backend-hpa
  namespace: rent-a-ride

spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend

  minReplicas: 1
  maxReplicas: 5

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

The HPA therefore maintains:

```text
Minimum replicas: 1
Maximum replicas: 5
CPU target: 50%
```

Only the backend was configured with HPA because the task requirement allows HPA on the frontend and/or backend.

---

## 9. HPA Verification

HPA status was checked using:

```bash
kubectl get hpa -n rent-a-ride
```

Final state:

```text
NAME          REFERENCE            TARGETS       MINPODS   MAXPODS   REPLICAS
backend-hpa   Deployment/backend   cpu: 2%/50%   1         5         1
```

This confirms that:

* Metrics Server is providing CPU metrics.
* HPA is successfully calculating utilization.
* The backend is currently running with one replica.
* The current CPU utilization is below the 50% target.

---

## 10. Load Testing

A temporary load-generator pod was created inside the Kubernetes namespace:

```bash
kubectl run load-generator \
  -n rent-a-ride \
  --image=busybox:1.36 \
  --restart=Never \
  -- sh -c 'while true; do wget -q -O- http://backend-service:3000/health > /dev/null; done'
```

The load generator continuously requested the backend health endpoint.

This increased backend CPU utilization and triggered the HPA.

---

## 11. HPA Scale-Up Verification

During load testing, the HPA detected CPU utilization significantly above the configured 50% target.

The HPA events recorded the following scaling actions:

```text
New size: 2; reason: cpu resource utilization above target
New size: 4; reason: cpu resource utilization above target
New size: 5; reason: cpu resource utilization above target
```

Therefore, the backend successfully scaled:

```text
1 → 2 → 4 → 5 replicas
```

This confirmed that the Horizontal Pod Autoscaler was actively responding to increased CPU utilization.

The configured maximum of 5 replicas was also respected.

---

## 12. HPA Scale-Down Verification

After the load generator was stopped and deleted:

```bash
kubectl delete pod load-generator -n rent-a-ride
```

CPU utilization decreased.

The HPA recorded:

```text
New size: 1; reason: All metrics below target
```

The backend therefore successfully scaled back down:

```text
5 → 1 replica
```

Final HPA state:

```text
CPU: 2% / 50%
Current replicas: 1
Desired replicas: 1
```

This verified both scale-up and scale-down behavior.

---

## 13. Final Resource Usage

Current resource utilization was verified using:

```bash
kubectl top pods -n rent-a-ride
```

Final observed usage:

```text
NAME                        CPU(cores)   MEMORY(bytes)
backend-d65979cd5-wtff4     2m           45Mi
frontend-6476fd66b8-2kkfb   0m           3Mi
mongodb-0                   5m           84Mi
```

This confirmed that the Metrics Server was successfully collecting live CPU and memory metrics from the application pods.

---

## 14. Final Application Verification

The complete application was checked using:

```bash
kubectl get all -n rent-a-ride
```

Final state:

```text
backend     1/1   Running
frontend    1/1   Running
mongodb     1/1   Running
```

Services:

```text
backend-service   ClusterIP   3000/TCP
mongodb-service   ClusterIP   27017/TCP
```

Deployments:

```text
backend    1/1
frontend   1/1
```

MongoDB StatefulSet:

```text
mongodb   1/1
```

HPA:

```text
backend-hpa   Deployment/backend   cpu: 2%/50%   1/5 replicas
```

All critical application components were running successfully.

---

## 15. Troubleshooting

### Issue 1 — Metrics API unavailable

Initial command:

```bash
kubectl top pods -n rent-a-ride
```

Returned:

```text
error: Metrics API not available
```

### Root Cause

Metrics Server was not installed in the Kind cluster.

### Solution

Metrics Server was installed and configured.

---

### Issue 2 — Metrics Server TLS certificate error

Metrics Server initially failed to scrape the Kind nodes because the kubelet certificates did not contain the expected IP SANs.

Error:

```text
tls: failed to verify certificate
```

### Solution

The Metrics Server deployment was configured with:

```text
--kubelet-insecure-tls
```

After this change, Metrics Server successfully became ready and metrics became available.

---

### Issue 3 — HPA initially showed unknown CPU

Initially:

```text
cpu: <unknown>/50%
```

### Root Cause

Metrics Server was not yet providing usable CPU metrics.

### Solution

After Metrics Server became healthy, the HPA successfully reported CPU utilization:

```text
cpu: 2%/50%
```

---

### Issue 4 — Load generator caused terminal commands to appear stuck

The load generator continuously generated requests, which caused increased system activity.

The temporary pod was eventually deleted:

```bash
kubectl delete pod load-generator -n rent-a-ride
```

After removing the load, the HPA scaled the backend back down to one replica.

---

## 16. Architecture Flow

```text
                    ┌─────────────────────┐
                    │       Frontend      │
                    │       1 Pod         │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │       Backend       │
                    │     Deployment      │
                    │                     │
                    │   HPA: CPU 50%      │
                    │   Min: 1            │
                    │   Max: 5            │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │      MongoDB        │
                    │    StatefulSet      │
                    │       1 Pod         │
                    └─────────────────────┘

                    Metrics Server
                          │
                          ▼
                    CPU / Memory
                          │
                          ▼
                       Backend HPA
```

---

## 17. Final Outcome

Task 11 successfully implemented advanced Kubernetes workload management.

The following requirements were completed:

* Kubernetes cluster using Kind — **Completed**
* Dedicated application namespace — **Completed**
* Frontend deployment — **Completed**
* Backend deployment — **Completed**
* CPU and memory requests — **Completed**
* CPU and memory limits — **Completed**
* Backend health endpoint — **Completed**
* Liveness probe — **Completed**
* Readiness probe — **Completed**
* Metrics Server — **Completed**
* Backend Horizontal Pod Autoscaler — **Completed**
* Load testing — **Completed**
* HPA scale-up verification — **Completed**
* HPA scale-down verification — **Completed**
* Resource monitoring using `kubectl top` — **Completed**
* Final application verification — **Completed**
* Troubleshooting and documentation — **Completed**

### Key Result

The most important validation was the real autoscaling behavior:

```text
Normal load
     ↓
1 backend pod
     ↓
Load generated
     ↓
CPU exceeded 50%
     ↓
1 → 2 → 4 → 5 pods
     ↓
Load removed
     ↓
CPU decreased
     ↓
5 → 1 pod
```

This demonstrates that Kubernetes was not only configured for autoscaling but successfully **detected workload changes and dynamically adjusted the backend replica count**.

## Conclusion

Task 11 provided practical experience with Kubernetes resource management, health monitoring, Metrics Server, Horizontal Pod Autoscaling, and workload testing.

The backend remained available while Kubernetes automatically adjusted the number of replicas according to CPU utilization. The successful `1 → 5 → 1` scaling cycle confirmed that the HPA configuration and metrics pipeline were functioning correctly in the Kind environment.


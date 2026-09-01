# Task 13 — Helm Installation & Application Deployment Using Helm Charts

**Project:** Rent-a-Ride (MERN stack)
**Environment:** AWS EC2, Ubuntu, Kind Kubernetes cluster
**Branch:** `feature/task13-helm`

## Objective

Install Helm, convert the existing Kubernetes manifests into a Helm chart, manage configuration through `values.yaml`, and deploy/upgrade the application using Helm — including Ingress access.

---

## 1. Helm Installation

Helm was installed on the EC2 instance using the official install script.

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

Result:

```text
version.BuildInfo{Version:"v3.21.4", ...}
```

---

## 2. Helm Chart Creation

A new chart was scaffolded and unnecessary default templates (`httproute.yaml`, `serviceaccount.yaml`, `tests/`) were removed.

```bash
helm create rent-a-ride-chart
```

Final chart structure:

```text
rent-a-ride-chart/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── backend-deployment.yaml
│   ├── backend-service.yaml
│   ├── backend-hpa.yaml
│   ├── frontend-deployment.yaml
│   ├── frontend-service.yaml
│   ├── mongodb-statefulset.yaml
│   ├── mongodb-service.yaml
│   ├── mongodb-pvc.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── ingress.yaml
│   ├── NOTES.txt
│   └── _helpers.tpl
└── charts/
```

---

## 3. Converting Kubernetes Manifests to Helm Templates

All existing manifests from `k8s/` were converted into Helm templates using Go templating syntax (`{{ .Values.xxx }}`) in place of hardcoded values.

Converted resources:

* Backend Deployment, Service, HPA
* Frontend Deployment, Service
* MongoDB StatefulSet, Service, PVC
* ConfigMap (`app-config`)
* Secret (`app-secrets`)
* Ingress (`rent-a-ride-ingress`)

---

## 4. Parameterizing Configuration (`values.yaml`)

Hardcoded values were moved into `values.yaml`, including:

* Image repository/tag for backend, frontend, mongodb
* Replica counts (backend, frontend, mongodb)
* Container/service ports
* Resource requests and limits
* MongoDB connection URI
* HPA `minReplicas`, `maxReplicas`, target CPU utilization
* Ingress `enabled` flag and `ingressClassName`
* ConfigMap `BACKEND_URL`
* Secret values (`ACCESS_TOKEN`, `REFRESH_TOKEN`)

---

## 5. Validating the Helm Chart

```bash
helm lint .
```

Result:

```text
1 chart(s) linted, 0 chart(s) failed
```

Local rendering was checked before deployment:

```bash
helm template rent-a-ride . > /tmp/rendered.yaml
```

The rendered output was verified against the original `k8s/` manifests to confirm namespace, image tags, ports, and env values matched correctly.

---

## 6. Deploying the Application Using Helm

The old raw manifests (deployed via `kubectl apply -f k8s/`) were removed first to avoid resource name conflicts:

```bash
kubectl delete -f k8s/backend-deployment.yaml
kubectl delete -f k8s/backend-service.yaml
kubectl delete -f k8s/frontend-deployment.yaml
kubectl delete -f k8s/frontend-service.yaml
kubectl delete -f k8s/mongodb-deployment.yaml
kubectl delete -f k8s/mongodb-service.yaml
kubectl delete -f k8s/mongodb-pvc.yaml
kubectl delete -f k8s/backend-hpa.yaml
kubectl delete -f k8s/ingress.yaml
kubectl delete configmap app-config -n rent-a-ride
kubectl delete secret app-secrets -n rent-a-ride
```

The chart was then installed:

```bash
helm install rent-a-ride . -n rent-a-ride
```

Result:

```text
STATUS: deployed
REVISION: 1
```

Verification:

```bash
helm list -n rent-a-ride
kubectl get pods -n rent-a-ride
kubectl get svc -n rent-a-ride
kubectl get hpa -n rent-a-ride
kubectl get ingress -n rent-a-ride
kubectl get configmap,secret -n rent-a-ride
```

All pods reported `1/1 Running`: `backend`, `frontend`, `mongodb-0`.

---

## 7. Managing Helm Releases (Upgrade Test)

To validate the upgrade workflow, `hpa.maxReplicas` was changed from `5` to `6` in `values.yaml`, then applied without touching any raw manifest:

```bash
helm upgrade rent-a-ride . -n rent-a-ride
```

Result:

```text
Release "rent-a-ride" has been upgraded. Happy Helming!
REVISION: 2
```

```bash
kubectl get hpa -n rent-a-ride
```

```text
backend-hpa   Deployment/backend   cpu: 2%/50%   1   6   1
```

---

## 8. Ingress Configuration via Helm

The Ingress resource is templated and controlled by `values.yaml`:

```yaml
ingress:
  enabled: true
  className: nginx
```

Routing rules (same as original `k8s/ingress.yaml`):

```text
/api  → backend-service:3000
/     → frontend-service:8080
```

---

## 9. Accessing the Application Through Ingress

```bash
curl -I http://localhost:30080
```

Result:

```text
HTTP/1.1 200 OK
```

```bash
curl -I http://localhost:30080/api
```

Result: request reached the Express backend (`X-Powered-By: Express`), confirming the `/api` route is correctly proxied to `backend-service`. The `404` returned is expected since `/api` alone is not a defined backend endpoint.

---

## 10. Troubleshooting Notes

* **`helm lint` failure** — default `NOTES.txt` referenced `.Values.httpRoute.enabled`, which does not exist in this chart's `values.yaml`. Fixed by replacing `NOTES.txt` with a simple custom message.
* **Old resources reappearing after delete** — caused by Kubernetes pod termination delay (`Terminating` state), not an actual controller recreating resources. Resolved by waiting a few seconds before re-checking (`kubectl get all -n rent-a-ride`).

---

## 11. Outcome

* Helm installed and verified ✅
* Helm chart created for the full application stack (backend, frontend, mongodb) ✅
* Kubernetes manifests fully converted into reusable Helm templates ✅
* Configuration parameterized via `values.yaml` ✅
* Chart validated using `helm lint` and `helm template` ✅
* Application deployed successfully using `helm install` ✅
* Live upgrade tested and verified using `helm upgrade` ✅
* Ingress accessible for both frontend and backend routes ✅

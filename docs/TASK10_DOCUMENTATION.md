# Task 10 — Advanced Kubernetes Configuration

**Project:** Rent-a-Ride (MERN stack)
**Environment:** AWS EC2, Ubuntu, eu-north-1 (Stockholm), Kind Kubernetes cluster
**Branch:** `feature/task10-k8s-advanced`

## Objective

Extend the existing Kubernetes deployment by adding ConfigMaps, Secrets, StatefulSet-based MongoDB, and persistent storage. Verify configuration management, service communication, and database data persistence.

---

## 1. ConfigMap — Application Configuration

A Kubernetes `ConfigMap` was created for non-sensitive application configuration.

```yaml
BACKEND_URL: http://backend-service:3000
```

The backend consumes `BACKEND_URL` using `configMapKeyRef` instead of hardcoding the configuration directly into the container.

---

## 2. Secrets — Sensitive Configuration

A Kubernetes Secret named `app-secrets` was used for sensitive backend credentials.

The backend receives:

* `ACCESS_TOKEN`
* `REFRESH_TOKEN`

Both values are injected using `secretKeyRef`, keeping the actual credentials separate from the Deployment configuration.

---

## 3. MongoDB — StatefulSet

MongoDB was converted from a Deployment to a Kubernetes `StatefulSet`.

```text
mongodb-0
```

StatefulSet provides a stable identity for the database pod and is more appropriate for stateful workloads than a standard Deployment.

MongoDB continues to communicate through:

```text
mongodb-service:27017
```

---

## 4. Persistent Storage

MongoDB uses a PersistentVolumeClaim:

```text
mongodb-claim
```

The PVC is backed by a dynamically provisioned PersistentVolume using the default `standard` StorageClass.

```text
StorageClass → PV → PVC → MongoDB
```

Verified:

```bash
kubectl get storageclass
kubectl get pv
kubectl get pvc -n rent-a-ride
```

PVC status:

```text
Bound
```

---

## 5. Data Persistence Test

A test document was inserted into MongoDB:

```javascript
{ task: "task10", status: "persistence-test" }
```

The MongoDB pod was recreated through the StatefulSet.

The document was queried again after the pod restarted and was still available, confirming that the database data persisted through the pod recreation.

---

## 6. Issues Encountered

### Issue 1 — MongoDB CrashLoopBackOff

**Root cause:** The new StatefulSet pod attempted to use persistent storage that was already being used by another MongoDB instance.

**Fix:** The old MongoDB pod was removed, allowing the StatefulSet-managed pod to use the persistent storage correctly.

### Issue 2 — NodePort Already Allocated

`frontend-service` could not reuse NodePort `30080` because the port was already allocated by the existing service.

The existing working NodePort configuration was retained.

### Issue 3 — `kind-config.yaml` Applied as Kubernetes Manifest

The Kind cluster configuration was accidentally passed to `kubectl`.

**Root cause:** `kind-config.yaml` is a Kind configuration file, not a Kubernetes resource manifest.

It should be used by Kind during cluster creation, not with `kubectl apply`.

---

## 7. Verification

All application components were verified inside the `rent-a-ride` namespace.

```bash
kubectl get all -n rent-a-ride
kubectl get pvc -n rent-a-ride
kubectl get pv
kubectl get storageclass
kubectl get configmap -n rent-a-ride
kubectl get secret -n rent-a-ride
```

Application connectivity:

```bash
curl -I http://localhost:30080
```

Result:

```text
HTTP/1.1 200 OK
```

Backend logs confirmed successful MongoDB connection.

MongoDB persistence test successfully returned the previously inserted document.

---

## 8. Outcome

* ConfigMap implemented for non-sensitive configuration ✅
* Kubernetes Secret implemented for sensitive credentials ✅
* Backend consumes ConfigMap and Secret values ✅
* MongoDB running as StatefulSet ✅
* PVC, PV and StorageClass configured ✅
* Database persistence verified after pod recreation ✅
* Frontend → Backend → MongoDB communication verified ✅
* Frontend accessible through NodePort `30080` ✅
* Kubernetes resources running successfully in `rent-a-ride` namespace ✅


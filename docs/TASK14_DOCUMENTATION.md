# Task 14 — GitOps Deployment Using ArgoCD with Helm & Image Updater

**Project:** Rent-a-Ride (MERN stack)
**Environment:** AWS EC2, Ubuntu, Kind Kubernetes cluster
**Branch:** `feature/task14-argocd-image-updater`

## Objective

Implement a GitOps workflow using ArgoCD: connect the GitHub repository to the Kubernetes cluster, deploy the application using the existing Helm chart, verify automated Git-to-cluster synchronization, and configure ArgoCD Image Updater for automatic container image updates.

---

## 1. ArgoCD Installation

ArgoCD was installed in the `argocd` namespace. Pods and services were verified as running.

---

## 2. Connecting Git Repository & Creating ArgoCD Application

An ArgoCD Application named `rent-a-ride` was created pointing to:

* Repo: `https://github.com/804ahmedkhan07-oss/Rent-A-Ride-Cloud-DevOps`
* Branch: `dev`
* Path: `rent-a-ride-chart` (the Helm chart from Task 13)
* Destination namespace: `rent-a-ride`
* Sync policy: automated (`prune: true`, `selfHeal: true`)

---

## 3. Deploying via ArgoCD

The Application was synced, deploying all Helm-templated resources (backend, frontend, mongodb, hpa, ingress, configmap, secret) into the `rent-a-ride` namespace. Status reported `Healthy` / `Synced`.

---

## 4. Verifying the GitOps Workflow

A configuration change (HPA `maxReplicas` adjustment) was committed and pushed to the tracked branch. ArgoCD automatically detected the Git change and re-synced the cluster without any manual `kubectl` or `helm` command, confirming the GitOps loop:

```text
Git commit → ArgoCD detects change → auto-sync → Kubernetes updated
```

---

## 5. Exporting the ArgoCD Application to Git

The Application definition originally existed only as a live cluster object (created via `kubectl`/dashboard), which breaks the GitOps principle of Git being the single source of truth. It was exported and committed to the repository as `argocd-application.yaml`, so the Application (including Image Updater annotations) can be recreated from Git if ever deleted.

```bash
kubectl get application rent-a-ride -n argocd -o yaml
```

---

## 6. Installing ArgoCD Image Updater

ArgoCD Image Updater `v1.3.0` was installed in the `argocd` namespace. Docker Hub registry access was configured via a secret (`dockerhub-secret`) referenced in the Image Updater config.

---

## 7. Configuring Image Update Policy

Image Updater `v1.3.0` uses a CRD-based model (`ImageUpdater` custom resource) rather than relying solely on Application annotations. Initial attempts using only annotations did not trigger processing (`No ImageUpdater CRs to process`).

**Fix:** An `ImageUpdater` CR was created referencing the existing Application and enabling `useAnnotations: true`, so the previously configured annotations (image list, update strategy, Helm value paths) are reused:

```yaml
apiVersion: argocd-image-updater.argoproj.io/v1alpha1
kind: ImageUpdater
metadata:
  name: rent-a-ride-imageupdater
  namespace: argocd
spec:
  applicationRefs:
    - namePattern: rent-a-ride
      useAnnotations: true
```

Tracked images (via existing annotations):
* `backend=ahmedmateen07/rent-a-ride-backend` → updates `backend.image.tag` in Helm values
* `frontend=ahmedmateen07/rent-a-ride-frontend` → updates `frontend.image.tag` in Helm values

---

## 8. Verifying Image Updater Operation

After applying the `ImageUpdater` CR, controller logs confirmed successful processing:

```text
Starting image update cycle, considering 1 application(s) for update
Processing results: applications=1 images_considered=2 images_skipped=0 images_updated=0 errors=0
```

This confirms the Image Updater correctly discovered the `rent-a-ride` Application and both tracked images (backend, frontend), with zero errors. `images_updated=0` is expected since no new image digest was available on Docker Hub at test time.

---

## 9. Troubleshooting Notes

* **Issue:** Image Updater logs repeatedly showed `No ImageUpdater CRs to process` despite correct annotations on the Application.
  **Root cause:** Image Updater `v1.3.0` requires an explicit `ImageUpdater` custom resource; annotations alone (used in older versions) are not sufficient.
  **Fix:** Created an `ImageUpdater` CR with `useAnnotations: true` to reuse existing annotation-based configuration.

* **Issue:** ArgoCD Application existed only in-cluster, not in Git.
  **Fix:** Exported to `argocd-application.yaml` and committed to the repository.

---

## 10. Outcome

* ArgoCD installed and connected to GitHub repository ✅
* ArgoCD Application deploying the Task 13 Helm chart ✅
* GitOps auto-sync verified (Git change → automatic cluster update) ✅
* ArgoCD Application definition committed to Git ✅
* ArgoCD Image Updater installed and correctly configured via CRD ✅
* Image Updater actively monitoring backend and frontend images with zero errors ✅

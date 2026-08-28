# TASK09 — Kubernetes Documentation

## Objective

Deploy Rent-A-Ride application on Kubernetes using Kind.

## Cluster

* 1 Control Plane
* 2 Worker Nodes
* NodePort `30080` mapped to EC2

## Components

* Frontend — Nginx
* Backend — Node.js
* Database — MongoDB
* MongoDB PVC — `1Gi`
* Services — ClusterIP + NodePort

## Kubernetes Files

* `backend-deployment.yaml`
* `backend-service.yaml`
* `frontend-deployment.yaml`
* `frontend-service.yaml`
* `mongodb-deployment.yaml`
* `mongodb-service.yaml`
* `mongodb-pvc.yaml`
* `kind-config.yaml`

## Configuration

Backend connects to MongoDB through:

`mongodb://mongodb-service:27017/rent-a-ride`

Frontend Nginx proxies API requests to:

`http://backend-service:3000/api/`

## Verification

* All pods running successfully
* All Services have valid endpoints
* MongoDB PVC is Bound
* Frontend returns `HTTP 200 OK`
* Backend successfully scaled to 2 replicas

## Result

Rent-A-Ride is running successfully on the Kind Kubernetes cluster with MongoDB persistence, Kubernetes service discovery, NodePort access, and backend scaling.


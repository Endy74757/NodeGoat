# NodeGoat k8s manifests

This directory contains Kubernetes manifests to run NodeGoat and MongoDB in-cluster.

Files added:
- `mongo-headless-svc.yaml` - a headless Service (`clusterIP: None`) named `mongo` used by the StatefulSet.
- `mongo-statefulset.yaml` - a StatefulSet (3 replicas) with `volumeClaimTemplates` for persistent storage.

Quick apply (namespace `nodegoat` must exist):

1. Create namespace (if not already present):
   kubectl create namespace nodegoat

2. Apply the mongo manifests:
   kubectl apply -f mongo-headless-svc.yaml -n nodegoat
   kubectl apply -f mongo-statefulset.yaml -n nodegoat

3. Wait for pods:
   kubectl rollout status statefulset/mongo -n nodegoat

Notes & next steps:
- If you want the NodeGoat app to auto-initialize DB with `artifacts/db-reset.js`, run the `nodegoat-db-init` Job after Mongo is ready (or apply the `nodegoat-job.yaml`).
- For Docker Desktop Kubernetes: build the image locally and use the `nodegoat:local` tag. Example:

  ```bash
  cd NodeGoat
  docker build -t nodegoat:local .
  kubectl create namespace nodegoat
  kubectl apply -f k8s/ -n nodegoat
  kubectl apply -f k8s/nodegoat-job.yaml -n nodegoat
  ```

- Consider setting `storageClassName` in `mongo-statefulset.yaml` to match your cluster's StorageClass if PVCs do not bind automatically.
- If you prefer a single primary + secondaries replica set configuration with proper replication, I can update the StatefulSet and init scripts to initialize a replica set.

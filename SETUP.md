# Redis Operator Local Setup

Testing Redis Operator on a local Kubernetes cluster using kind.

## Environment

- Docker 29.1.3
- kind 0.31.0
- kubectl v1.34.1
- helm (required — install: `brew install helm`)

## Why kind

Redis Cluster requires 3 nodes minimum. kind runs each node as a real Docker container, giving a proper multi-node cluster locally.

---

## Step 1: Install kind

```bash
brew install kind
```

## Step 2: Create cluster config

File: `kind-config.yaml` (already in this folder)

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
```

1 control-plane + 3 workers matches Redis `clusterSize: 3`.

## Step 3: Create the cluster

```bash
kind create cluster --name redis-test --config kind-config.yaml
```

Verify nodes are up:

```bash
kubectl get nodes
```

Expected: 4 nodes (1 control-plane + 3 workers), all `Ready`.

---

## Step 4: Add Helm repo + install Redis Operator

```bash
helm repo add ot-helm https://ot-container-kit.github.io/helm-charts/
helm repo update

helm install redis-operator ot-helm/redis-operator \
  --namespace ot-operators \
  --create-namespace \
  --set featureGates.GenerateConfigInInitContainer=true
```

Verify operator pod is running:

```bash
kubectl get pods -n ot-operators
```

Expected: `redis-operator-xxxx` pod in `Running` state.

---

## Step 5: Deploy Redis Cluster

File: `redis-cluster.yaml` (already in this folder)

```bash
kubectl apply -f redis-cluster.yaml
```

Watch pods come up:

```bash
kubectl get pods -n ot-operators -w
```

Expected: 6 pods total — 3 leaders + 3 followers, all `Running`.

Check cluster CR status:

```bash
kubectl get rediscluster -n ot-operators
```

---

## Step 6: Verify cluster health

Exec into a leader pod and check cluster nodes:

```bash
kubectl exec -it redis-cluster-leader-0 -n ot-operators -- redis-cli cluster nodes
```

Expected: 6 nodes listed, 3 as `master`, 3 as `slave`, all `connected`.

---

## Step 7: Test self-healing (the whole point)

Seed some keys first:

```bash
kubectl exec -it redis-cluster-leader-0 -n ot-operators -- redis-cli set testkey "hello"
```

Delete a leader pod to simulate failure:

```bash
kubectl delete pod redis-cluster-leader-0 -n ot-operators
```

Watch operator recover it automatically:

```bash
kubectl get pods -n ot-operators -w
```

After recovery, verify a follower was promoted and cluster is healthy:

```bash
kubectl exec -it redis-cluster-leader-0 -n ot-operators -- redis-cli cluster nodes
kubectl exec -it redis-cluster-leader-0 -n ot-operators -- redis-cli get testkey
```

Expected: key still exists, cluster back to 3 masters + 3 slaves.

---

## Cleanup

Delete just the cluster (keeps kind + helm installed):

```bash
kind delete cluster --name redis-test
```

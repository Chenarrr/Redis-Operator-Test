# Redis Operator Local Setup

Testing Redis Enterprise Operator on a local Kubernetes cluster using kind.

## Environment

- Docker 29.1.3
- kind 0.31.0
- kubectl v1.34.1

## Why kind

Redis Enterprise Cluster requires 3 nodes minimum. kind runs each node as a real Docker container, giving a proper multi-node cluster locally.

## Step 1: Install kind

```bash
brew install kind
```

## Step 2: Create cluster config

File: `kind-config.yaml`

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
kind create cluster --name redis-lab --config kind-config.yaml
```

## Next Steps

- [ ] Create namespace `ot-operators`
- [ ] Install Redis Operator via Helm
- [ ] Deploy RedisCluster CRD (clusterSize: 3)
- [ ] Verify pods with `kubectl get pods -n ot-operators -w`
- [ ] Test self-healing: delete a pod, watch operator recover it

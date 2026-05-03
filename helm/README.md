# Redis Helm Chart Test

Bitnami Redis cluster via Helm — no operator, plain StatefulSet.

## Requirements

- Docker
- kind — `brew install kind`
- kubectl
- helm — `brew install helm`

## Setup

```bash
# Create cluster
kind create cluster --name redis-helm --config kind-config.yaml

# Add Bitnami repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Install Redis cluster
helm install redis-helm bitnami/redis-cluster \
  --namespace redis-helm \
  --create-namespace \
  -f values.yaml
```

## Verify

```bash
kubectl get pods -n redis-helm
```

## Test Self-Healing (vs Operator)

```bash
# Kill a master
kubectl delete pod redis-helm-redis-cluster-0 -n redis-helm

# Watch — pod restarts but cluster may need manual fix
kubectl get pods -n redis-helm -w

# Check cluster state
kubectl exec -it redis-helm-redis-cluster-0 -n redis-helm -- redis-cli cluster nodes
```

## Cleanup

```bash
kind delete cluster --name redis-helm
```

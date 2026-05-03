# Redis Operator Local Test

Testing Redis Operator self-healing on a local kind cluster.

## Requirements

- Docker
- kind — `brew install kind`
- kubectl
- helm — `brew install helm`

## Setup

```bash
# Create cluster (1 control-plane + 3 workers)
kind create cluster --name redis-test --config kind-config.yaml

# Install operator
helm repo add ot-helm https://ot-container-kit.github.io/helm-charts/
helm repo update
helm install redis-operator ot-helm/redis-operator \
  --namespace ot-operators \
  --create-namespace \
  --set featureGates.GenerateConfigInInitContainer=true

# Deploy Redis cluster (operator creates 3 leaders + 3 followers automatically)
kubectl apply -f redis-cluster.yaml
```

## Verify

```bash
kubectl get pods -n ot-operators
kubectl exec -it redis-cluster-leader-0 -n ot-operators -- redis-cli cluster nodes
```

## Test Self-Healing

```bash
# Kill a leader
kubectl delete pod redis-cluster-leader-0 -n ot-operators

# Watch operator recover it
kubectl get pods -n ot-operators

# Check promotion — follower may have become master
kubectl exec -it redis-cluster-leader-0 -n ot-operators -- redis-cli cluster nodes
```

## Cleanup

```bash
kind delete cluster --name redis-test
```

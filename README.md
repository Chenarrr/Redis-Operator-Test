# Redis Operator vs Helm — Local Test

Comparing Redis Operator self-healing against plain Bitnami Helm chart on local kind clusters.

## Requirements

- Docker, kind, kubectl, helm

```bash
brew install kind helm
```

## Structure

```
operator/   — Redis Operator (ot-container-kit), cluster: redis-test
helm/       — Bitnami Redis cluster, cluster: redis-helm
```

---

## Operator Setup (`operator/`)

```bash
kind create cluster --name redis-test --config operator/kind-config.yaml

helm repo add ot-helm https://ot-container-kit.github.io/helm-charts/
helm repo update
helm install redis-operator ot-helm/redis-operator \
  --namespace ot-operators --create-namespace \
  --set featureGates.GenerateConfigInInitContainer=true

kubectl apply -f operator/redis-cluster.yaml
kubectl get pods -n ot-operators
```

## Helm Setup (`helm/`)

```bash
kind create cluster --name redis-helm --config helm/kind-config.yaml

helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install redis-helm bitnami/redis-cluster \
  --namespace redis-helm --create-namespace \
  -f helm/values.yaml

kubectl get pods -n redis-helm
```

---

## Self-Healing Test

Kill a master in each and compare:

**Operator** — auto recovers, promotes replica, re-joins pod as slave:
```bash
kubectl delete pod redis-cluster-leader-0 -n ot-operators
kubectl get pods -n ot-operators -w
kubectl exec -it redis-cluster-leader-0 -n ot-operators -- redis-cli cluster nodes
```

**Helm** — pod restarts but cluster may need manual fix:
```bash
kubectl delete pod redis-helm-redis-cluster-0 -n redis-helm
kubectl get pods -n redis-helm -w
kubectl exec -it redis-helm-redis-cluster-0 -n redis-helm -- redis-cli cluster nodes
```

---

## Cleanup

```bash
kind delete cluster --name redis-test
kind delete cluster --name redis-helm
```

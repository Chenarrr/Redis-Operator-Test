# Redis Operator vs Helm — Local Test

Comparing Redis Operator self-healing against a plain Helm chart on local kind clusters.

---

## Redis Modes Explained

### Redis Cluster (what the Operator runs)

Data is split across multiple masters using **hash slots**. All 16384 slots are divided equally:

```
master-0  →  slots 0–5460      ~33% of your data
master-1  →  slots 5461–10922  ~33% of your data
master-2  →  slots 10923–16383 ~33% of your data
```

Each master has 1 replica watching it. If a master dies, its replica promotes automatically.

- Scales horizontally — add more masters = more capacity
- No single point of failure
- Used in production at scale

### Redis HA / Sentinel (what the Helm chart runs)

One master holds **all** the data. Replicas just copy it. Sentinel is a separate process that watches the master and promotes a replica if it dies.

```
master  →  holds everything
replica-0  →  copy
replica-1  →  copy
sentinel  →  watching, triggers failover
```

- Simpler setup
- Does not scale horizontally
- Good for smaller workloads

### Key Difference

| | Cluster | HA / Sentinel |
|---|---|---|
| Data distribution | Split across masters | All on one master |
| Scales with | More masters | Bigger machine |
| Failover handled by | Cluster protocol | Sentinel process |
| Setup complexity | Higher | Lower |

---

## What We're Testing

**The operator knows Redis internals.** When a pod dies it:
1. Detects which slots are lost
2. Promotes the right replica
3. Re-joins the recovered pod as a replica
4. Does this automatically, in seconds

**The Helm chart just deploys.** When a pod dies:
- Kubernetes restarts it (basic)
- But it knows nothing about Redis cluster state
- Slot rebalancing, re-join, promotion — all manual

---

## Requirements

- Docker, kind, kubectl, helm

```bash
brew install kind helm
```

---

## Structure

```
operator/   — Redis Operator (ot-container-kit), cluster: redis-test
helm/       — Redis HA via dandydev chart, cluster: redis-helm
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

helm repo add dandydev https://dandydeveloper.github.io/charts/
helm repo update
helm install redis-ha dandydev/redis-ha \
  --namespace redis-helm --create-namespace \
  -f helm/values.yaml

kubectl get pods -n redis-helm
```

---

## Self-Healing Test

**Operator** — kills a leader, watch it recover and re-join automatically:

```bash
kubectl delete pod redis-cluster-leader-0 -n ot-operators --context kind-redis-test
kubectl get pods -n ot-operators --context kind-redis-test -w
kubectl exec -it redis-cluster-leader-0 -n ot-operators --context kind-redis-test \
  -- redis-cli cluster nodes
```

**Helm** — kills master, Sentinel promotes a replica but cluster needs manual re-join:

```bash
kubectl delete pod redis-ha-server-0 -n redis-helm --context kind-redis-helm
kubectl get pods -n redis-helm --context kind-redis-helm -w
```

---

## Cleanup

```bash
kind delete cluster --name redis-test
kind delete cluster --name redis-helm
```

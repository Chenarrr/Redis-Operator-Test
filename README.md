# Redis Operator vs Helm — Local Test

Comparing Redis Operator self-healing against a plain Helm chart on local kind clusters.

---

## The Three Approaches

### 1. Bitnami Redis Cluster (industry standard — now paywalled)

What your supervisor used. Bitnami's `redis-cluster` Helm chart deployed Redis in
**Cluster mode** — data split across 3 masters using hash slots, each with 1 replica.

```
master-0  →  slots 0–5460      + replica
master-1  →  slots 5461–10922  + replica
master-2  →  slots 10923–16383 + replica
```

Bitnami pulled all free images on **August 28, 2025**. Everything moved behind a paid
subscription. Old tags deleted. Can't use it without paying.

---

### 2. Helm HA — dandydev/redis-ha (what we use instead)

Same Helm approach, different mode: **Sentinel HA**.

One master holds all data. Replicas copy it. Sentinel is a separate process that
watches the master — if it dies, sentinel votes and promotes a replica.

```
server-0  →  master   (redis + sentinel + config)
server-1  →  replica  (redis + sentinel + config)
server-2  →  replica  (redis + sentinel + config)
```

Each pod has 3 containers — redis, sentinel, configmap watcher.

**Helm just deploys.** After install, it walks away. If a pod dies:
- Kubernetes restarts the container (basic)
- Sentinel may promote a replica
- But re-join, slot rebalancing, cluster repair — all manual

---

### 3. Redis Operator — ot-container-kit (what we test)

Operator runs a **reconciliation loop forever**:

```
observe current state → compare to desired → fix diff → repeat
```

Deployed in **Cluster mode** — 3 masters, 3 followers:

```
leader-0  →  slots 0–5460      + follower
leader-1  →  slots 5461–10922  + follower
leader-2  →  slots 10923–16383 + follower
```

When a pod dies:
1. Operator detects missing pod instantly
2. Recreates it
3. Re-joins it to the cluster
4. Rebalances slots if needed
5. All automatic, in seconds

**This is what Helm cannot do.**

---

## Key Difference

| | Helm HA | Operator |
|---|---|---|
| Knows Redis internals | No | Yes |
| Self-healing | Basic (restart only) | Full (re-join, rebalance) |
| Failover | Sentinel (manual setup) | Built-in |
| Day-2 ops | Manual | Automated |

---

## Requirements

```bash
brew install kind helm
```

---

## Structure

```
operator/   — Redis Operator cluster: redis-test  (3 leaders, no replicas)
helm/       — Redis HA Helm cluster: redis-helm   (1 master + 2 replicas)
```

---

## Operator Setup

```bash
kind create cluster --name redis-test --config operator/kind-config.yaml

helm repo add ot-helm https://ot-container-kit.github.io/helm-charts/
helm repo update
helm install redis-operator ot-helm/redis-operator \
  --namespace ot-operators --create-namespace \
  --set featureGates.GenerateConfigInInitContainer=true

kubectl apply -f operator/redis-cluster.yaml
kubectl get pods -n ot-operators --context kind-redis-test
```

## Helm Setup

```bash
kind create cluster --name redis-helm --config helm/kind-config.yaml

helm repo add dandydev https://dandydeveloper.github.io/charts/
helm repo update
helm install redis-ha dandydev/redis-ha \
  --namespace redis-helm --create-namespace \
  -f helm/values.yaml

kubectl get pods -n redis-helm --context kind-redis-helm
```

---

## Self-Healing Test

### Commands

**Operator** — seed a key, kill a leader, verify recovery:

```bash
kubectl exec -it redis-cluster-leader-0 -n ot-operators --context kind-redis-test \
  -- redis-cli set testkey "value"

kubectl delete pod redis-cluster-leader-0 -n ot-operators --context kind-redis-test
kubectl get pods -n ot-operators --context kind-redis-test -w

kubectl exec -it redis-cluster-leader-0 -n ot-operators --context kind-redis-test \
  -- redis-cli cluster nodes
```

**Helm** — kill master, watch sentinel promote:

```bash
kubectl delete pod redis-ha-server-0 -n redis-helm --context kind-redis-helm
kubectl get pods -n redis-helm --context kind-redis-helm -w

kubectl exec redis-ha-server-1 -n redis-helm --context kind-redis-helm \
  -c sentinel -- redis-cli -p 26379 sentinel masters
```

---

### Results (observed)

**Operator:**
- Pod recovered in ~19 seconds
- follower-0 promoted to master automatically
- leader-0 came back as slave
- Data redirected via `MOVED` — cluster protocol handles it, client follows automatically
- Zero manual intervention

**Helm HA:**
- server-0 restarted by Kubernetes
- Sentinel negotiated new master among remaining pods
- Recovery slower — sentinel needs election timeout before promoting
- No cluster slot management (HA mode, not cluster mode)
- If sentinel quorum fails — manual `redis-cli sentinel failover` needed

---

## Cleanup

```bash
kind delete cluster --name redis-test
kind delete cluster --name redis-helm
```

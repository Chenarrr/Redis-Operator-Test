# Redis Operator vs Bitnami Helm — PoC

Proof of concept comparing Redis self-healing between the **ot-container-kit Redis Operator**
and the **Bitnami Redis Cluster Helm chart** on local Kubernetes clusters.

---

## What We Built

Two fully independent Kubernetes clusters running simultaneously on a single laptop using **kind**
(Kubernetes in Docker). Each cluster runs a production-grade Redis setup — one managed by an
operator, one managed by Helm.

```
Laptop
├── Docker
│   ├── Cluster 1: redis-test      (Operator)
│   │   ├── redis-test-control-plane
│   │   ├── redis-test-worker
│   │   └── redis-test-worker2
│   └── Cluster 2: redis-helm      (Bitnami Helm)
│       ├── redis-helm-control-plane
│       ├── redis-helm-worker
│       └── redis-helm-worker2
```

6 Docker containers total. Each container = a Kubernetes node running inside Docker.
Control-plane nodes handle scheduling only. Workers run the Redis pods.

---

## Infrastructure

### Why kind over minikube

kind runs each Kubernetes node as a real Docker container — proper multi-node cluster locally.
Redis Cluster requires a minimum of 3 masters, which needs real node separation for anti-affinity.

### Cluster topology (both clusters identical)

```
1 control-plane  →  scheduling only, no workloads
1 worker         →  Redis pods
1 worker         →  Redis pods
```

---

## Pod Distribution

### Cluster 1 — Operator (`redis-test`)

```
redis-test-worker
├── redis-cluster-leader-0     (master → slots 0–5460)
├── redis-cluster-leader-2     (master → slots 10923–16383)
├── redis-cluster-follower-0   (replica)
└── redis-cluster-follower-2   (replica)

redis-test-worker2
├── redis-cluster-leader-1     (master → slots 5461–10922)
├── redis-cluster-follower-1   (replica)
└── redis-operator             (controller — watches cluster 24/7)
```

**Total: 7 pods** — 3 leaders + 3 followers + 1 operator

### Cluster 2 — Bitnami Helm (`redis-helm`)

```
redis-helm-worker
├── redis-bitnami-redis-cluster-0   (master → slots 0–5460)
├── redis-bitnami-redis-cluster-2   (master → slots 10923–16383)
└── redis-bitnami-redis-cluster-5   (replica of cluster-1)

redis-helm-worker2
├── redis-bitnami-redis-cluster-1   (master → slots 5461–10922)
├── redis-bitnami-redis-cluster-3   (replica of cluster-2)
└── redis-bitnami-redis-cluster-4   (replica of cluster-0)
```

**Total: 6 pods** — 3 masters + 3 replicas

---

## How Redis Cluster Works

16384 hash slots split across 3 masters. Every key hashes to a slot, every slot belongs to a master.

```
master-0  →  slots 0–5460      ~33% of data
master-1  →  slots 5461–10922  ~33% of data
master-2  →  slots 10923–16383 ~33% of data
```

Each master has 1 replica watching it. If a master dies, its replica promotes via cluster gossip vote.

---

## The Difference

### Bitnami Helm

Helm deploys Redis and walks away. No reconciliation loop. When a pod dies:
- Kubernetes restarts the container
- Cluster gossip protocol re-elects master among surviving nodes
- **Complex failures** (node loss, split-brain, network partition) need `redis-cli cluster fix` manually

### Redis Operator

Operator runs a reconciliation loop forever:

```
observe current state → compare to desired YAML → fix the diff → repeat
```

When a pod dies:
- Operator detects it instantly (watches Kubernetes API — no timeout)
- Recreates the pod
- Re-joins it to the cluster automatically
- Rebalances slots if needed
- **All failures** handled automatically including complex ones

---

## Comparison Table

| | **Bitnami Helm** | **Redis Operator** |
|---|---|---|
| Mode | Cluster (sharded) | Cluster (sharded) |
| Masters | 3 | 3 |
| Replicas | 3 (1 per master) | 3 (1 per leader) |
| Total pods | 6 | 7 (includes operator) |
| Data distribution | Hash slots across masters | Hash slots across leaders |
| Failover mechanism | Cluster gossip protocol | Operator + cluster protocol |
| Failure detection | Gossip timeout (~15s) | Instant (K8s API watch) + periodic loop every ~12s |
| Self-healing | Pod restart only (promotion only if down > 15s) | Re-join + rebalance + promote (always) |
| Complex failure recovery | Manual | Automatic |
| Scaling | Manual resharding | Operator handles it |
| Version upgrades | Manual coordination | Rolling update via operator |
| Day-2 ops | Manual | Automated |
| Recovery speed (observed) | ~25 seconds | ~19 seconds |
| Data survived recovery | Yes | Yes |
| Image | bitnamilegacy/redis-cluster | quay.io/opstree/redis:v7.0.15 |
| Free to use | Was (paywalled Aug 2025) | Yes |
| Production verdict | Requires ops team | Recommended |

---

## What Happened During Testing

### Test: Kill a master pod in each cluster

**Seeded a key before killing:**

```bash
# Operator
kubectl exec -it redis-cluster-leader-0 -n ot-operators --context kind-redis-test \
  -- redis-cli set testkey "value"

# Bitnami
kubectl exec -it redis-bitnami-redis-cluster-0 -n redis-helm --context kind-redis-helm \
  -- redis-cli -c set testkey "bitnami-value"
```

**Killed the master pod in both clusters simultaneously.**

**Operator result:**
- leader-0 detected missing instantly
- follower-0 promoted to master automatically
- leader-0 came back as slave in ~19 seconds
- Data accessible via `MOVED` redirect — cluster protocol routes client automatically
- Zero manual steps

**Bitnami result:**
- Relies on gossip timeout (cluster-node-timeout=15000ms) to declare a node dead
- If the pod restarts faster than 15s (cached image = ~7s), gossip never fires — **no promotion, pod comes back as master**
- Only promotes if the pod stays down longer than 15s (e.g. node-level failure, slow image pull)
- Data survived either way
- Zero manual steps for basic failure
- Would need manual intervention for split-brain or node-level failure

### Promotion observed (Operator)

```
Before kill:   leader-0 = master,  follower-0 = slave
After kill:    leader-0 = slave,   follower-0 = master  ← promoted
```

The pod names (leader/follower) are just labels assigned at creation — roles flip based on cluster state.

---

## Image Situation

Bitnami paywalled all container images on **August 28, 2025** — deleted from Docker Hub entirely.
We found `bitnamilegacy/redis-cluster:latest` — the last free image Bitnami pushed before the paywall
(built May 30, 2025). Used with the official Bitnami chart via `--set image.*` overrides.

---

## Setup

**Requirements:**

```bash
brew install kind helm
```

**Operator cluster:**

```bash
kind create cluster --name redis-test --config operator/kind-config.yaml

helm repo add ot-helm https://ot-container-kit.github.io/helm-charts/
helm repo update
helm install redis-operator ot-helm/redis-operator \
  --namespace ot-operators --create-namespace \
  --set featureGates.GenerateConfigInInitContainer=true

kubectl apply -f operator/redis-cluster.yaml
kubectl get pods -n ot-operators --context kind-redis-test -w
```

**Bitnami Helm cluster:**

```bash
kind create cluster --name redis-helm --config helm/kind-config.yaml

helm install redis-bitnami oci://registry-1.docker.io/bitnamicharts/redis-cluster \
  --version 10.3.0 \
  --namespace redis-helm --create-namespace \
  --set auth.enabled=false \
  --set image.registry=docker.io \
  --set image.repository=bitnamilegacy/redis-cluster \
  --set image.tag=latest

kubectl get pods -n redis-helm --context kind-redis-helm -w
```

---

## Watch Operator Reconciliation Loop

```bash
kubectl logs -f redis-operator-58c8bb9c8b-dcxk8 -n ot-operators --context kind-redis-test \
  | grep --line-buffered "Number of Redis nodes\|reconcileID" \
  | awk '{print $2, $NF}'
```

Fires every ~10 seconds (hardcoded). Interval not configurable via Helm or flags — would require recompiling the operator.

---

## Self-Healing Test Commands

**Operator:**

```bash
kubectl exec -it redis-cluster-leader-0 -n ot-operators --context kind-redis-test \
  -- redis-cli set testkey "value"

kubectl delete pod redis-cluster-leader-0 -n ot-operators --context kind-redis-test

kubectl get pods -n ot-operators --context kind-redis-test -w

kubectl exec redis-cluster-leader-0 -n ot-operators --context kind-redis-test \
  -- redis-cli cluster nodes 2>/dev/null | \
  awk '{split($2,a,","); split(a[2],b,"."); printf "%-35s %-20s %s\n", b[1], $3, $9}'
```

**Bitnami Helm:**

```bash
kubectl exec -it redis-bitnami-redis-cluster-0 -n redis-helm --context kind-redis-helm \
  -- redis-cli -c set testkey "value"

kubectl delete pod redis-bitnami-redis-cluster-0 -n redis-helm --context kind-redis-helm

kubectl get pods -n redis-helm --context kind-redis-helm -w

kubectl exec redis-bitnami-redis-cluster-1 -n redis-helm --context kind-redis-helm \
  -- redis-cli -c cluster nodes 2>/dev/null | \
  while read line; do
    ip=$(echo "$line" | awk '{split($2,a,":"); print a[1]}')
    pod=$(kubectl get pods -n redis-helm --context kind-redis-helm --no-headers \
      -o custom-columns="NAME:.metadata.name,IP:.status.podIP" 2>/dev/null | \
      awk -v i="$ip" '$2==i{print $1}')
    role=$(echo "$line" | awk '{print $3}')
    slots=$(echo "$line" | awk '{print $9}')
    printf "%-40s %-20s %s\n" "${pod:-$ip}" "$role" "$slots"
  done
```

---

## Cleanup

```bash
kind delete cluster --name redis-test
kind delete cluster --name redis-helm
```

---

## References

| Resource | Link |
|---|---|
| Redis Operator guide | https://ot-container-kit.github.io/redis-operator/guide/ |
| RedisCluster config reference | https://ot-container-kit.github.io/redis-operator/guide/redis-cluster-config.html |
| GitHub — redis-operator | https://github.com/OT-CONTAINER-KIT/redis-operator |
| Redis Cluster spec (hash slots, gossip, failover) | https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/ |
| kind (Kubernetes in Docker) | https://github.com/kubernetes-sigs/kind |
| Bitnami redis-cluster Helm chart | https://github.com/bitnami/charts/tree/main/bitnami/redis-cluster |

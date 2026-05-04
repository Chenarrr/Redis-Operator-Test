# Redis Operator vs Bitnami Helm Sentinel — PoC

Side-by-side comparison of Redis self-healing on local Kubernetes clusters.

| | OT-Container-Kit Redis Operator | Bitnami Redis Helm (Sentinel) |
|---|---|---|
| Operator version | v0.24.0 | — |
| Chart version | ot-helm/redis-operator | bitnamicharts/redis v20.6.3 |
| Redis version | 7.0.15 | 8.2.1 |
| Image | quay.io/opstree/redis:v7.0.15 | bitnamilegacy/redis:8.2.1 |
| Mode | Cluster (sharded) | Sentinel (single master) |

---

## Repo Structure

```
redis-operator-vs-sentinel/
├── ot-operator/
│   ├── kind-config.yaml          # kind cluster topology (1 control-plane + 2 workers)
│   └── redis-cluster-crd.yaml   # RedisCluster CRD manifest (ot-container-kit)
├── bitnami-sentinel/
│   ├── kind-config.yaml          # kind cluster topology (1 control-plane + 2 workers)
│   └── helm-values.yaml          # Bitnami redis chart values (sentinel mode)
└── README.md
```

---

## What We Built

Two independent Kubernetes clusters running simultaneously on a single laptop using **kind** v0.27+
(Kubernetes in Docker). Each cluster runs a production-grade Redis setup.

```
Laptop
├── Docker
│   ├── Cluster 1: redis-test        (OT Operator — Cluster mode)
│   │   ├── redis-test-control-plane
│   │   ├── redis-test-worker
│   │   └── redis-test-worker2
│   └── Cluster 2: redis-helm        (Bitnami Sentinel)
│       ├── redis-helm-control-plane
│       ├── redis-helm-worker
│       └── redis-helm-worker2
```

6 Docker containers total. Each container = a Kubernetes node.
Control-plane nodes handle scheduling only — workers run Redis pods.

---

## Pod Distribution

### Cluster 1 — OT Operator (`redis-test`)

```
redis-test-worker
├── redis-cluster-leader-0     (master → slots 0–5460)
├── redis-cluster-leader-2     (master → slots 10923–16383)
├── redis-cluster-follower-0   (replica)
└── redis-cluster-follower-2   (replica)

redis-test-worker2
├── redis-cluster-leader-1     (master → slots 5461–10922)
├── redis-cluster-follower-1   (replica)
└── redis-operator             (controller — reconciles every ~10s)
```

**7 pods** — 3 leaders + 3 followers + 1 operator controller

### Cluster 2 — Bitnami Helm Sentinel (`redis-helm`)

```
redis-helm-worker
├── redis-bitnami-node-0   (master)  [2 containers: redis + sentinel]
└── redis-bitnami-node-2   (slave)   [2 containers: redis + sentinel]

redis-helm-worker2
└── redis-bitnami-node-1   (slave)   [2 containers: redis + sentinel]
```

**3 pods, 6 containers** — 1 master + 2 slaves, each pod runs redis + sentinel sidecar

---

## Architecture

### OT Operator — Redis Cluster (sharded)

Data split across 3 masters via 16384 hash slots. Every key hashes to a slot.

```
leader-0  →  slots 0–5460      (~33% of data)
leader-1  →  slots 5461–10922  (~33% of data)
leader-2  →  slots 10923–16383 (~33% of data)
```

The operator pod runs a reconciliation loop 24/7:

```
observe state → compare to redis-cluster-crd.yaml → fix diff → repeat (every ~10s)
```

On pod death: operator detects instantly via K8s API watch → promotes replica → re-joins
recovered pod as slave. Always fires regardless of restart speed.

### Bitnami Sentinel (single master)

One master holds **all** data. Two slaves mirror it. Sentinel processes (one per pod)
watch the master and vote on failover (quorum = 2/3).

```
master  →  holds 100% of data
slave-1 →  full copy
slave-2 →  full copy
```

Failover only fires if master is unreachable for `down-after-milliseconds` (60s default).
If pod restarts faster than 60s (cached image = ~7s), sentinel never declares it dead —
pod returns as master, no promotion happens.

---

## Comparison

| | **Bitnami Helm Sentinel** | **OT Operator** |
|---|---|---|
| Mode | Sentinel (single master) | Cluster (sharded) |
| Architecture | 1 master + 2 slaves | 3 leaders + 3 followers |
| Total pods | 3 | 7 (includes operator) |
| Containers per pod | 2 (redis + sentinel) | 1 |
| Data distribution | All on master (no sharding) | Hash slots — scales horizontally |
| Max capacity | Single node RAM | Add more leaders to expand |
| Failure detection | down-after-ms timeout (60s) | Instant (K8s API watch) |
| Reconciliation loop | None — Helm walks away | Every ~10s + event-driven |
| Failover guarantee | Only if down > 60s | Always, regardless of restart speed |
| Complex failure recovery | Manual (`sentinel reset`) | Automatic |
| Scaling | Manual | Operator handles resharding |
| Rolling upgrades | Manual | Operator coordinates |
| Day-2 ops | Manual | Automated |
| Data survived recovery | Yes | Yes |
| Redis version | 8.2.1 | 7.0.15 |
| Image source | bitnamilegacy (last free, Aug 2025) | quay.io/opstree (always free) |
| Production verdict | Simple HA, small datasets | Recommended for scale |

---

## Official Open Source Sentinel Operator

The same OT-Container-Kit operator supports `RedisSentinel` (same binary, different CRD kind):

```yaml
apiVersion: redis.redis.opstreelabs.in/v1beta2
kind: RedisSentinel
```

Docs: https://ot-container-kit.github.io/redis-operator/guide/

---

## Prerequisites

```bash
brew install kind helm
```

Versions used:

| Tool | Version |
|---|---|
| kind | v0.27+ |
| helm | v3.x |
| kubectl | v1.32+ |
| Docker | v29+ |

---

## Setup — OT Operator Cluster

```bash
# 1. Create the kind cluster
kind create cluster --name redis-test --config ot-operator/kind-config.yaml

# 2. Add the OT-Container-Kit Helm repo
helm repo add ot-helm https://ot-container-kit.github.io/helm-charts/
helm repo update

# 3. Install the operator (v0.24.0)
helm install redis-operator ot-helm/redis-operator \
  --namespace ot-operators \
  --create-namespace \
  --set featureGates.GenerateConfigInInitContainer=true

# 4. Deploy the RedisCluster CRD (3 leaders + 3 followers, Redis v7.0.15)
kubectl apply -f ot-operator/redis-cluster-crd.yaml

# 5. Watch pods come up (~2 min)
kubectl get pods -n ot-operators --context kind-redis-test -w
```

Expected output (7 pods):

```
redis-cluster-follower-0   1/1   Running
redis-cluster-follower-1   1/1   Running
redis-cluster-follower-2   1/1   Running
redis-cluster-leader-0     1/1   Running
redis-cluster-leader-1     1/1   Running
redis-cluster-leader-2     1/1   Running
redis-operator-xxx         1/1   Running
```

---

## Setup — Bitnami Sentinel Cluster

```bash
# 1. Create the kind cluster
kind create cluster --name redis-helm --config bitnami-sentinel/kind-config.yaml

# 2. Install Bitnami redis chart (v20.6.3, Redis 8.2.1, Sentinel mode)
#    Uses bitnamilegacy images — last free Bitnami images before Aug 28 2025 paywall
helm install redis-bitnami oci://registry-1.docker.io/bitnamicharts/redis \
  --version 20.6.3 \
  --namespace redis-helm \
  --create-namespace \
  -f bitnami-sentinel/helm-values.yaml

# 3. Watch pods come up (~3 min — pulls two images)
kubectl get pods -n redis-helm --context kind-redis-helm -w
```

Expected output (3 pods, 2 containers each):

```
redis-bitnami-node-0   2/2   Running
redis-bitnami-node-1   2/2   Running
redis-bitnami-node-2   2/2   Running
```

---

## Verify Cluster State

### OT Operator — cluster roles and slot assignments

```bash
kubectl exec redis-cluster-leader-0 -n ot-operators --context kind-redis-test \
  -- redis-cli cluster nodes 2>/dev/null | \
  awk '{split($2,a,","); split(a[2],b,"."); printf "%-35s %-20s %s\n", b[1], $3, $9}'
```

### Bitnami Sentinel — which pod is master

```bash
for pod in redis-bitnami-node-0 redis-bitnami-node-1 redis-bitnami-node-2; do
  role=$(kubectl exec "$pod" -n redis-helm --context kind-redis-helm \
    -c redis -- redis-cli role 2>/dev/null | head -1)
  printf "%-25s %s\n" "$pod" "$role"
done
```

### Bitnami Sentinel — sentinel view

```bash
kubectl exec redis-bitnami-node-0 -n redis-helm --context kind-redis-helm \
  -c sentinel -- redis-cli -p 26379 sentinel masters 2>/dev/null | \
  grep -E "^name$|^ip$|^flags$|^num-slaves$|^quorum$|^down-after" | paste - -
```

---

## Watch Operator Reconciliation Loop

The operator reconciles every ~10 seconds (hardcoded — not configurable via Helm or flags):

```bash
kubectl logs -f redis-operator-58c8bb9c8b-dcxk8 \
  -n ot-operators \
  --context kind-redis-test \
  | grep --line-buffered "Number of Redis nodes\|reconcileID" \
  | awk '{print $2, $NF}'
```

---

## Self-Healing Test

### Step 1 — seed a key in both clusters

```bash
# OT Operator
kubectl exec redis-cluster-leader-0 \
  -n ot-operators \
  --context kind-redis-test \
  -- redis-cli set testkey "operator-value"

# Bitnami Sentinel
kubectl exec redis-bitnami-node-0 \
  -n redis-helm \
  --context kind-redis-helm \
  -c redis -- redis-cli set testkey "sentinel-value"
```

### Step 2 — kill the master pod in both

```bash
# OT Operator — kill leader-0
kubectl delete pod redis-cluster-leader-0 \
  -n ot-operators \
  --context kind-redis-test

# Bitnami Sentinel — kill master
kubectl delete pod redis-bitnami-node-0 \
  -n redis-helm \
  --context kind-redis-helm
```

### Step 3 — watch recovery

```bash
# OT Operator
kubectl get pods -n ot-operators --context kind-redis-test -w

# Bitnami Sentinel
kubectl get pods -n redis-helm --context kind-redis-helm -w
```

### Step 4 — verify roles and data after recovery

```bash
# OT Operator — check new cluster roles
kubectl exec redis-cluster-leader-1 \
  -n ot-operators \
  --context kind-redis-test \
  -- redis-cli cluster nodes 2>/dev/null | \
  awk '{split($2,a,","); split(a[2],b,"."); printf "%-35s %-20s %s\n", b[1], $3, $9}'

# OT Operator — data survived?
kubectl exec redis-cluster-leader-0 \
  -n ot-operators \
  --context kind-redis-test \
  -- redis-cli get testkey 2>/dev/null

# Bitnami Sentinel — check new roles
for pod in redis-bitnami-node-0 redis-bitnami-node-1 redis-bitnami-node-2; do
  role=$(kubectl exec "$pod" -n redis-helm --context kind-redis-helm \
    -c redis -- redis-cli role 2>/dev/null | head -1)
  printf "%-25s %s\n" "$pod" "$role"
done

# Bitnami Sentinel — data survived?
kubectl exec redis-bitnami-node-1 \
  -n redis-helm \
  --context kind-redis-helm \
  -c redis -- redis-cli get testkey 2>/dev/null
```

### What to expect

| | OT Operator | Bitnami Sentinel |
|---|---|---|
| Detection | Instant (K8s API watch) | After 60s timeout |
| Killed pod returns as | Slave (operator enforces it) | Master (if restart < 60s) |
| Promotion fires | Always | Only if pod down > 60s |
| Data | Survived | Survived |

---

## Image Situation

Bitnami paywalled all container images on **August 28, 2025** — deleted from Docker Hub.

- `bitnamilegacy/redis:8.2.1` — last free Redis image Bitnami pushed (built Aug 22, 2025)
- `bitnamilegacy/redis-sentinel:8.2.1` — last free Redis Sentinel image (built Aug 22, 2025)

Used with official `bitnamicharts/redis` chart v20.6.3 via `global.security.allowInsecureImages: true`
and image overrides in `bitnami-sentinel/helm-values.yaml`.

OT Operator uses `quay.io/opstree/redis:v7.0.15` — always free, no paywall.

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
| OT Operator guide | https://ot-container-kit.github.io/redis-operator/guide/ |
| OT Operator — RedisCluster config | https://ot-container-kit.github.io/redis-operator/guide/redis-cluster-config.html |
| GitHub — redis-operator (v0.24.0) | https://github.com/OT-CONTAINER-KIT/redis-operator |
| Bitnami redis Helm chart | https://github.com/bitnami/charts/tree/main/bitnami/redis |
| Redis Sentinel docs | https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/ |
| Redis Cluster spec | https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/ |
| kind (Kubernetes in Docker) | https://github.com/kubernetes-sigs/kind |

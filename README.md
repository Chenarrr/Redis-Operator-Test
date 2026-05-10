# Redis Operator vs Bitnami Helm Sentinel

Side-by-side comparison of Redis HA, sentinel behavior, backup, and migration on Kubernetes.

| | OT-Container-Kit Redis Operator | Bitnami Redis Helm (Sentinel) |
|---|---|---|
| Operator version | v0.24.0 | — |
| Chart version | ot-helm/redis-operator | bitnamicharts/redis v20.6.3 |
| Redis version | 8.6.2 | 8.2.1 |
| Image | quay.io/opstree/redis:v8.6.2 | bitnamilegacy/redis:8.2.1 |
| Sentinel image | quay.io/opstree/redis-sentinel:v8.6.2 | bitnamilegacy/redis-sentinel:8.2.1 |
| Mode | Sentinel (operator-managed) | Sentinel (Helm-managed) |

---

## Repo Structure

```
├── ot-operator/
│   ├── kind-config.yaml             # kind cluster topology (1 control-plane + 2 workers)
│   ├── redis-replication-crd.yaml   # RedisReplication CRD (1 master + 2 replicas)
│   └── redis-sentinel-crd.yaml      # RedisSentinel CRD (3 sentinel pods)
├── bitnami-sentinel/
│   ├── kind-config.yaml             # kind cluster topology (1 control-plane + 2 workers)
│   └── helm-values.yaml             # Bitnami redis chart values (sentinel mode)
└── README.md
```

---

## What We Built

Two independent Kubernetes clusters running simultaneously on a single machine using **kind** v0.27+
(Kubernetes in Docker).

```
Machine
├── Docker
│   ├── Cluster 1: redis-test        (OT Operator — Sentinel mode)
│   │   ├── redis-test-control-plane
│   │   ├── redis-test-worker
│   │   └── redis-test-worker2
│   └── Cluster 2: redis-helm        (Bitnami — Sentinel mode)
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
├── redis-replication-0   (master)
├── redis-replication-1   (replica)
├── redis-sentinel-0      (dedicated sentinel pod)
└── redis-sentinel-1      (dedicated sentinel pod)

redis-test-worker2
├── redis-replication-2   (replica)
├── redis-sentinel-2      (dedicated sentinel pod)
└── redis-operator        (controller — reconciles K8s state every ~10s)
```

**7 pods** — 1 master + 2 replicas (RedisReplication) + 3 sentinels (RedisSentinel) + 1 operator controller

### Cluster 2 — Bitnami Helm Sentinel (`redis-helm`)

```
redis-helm-worker
├── redis-bitnami-node-0   (master)  [2 containers: redis + sentinel sidecar]
└── redis-bitnami-node-2   (slave)   [2 containers: redis + sentinel sidecar]

redis-helm-worker2
└── redis-bitnami-node-1   (slave)   [2 containers: redis + sentinel sidecar]
```

**3 pods, 6 containers** — 1 master + 2 slaves, each pod runs redis + sentinel as a sidecar


---

## Comparison

| | **Bitnami Helm Sentinel** | **OT Operator Sentinel** |
|---|---|---|
| Mode | Sentinel (Helm-managed) | Sentinel (operator-managed) |
| Architecture | 1 master + 2 replicas + sentinel sidecar per pod | 1 master + 2 replicas (RedisReplication) + 3 sentinel pods (RedisSentinel) |
| Total pods | 3 | 7 (3 replication + 3 sentinel + 1 operator) |
| Containers per pod | 2 (redis + sentinel sidecar) | 1 |
| Sentinel placement | Co-located in each Redis pod | Dedicated separate pods |
| Who does failover | Sentinel | Sentinel (same protocol, same 60s timeout) |
| Sentinels lost on master crash | 1 (co-located sidecar dies) | 0 (separate pods survive) |
| Sentinels alive after crash | 2/3 | 3/3 |
| Quorum after crash | Barely met (2/2 needed) | Comfortably met (3/2 needed) |
| Recovery of crashed pod | Returns as master (if < 60s) or slave | Always returns as replica (operator enforces) |
| Reconciliation loop | None — Helm walks away | Every ~10s + event-driven |
| Rolling upgrades | Manual | Operator coordinates |
| Day-2 ops | Manual | Automated |
| Image source | bitnamilegacy (last free, Aug 2025) | quay.io/opstree (always free, actively updated) |
| Production fit | Simple HA, minimal overhead | Recommended for automated day-2 ops |

---

## Performance

Both use Sentinel architecture — single master handles all writes, replicas serve reads.
Throughput profile is identical. Difference is in operations, not performance.

| | Both (Sentinel) |
|---|---|
| Write throughput | Single master — scales vertically only |
| Read throughput | Replicas can serve reads (`READONLY`) |
| Horizontal scale | No — limited to one node's RAM |

**Client connection — same for both:**

| Client | Connection |
|---|---|
| redis-cli | `redis-cli -h <sentinel-host> -p 26379` |
| Python (redis-py) | `Sentinel([("host", 26379)]).master_for("mymaster")` |
| Node.js (ioredis) | `new Redis({ sentinels: [...], name: "mymaster" })` |
| Java (Lettuce) | `RedisSentinelClient.create("redis-sentinel://host:26379/mymaster")` |

---

## Backup

### Option 1 — RDB Snapshot (built-in, simplest)

Redis writes a point-in-time `.rdb` file to disk at `/data/dump.rdb`.

```bash
# OT Operator — trigger save on master
kubectl exec redis-replication-0 -n ot-operators --context kind-redis-test \
  -- redis-cli bgsave

# Check completed
kubectl exec redis-replication-0 -n ot-operators --context kind-redis-test \
  -- redis-cli lastsave

# Bitnami — trigger save on master
kubectl exec redis-bitnami-node-0 -n redis-helm --context kind-redis-helm \
  -c redis -- redis-cli bgsave
```

Restore: copy `dump.rdb` into the PVC and restart the pod.

### Option 2 — AOF (Append-Only File)

Logs every write operation. Near-zero data loss, larger files.

Enable in OT Operator via `redis-replication-crd.yaml`:

```yaml
spec:
  redisConfig:
    additionalRedisConfig: |
      appendonly yes
      appendfsync everysec
```

Enable in Bitnami via `helm-values.yaml`:

```yaml
commonConfiguration: |
  appendonly yes
  appendfsync everysec
```

### Option 3 — PVC Snapshot (Kubernetes-native)

If storage class supports `VolumeSnapshot`:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: redis-replication-0-backup
  namespace: ot-operators
spec:
  volumeSnapshotClassName: csi-hostpath-snapclass
  source:
    persistentVolumeClaimName: redis-replication-0
EOF
```

Works for both setups — any PVC-backed Redis.



### Option 4 — RIOT (key-level export, best for cross-cluster migration)

```bash
brew install redis/tap/riot

riot replicate \
  --source-uri redis-sentinel://sentinel-host:26379?master=mymaster \
  --target-uri redis-sentinel://new-sentinel-host:26379?master=mymaster \
  --mode live
```

### Backup comparison

| Method | Granularity | Recovery time | Requires | Best for |
|---|---|---|---|---|
| RDB snapshot | Point-in-time | Fast | Nothing extra | Simple, periodic backups |
| AOF | Near real-time | Slower (replay log) | Disk space | Zero data loss requirement |
| PVC snapshot | Point-in-time | Fast | CSI driver | Kubernetes-native workflows |
| RIOT | Key-level | Depends on size | Extra tooling | Cross-cluster migration |

---

## Prerequisites

```bash
brew install kind helm
```

| Tool | Version |
|---|---|
| kind | v0.27+ |
| helm | v3.x |
| kubectl | v1.32+ |
| Docker | v29+ |

---

## Setup — OT Operator Sentinel

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

# 4. Deploy RedisReplication (1 master + 2 replicas, Redis v8.6.2)
kubectl apply -f ot-operator/redis-replication-crd.yaml

# 5. Deploy RedisSentinel (3 sentinel pods)
kubectl apply -f ot-operator/redis-sentinel-crd.yaml

# 6. Watch pods come up (~2 min)
kubectl get pods -n ot-operators --context kind-redis-test -w
```

Expected (7 pods):

```
redis-operator-xxx           1/1   Running
redis-replication-0          1/1   Running
redis-replication-1          1/1   Running
redis-replication-2          1/1   Running
redis-sentinel-sentinel-0    1/1   Running
redis-sentinel-sentinel-1    1/1   Running
redis-sentinel-sentinel-2    1/1   Running
```

---

## Setup — Bitnami Sentinel

```bash
# 1. Create the kind cluster
kind create cluster --name redis-helm --config bitnami-sentinel/kind-config.yaml

# 2. Install Bitnami redis chart (v20.6.3, Redis 8.2.1, Sentinel mode)
helm install redis-bitnami oci://registry-1.docker.io/bitnamicharts/redis \
  --version 20.6.3 \
  --namespace redis-helm \
  --create-namespace \
  -f bitnami-sentinel/helm-values.yaml

# 3. Watch pods come up (~3 min — pulls two images)
kubectl get pods -n redis-helm --context kind-redis-helm -w
```

Expected (3 pods, 2 containers each):

```
redis-bitnami-node-0   2/2   Running
redis-bitnami-node-1   2/2   Running
redis-bitnami-node-2   2/2   Running
```

---

## Verify Cluster State

### OT Operator — replication roles

```bash
for pod in redis-replication-0 redis-replication-1 redis-replication-2; do
  role=$(kubectl exec "$pod" -n ot-operators --context kind-redis-test \
    -- redis-cli role 2>/dev/null | head -1)
  printf "%-25s %s\n" "$pod" "$role"
done
```

### OT Operator — sentinel view

```bash
kubectl exec redis-sentinel-sentinel-0 -n ot-operators --context kind-redis-test \
  -- redis-cli -p 26379 sentinel masters 2>/dev/null | \
  grep -E "^name$|^ip$|^flags$|^num-slaves$|^quorum$|^down-after" | paste - -
```

### Bitnami — which pod is master

```bash
for pod in redis-bitnami-node-0 redis-bitnami-node-1 redis-bitnami-node-2; do
  role=$(kubectl exec "$pod" -n redis-helm --context kind-redis-helm \
    -c redis -- redis-cli role 2>/dev/null | head -1)
  printf "%-25s %s\n" "$pod" "$role"
done
```

### Bitnami — sentinel view

```bash
kubectl exec redis-bitnami-node-1 -n redis-helm --context kind-redis-helm \
  -c sentinel -- redis-cli -p 26379 sentinel masters 2>/dev/null | \
  grep -E "^name$|^ip$|^flags$|^num-slaves$|^quorum$|^down-after" | paste - -
```

---

## Watch Operator Reconciliation Loop

Fires every ~10 seconds (hardcoded — not configurable via Helm or flags):

```bash
kubectl logs -f \
  $(kubectl get pods -n ot-operators --context kind-redis-test \
    -l app=redis-operator -o jsonpath='{.items[0].metadata.name}') \
  -n ot-operators --context kind-redis-test \
  | grep --line-buffered "Number of Redis nodes\|reconcileID" \
  | awk '{print $2, $NF}'
```

---

## Self-Healing Test

### Step 1 — seed a key

```bash
# OT Operator — seed to current master
kubectl exec redis-replication-0 \
  -n ot-operators --context kind-redis-test \
  -- redis-cli set testkey "operator-value"

# Bitnami — find master, then seed
for pod in redis-bitnami-node-0 redis-bitnami-node-1 redis-bitnami-node-2; do
  role=$(kubectl exec "$pod" -n redis-helm --context kind-redis-helm \
    -c redis -- redis-cli role 2>/dev/null | head -1)
  [ "$role" = "master" ] && \
    kubectl exec "$pod" -n redis-helm --context kind-redis-helm \
      -c redis -- redis-cli set testkey "sentinel-value" && break
done
```

### Step 2 — kill the master

```bash
# OT Operator
kubectl delete pod redis-replication-0 -n ot-operators --context kind-redis-test

# Bitnami — kill whichever pod is currently master
```

### Step 3 — watch recovery

```bash
kubectl get pods -n ot-operators --context kind-redis-test -w
kubectl get pods -n redis-helm   --context kind-redis-helm -w
```

### Step 4 — verify roles and data

```bash
# OT Operator — new roles
for pod in redis-replication-0 redis-replication-1 redis-replication-2; do
  role=$(kubectl exec "$pod" -n ot-operators --context kind-redis-test \
    -- redis-cli role 2>/dev/null | head -1)
  printf "%-25s %s\n" "$pod" "$role"
done

# OT Operator — data survived (run on new master)
for pod in redis-replication-0 redis-replication-1 redis-replication-2; do
  role=$(kubectl exec "$pod" -n ot-operators --context kind-redis-test \
    -- redis-cli role 2>/dev/null | head -1)
  [ "$role" = "master" ] && \
    kubectl exec "$pod" -n ot-operators --context kind-redis-test \
      -- redis-cli get testkey 2>/dev/null && break
done

# Bitnami — new roles
for pod in redis-bitnami-node-0 redis-bitnami-node-1 redis-bitnami-node-2; do
  role=$(kubectl exec "$pod" -n redis-helm --context kind-redis-helm \
    -c redis -- redis-cli role 2>/dev/null | head -1)
  printf "%-25s %s\n" "$pod" "$role"
done

# Bitnami — data survived (run on new master)
for pod in redis-bitnami-node-0 redis-bitnami-node-1 redis-bitnami-node-2; do
  role=$(kubectl exec "$pod" -n redis-helm --context kind-redis-helm \
    -c redis -- redis-cli role 2>/dev/null | head -1)
  [ "$role" = "master" ] && \
    kubectl exec "$pod" -n redis-helm --context kind-redis-helm \
      -c redis -- redis-cli get testkey 2>/dev/null && break
done
```

### Expected results

| | OT Operator | Bitnami Sentinel |
|---|---|---|
| Who triggers failover | Sentinel (all 3 survive crash) | Sentinel (2 survive crash) |
| Failover timer | 60s (same sentinel protocol) | 60s (same sentinel protocol) |
| Killed pod returns as | Replica always (operator enforces) | Replica if promotion fired; master if pod recovered < 60s |
| Data survived | Yes | Yes |

---

## Image Situation

Bitnami paywalled all container images on **August 28, 2025**.

- `bitnamilegacy/redis:8.2.1` — last free Redis image (Aug 22, 2025)
- `bitnamilegacy/redis-sentinel:8.2.1` — last free Sentinel image (Aug 22, 2025)

Used with `bitnamicharts/redis` v20.6.3 via `global.security.allowInsecureImages: true`
in `bitnami-sentinel/helm-values.yaml`.

OT Operator uses `quay.io/opstree/redis:v8.6.2` — always free, actively maintained.

---

## References

| Resource | Link |
|---|---|
| OT Operator guide | https://ot-container-kit.github.io/redis-operator/guide/ |
| OT Operator — RedisSentinel config | https://ot-container-kit.github.io/redis-operator/guide/redis-sentinel-config.html |
| GitHub — redis-operator (v0.24.0) | https://github.com/OT-CONTAINER-KIT/redis-operator |
| Bitnami redis Helm chart | https://github.com/bitnami/charts/tree/main/bitnami/redis |
| Redis Sentinel docs | https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/ |
| RIOT — Redis migration tool | https://developer.redis.com/riot/ |

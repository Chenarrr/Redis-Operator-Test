# Redis Operator vs Bitnami Helm Sentinel

Side-by-side comparison of Redis self-healing, performance, backup, and migration on Kubernetes.

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
├── ot-operator/
│   ├── kind-config.yaml          # kind cluster topology (1 control-plane + 2 workers)
│   └── redis-operator-crd.yaml   # RedisCluster CRD manifest (ot-container-kit)
├── bitnami-sentinel/
│   ├── kind-config.yaml          # kind cluster topology (1 control-plane + 2 workers)
│   └── helm-values.yaml          # Bitnami redis chart values (sentinel mode)
└── README.md
```

---

## What We Built

Two independent Kubernetes clusters running simultaneously on a single machine using **kind** v0.27+
(Kubernetes in Docker).

```
Machine
├── Docker
│   ├── Cluster 1: redis-test        (OT Operator — Redis Cluster mode)
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
observe state → compare to redis-operator-crd.yaml → fix diff → repeat (every ~10s)
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

Failover fires if master is unreachable for `down-after-milliseconds` (60s default).
Even with cached images, pod restart + Redis initialization can exceed 60s — sentinel will promote
in that case. If the Redis process comes back within 60s, the old master returns and no promotion fires.

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
| Production fit | Simple HA, small datasets | Recommended for scale |

---

## Performance

### Throughput

| | Sentinel | Operator (Cluster) |
|---|---|---|
| Write throughput | Limited to single master | Scales with number of leaders — 3 masters = ~3x write capacity |
| Read throughput | Slaves can serve reads (`READONLY`) | Any node can serve reads for its slot range |
| Horizontal scale | No — only vertical (bigger node) | Yes — add more master/follower pairs |
| Memory limit | One node's RAM | Combined RAM of all masters |

### Latency

**Sentinel** — single hop. Client connects directly to master, no routing layer.

**Cluster** — may add one extra hop. If a key lives on `leader-1` but the client sends
the request to `leader-0`, Redis returns a `MOVED` redirect and the client retries on
the correct node. Cluster-aware clients handle this transparently, but it adds ~1ms in
the worst case.

```
Client → leader-0 → MOVED 5500 leader-1:6379 → Client → leader-1 → OK
```

### Client requirements

| | Sentinel | Cluster |
|---|---|---|
| Standard redis-cli | `redis-cli -h <host> -p 6379` | `redis-cli -c -h <host>` (`-c` enables cluster mode) |
| Python (redis-py) | `Redis(host, port)` | `RedisCluster(host, port)` |
| Node.js (ioredis) | `new Redis({host, port})` | `new Redis.Cluster([{host, port}])` |
| Java (Lettuce) | `RedisClient.create(uri)` | `RedisClusterClient.create(uri)` |

Sentinel mode is simpler to connect to. Cluster mode requires cluster-aware clients — not all
libraries support it equally. This is a real migration cost.

### When to use which

| Use Sentinel when | Use Cluster (Operator) when |
|---|---|
| Dataset fits in one node's RAM | Dataset exceeds one node's RAM |
| Simple client setup matters | High write throughput is needed |
| Team is small, ops overhead matters | Team can operate Kubernetes properly |
| < 10 GB data | > 10 GB or growing fast |
| Low traffic, moderate availability | High availability + auto day-2 ops |

---

## Backup

### Option 1 — RDB Snapshot (built-in, simplest)

Redis writes a point-in-time `.rdb` file to disk. Trigger manually or configure automatic saves.

```bash
# Trigger a background save on the operator master
kubectl exec redis-cluster-leader-0 -n ot-operators --context kind-redis-test \
  -- redis-cli bgsave

# Check it completed
kubectl exec redis-cluster-leader-0 -n ot-operators --context kind-redis-test \
  -- redis-cli lastsave

# Same for sentinel
kubectl exec redis-bitnami-node-0 -n redis-helm --context kind-redis-helm \
  -c redis -- redis-cli bgsave
```

RDB files live in the PVC at `/data/dump.rdb`. Restore by replacing the file and restarting Redis.

### Option 2 — AOF (Append-Only File)

AOF logs every write operation. More durable than RDB (no data loss window) but larger files.
Enable in operator via `redis-operator-crd.yaml`:

```yaml
spec:
  redisConfig:
    additionalRedisConfig: |
      appendonly yes
      appendfsync everysec
```

### Option 3 — PVC Snapshot (Kubernetes-native)

If your storage class supports `VolumeSnapshot`, snapshot the PVC directly:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: redis-leader-0-backup
  namespace: ot-operators
spec:
  volumeSnapshotClassName: csi-hostpath-snapclass
  source:
    persistentVolumeClaimName: redis-cluster-leader-0
EOF
```

Restore by creating a new PVC from the snapshot and pointing the pod at it.
Works for both operator and sentinel — any PVC-backed Redis.

### Option 4 — Velero (full cluster backup)

Velero backs up Kubernetes objects + PVC data together. Useful when you want to restore
the entire Redis deployment, not just data.

```bash
# Install Velero (requires object storage — S3, GCS, Azure Blob)
velero install --provider aws --bucket my-bucket --secret-file ./credentials

# Backup
velero backup create redis-backup --include-namespaces ot-operators

# Restore
velero restore create --from-backup redis-backup
```

### Option 5 — redis-dump / RIOT (key-level export)

Export all keys to a file. Useful for cross-cluster migrations.

```bash
# Using redis-cli --scan to dump all keys
kubectl exec redis-cluster-leader-0 -n ot-operators --context kind-redis-test \
  -- redis-cli --cluster call \
  redis-cluster-leader-0.redis-cluster-leader-headless.ot-operators.svc:6379 \
  BGSAVE
```

Or use [RIOT](https://developer.redis.com/riot/) (Redis I/O Tools) for full key-level
export/import between any two Redis instances.

### Backup comparison

| Method | Granularity | Recovery time | Requires | Best for |
|---|---|---|---|---|
| RDB snapshot | Point-in-time | Fast | Nothing extra | Simple, periodic backups |
| AOF | Near real-time | Slower (replay log) | Disk space | Zero data loss requirement |
| PVC snapshot | Point-in-time | Fast | CSI driver | Kubernetes-native workflows |
| Velero | Full cluster | Medium | Object storage | Disaster recovery |
| RIOT / redis-dump | Key-level | Depends on size | Extra tooling | Cross-cluster migration |

---

## Migration

### Bitnami Sentinel → OT Operator Sentinel

Same topology (1 master + replicas + sentinel processes), just switching management from Bitnami Helm to the OT-Container-Kit operator. No client changes needed — connection string stays sentinel-style.

**Step 1 — Deploy OT operator on its own cluster (or namespace)**

```bash
kind create cluster --name redis-test --config ot-operator/kind-config.yaml
helm repo add ot-helm https://ot-container-kit.github.io/helm-charts/ && helm repo update
helm install redis-operator ot-helm/redis-operator \
  --namespace ot-operators --create-namespace \
  --set featureGates.GenerateConfigInInitContainer=true
```

**Step 2 — Apply the RedisSentinel CRD (not RedisCluster)**

```yaml
# ot-operator/redis-sentinel-crd.yaml
apiVersion: redis.redis.opstreelabs.in/v1beta2
kind: RedisSentinel
metadata:
  name: redis-sentinel
  namespace: ot-operators
spec:
  clusterSize: 3
  redisSentinelConfig:
    masterGroupName: mymaster
    quorum: "2"
    downAfterMilliseconds: "60000"
  kubernetesConfig:
    image: quay.io/opstree/redis:v7.0.15
    imagePullPolicy: IfNotPresent
```

```bash
kubectl apply -f ot-operator/redis-sentinel-crd.yaml
kubectl get pods -n ot-operators --context kind-redis-test -w
```

**Step 3 — Migrate data using RIOT (live sync)**

```bash
brew install redis/tap/riot

riot replicate \
  --source-uri redis-sentinel://sentinel-host:26379?master=mymaster \
  --target-uri redis-sentinel://new-sentinel-host:26379?master=mymaster \
  --mode live
```

Live mode keeps syncing until you cut over — zero downtime window.

**Step 4 — Switch app config, decommission Bitnami**

Update connection string from Bitnami sentinel host → OT operator sentinel host. Same sentinel protocol, same port (26379), same master name.

```bash
helm uninstall redis-bitnami -n redis-helm
kind delete cluster --name redis-helm
```

**Key advantages over migrating to Cluster mode:**
- No client library changes — sentinel protocol is identical
- No hash tag requirements on keys
- No slot assignment complexity
- No cluster-aware client requirement
- Same `down-after-milliseconds` tuning applies

**Key risks:**
- `down-after-milliseconds` resets to operator default — re-apply your tuning in the CRD spec
- Sentinel quorum must be set correctly (minimum `(replicas/2)+1`, so 2 for 3 replicas)
- Test failover in staging before production cutover

---

## Official Open Source Sentinel Operator

The same OT-Container-Kit operator supports `RedisSentinel` (same binary, different CRD kind):

```yaml
apiVersion: redis.redis.opstreelabs.in/v1beta2
kind: RedisSentinel
```

Gives you operator-managed Sentinel — automated failover detection, rolling upgrades, and
reconciliation on top of the standard Sentinel protocol.

Docs: https://ot-container-kit.github.io/redis-operator/guide/

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
kubectl apply -f ot-operator/redis-operator-crd.yaml

# 5. Watch pods come up (~2 min)
kubectl get pods -n ot-operators --context kind-redis-test -w
```

Expected (7 pods):

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

Expected (3 pods, 2 containers each):

```
redis-bitnami-node-0   2/2   Running
redis-bitnami-node-1   2/2   Running
redis-bitnami-node-2   2/2   Running
```

---

## Verify Cluster State

### OT Operator — roles and slot assignments

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

Fires every ~10 seconds (hardcoded — not configurable via Helm or flags):

```bash
kubectl logs -f redis-operator-58c8bb9c8b-dcxk8 \
  -n ot-operators \
  --context kind-redis-test \
  | grep --line-buffered "Number of Redis nodes\|reconcileID" \
  | awk '{print $2, $NF}'
```

---

## Self-Healing Test

### Step 1 — seed a key

```bash
# OT Operator
kubectl exec redis-cluster-leader-0 \
  -n ot-operators --context kind-redis-test \
  -- redis-cli set testkey "operator-value"

# Bitnami Sentinel
kubectl exec redis-bitnami-node-0 \
  -n redis-helm --context kind-redis-helm \
  -c redis -- redis-cli set testkey "sentinel-value"
```

### Step 2 — kill the master

```bash
kubectl delete pod redis-cluster-leader-0 -n ot-operators --context kind-redis-test
kubectl delete pod redis-bitnami-node-0   -n redis-helm   --context kind-redis-helm
```

### Step 3 — watch recovery

```bash
kubectl get pods -n ot-operators --context kind-redis-test -w
kubectl get pods -n redis-helm   --context kind-redis-helm -w
```

### Step 4 — verify roles and data

```bash
# Operator — new roles
kubectl exec redis-cluster-leader-1 -n ot-operators --context kind-redis-test \
  -- redis-cli cluster nodes 2>/dev/null | \
  awk '{split($2,a,","); split(a[2],b,"."); printf "%-35s %-20s %s\n", b[1], $3, $9}'

# Operator — data survived?
kubectl exec redis-cluster-leader-0 -n ot-operators --context kind-redis-test \
  -- redis-cli get testkey 2>/dev/null

# Sentinel — new roles
for pod in redis-bitnami-node-0 redis-bitnami-node-1 redis-bitnami-node-2; do
  role=$(kubectl exec "$pod" -n redis-helm --context kind-redis-helm \
    -c redis -- redis-cli role 2>/dev/null | head -1)
  printf "%-25s %s\n" "$pod" "$role"
done

# Sentinel — data survived?
kubectl exec redis-bitnami-node-1 -n redis-helm --context kind-redis-helm \
  -c redis -- redis-cli get testkey 2>/dev/null
```

### Expected results

| | OT Operator | Bitnami Sentinel |
|---|---|---|
| Detection | Instant (K8s event watch) | After 60s timeout |
| Killed pod returns as | Slave (operator enforces) | Slave if promotion fired; Master if redis recovered < 60s |
| Promotion fires | Always | If redis process unreachable > 60s (includes startup time) |
| Data survived | Yes | Yes |

---

## Image Situation

Bitnami paywalled all container images on **August 28, 2025** — deleted from Docker Hub.

- `bitnamilegacy/redis:8.2.1` — last free Redis image (Aug 22, 2025)
- `bitnamilegacy/redis-sentinel:8.2.1` — last free Sentinel image (Aug 22, 2025)

Used with `bitnamicharts/redis` v20.6.3 via `global.security.allowInsecureImages: true`
in `bitnami-sentinel/helm-values.yaml`.

OT Operator uses `quay.io/opstree/redis:v7.0.15` — always free.

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
| RIOT — Redis migration tool | https://developer.redis.com/riot/ |
| Velero — K8s backup | https://velero.io/ |
| kind (Kubernetes in Docker) | https://github.com/kubernetes-sigs/kind |

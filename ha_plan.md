# High Availability Implementation Plan — mini_k8s

## Overview

This document details how to make every component in the mini_k8s control plane highly available. The goal: any single component failure should not cause system-wide outage. Components recover automatically or fail over to standby instances.

**HA Principles:**
1. **No Single Point of Failure** — every control plane component runs 2+ instances
2. **Leader Election** — only one active Scheduler/Controller Manager at a time (Redis SETNX + TTL)
3. **Stateless API Server** — multiple instances behind a load balancer, shared state in Postgres/Redis
4. **Automatic Failover** — standby promotes to leader within lease TTL (15s)
5. **Split-Brain Prevention** — fencing via atomic Redis operations, no two leaders can hold the same lease

---

## Architecture: HA Topology

```
                        ┌─────────────┐
                        │   Clients    │
                        │ (kubectl-mini│
                        │  chaos, obs) │
                        └──────┬───────┘
                               │
                        ┌──────▼───────┐
                        │ Load Balancer│
                        │  (nginx/HA)  │
                        └──┬───┬───┬───┘
                           │   │   │
                ┌──────────▼┐ ┌▼────────┐ ┌──────────▼┐
                │API Server 1│ │API Srv 2│ │API Server 3│
                │  (Go)      │ │  (Go)   │ │  (Go)      │
                └─────┬──────┘ └────┬────┘ └─────┬──────┘
                      │             │             │
              ┌───────▼─────────────▼─────────────▼───────┐
              │              Shared Data Layer             │
              │  ┌─────────────────┐  ┌─────────────────┐ │
              │  │ PostgreSQL      │  │ Redis            │ │
              │  │ Primary ──►Replica│ │ Sentinel (3-node)│ │
              │  └─────────────────┘  └─────────────────┘ │
              └───────────────────────────────────────────┘
                               │
            ┌──────────────────┼──────────────────┐
            ▼                  ▼                   ▼
    ┌───────────────┐ ┌───────────────┐  ┌───────────────┐
    │Scheduler 1    │ │Scheduler 2    │  │Scheduler 3    │
    │(LEADER)       │ │(STANDBY)      │  │(STANDBY)      │
    └───────────────┘ └───────────────┘  └───────────────┘

    ┌───────────────┐ ┌───────────────┐  ┌───────────────┐
    │CM 1 (LEADER)  │ │CM 2 (STANDBY) │  │CM 3 (STANDBY) │
    └───────────────┘ └───────────────┘  └───────────────┘

    ┌───────────────┐ ┌───────────────┐
    │Node Agent 1   │ │Node Agent 2   │  ... (N agents)
    └───────────────┘ └───────────────┘
```

---

## Component 1: API Server HA

### Strategy: Stateless + Shared State

The API Server is **stateless** — all state lives in Postgres + Redis. Run N instances behind a load balancer.

### Implementation Tasks

- [ ] **Remove in-memory state** — ensure no request-scoped state leaks between goroutines beyond `context.Context`
- [ ] **Connection pooling** — `pgxpool` for Postgres, `redis.UniversalClient` for Redis (both handle reconnection)
- [ ] **Health endpoint** — `GET /healthz` returns 200 if Postgres + Redis reachable
  ```go
  func healthzHandler(w http.ResponseWriter, r *http.Request) {
      // Ping Postgres
      // Ping Redis
      // Return 200 or 503
  }
  ```
- [ ] **Readiness probe** — `GET /readyz` — returns 200 only after migrations complete and caches warm
- [ ] **Idempotency** — all write operations must be idempotent (use `resourceVersion` CAS)
- [ ] **Docker Compose** — 3 API Server instances on ports 8080, 8081, 8082
- [ ] **Nginx load balancer** — round-robin upstream config with health checks

### Failure Modes

| Failure | Detection | Recovery |
|---------|-----------|----------|
| API Server crash | Health check fails | LB stops routing, other instances serve |
| Postgres down | Health check fails, all instances 503 | Postgres HA failover (see below) |
| Redis down | Health check fails | Redis Sentinel promotes replica |

### Config

```yaml
# api-server config
server:
  port: 8080
  replicas: 3
postgres:
  host: postgres-primary
  port: 5432
  pool_max_conns: 20
redis:
  addrs: ["redis-sentinel-1:26379", "redis-sentinel-2:26379", "redis-sentinel-3:26379"]
  sentinel_master: "mymaster"
```

---

## Component 2: Scheduler HA

### Strategy: Leader Election via Redis

Only the leader runs the scheduling loop. Standbys hold a warm cache and watch for leader lease expiry.

### Leader Election Protocol

```
LEADER (Scheduler 1):
  1. SETNX k8s:leader:scheduler "<instance-id>" EX 15
  2. If acquired → run scheduling loop
  3. Every 5s → SET k8s:leader:scheduler "<instance-id>" EX 15 (renew)
     - Use Lua script for atomic check-and-renew:
       if redis.call("GET", KEYS[1]) == ARGV[1] then
         return redis.call("SET", KEYS[1], ARGV[1], "EX", ARGV[2])
       else
         return 0
       end
  4. On graceful shutdown → DEL k8s:leader:scheduler

STANDBY (Scheduler 2):
  1. Every 5s → GET k8s:leader:scheduler
  2. If key missing → attempt SETNX (try to become leader)
  3. Meanwhile: maintain warm cache via LIST+WATCH (no scheduling)
```

### Implementation Tasks

- [ ] **LeaderElector struct** (reusable for Scheduler + CM):
  ```go
  type LeaderElector struct {
      redis      *redis.Client
      key        string
      instanceID string
      ttl        time.Duration
      isLeader   atomic.Bool
      callbacks  LeaderCallbacks
  }

  type LeaderCallbacks interface {
      OnStartedLeading(ctx context.Context)
      OnStoppedLeading()
      OnNewLeader(identity string)
  }
  ```
- [ ] **Graceful handoff** — leader sets TTL short (15s), on shutdown DEL the key, standbys detect within 5s
- [ ] **Warm standby cache** — non-leader still LIST+WATCH pods to keep cache hot
- [ ] **Metrics** — expose `leader_election_status` gauge (1=leader, 0=standby)
- [ ] **Multiple scheduler replicas** — Docker Compose: 3 scheduler instances

### Failure Modes

| Failure | Detection | Recovery |
|---------|-----------|----------|
| Leader crashes | Lease expires (15s) | Standby acquires lease in 5s polling cycle |
| Leader network partition | Lease expires | Standby promotes; old leader can't renew (fencing) |
| All schedulers down | Pods stay Pending | Alert — manual intervention or auto-restart |
| Redis down | Can't acquire lease | All schedulers stop; failover Redis first |

### Split-Brain Prevention

- **Atomic lease renewal**: Lua script checks instance ID before renewing — if another leader took over, renewal fails
- **No local decisions without lease**: scheduling loop checks `isLeader.Load()` before each reconcile iteration
- **Short TTL**: 15s max window where old leader might still act (acceptable for simulation)

---

## Component 3: Controller Manager HA

### Strategy: Same Leader Election as Scheduler

Each controller type (ReplicaSet, Deployment) uses the same `LeaderElector` pattern.

### Implementation Tasks

- [ ] **Per-controller leader election** — each controller can independently elect a leader:
  ```
  k8s:leader:replicaset-controller
  k8s:leader:deployment-controller
  ```
  This allows ReplicaSet controller on instance 1 and Deployment controller on instance 2.
- [ ] **Shared LeaderElector** — reuse the struct from Scheduler
- [ ] **Work queue sharding** (future): if leader election overhead too high, shard by namespace/name hash
- [ ] **Resync on leader promotion** — new leader does full LIST to rebuild state before starting reconcile

### Failure Modes

| Failure | Detection | Recovery |
|---------|-----------|----------|
| Leader CM crashes | Lease expires | Standby promotes, resyncs, resumes reconciliation |
| Partial failure (1 controller stuck) | Reconciliation timeout | Restart that controller goroutine |

---

## Component 4: Node Agent Resilience

### Strategy: Reconnect + State Recovery

Node Agents are not HA-replicated (each represents a physical/virtual node). Instead, they must be resilient to API Server restarts and network failures.

### Implementation Tasks

- [ ] **Exponential backoff reconnection**:
  ```go
  func (n *NodeAgent) reconnectWithBackoff(ctx context.Context) {
      backoff := time.Second
      maxBackoff := 30 * time.Second
      for {
          err := n.connect(ctx)
          if err == nil {
              backoff = time.Second // reset on success
              return
          }
          time.Sleep(backoff + jitter(backoff))
          backoff = min(backoff*2, maxBackoff)
      }
  }
  ```
- [ ] **Heartbeat with retry** — if heartbeat PUT fails, retry 3x before reconnecting
- [ ] **Pod state reconciliation** — on reconnect, GET assigned pods from API Server, compare with local Docker state, reconcile drift
  ```go
  func (n *NodeAgent) reconcilePodState(ctx context.Context) {
      // 1. List pods assigned to this node from API Server
      // 2. List running containers via Docker SDK
      // 3. Diff: start missing, stop orphans
  }
  ```
- [ ] **Node status updates** — include `lastHeartbeatTime` so API Server can detect stale nodes
- [ ] **Node controller** (in Controller Manager) — mark nodes `NotReady` after 40s without heartbeat, evict pods after 5min

### Failure Modes

| Failure | Detection | Recovery |
|---------|-----------|----------|
| Node Agent crash | Heartbeat stops (40s) | Node marked NotReady, pods evicted after 5min |
| API Server down (all instances) | Heartbeat PUT fails | Node Agent reconnects with backoff |
| Docker daemon down | Container operations fail | Report pod status as Failed, let controller recreate |
| Network partition | Heartbeat fails | Node marked NotReady, pods rescheduled elsewhere |

---

## Component 5: PostgreSQL HA

### Strategy: Primary-Replica with Streaming Replication

For the simulation, use a simple primary + 1 replica setup.

### Implementation Tasks

- [ ] **PostgreSQL Primary** — accepts all reads/writes
- [ ] **PostgreSQL Replica** — streaming replication from primary, read-only
- [ ] **Failover script** — promote replica to primary if primary unreachable
- [ ] **Connection string failover** — `pgx` config with multiple hosts:
  ```go
  connStr := "host=pg-primary,pg-replica port=5432 dbname=mini_k8s target_session_attrs=read-write"
  ```
- [ ] **Docker Compose** — 2 Postgres containers with shared volume for WAL shipping

### Docker Compose Config

```yaml
postgres-primary:
  image: postgres:16
  environment:
    POSTGRES_DB: mini_k8s
    POSTGRES_REPLICATION_USER: replicator
  command: >
    postgres
    -c wal_level=replica
    -c max_wal_senders=3
    -c hot_standby=on

postgres-replica:
  image: postgres:16
  environment:
    PGHOST: postgres-primary
    PGUSER: replicator
  command: >
    sh -c "
      pg_basebackup -h postgres-primary -D /var/lib/postgresql/data -U replicator -Fp -Xs -R
      postgres -D /var/lib/postgresql/data
    "
```

---

## Component 6: Redis HA

### Strategy: Redis Sentinel (3 Sentinels + 1 Primary + 2 Replicas)

Sentinel monitors Redis master, promotes replica on failure.

### Implementation Tasks

- [ ] **Redis Primary** — handles all writes (leader election, event streams)
- [ ] **Redis Replicas** — 2 replicas for read scaling + failover candidates
- [ ] **Redis Sentinel** — 3 sentinel instances monitoring the master
- [ ] **Client config** — `go-redis` sentinel-aware client:
  ```go
  client := redis.NewFailoverClient(&redis.FailoverOptions{
      MasterName:    "mymaster",
      SentinelAddrs: []string{"sentinel-1:26379", "sentinel-2:26379", "sentinel-3:26379"},
  })
  ```

### Docker Compose Config

```yaml
redis-primary:
  image: redis:7
  command: redis-server --appendonly yes

redis-replica-1:
  image: redis:7
  command: redis-server --replicaof redis-primary 6379

redis-replica-2:
  image: redis:7
  command: redis-server --replicaof redis-primary 6379

redis-sentinel-1:
  image: redis:7
  command: >
    redis-sentinel /etc/redis/sentinel.conf
  volumes:
    - ./sentinel.conf:/etc/redis/sentinel.conf
```

`sentinel.conf`:
```
sentinel monitor mymaster redis-primary 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 10000
sentinel parallel-syncs mymaster 1
```

---

## Component 7: Load Balancing

### Strategy: Nginx as External Load Balancer

### Implementation Tasks

- [ ] **Nginx config** — round-robin across API Server instances with health checks:
  ```nginx
  upstream api_servers {
      server apiserver-1:8080 max_fails=3 fail_timeout=30s;
      server apiserver-2:8080 max_fails=3 fail_timeout=30s;
      server apiserver-3:8080 max_fails=3 fail_timeout=30s;
  }

  server {
      listen 80;
      location / {
          proxy_pass http://api_servers;
          proxy_connect_timeout 5s;
          proxy_next_upstream error timeout http_502 http_503;
      }
      location /healthz {
          proxy_pass http://api_servers/healthz;
      }
  }
  ```
- [ ] **Sticky sessions for SSE** — use `ip_hash` for `/api/v1/watch/` routes so SSE connections stay on one instance
- [ ] **gRPC load balancing** — client-side round-robin in Scheduler/CM Go clients

---

## Component 8: Health Checks & Monitoring

### Implementation Tasks

- [ ] **Component health endpoints**:
  - `GET /healthz` — liveness (process alive)
  - `GET /readyz` — readiness (dependencies reachable)
- [ ] **Aggregated health dashboard** — Python observability sidecar checks all components
- [ ] **Metrics** — Prometheus-style `/metrics` on each Go component:
  ```
  leader_election_status{component="scheduler"} 1
  leader_election_status{component="controller-manager"} 0
  api_server_requests_total{method="POST",status="200"} 1234
  node_heartbeat_latency_seconds{node="node-1"} 0.05
  ```
- [ ] **Alerting rules** (in observability sidecar):
  - No leader for scheduler > 30s → CRITICAL
  - Node heartbeat missing > 40s → WARNING
  - API Server all instances unhealthy → CRITICAL

---

## Component 9: Failure Testing (Chaos Scenarios)

### Test Scenarios

- [ ] **Kill leader scheduler** — verify standby promotes within 20s, pods continue scheduling
- [ ] **Kill all API Servers** — verify node agents reconnect with backoff, system recovers when API comes back
- [ ] **Kill Redis primary** — verify Sentinel promotes replica, leader election continues
- [ ] **Kill Postgres primary** — verify replica promotes, API Servers reconnect
- [ ] **Network partition** — simulate by blocking Redis from leader, verify new leader elected, old leader stops
- [ ] **Node Agent crash** — verify node marked NotReady after 40s, pods evicted after 5min
- [ ] **Rolling restart** — restart API Servers one by one, verify zero downtime

### YAML Scenarios

```yaml
# chaos/ha-scenarios.yaml
scenarios:
  - name: "scheduler-leader-failover"
    steps:
      - action: exec
        command: "docker stop scheduler-1"
      - action: wait
        duration: 20s
      - action: verify
        check: "scheduler leader exists"
      - action: verify
        check: "pods continue to be scheduled"

  - name: "api-server-rolling-restart"
    steps:
      - action: exec
        command: "docker restart apiserver-1"
      - action: wait
        duration: 5s
      - action: verify
        check: "api responding"
      - action: exec
        command: "docker restart apiserver-2"
      - action: wait
        duration: 5s
      - action: verify
        check: "api responding"
      - action: exec
        command: "docker restart apiserver-3"
      - action: wait
        duration: 5s
      - action: verify
        check: "api responding"

  - name: "redis-failover"
    steps:
      - action: exec
        command: "docker stop redis-primary"
      - action: wait
        duration: 15s
      - action: verify
        check: "redis writeable"
      - action: verify
        check: "leader election functional"
```

---

## Docker Compose: Full HA Deployment

```yaml
# deploy/docker-compose.ha.yml
version: "3.8"

services:
  # ── Data Layer ──
  postgres-primary:
    image: postgres:16
    environment:
      POSTGRES_DB: mini_k8s
      POSTGRES_PASSWORD: mini_k8s
    ports: ["5432:5432"]
    volumes: ["pg-data:/var/lib/postgresql/data"]

  postgres-replica:
    image: postgres:16
    environment:
      PGPASSWORD: mini_k8s
    depends_on: [postgres-primary]

  redis-primary:
    image: redis:7
    ports: ["6379:6379"]

  redis-replica-1:
    image: redis:7
    command: redis-server --replicaof redis-primary 6379
    depends_on: [redis-primary]

  redis-replica-2:
    image: redis:7
    command: redis-server --replicaof redis-primary 6379
    depends_on: [redis-primary]

  redis-sentinel-1:
    image: redis:7
    command: redis-sentinel /etc/redis/sentinel.conf
    volumes: ["./sentinel.conf:/etc/redis/sentinel.conf"]

  redis-sentinel-2:
    image: redis:7
    command: redis-sentinel /etc/redis/sentinel.conf
    volumes: ["./sentinel.conf:/etc/redis/sentinel.conf"]

  redis-sentinel-3:
    image: redis:7
    command: redis-sentinel /etc/redis/sentinel.conf
    volumes: ["./sentinel.conf:/etc/redis/sentinel.conf"]

  # ── Load Balancer ──
  nginx:
    image: nginx:alpine
    ports: ["80:80"]
    volumes: ["./nginx.conf:/etc/nginx/nginx.conf"]
    depends_on: [apiserver-1, apiserver-2, apiserver-3]

  # ── API Servers ──
  apiserver-1:
    build: .
    command: ./apiserver
    environment:
      SERVER_PORT: 8080
      DB_HOST: postgres-primary
      REDIS_SENTINELS: "redis-sentinel-1:26379,redis-sentinel-2:26379,redis-sentinel-3:26379"

  apiserver-2:
    build: .
    command: ./apiserver
    environment:
      SERVER_PORT: 8080
      DB_HOST: postgres-primary
      REDIS_SENTINELS: "redis-sentinel-1:26379,redis-sentinel-2:26379,redis-sentinel-3:26379"

  apiserver-3:
    build: .
    command: ./apiserver
    environment:
      SERVER_PORT: 8080
      DB_HOST: postgres-primary
      REDIS_SENTINELS: "redis-sentinel-1:26379,redis-sentinel-2:26379,redis-sentinel-3:26379"

  # ── Schedulers ──
  scheduler-1:
    build: .
    command: ./scheduler
    environment:
      INSTANCE_ID: scheduler-1
      API_SERVER: http://nginx:80

  scheduler-2:
    build: .
    command: ./scheduler
    environment:
      INSTANCE_ID: scheduler-2
      API_SERVER: http://nginx:80

  scheduler-3:
    build: .
    command: ./scheduler
    environment:
      INSTANCE_ID: scheduler-3
      API_SERVER: http://nginx:80

  # ── Controller Managers ──
  controller-manager-1:
    build: .
    command: ./controller-manager
    environment:
      INSTANCE_ID: cm-1
      API_SERVER: http://nginx:80

  controller-manager-2:
    build: .
    command: ./controller-manager
    environment:
      INSTANCE_ID: cm-2
      API_SERVER: http://nginx:80

  # ── Node Agents ──
  node-agent-1:
    build: .
    command: ./node-agent
    environment:
      NODE_NAME: node-1
      API_SERVER: http://nginx:80
    volumes: ["/var/run/docker.sock:/var/run/docker.sock"]

  node-agent-2:
    build: .
    command: ./node-agent
    environment:
      NODE_NAME: node-2
      API_SERVER: http://nginx:80
    volumes: ["/var/run/docker.sock:/var/run/docker.sock"]

volumes:
  pg-data:
```

---

## Implementation Order

| Step | Component | What | Depends On |
|------|-----------|------|------------|
| 1 | **LeaderElector** | Reusable Go struct with Redis SETNX + Lua script | Phase 1 (Redis) |
| 2 | **API Server health endpoints** | `/healthz`, `/readyz` | Phase 1 |
| 3 | **API Server multi-instance** | 3 instances, ensure statelessness | Step 2 |
| 4 | **Nginx LB** | Round-robin + sticky for SSE | Step 3 |
| 5 | **Scheduler leader election** | Integrate LeaderElector, warm standby | Step 1, Phase 3 |
| 6 | **Controller Manager leader election** | Integrate LeaderElector per controller | Step 1, Phase 4 |
| 7 | **Node Agent resilience** | Backoff reconnect, state reconciliation | Phase 5 |
| 8 | **Redis Sentinel** | 3 sentinels, failover client config | Phase 2 |
| 9 | **PostgreSQL replication** | Primary + replica, failover | Phase 1 |
| 10 | **HA Docker Compose** | Full multi-instance deployment | Steps 3-9 |
| 11 | **Chaos HA scenarios** | Kill/restart/failover tests | Step 10 |

---

## Key Go Patterns for HA

```go
// 1. Leader Election with atomic renewal
func (le *LeaderElector) renew(ctx context.Context) error {
    script := redis.NewScript(`
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("SET", KEYS[1], ARGV[1], "EX", ARGV[2])
        end
        return 0
    `)
    result, err := script.Run(ctx, le.redis, []string{le.key}, le.instanceID, le.ttl.Seconds()).Int()
    if err != nil || result == 0 {
        le.isLeader.Store(false)
        le.callbacks.OnStoppedLeading()
        return errors.New("lost leadership")
    }
    return nil
}

// 2. Backoff reconnect for Node Agents
func backoff(attempt int) time.Duration {
    d := time.Second * time.Duration(1<<min(attempt, 5))
    return d + time.Duration(rand.Int63n(int64(d)/2)) // jitter
}

// 3. Health check combining multiple dependencies
func healthCheck(pg *pgxpool.Pool, redis *redis.Client) error {
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()
    if err := pg.Ping(ctx); err != nil {
        return fmt.Errorf("postgres: %w", err)
    }
    if err := redis.Ping(ctx).Err(); err != nil {
        return fmt.Errorf("redis: %w", err)
    }
    return nil
}
```

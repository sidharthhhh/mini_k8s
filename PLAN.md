# Kubernetes Control Plane Simulator - Project Plan

## Goal
Build a production-style Kubernetes Control Plane Simulator in Python to master K8s internals, FastAPI, gRPC, and distributed systems.

## Phases

### Phase 1: API Server (The Brain)
- [ ] **Core Setup**: FastAPI project structure, Pydantic models for Pod/Node/Deployment.
- [ ] **Data Store**: Postgres for persistence with `resourceVersion` column.
- [ ] **Optimistic Locking**: Implement CAS (Compare-And-Swap) logic in DB updates.
- [ ] **REST APIs**: CRUD endpoints with version checks (`409 Conflict` on mismatch).
- [ ] **Rate Limiting**: Add simple middleware to throttle requests.
- [ ] **Authentication**: JWT middleware.
- [ ] **Dockerization**: Dockerfile and docker-compose for API Server + Postgres.

### Phase 2: Internal Event Bus (The Nervous System)
- [ ] **Watch Mechanism**: Redis Streams implementation.
- [ ] **Event Publisher**: Hook into CRUD operations to publish generic `(Action, Kind, Body)` events.
- [ ] **Watch Endpoint**: `/api/v1/watch/...` streaming response (SSE).

### Phase 3: Scheduler (The Decider)
- [ ] **Leader Election**: Implement Redis-based locking/lease sidecar.
- [ ] **Informer Pattern**: Build a generic `SharedInformer` class that:
    -   Lists all objects initially.
    -   Watches for updates.
    -   Maintains a local in-memory cache.
- [ ] **Scheduling Logic**: Read from Local Cache (Informer), Write to API Server.
- [ ] **Algorithms**: Round-robin, Bin-packing (CPU/Memory).

### Phase 4: Controller Manager (The State Enforcer)
- [ ] **ReplicaSet Controller**: Use Informer to count Pods vs ReplicaSet spec.
- [ ] **Deployment Controller**: Implement Rolling Update logic.
- [ ] **Reconciliation**: Periodic loop (Resync) ensuring cache consistency.

### Phase 5: Node Agent (The Worker)
- [ ] **Heartbeats**: Report node status / Leases.
- [ ] **Pod Lifecycle**: Simulate pulling images, starting containers (state transitions).

### Phase 6: Chaos & Resilience
- [ ] **Failure Injection**: Kill components, network partitions.
- [ ] **Testing**: Verify system recovery.

### Phase 7: Scalability Experimentation
- [ ] **Load Testing**: Use Locust or K6 to hammer the API Server.
- [ ] **Metrics**: Expose Prometheus metrics (/metrics) from all components.
- [ ] **Performance Tuning**: Adjust Rate Limits, Connection Pools.

---

## Current Focus: Phase 1 - API Server Architecture
We will start by laying out the domain models and the basic FastAPI application.

# Kubernetes Control Plane Simulator - Project Plan

## Goal
Build a production-style Kubernetes Control Plane Simulator in Python to master K8s internals, FastAPI, gRPC, and distributed systems.

## Phases

### Phase 1: API Server (The Brain)
- [ ] **Core Setup**: FastAPI project structure, Pydantic models for Pod/Node/Deployment.
- [ ] **Data Store**: Postgres for persistence (simulating etcd).
- [ ] **REST APIs**: CRUD endpoints for `/api/v1/pods`, `/api/v1/nodes`.
- [ ] **Concurrency**: Optimistic locking (`resourceVersion`) and atomic updates.
- [ ] **Authentication**: JWT middleware.
- [ ] **Dockerization**: Dockerfile and docker-compose for API Server + Postgres.

### Phase 2: Internal Event Bus (The Nervous System)
- [ ] **Watch Mechanism**: Redis Streams implementation.
- [ ] **API Watch endpoint**: `/api/v1/watch/...` using SSE or streaming response.
- [ ] **Event Emitting**: API server publishes events on CRUD.

### Phase 3: Scheduler (The Decider)
- [ ] **gRPC Setup**: Define Protobufs for scheduling.
- [ ] **Watch Pending**: Scheduler watches API server for `Pending` pods.
- [ ] **Scheduling Algorithms**: Round-robin, Bin-packing (CPU/Memory).
- [ ] **Binding**: Call API Server to bind Pod to Node.

### Phase 4: Controller Manager (The State Enforcer)
- [ ] **ReplicaSet Controller**: Ensure `replicas` count matches desired state.
- [ ] **Reconciliation Loop**: Periodic check + Event watcher.
- [ ] **Deployment Controller**: Rolling updates simulation.

### Phase 5: Node Agent (The Worker)
- [ ] **Heartbeats**: Report node status to API Server.
- [ ] **Pod Lifecycle**: Simulate pulling images, starting containers (state transitions).

### Phase 6: Chaos & Resilience
- [ ] **Failure Injection**: Kill components, network partitions.
- [ ] **Testing**: Verify system recovery.

---

## Current Focus: Phase 1 - API Server Architecture
We will start by laying out the domain models and the basic FastAPI application.

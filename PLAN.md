# Kubernetes Control Plane Simulator — Project Plan

## Goal
Build a **production-style Kubernetes Control Plane Simulator** using **Go + Python** to master K8s internals, distributed systems, gRPC, and systems programming — learning both languages organically through the project.

---

## Language Assignment Philosophy

| Component | Language | Reason to Learn |
|---|---|---|
| **API Server** | **Go** | Goroutines, net/http, interfaces, structs — core Go concepts |
| **Scheduler** | **Go** | gRPC client, concurrent goroutines, channels — Go concurrency model |
| **Controller Manager** | **Go** | Reconciliation loops with goroutines & channels — idiomatic Go |
| **Node Agent (Kubelet)** | **Go** | Goroutines, OS-level operations, gRPC streaming — systems Go |
| **Chaos Injector** | **Python** | asyncio, httpx, chaos scripting — Python shines for scripting & tooling |
| **CLI Tool (kubectl-mini)** | **Python** | Click/Typer, rich terminal output — Python great for CLIs |
| **Observability Dashboard** | **Python** | FastAPI + WebSockets for live dashboard — Python for rapid UI/API |

> **Rule of thumb**: Go owns the **critical path** (performance, concurrency, correctness). Python owns **tooling, scripting, and the human interface layer**.

---

## Phases

### Phase 1 — API Server in Go (The Brain)
**Language: Go** | *Goal: Learn Go structs, interfaces, HTTP, middleware, and Postgres*

- [ ] **Project Setup**: Go modules (`go mod init`), folder structure (`cmd/`, `internal/`, `pkg/`).
- [ ] **Domain Models**: Go structs for `Pod`, `Node`, `Deployment` with JSON tags.
- [ ] **Data Store**: PostgreSQL via `pgx` driver (raw SQL — no ORM, to learn SQL properly).
- [ ] **REST API**: `net/http` or `chi` router — CRUD for `/api/v1/pods`, `/api/v1/nodes`.
- [ ] **Optimistic Concurrency**: `resourceVersion` as integer, `409 Conflict` on mismatch.
- [ ] **Authentication**: JWT middleware in Go (`golang-jwt/jwt`).
- [ ] **Dockerization**: Multi-stage Go Dockerfile + `docker-compose`.

**Go concepts you'll learn**: structs, interfaces, error handling, `context`, middleware chains, goroutines (for async handlers), `sync.Mutex`.

---

### Phase 2 — Internal Event Bus (The Nervous System)
**Language: Go** | *Goal: Learn Go channels, Redis Streams, and concurrent pub/sub*

- [ ] **Redis Streams**: Go `redis/go-redis` client to publish events on every CRUD write.
- [ ] **Watch Endpoint**: `GET /api/v1/watch/pods` — Server-Sent Events (SSE) using goroutines + channels.
- [ ] **Fan-out**: Each `WATCH` client gets its own goroutine subscribed to a Redis consumer group.

**Go concepts you'll learn**: goroutines, `select` statement, channels, `context.WithCancel`, SSE streaming.

---

### Phase 3 — Scheduler in Go (The Decider)
**Language: Go** | *Goal: Learn gRPC in Go, goroutines for control loops, channels*

- [ ] **Protobuf Definitions**: Define `.proto` files for `BindPodRequest`, `SchedulerService`.
- [ ] **Code Generation**: `protoc` + `protoc-gen-go-grpc` to generate Go stubs.
- [ ] **Informer Pattern**: At startup, LIST all Pending pods → WATCH for new ones via SSE.
- [ ] **Local Cache**: Go `map[string]Pod` behind a `sync.RWMutex` as in-memory state.
- [ ] **Scheduling Algorithms**: Round-robin (simple), Bin-packing (CPU/Memory fit).
- [ ] **gRPC Bind Call**: Call API Server over gRPC to `BindPod(podName, nodeName)`.
- [ ] **Leader Election**: Redis-based distributed lock (`SETNX` with TTL) — only leader schedules.

**Go concepts you'll learn**: gRPC, protobuf, `sync.RWMutex`, goroutines for watch loops, interface-based algorithm design.

---

### Phase 4 — Controller Manager in Go (The State Enforcer)
**Language: Go** | *Goal: Learn Go reconciliation loops, goroutines, and worker queues*

- [ ] **ReplicaSet Controller**: Watch pods → if `running < desired`, create new pods via API.
- [ ] **Reconciliation Loop**: Goroutine per controller — infinite loop with `time.Ticker` + watch events.
- [ ] **Work Queue**: Buffered channel as work queue (like `client-go`'s `workqueue`).
- [ ] **Deployment Controller**: Rolling update simulation — replace pods one by one.
- [ ] **Leader Election**: Same Redis lock mechanism as Scheduler.

**Go concepts you'll learn**: goroutines + `for` loops, buffered channels as queues, `time.Ticker`, `sync.WaitGroup`.

---

### Phase 5 — Node Agent in Go (The Worker / Kubelet Sim)
**Language: Go** | *Goal: Learn Go OS interaction, goroutines for daemon processes*

- [ ] **Registration**: POST `/api/v1/nodes` on startup with capacity (CPU, Memory).
- [ ] **Heartbeat Loop**: Goroutine sending PUT `/api/v1/nodes/{id}/status` every 10s.
- [ ] **Pod Watch Loop**: Watch pods assigned to this node via SSE.
- [ ] **Pod Lifecycle Simulation**: State machine — `Pending → ContainerCreating → Running → Succeeded/Failed`.
- [ ] **Docker Integration**: Use Docker SDK for Go (`docker/docker`) to actually start/stop containers.

**Go concepts you'll learn**: goroutines for daemons, `select` for multi-event loop, Docker SDK, state machines.

---

### Phase 6 — CLI Tool in Python (kubectl-mini)
**Language: Python** | *Goal: Learn Python Click/Typer, httpx, rich terminal output*

- [ ] **CLI Framework**: `Typer` for command definitions, `rich` for pretty tables.
- [ ] **Commands**: `get pods`, `get nodes`, `apply -f pod.yaml`, `describe pod <name>`, `logs pod <name>`.
- [ ] **Config**: Read from `~/.minikube-config.yaml` (current-context, server URL, token).
- [ ] **Watch Mode**: `kubectl-mini get pods --watch` using SSE streaming with `httpx`.

**Python concepts you'll learn**: CLI design with Typer, YAML parsing, `httpx` async client, rich terminal formatting.

---

### Phase 7 — Chaos & Observability in Python
**Language: Python** | *Goal: Learn Python asyncio, FastAPI WebSockets, scripting*

- [ ] **Chaos Injector**: Python script using `asyncio` + `httpx` to randomly kill pods/nodes, inject network delays.
- [ ] **Scenario Runner**: YAML-based chaos scenarios (e.g., `kill_node: node-1 after: 30s`).
- [ ] **Observability API**: FastAPI app exposing `/metrics`, `/events` with WebSocket live feed.
- [ ] **Dashboard**: Simple HTML + JS dashboard consuming the WebSocket feed.

**Python concepts you'll learn**: `asyncio`, `httpx` async, FastAPI, WebSockets, YAML config parsing.

---

## Infra & Shared

- [ ] **Protobuf**: Single `proto/` directory, generate both Go and Python stubs.
- [ ] **Docker Compose**: One compose file wiring API Server, Scheduler, CM, Node Agents, Redis, Postgres.
- [ ] **Makefile**: Targets for `build`, `proto-gen`, `run`, `test`, `chaos`.

---

## Learning Milestones

```
Phase 1 ✅ → You know Go HTTP servers, structs, Postgres
Phase 2 ✅ → You know Go goroutines, channels, Redis
Phase 3 ✅ → You know gRPC in Go, concurrent caches
Phase 4 ✅ → You know Go reconciliation loops, worker queues
Phase 5 ✅ → You know Go daemons, Docker SDK
Phase 6 ✅ → You know Python CLI, httpx, Typer
Phase 7 ✅ → You know Python asyncio, FastAPI, chaos engineering
```

---

## Current Focus: Phase 1 — API Server in Go
Start with: `go mod init`, folder structure, first `Pod` struct, Postgres connection.

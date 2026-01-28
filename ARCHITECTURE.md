# Architecture Design

## System Overview

This project simulates a production-grade Kubernetes Control Plane. It aims to capture the *behavior* and *internal mechanics* of Kubernetes using Python-native technologies (FastAPI, SQLAlchemy, Redis).

**Key Design Principles:**
1.  **Hub-and-Spoke Topology**: All components communicate *only* via the API Server.
2.  **Event-Driven & Level-Triggered**: Components react to state changes (Events) but essentially reconcile towards a desired state (Level-Triggered).
3.  **Optimistic Concurrency**: No global locks. Use `resourceVersion` and CAS (Compare-And-Swap) logic.
4.  **High Availability Ready**: Designed so components (Scheduler, Controllers) can have multiple replicas with Leader Election.

```mermaid
graph TD
    %% Actors
    User((User / Kubectl))
    
    %% Core Components
    subgraph Control_Plane [Control Plane]
        API["API Server Group (FastAPI) <br/> Auth, Validation, OCC"]
        
        subgraph Sched_Group [Scheduler Cluster]
            Sched1["Scheduler 1 <br/> (Leader)"]
            Sched2["Scheduler 2 <br/> (Standby)"]
        end
        
        subgraph CM_Group [Controller Manager Cluster]
            CM1["ReplicaSet Controller <br/> (Leader)"]
            CM2["Deployment Controller <br/> (Standby)"]
        end
    end

    %% Data Store Simulation
    subgraph Data_Store [Etcd Simulation]
        PG[("PostgreSQL <br/> Durable Key-Value Store")]
        Redis[("Redis <br/> 1. Pub/Sub (Watch) <br/> 2. Dist. Lock (Leases)")]
    end

    %% Worker Nodes
    subgraph Worker_Nodes [Data Plane]
        Node1["Node Agent 1"]
        Node2["Node Agent 2"]
    end

    %% Interactions
    User -->|REST / HTTP2| API
    
    %% Storage
    API -->|SQL (ACID)| PG
    API -->|Publish Events| Redis
    
    %% Watch & Cache (The "Informer" Pattern)
    Redis -.-> |Stream Events| Sched1
    Redis -.-> |Stream Events| CM1
    Redis -.-> |Stream Events| Node1
    
    %% Leader Election
    Sched1 -.-> |Acquire Lock| Redis
    Sched2 -.-> |Check Lock| Redis
    
    %% Control Loops
    Sched1 --> |gRPC: Bind Pod| API
    CM1 --> |gRPC: Update Status| API
    Node1 --> |Heartbeat / Status| API

    %% Styling
    classDef component fill:#326ce5,stroke:#fff,stroke-width:2px,color:white;
    classDef db fill:#ff9900,stroke:#333,stroke-width:2px;
    classDef worker fill:#28a745,stroke:#fff,stroke-width:2px,color:white;
    
    class API,Sched1,Sched2,CM1,CM2 component;
    class PG,Redis db;
    class Node1,Node2 worker;
```

## detailed Component Breakthrough

### 1. API Server (The Brain)
-   **Tech**: FastAPI (Async), Uvicorn.
-   **State Management (Postgres)**:
    -   All objects have `kind`, `metadata` (name, uid, **resourceVersion**), `spec`, and `status`.
    -   **Optimistic Locking**: Updates *must* match the current `resourceVersion`. If DB version > Request version -> `409 Conflict`.
-   **Event System (Redis Streams)**:
    -   On successful DB write, API publishes `(Type, Object)` event to Redis Stream `k8s_events`.
-   **Watch Endpoint**:
    -   `GET /api/v1/watch/pods` -> Upgrades connection (SSE or Chunked) -> Subscribes to Redis -> Stream updates to client.

### 2. Scheduler & Controllers (The Logic)
These components use the **Informer Pattern** to avoid hammering the DB.

-   **Informer / Local Cache**:
    1.  **List**: At startup, fetch *all* objects of interest from API.
    2.  **Watch**: Open a long-lived stream from `latest_resource_version`.
    3.  **Cache**: update local dictionary/store based on events.
    4.  **Reconcile**: Business logic reads from *Local Cache* (fast) but writes to *API Server* (safe).

-   **Leader Election**:
    -   Mechanism: `Lease` object (or Redis Lock with TTL).
    -   Loop: Attempt to acquire lock every 5s. If `leader` exists and is healthy, become `Standby`. If `leader` lease expires, acquire and become `Active`.

### 3. Node Agent (Kubelet Simulator)
-   **Registration**: POST /api/v1/nodes on startup.
-   **Heartbeat**: PUT /api/v1/nodes/status every 10s (Lease renewal).
-   **Pod Loop**:
    -   Watch Pods where `spec.nodeName == MyNodeID`.
    -   Compare `spec` vs running containers.
    -   Create/Kill Docker containers accordingly.
    -   Update `pod.status.phase` (Pending -> Running).

## Scalability & Reliability Features implemented

1.  **Resource Versioning & Optimistic Locking**: Prevents "Lost Updates" when multiple controllers try to update the same object.
2.  **Leader Election**: Ensures we can run high-availability control planes without split-brain scheduling.
3.  **Level-Triggered Reconciliation**: Even if an event is missed, the periodic "Resync" loop ensures the state eventually matches.
4.  **Rate Limiting**: API Server protects itself from "Retry Storms" using token buckets.

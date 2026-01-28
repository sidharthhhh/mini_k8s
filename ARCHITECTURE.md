# Architecture Design

## System Overview

The system mimics Kubernetes' decoupled architecture. Components communicate via API Server, never directly accessing the database.

```mermaid
graph TD
    %% Actors
    User((User / Kubectl))
    
    %% Core Components
    subgraph Control_Plane [Control Plane]
        API[API Server <br/>(FastAPI)]
        Sched[Scheduler Service <br/>(gRPC)]
        CM[Controller Manager <br/>(gRPC)]
    end

    %% Data Store Simulation
    subgraph Data_Store [Etcd Simulation]
        PG[(PostgreSQL <br/> Durable State)]
        Redis[(Redis Streams <br/> Event Bus)]
    end

    %% Worker Nodes
    subgraph Worker_Nodes [Data Plane]
        Node1[Node Agent 1 <br/> (Kubelet Clone)]
        Node2[Node Agent 2 <br/> (Kubelet Clone)]
        Docker1[Docker Engine]
        Docker2[Docker Engine]
    end

    %% Data Flows
    %% 1. Submission
    User -- "1. POST /pods (Desired State)" --> API
    
    %% 2. Persistence & Events
    API -- "2. INSERT Pod (Pending)" --> PG
    API -- "3. Publish Event (ADDED)" --> Redis
    
    %% 3. Watch Mechanism
    Redis -.-> |"4. Stream Event (New Pod)"| Sched
    Redis -.-> |"Stream Event"| CM
    Redis -.-> |"Stream Event"| Node1
    
    %% 4. Scheduling Logic
    Sched -- "5. Bind Pod to Node1" --> API
    API -- "6. UPDATE Pod (nodeName=Node1)" --> PG
    API -- "7. Publish Event (MODIFIED)" --> Redis

    %% 5. Node Execution
    Redis -.-> |"8. Watch (Pod Assigned)"| Node1
    Node1 -- "9. Start Container" --> Docker1
    Node1 -- "10. Update Status (Running)" --> API
    
    %% 6. Reconciliation (Background)
    CM -- "Loop: Check Desired vs Actual" --> API
    
    %% Styling
    classDef component fill:#326ce5,stroke:#fff,stroke-width:2px,color:white;
    classDef db fill:#ff9900,stroke:#333,stroke-width:2px;
    classDef worker fill:#28a745,stroke:#fff,stroke-width:2px,color:white;
    
    class API,Sched,CM component;
    class PG,Redis db;
    class Node1,Node2 worker;
```

## Key Components

### 1. API Server (FastAPI)
- **Role**: Central management point. Only component that talks to DB.
- **Responsibility**: Authenticates requests, validates data, persists to Postgres, emits events to Redis.
- **Tech**: Python FastAPI, SQLAlchemy (Async), Redis-py.

### 2. Etcd Simulation (Postgres + Redis)
- **Why?**: Real etcd is key-value with watch. We simulate this:
    - **Postgres**: Durable storage and strong consistency.
    - **Redis Streams**: "Watch" mechanism. When data changes in PG, API pushes an event to Redis. Clients "watch" via API, which streams from Redis.

### 3. Scheduler (gRPC + Python)
- **Role**: Assigns unscheduled pods to nodes.
- **Workflow**: 
    1. Watches for Pods with `spec.nodeName` is Empty.
    2. Filters feasible nodes.
    3. Scores nodes.
    4. Calls `Bind` API on API Server.

### 4. Controller Manager (gRPC + Python)
- **Role**: Reconciles state (Desired vs Actual).
- **Loop**:
    1. Watch for changes (e.g., Deployment created).
    2. Check current state (Count Pods).
    3. Take action (Create/Delete Pods).

## Data Flow: Pod Creation

1. **User** POSTs Pod to `/api/v1/pods`.
2. **API Server** validates payload (Pydantic).
3. **API Server** saves to **Postgres** (Status = `Pending`).
4. **API Server** publishes `ADDED` event to **Redis**.
5. **Scheduler** (watching Redis) sees `ADDED` Pod.
6. **Scheduler** calculates best Node.
7. **Scheduler** calls API `PATCH /pods/<id>` to set `nodeName`.
8. **Node Agent** (watching for Pods on its Node) sees update.
9. **Node Agent** starts container (simulated) and updates Pod Status to `Running`.

# 🏛️ Architecture: System Overview & Topology

## 1. Domain Decomposition & Service Boundaries

The platform is designed around domain-driven microservices implemented primarily in **Go (Golang)** to maximize concurrency, minimize latency, and keep server resource footprints minimal.

```mermaid
classDiagram
    class CoreAPIService {
        +AuthenticateUser(credentials)
        +ValidateSession(token)
        +ManageBillingQuotas(tenantId)
        +Store: MySQL 8.0 (ACID)
    }

    class MarketingService {
        +CompileCampaignWorkflow(blueprint)
        +ScheduleTaskBatch(tasks)
        +AuditExecutionLogs(runId)
        +Store: MongoDB 6.0 (Documents)
    }

    class ChatbotService {
        +HandleWebhook(provider, payload)
        +ExecuteMultiTurnAgent(history, tools)
        +QueryVectorKnowledge(prompt)
        +Store: MongoDB + Vector Store
    }

    class TaskOrchestrator {
        +PollPendingTasks()
        +LeaseTaskToWorker(workerId, taskId)
        +MonitorWorkerHeartbeats()
        +Store: Redis (Streams + Locks)
    }

    CoreAPIService <--> TaskOrchestrator : Token & Quota Checks
    MarketingService --> TaskOrchestrator : Push Execution Pipelines
    ChatbotService --> TaskOrchestrator : Trigger Automation Workflows
```

### Microservice Directory & Roles:

| Service Name | Primary Language | Storage Engine | Responsibility |
| :--- | :--- | :--- | :--- |
| **`svc-core-api`** | Go 1.22 | **MySQL 8.0** | Master identity, RBAC, tenant organization hierarchy, subscription/billing quotas, and token revocation tables. |
| **`svc-business-marketing`** | Go 1.22 | **MongoDB 6.0** | Flexible schema storage for marketing campaigns, dynamic node workflows, automation templates, and audit logs. |
| **`svc-business-chatbot`** | Go 1.22 | **MongoDB + Redis** | High-throughput multi-channel webhook ingestion (Facebook Messenger, Telegram Bot API), LLM tool execution, RAG embeddings. |
| **`orchestrator-go`** | Go 1.22 | **Redis (In-Memory)** | Stateless worker fleet manager, task lease allocation, heartbeat watchdog, and dead-letter queue reclaimer. |
| **`wkr-marketing`** | Python 3.11 | Local SQLite / Mem | Autonomous execution agent using Playwright browser engine, proxy routing, and local anti-detect fingerprinting. |

---

## 2. Polyglot Storage Strategy

A single database paradigm cannot simultaneously support ACID financial transactions, flexible nested workflow graphs, and microsecond-latency task coordination. The platform uses a targeted **Polyglot Persistence Model**:

```mermaid
flowchart LR
    subgraph DataPersistence["Targeted Persistence Strategy"]
        MYSQL[("MySQL 8.0\nRelational Database")]
        MONGO[("MongoDB 6.0\nDocument Store")]
        REDIS[("Redis 7.2\nIn-Memory Store")]
    end

    subgraph UseCases["System Use Cases"]
        U1["Tenants, Users, Billing, Quotas\n• Strict ACID Constraints\n• Relational Integrity"]
        U2["Workflows, Node Graphs, Logs\n• Polymorphic Schemas\n• High Write Throughput"]
        U3["Task Streams, Distributed Leases\n• Sub-millisecond Read/Write\n• Pub/Sub & Streaming"]
    end

    U1 --> MYSQL
    U2 --> MONGO
    U3 --> REDIS
```

### Why Redis Streams Instead of Heavy Message Brokers (Kafka/RabbitMQ)?
1. **Lightweight Operational Overhead:** The infrastructure operates in resource-constrained environments (K3s edge clusters) where running a dedicated Kafka JVM cluster with ZooKeeper/KRaft would consume multiple gigabytes of memory just for idle state.
2. **Consumer Group Primitives:** Redis Streams natively provides consumer groups, automatic message tracking via Pending Entries Lists (PEL), and atomic acknowledgement (`XACK`), matching Kafka’s core semantic capabilities at a fraction of the RAM cost.
3. **Unified Cache & Broker:** Leveraging Redis for both distributed locking (`SET NX EX`) and streaming queues eliminated the complexity of cross-service two-phase commits.

---

## 3. Request Lifecycle: End-to-End Sequence

```mermaid
sequenceDiagram
    autonumber
    actor User as Client / Browser
    participant Edge as Cloudflare Pages (_middleware.ts)
    participant Core as svc-core-api (Go)
    participant Mkt as svc-business-marketing (Go)
    participant Redis as Redis (Streams & Lock)
    participant Orch as orchestrator-go
    participant MQTT as MQTT Broker (EMQX)
    participant Worker as wkr-marketing (Desktop)

    User->>Edge: POST /api/v1/campaigns/trigger
    Note over Edge: Edge validates JWT & strips internal headers
    Edge->>Mkt: Forward sanitized request
    Mkt->>Core: Verify tenant execution quota
    Core-->>Mkt: Quota approved (200 OK)
    Mkt->>Redis: XADD stream:tasks {task_id, payload, lease_duration}
    Mkt-->>User: 202 Accepted {run_id: "run_xyz"}

    Orch->>Redis: XREADGROUP GROUP orch_workers
    Redis-->>Orch: Return task batch
    Orch->>Redis: Acquire Lease Lock (worker_id, task_id, TTL: 60s)
    Orch->>MQTT: Publish command to topic `worker/{id}/cmd`
    MQTT->>Worker: Deliver task via persistent duplex TCP connection
    
    loop Every 20 seconds during execution
        Worker->>MQTT: Publish Heartbeat `worker/{id}/heartbeat`
        MQTT->>Orch: Forward Heartbeat
        Orch->>Redis: Extend Lease Lock TTL (+60s)
    end

    Worker->>MQTT: Publish Task Completed event
    MQTT->>Orch: Forward Completion
    Orch->>Redis: XACK stream:tasks + Release Lease Lock
    Orch->>Redis: XADD stream:events {status: "COMPLETED"}
```

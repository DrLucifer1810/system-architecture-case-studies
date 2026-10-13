# ⚡ Architecture: Event-Driven Task Orchestration & Distributed Leases

## 1. Overview of Task Scheduling Architecture

Long-running automation tasks (such as crawling multiple dynamic pages or batch-messaging verified prospects) introduce unique challenges that traditional synchronous REST endpoints cannot address:
- Tasks may run for minutes or hours.
- Workers can suddenly lose connectivity, encounter hardware reboots, or exhaust memory.
- If a task fails midway, it must either be safely retried without duplicate execution of completed side-effects, or sent to an administrative Dead-Letter Queue (DLQ).

The platform implements an **Event-Driven Task Orchestrator** in Go powered by **Redis Streams** and **Atomic Distributed Leases**.

---

## 2. Redis Streams Topology & Consumer Groups

```mermaid
flowchart LR
    subgraph Producers["Event Producers"]
        P1["svc-business-marketing"]
        P2["svc-business-chatbot"]
    end

    subgraph RedisStream["Redis Stream: stream:tasks"]
        M1["Task #1001"]
        M2["Task #1002"]
        M3["Task #1003"]
    end

    subgraph ConsumerGroup["Consumer Group: cg:task-orchestrators"]
        C1["Orchestrator Instance 1"]
        C2["Orchestrator Instance 2"]
    end

    subgraph PEL["Pending Entries List (PEL)"]
        PEL1["Task #1001 (Leased to Inst 1, Idle: 40s)"]
        PEL2["Task #1002 (Leased to Inst 2, Idle: 120s - STALE!)"]
    end

    subgraph Sweeper["Auto-Claim Sweeper Routine"]
        SWEEP["XAUTOCLAIM / XCLAIM Watchdog"]
    end

    P1 -->|XADD| RedisStream
    P2 -->|XADD| RedisStream
    RedisStream -->|XREADGROUP| C1
    RedisStream -->|XREADGROUP| C2
    C1 -.->|Track in| PEL
    C2 -.->|Track in| PEL
    SWEEP -->|Audit Idle Messages| PEL
    SWEEP -->|Reclaim Stale Tasks| C1
```

---

## 3. The Distributed Worker Lease Protocol

To prevent multiple workers from executing the same campaign or account simultaneously (which could result in anti-spam security bans by third-party platforms), every task execution requires a **Distributed Lease**:

```mermaid
sequenceDiagram
    autonumber
    participant Orch as orchestrator-go
    participant Redis as Redis Server
    participant Worker as Worker Node

    Orch->>Redis: XREADGROUP GROUP cg:tasks Orch-1 COUNT 1
    Redis-->>Orch: Task {ID: "1001", Payload: {...}}

    Note over Orch,Redis: Acquire Distributed Lease
    Orch->>Redis: SET lock:account:acc_889 "orch-1" NX EX 120
    alt Lock Acquisition Failed (Another worker holds lease)
        Redis-->>Orch: NIL
        Orch->>Redis: Delay ACK and defer task to backoff stream
    else Lock Acquisition Succeeded
        Redis-->>Orch: OK
        Orch->>Worker: Dispatch Task Command via MQTT
    end

    loop Worker Execution Heartbeat (Every 30s)
        Worker->>Orch: Heartbeat {task_id: "1001", status: "running"}
        Note over Orch,Redis: Atomic Lua Script to Extend Lease
        Orch->>Redis: EVALSHA renew_script (Verify owner == "orch-1" & EX 120)
        Redis-->>Orch: Renewed OK
    end

    Worker->>Orch: Task Completed {status: "success"}
    Orch->>Redis: DEL lock:account:acc_889
    Orch->>Redis: XACK stream:tasks cg:tasks "1001"
    Orch->>Redis: XDEL stream:tasks "1001"
```

### Atomic Lease Renewal Lua Script:
To guarantee that a slow worker whose lease has already expired does not inadvertently extend or delete a lease newly acquired by another worker, lease renewal and release use Lua scripts:

```lua
-- Atomic Lease Renewal Script
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("expire", KEYS[1], ARGV[2])
else
    return 0
end
```

---

## 4. Crash Recovery & Dead-Letter Queue (DLQ)

If an orchestrator or worker process crashes abruptly:
1. The message remains unacknowledged in the **Pending Entries List (PEL)**.
2. A background sweeper routine executes `XPENDING stream:tasks cg:tasks` every 30 seconds.
3. Any message that has been idle for more than `2 * Heartbeat_Interval` (e.g. > 60 seconds) is reclaimed using `XAUTOCLAIM`.
4. If a task’s delivery counter exceeds the maximum retry threshold (`retry_count >= 3`), the sweeper removes it from `stream:tasks`, writes it to `stream:tasks:dlq`, and emits an alert to the administrative dashboard.

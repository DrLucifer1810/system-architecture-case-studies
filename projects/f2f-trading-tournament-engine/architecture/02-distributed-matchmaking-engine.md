# 📐 Architectural Specification 02: Distributed Matchmaking & Tournament Engine

## 1. Multi-Stage Tournament Progression Mechanics

F2F Arena supports a hybrid two-stage tournament progression model designed to maximize trader engagement and competitive fairness:

```mermaid
flowchart TD
    subgraph Stage_1 ["Stage 1: Points Group Stage (Swiss / Round-Robin)"]
        P1["Trader Pool Registration<br/>(Fixed Entry Fee / Account Checkout)"]
        P2["Group Round 1 (Points Accumulation)"]
        P3["Group Round 2 (Points Accumulation)"]
        P4["Leaderboard Calculation<br/>(Top N Qualified by Points & Profit Factor)"]
    end

    subgraph Stage_2 ["Stage 2: Knockout Elimination (Single / Double Elimination)"]
        K1["Bracket Seed Generation<br/>(Seed #1 vs Seed #N)"]
        K2["Quarter-Finals (1v1 Live Trading Match)"]
        K3["Semi-Finals (1v1 Live Trading Match)"]
        K4["Grand Final & Podium Distribution"]
    end

    P1 --> P2 --> P3 --> P4
    P4 -->|"Top 8 / Top 16 Seeds"| K1
    K1 --> K2 --> K3 --> K4
```

---

## 2. Automated Tournament State Machine (`tournament_management.py`)

Tournaments transition through a deterministic lifecycle without human supervisor intervention:

$$\text{Upcoming} \xrightarrow{\text{event\_time} \le \text{now}} \text{Live (Round 1)} \xrightarrow{\Delta t_{\text{round}}} \text{Round Transition} \xrightarrow{\text{Final Round}} \text{Completed}$$

```python
async def check_and_start_tournaments():
    \"\"\"
    Scans the database for tournaments that have reached their scheduled start time,
    automatically generates round 1 pairings, and assigns demo trading accounts.
    \"\"\"
    pool = await get_pool()
    async with pool.acquire() as conn:
        async with conn.cursor(DictCursor) as cursor:
            query = \"\"\"
                SELECT id, event_time, status, total_rounds, total_round_duration_minutes 
                FROM tournament_info 
                WHERE status = 'upcoming' AND event_time <= UTC_TIMESTAMP()
            \"\"\"
            await cursor.execute(query)
            tournaments = await cursor.fetchall()
            
            for t in tournaments:
                await start_tournament_round_one(cursor, t["id"])
```

---

## 3. Bracket Generation & Pairing Logic

For knockout rounds, contestant pairings are generated using standard tournament seeding algorithms:
- Seed $i$ is paired with Seed $(N - i + 1)$ to ensure that the highest-ranked contestants do not eliminate each other in preliminary rounds.
- Ties in score are broken deterministically by:
  1. **Total Profit / ROI Percentage:** $\frac{\text{Ending Equity} - \text{Starting Balance}}{\text{Starting Balance}} \times 100\%$
  2. **Maximum Drawdown ($\text{MDD}$):** Lowest peak-to-trough decline wins.
  3. **Total Traded Volume:** Highest lot volume executed during the round.

# 🔄 Architecture: Dynamic Macro-Orchestration DAG & Autonomous Quality Control (AQC)

## 1. Beyond Static SOPs: The Dynamic State Graph

Traditional automation tools rely on static, hardcoded Standard Operating Procedures (SOPs). If a web UI changes by a single button or an authentication wall appears, static pipelines crash.

Fusio implements an adaptive **Dynamic State Graph** (similar to LangGraph) managed by the `CognitiveOrchestrator`:
- Nodes in the graph are not hardcoded functions; they are dynamically generated sub-agent goals.
- Transitions between nodes are evaluated at runtime by a **State Graph Supervisor** inspecting the shared **Session Blackboard**.
- The graph supports loops, dynamic branching, and recursive node insertion.

```mermaid
stateDiagram-v2
    [*] --> Intent_Classification
    
    Intent_Classification --> Zero_Shot_Discovery : Target tools/apps unspecified (Blind Task)
    Intent_Classification --> Generate_Initial_DAG : Context & tools recognized
    
    Zero_Shot_Discovery --> Generate_Initial_DAG : Schema & docs learned into Blackboard
    
    state DAG_Execution_Loop {
        [*] --> Select_Next_Node
        Select_Next_Node --> Execute_SubAgent : Pass scoped memory & evaluation_criteria
        Execute_SubAgent --> AQC_Verification : Inspect output & screenshots
        
        AQC_Verification --> Select_Next_Node : Score >= 8/10 (Pass)
        AQC_Verification --> Self_Healing_Loop : Score < 8/10 or BlockedException
        
        state Self_Healing_Loop {
            [*] --> Triage_Error
            Triage_Error --> Spawn_Rescue_Agent : UI obstruction (Popups/Login)
            Triage_Error --> Dynamic_Recovery_DAG : Cognitive Gap (Changed Web Interface)
            Spawn_Rescue_Agent --> Resolve_Blocker
            Dynamic_Recovery_DAG --> Patch_App_Dossier
            Resolve_Blocker --> [*]
            Patch_App_Dossier --> [*]
        }
        
        Self_Healing_Loop --> Execute_SubAgent : Retry with refreshed context
    }

    DAG_Execution_Loop --> [*] : Supervisor signals FINISH
```

---

## 2. Zero-Shot Discovery (Node 0)

When a user provides an open-ended command without specifying tools or application endpoints (e.g. *"Create a 60-second animated video explaining photosynthesis from text"*):
1. The `CognitiveOrchestrator` does not reject the command or hallucinate non-existent API parameters.
2. It dynamically prepends **Node 0 (Zero-Shot Discovery)** to the execution plan.
3. Node 0 activates browser and search tools to investigate currently available services, evaluate API/UI ergonomics, crawl documentation, and write a structured capability profile into the **App Skill Dossier**.
4. Once Node 0 finishes, the orchestrator recursively calls itself, synthesizing execution nodes (Scriptwriting -> Voiceover Synthesis -> Video Stitching) grounded in the newly acquired knowledge.

---

## 3. Autonomous Quality Control (AQC)

A critical failure mode in multi-agent workflows is the **compounding error cascade**: an early node produces a slightly flawed prompt or image, which causes subsequent nodes to produce unusable output.

Fusio implements **Autonomous Quality Control (AQC)**:

```mermaid
sequenceDiagram
    autonumber
    participant Orch as CognitiveOrchestrator
    participant Node as Sub-Agent (Node 2: Image Generation)
    participant Supervisor as Supervisor LLM (AQC Inspector)
    participant BB as Session Blackboard

    Orch->>Node: Execute Node with `evaluation_criteria`:<br/>{"consistent_character": true, "aspect_ratio": "16:9"}
    Node->>Node: Execute Playwright script & capture output asset
    Node-->>Supervisor: Submit output asset + screenshot
    
    Supervisor->>Supervisor: Evaluate asset against `evaluation_criteria`
    alt Score < 8/10 (e.g. Aspect ratio generated as 1:1 instead of 16:9)
        Supervisor-->>Node: REJECT {score: 5/10, reason: "Aspect ratio mismatch", suggested_patch: "Add --ar 16:9 flag"}
        Note over Node: Cyclical Feedback Loop:<br/>Node re-runs with corrective instruction
        Node->>Node: Re-execute with adjusted parameters
        Node-->>Supervisor: Resubmit corrected asset
        Supervisor-->>Orch: APPROVED (Score: 9.5/10)
    else Score >= 8/10
        Supervisor-->>Orch: APPROVED (Score: 9.0/10)
    end
    
    Orch->>BB: Write verified asset URI to Blackboard
    Orch->>Orch: Transition to Node 3 (Video Synthesis)
```

### Dynamic Criteria Enforcement:
Evaluation criteria are **never hardcoded**. The orchestrator extracts explicit constraints from the user prompt and translates them into machine-evaluable JSON criteria:
```json
{
  "node_id": "node_02_voiceover",
  "goal": "Generate professional voiceover audio",
  "evaluation_criteria": {
    "duration_min_sec": 55,
    "duration_max_sec": 65,
    "tone": "enthusiastic_educational",
    "required_language": "vi"
  }
}
```
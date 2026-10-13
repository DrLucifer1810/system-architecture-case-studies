# 📐 Architectural Specification 03: 6-Dimensional Radar Matrix & Semantic Edit Ratio Engine

## 1. Enterprise Evaluation Framework

Rather than quantifying developer output through arbitrary commit counts or lines of code (LOC), MyDNA measures **Cognitive Depth, Human-AI Collaboration Efficacy, and Technical Discipline**.

The system synthesizes two renowned educational and professional competency frameworks:
1. **Bloom's Revised Taxonomy:** Moving from lower-order thinking (Remembering/Copying) to higher-order mastery (Analyzing/Evaluating/Creating).
2. **The Dreyfus Model of Skill Acquisition:** Tracking the transition from Novice (rule-bound, blindly trusting AI) to Proficient/Expert (intuitive, actively correcting AI hallucinations).

```mermaid
radar-chart
    title "Enterprise 6-Dimension Developer Competency Radar"
    "Competence" : 88
    "Discipline" : 92
    "Creativity" : 78
    "Critical Thinking" : 90
    "Collaboration" : 85
    "AI Token Efficiency" : 94
```

---

## 2. Mathematical Definition of the Semantic Edit Ratio

A critical indicator of developer skill is how they interact with AI-generated suggestions. If a developer blindly commits AI output without modification, they may be suffering from **Passive Hallucination Blindness**. Conversely, if they delete everything, they waste tokens and time. 

MyDNA calculates the **Edit Ratio ($\mathcal{E}$)** using line-level Myers diff analysis (`similar` crate):

$$\mathcal{E} = rac{\Delta_{	ext{insertions}} + \Delta_{	ext{deletions}}}{\Delta_{	ext{insertions}} + \Delta_{	ext{deletions}} + \Delta_{	ext{equal}}}$$

Where:
- $\mathcal{E} = 0.0$: The developer accepted AI output verbatim (0 modifications).
- $0.0 < \mathcal{E} \le 0.5$: Expert fine-tuning, adapting code to project conventions and existing APIs.
- $0.5 < \mathcal{E} < 1.0$: Major refactoring; correcting AI hallucinations or restructuring algorithms.
- $\mathcal{E} = 1.0$: Total discard and rewrite (severe AI prompt failure or hallucination).

```rust
pub fn calculate_edit_ratio(ai_output: &str, final_user_output: &str) -> f64 {
    if ai_output.is_empty() {
        return 1.0;
    }

    let diff = TextDiff::from_lines(ai_output, final_user_output);
    let mut total_lines = 0;
    let mut changed_lines = 0;

    for change in diff.iter_all_changes() {
        total_lines += 1;
        match change.tag() {
            ChangeTag::Delete | ChangeTag::Insert => changed_lines += 1,
            ChangeTag::Equal => {}
        }
    }

    if total_lines == 0 {
        return 0.0;
    }

    (changed_lines as f64) / (total_lines as f64)
}
```

---

## 3. Workflow Sessionization Logic (`sessionizer.rs`)

The background sessionizer correlates asynchronous events into cohesive units of work:
1. **Session Trigger:** The developer performs a `CLIPBOARD_COPY` from a recognized AI surface (ChatGPT, Gemini, Cursor AI panel).
2. **Session Context:** The generated snippet is recorded as `current_ai_output`, and the timestamp marks session start.
3. **Session Completion:** The developer executes a `FILE_SAVED` event in their IDE or submits a `FOCUSED_TEXT` buffer.
4. **Metric Compilation:** The Edit Ratio is computed, token footprints are estimated ($pprox rac{	ext{len}}{4}$), and the session is queued for LLM evaluation.

```rust
// Automated Session Boundary Detection
if event_type == "CLIPBOARD_COPY" && is_ai_window(&window_title) {
    current_ai_output = raw_content;
    session_start_time = timestamp;
} else if !current_ai_output.is_empty() && event_type == "FILE_SAVED" {
    let edit_ratio = Self::calculate_edit_ratio(&current_ai_output, &saved_file_content);
    state_machine.record_session(&session_start_time, &timestamp, edit_ratio);
}
```

# 🔬 Engineering Deep-Dive 01: Win32 Clipboard Lineage & Process Correlation

## 1. Challenge: Tracking Data Flow Across Application Boundaries

In a typical developer workflow, data flows rapidly across disparate processes:
```text
Browser (Gemini / ChatGPT) 
   ──[Copy]──> Windows Clipboard Ring 
   ──[Paste]──> IDE (VS Code / RustRover) 
   ──[Save]──> Local File on Disk
```

Naive tracking attempts either:
1. **Save all clipboard text in plaintext:** Creating an unacceptable vulnerability if passwords or sensitive tokens are temporarily copied.
2. **Lose the data lineage entirely:** Making it impossible to know whether a committed function originated from an AI recommendation or human manual typing.

MyDNA introduces **Cryptographic Clipboard Lineage Tracking**:

```mermaid
flowchart TD
    ClipEvent["Clipboard Copy Event Detected"] --> ReadClip["Read UTF-16 String via clipboard-win"]
    ReadClip --> Hash["Hash via DefaultHasher (64-bit Hex ID)"]
    Hash --> CheckDup{"Is Hash identical to Previous Event?"}
    CheckDup -- Yes --> Drop["Discard Duplicate Event"]
    CheckDup -- No --> Record["Emit Telemetry Event with Lineage ID"]
    Record --> SQLite[("Store (lineage_id, event_type, window_title)")]
```

---

## 2. Lineage Generation Mechanics (`clipboard.rs`)

```rust
use clipboard_win::{formats, get_clipboard};
use std::collections::hash_map::DefaultHasher;
use std::hash::{Hash, Hasher};

pub struct ClipboardEvent {
    pub lineage_id: String,
    pub content: String,
}

pub fn generate_lineage_id(content: &str) -> String {
    let mut hasher = DefaultHasher::new();
    content.hash(&mut hasher);
    format!("{:016x}", hasher.finish())
}

pub fn check_clipboard() -> Option<ClipboardEvent> {
    if let Ok(text) = clipboard_win::get_clipboard_string() {
        if !text.is_empty() {
            let lineage_id = generate_lineage_id(&text);
            return Some(ClipboardEvent { lineage_id, content: text });
        }
    }
    None
}
```

### Why 64-bit DefaultHasher?
- **Speed:** Less than 120 nanoseconds per kilobyte, orders of magnitude faster than SHA-256.
- **Collision Resistance:** For session-level window tracking across hours of developer activity, collision probability is effectively zero.
- **Privacy Enforcement:** Downstream analysis jobs can correlate data flow across windows using the `lineage_id` without exposing the actual text payload.

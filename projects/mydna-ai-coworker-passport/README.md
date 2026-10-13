# 🧬 MyDNA Architecture Blueprint (English Edition)

[![Language: English](https://img.shields.io/badge/Language-English-blue?style=flat-square)](https://github.com/DrLucifer1810/system-architecture-case-studies/tree/main/projects/mydna-ai-coworker-passport)
[![Vietnamese Edition](https://img.shields.io/badge/Phiên%20bản-Tiếng%20Việt-red?style=flat-square)](https://github.com/DrLucifer1810/system-architecture-case-studies-vi/tree/main/projects/mydna-ai-coworker-passport)

## 📌 Executive Summary

**MyDNA** represents a paradigm shift in developer performance assessment and human-AI collaboration analytics. Built as an autonomous, privacy-preserving desktop daemon in Rust and Tauri v2, it passively captures fine-grained developer interactions at the operating system level, sanitizes all private data locally, and constructs an evidence-based **"AI-Coworker Skill Passport"**.

Modern enterprises and technical leaders face two major challenges in the era of generative AI:
1. **The Resume Fraud & Illusion of Competence:** Superficial code volume or blind copy-pasting from LLMs can mask fundamental knowledge deficiencies and introduce catastrophic architectural vulnerabilities.
2. **Privacy and Surveillance Concerns:** Conventional employee monitoring software captures invasive keystrokes and plain-text screenshots, violating privacy and creating security liabilities.

MyDNA solves both challenges through **Edge Telemetry with Zero-Knowledge Local Redaction** and a **6-Dimensional Enterprise Competency Matrix** based on Bloom's Taxonomy and the Dreyfus Model of Skill Acquisition.

---

## 🏛️ System Architecture Index

### 📐 Architectural Specifications
1. [**01. Native OS Telemetry Engine & Hybrid Desktop Stack**](./architecture/01-system-overview.md)  
   Win32 API integration, active window tracking, COM UIAutomation accessibility hooks, and thread-safe SQLite event streaming.
2. [**02. Client-Side Zero-Leakage Privacy Redaction (PIR)**](./architecture/02-privacy-redaction-pipeline.md)  
   High-throughput regex singletons sanitizing API keys, OAuth tokens, personal emails, and payment information before any model ingestion.
3. [**03. 6-Dimensional Radar Matrix & Semantic Edit Ratio Engine**](./architecture/03-evaluation-matrix-engine.md)  
   Myers diff lineage, edit ratio mathematical formulas, Bloom's Taxonomy problem-solving classification, and token efficiency analysis.
4. [**04. Embedded Localhost Model Context Protocol (MCP) Server**](./architecture/04-embedded-mcp-server.md)  
   Axum JSON-RPC 2.0 server on port 5050 exposing `/mcp/resources/user_dna` for seamless zero-cloud integration with Cursor, Claude Desktop, and Antigravity.

### 🔬 Engineering Deep-Dives
1. [**01. Win32 Clipboard Lineage & Process Correlation**](./deep-dives/01-win32-clipboard-lineage.md)  
   Lineage hashing with `DefaultHasher`, cross-application data flow tracking, and leak-free memory management.
2. [**02. Hot-Reloadable Multi-Agent DNA Profiler**](./deep-dives/02-hot-reload-multi-agent-dna.md)  
   CrewAI-inspired YAML prompt decomposition, specialized agents (`CodeAnalyzer`, `CommunicationAnalyzer`, `CareerDiagnostic`), and runtime prompt evolution.
3. [**03. Headless WebView Companion vs Playwright Automation**](./deep-dives/03-headless-companion-ipc.md)  
   Zero-binary overhead AI session automation via native OS WebView2, URL fragment hash IPC (`#bzt-{KIND}-{stamp}`), and anti-bot mitigation.

---

## 📊 Core Architectural Matrix

| Layer | Technology | Primary Role | Performance / SLA |
| :--- | :--- | :--- | :--- |
| **OS Telemetry** | Rust (`windows-rs`, `windows-sys`) | Win32 foreground window & process inspection | `< 2 ms` hook overhead |
| **Clipboard Lineage** | `clipboard-win`, `DefaultHasher` | Non-invasive data tracking via 64-bit hashing | Zero plaintext persistence |
| **Filesystem Watcher** | `notify` (Notify-rs) | Real-time workspace modification detection | Async event dispatch |
| **State Machine** | `rusqlite` (SQLite WAL) | Embedded transactional timeline & session storage | Single-file zero config |
| **Privacy Redaction** | `regex` with `OnceLock` | Local scrubbing of credentials and PII | `> 450 MB/s` throughput |
| **Diff & Metrics** | `similar` (TextDiff) | Line-level insertion/deletion classification | Deterministic $0.0 - 1.0$ ratio |
| **Agent Server** | `axum`, `tokio` (Port 5050) | Localhost MCP JSON-RPC 2.0 endpoint | `< 1.2 ms` response latency |
| **Desktop UI** | Tauri v2, Chart.js, HTML5 | Interactive Radar Chart & profile visualization | Native WebView2 runtime |

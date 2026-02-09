# Dynamic Agent Runtime Tool-Stack (D.A.R.T.S)

## 1. System Overview & Objective
**Objective:** To implement a dynamic, resource-efficient AI agent architecture that eliminates context pollution and token waste. Instead of loading a monolithic library of all available tools, this system utilizes a "Just-In-Time" (JIT) tool loading strategy.

**Core Philosophy:**
1.  **Diagnosis First:** No execution begins without a validated requirement list.
2.  **Lean Context:** The runtime environment contains *only* the tools necessary for the immediate task.
3.  **Standardized Handshake:** Communication between the *Diagnostician* and the *Executor* occurs via strict JSON manifests.

---

## 2. Architectural Workflow

The system operates in a strictly defined three-phase loop:

### Phase 1: Diagnosis & Strategy (The Architect)
* **Input:** User Prompt / Raw Task.
* **Role:** Analyze the request, break it down into sub-tasks, and identify required capabilities.
* **Output:** `tool_manifest.json` (See Section 3).
* **Constraint:** This agent has *read-only* access to the Tool Registry index (descriptions only, no code).

### Phase 2: Orchestration (The Loader)
* **Input:** `tool_manifest.json`.
* **Role:**
    1.  Parse the manifest.
    2.  Retrieve specific Python/API wrappers from the `/skills` library.
    3.  Construct a temporary execution environment (The "Stack").
    4.  Inject the selected tools into the Execution Agent's system prompt.

### Phase 3: Execution (The Specialist)
* **Input:** User Task + Loaded Tool Stack.
* **Role:** Execute the task using the provided tools.
* **Output:** Final Result.

---

## 3. Data Protocols (AI-Readable Schemas)

Future agents working on this repo must adhere to the following JSON schemas for inter-agent communication.

### 3.1 The Tool Registry Schema
The master index of all available skills (`registry_index.json`) follows this format:

```json
{
  "skills": [
    {
      "id": "financial_verification_v1",
      "category": "finance",
      "description": "Verifies corporate identity and basic stock data via AlphaVantage API.",
      "cost_rating": "low",
      "latency": "medium",
      "dependencies": ["requests", "pandas"]
    },
    {
      "id": "github_repo_analyzer",
      "category": "coding",
      "description": "Clones and scans a repository to generate a file tree and summary.",
      "cost_rating": "high",
      "dependencies": ["gitpython"]
    }
  ]
}
```

### 3.2 The Requirement Manifest Schema
The *Diagnostician* must output this strict JSON structure to request tools:

```json
{
  "task_id": "unique_id_123",
  "reasoning": "User wants to analyze Apple's 10-K; requires financial data and PDF parsing.",
  "required_stack": [
    {
      "skill_id": "sec_edgar_retriever",
      "priority": "critical"
    },
    {
      "skill_id": "pdf_text_extractor",
      "priority": "high"
    }
  ],
  "execution_constraints": {
    "max_steps": 10,
    "allow_web_search": false
  }
}
```

---

## 4. Repository Structure

The file system is organized to separate logic from capabilities:

```text
/
├── core/
│   ├── diagnostician.py   # Agent: Diagnoses task & writes manifest
│   ├── orchestrator.py    # Logic: Loads tools based on manifest
│   └── executor.py        # Agent: Uses the tools to solve task
├── skills/                # The "Warehouse" of capabilities
│   ├── finance/
│   │   └── stock_lookup.py
│   ├── coding/
│   │   └── git_ops.py
│   └── web/
│       └── search_wrapper.py
├── registry_index.json    # The menu available to the Diagnostician
├── tool_manifest.json     # The dynamic handover file (generated at runtime)
└── README.md              # System definition
```

---

## 5. Implementation Guidelines for AI Developers

If you are an AI reading this to generate code, follow these rules:

1.  **Tool Isolation:** Every script in `/skills` must be a standalone class that inherits from a `BaseTool` class.
2.  **Standardized Returns:** All tools must return a string or JSON dictionary. Never return raw objects or binary data to the LLM.
3.  **Error Handling:** If a tool fails (e.g., API downtime), it must return a clear error message string so the Execution Agent can attempt a retry or alternative strategy.

## 6. Future Expansion: The "Self-Correction" Loop
*Current Status: Planned*

If the Execution Agent fails because the `tool_manifest` was insufficient:
1.  The Executor returns a `FAILURE` status with a log.
2.  The Orchestrator passes the log back to the Diagnostician.
3.  The Diagnostician generates a *new* manifest with different or additional tools.

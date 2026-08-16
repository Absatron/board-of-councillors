> **Public portfolio edition.** This document describes the system architecture without deployment-specific identifiers, credentials, or personal health data. Workflow JSON exports in this repository are sanitized structural references — configure credentials, folder IDs, and API keys in your own n8n instance before use.

# Board of Councillors — Technical Design Document

**System:** Board of Councillors  
**Platform:** n8n (Self-hosted)  
**Main Workflow:** `Board of Councillors` (assign workflow ID on deploy)  
**Status:** Complete — Production  
**Created:** 16 February 2026  
**Implementation Finalized:** 12 August 2026  
**Document Date:** 12 August 2026

---

## 1. Executive Overview

The Board of Councillors is a multi-agent iterative deliberation system that enables a user to submit health-related queries and observe a structured, multi-round discussion between three AI expert agents — a **Neuroscientist**, a **Psychologist**, and a **Physiologist** — with an **Epistemologist Fact-Checker** verifying claims against scientific literature after each round. Each expert analyses the query through their disciplinary lens, grounded in the user's complete longitudinal health history, and converses with awareness of what the other experts have said.

The system is designed around the metaphor of a medical council: the user poses a question, and a panel of specialists deliberates — not in isolation, but in dialogue — before a synthesiser distils the discussion into a unified, actionable response. An **Academic Scout** then surfaces the most relevant recent paper from a curated literature database.

### 1.1 Core Design Principles

- **Multi-perspectival analysis** — Every query is examined through three distinct scientific lenses (neural hardware, psychological software, physiological body systems), ensuring no single-domain bias dominates
- **Evidence-grounded deliberation** — An Epistemologist Fact-Checker verifies expert claims against Semantic Scholar and Exa literature search after each deliberation round; experts can autonomously search domain-specific literature during analysis
- **Iterative deliberation** — Experts engage in configurable multi-round discussion, building on and responding to each other's analyses and fact-check feedback
- **Full context grounding** — All agents have access to the user's complete life history across clinical and lifestyle documents, preventing decontextualised advice
- **Observable process** — The user watches the discussion unfold in real time via dedicated Telegram bot personas, one per expert, making the reasoning process transparent rather than opaque
- **Living records** — The system simultaneously updates the user's longitudinal clinical records with new observations extracted from each query, maintaining an ever-growing knowledge base
- **Modular architecture** — Core capabilities are decomposed into callable subworkflows, enabling independent development, testing, and future extension without modifying the main orchestrator
- **Separation of concerns** — Dedicated agents and subworkflows handle routing, record management, deliberation, fact-checking, synthesis, and literature recommendation, each with clearly bounded responsibilities

### 1.2 User Experience Summary

1. The user sends a natural language health query via Telegram
2. A confirmation message acknowledges receipt
3. Three expert bots begin sending their analyses in parallel, each from their own Telegram identity
4. After each round, the Epistemologist fact-checks claims and stores feedback that experts see in subsequent rounds
5. If multiple deliberation rounds are configured, the experts respond again with awareness of the full discussion and fact-check critiques
6. A final synthesis bot delivers a unified, cross-domain response grounded in validated claims
7. An Academic Scout bot delivers a relevant recent paper recommendation from the curated literature database
8. Meanwhile, in the background, relevant clinical records are updated with any new observations from the query

### 1.3 Key Metrics

| Metric | Value |
| --- | --- |
| Total nodes (all workflows) | 131 across 7 workflows |
| Main orchestrator nodes | 42 |
| Subworkflow nodes | 89 (6 subworkflows) |
| AI agent nodes | 8 (`agent` v3.1) |
| AI sub-agent tools | 4 (domain specialist record managers) |
| OpenRouter LLM instances | 24 (primary/fallback pairs) |
| Google Docs integrations | 9 (dynamic discovery + specialist read/write) |
| Google Drive integrations | 2 (folder listing for dynamic document discovery) |
| PostgreSQL operations | 14 nodes across 3 tables |
| Telegram bot personas | 6 (main + 3 experts + synthesiser + academic scout) |
| Deliberation rounds | Configurable (default: 1, user-adjustable via `/setloops`) |
| External prompt files | 26 files in `prompts/` |

### 1.4 Workflow Inventory

| Workflow | ID | Nodes | Role |
| --- | --- | --- | --- |
| **Board of Councillors** (main) | — | 42 | Orchestration, loop control, synthesis, academic scout |
| **Consult the board** | — | 32 | Expert deliberation + Epistemologist fact-checking |
| **Get personal records** | — | 10 | Dynamic context aggregation |
| **Update personal records** | — | 25 | Health Record Manager + specialists |
| **Get latest scientific papers** | — | 13 | Scheduled RAG ingestion pipeline |
| **Semantic Scholar search** | — | 5 | Abstract and full-paper literature search |
| **Exa search** | — | 4 | Domain-filtered web literature search |

---

## 2. System Architecture

### 2.1 High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           TELEGRAM INTERFACE                                    │
│  ┌──────────┐  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐           │
│  │ Main Bot │  │ NeuroSci Bot │  │ Psych Bot    │  │ Physio Bot    │           │
│  │ (Main)   │  │ (Expert Out) │  │ (Expert Out) │  │ (Expert Out)  │           │
│  └────┬─────┘  └──────▲───────┘  └──────▲───────┘  └───────▲───────┘           │
│       │               │                 │                   │                 │
│  ┌────▼─────┐  ┌──────┴─────────────────┴───────────────────┘                 │
│  │ Trigger  │  │     Consult the board (Subworkflow)                            │
│  └────┬─────┘  └──────▲───────────────────────────────────┐                     │
│       │               │                                    │                    │
│  ┌────▼──────────────────────────────────────────────────────────────────────┐  │
│  │                    MAIN ORCHESTRATOR (Board of Councillors)               │  │
│  │  Command Router → Administrator → Get personal records (Subworkflow)      │  │
│  │       │              │                    │                               │  │
│  │       │              ├── Update personal records (Subworkflow, async)     │  │
│  │       │              └── Prepare Payload → Consult the board ──┐          │  │
│  │       │                                                          │          │  │
│  │       │         ┌────────────────────────────────────────────────┘          │  │
│  │       │         │  Loop: Get messages → Update payload → If ≤ max_loops   │  │
│  │       │         │  Exit: Conversation Synthesizer + Academic Scout          │  │
│  └───────┴─────────┴─────────────────────────────────────────────────────────┘  │
│                                                                                   │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │              CONSULT THE BOARD SUBWORKFLOW (per round)                     │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌─────────────┐                         │   │
│  │  │ Neuroscientist│ │ Psychologist │ │ Physiologist│  (+ Exa literature)    │   │
│  │  └──────┬───────┘ └──────┬───────┘ └──────┬──────┘                         │   │
│  │         └────────────┬───┴────────────────┘                              │   │
│  │                      ▼                                                     │   │
│  │         Wait → Epistemologist (+ Semantic Scholar tools)                   │   │
│  │                      ▼                                                     │   │
│  │         Store fact_check_feedback in council_messages                      │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                                                                   │
│  ┌────────────────────────────┐  ┌──────────────────────────────────────────┐   │
│  │ GET PERSONAL RECORDS       │  │ GET LATEST SCIENTIFIC PAPERS (scheduled) │   │
│  │ Google Drive folder list   │  │ Semantic Scholar → embed → PGVector      │   │
│  │ → fetch docs → merge       │  │ (weekly ingestion for Academic Scout)    │   │
│  └────────────────────────────┘  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Technology Stack

| Layer | Technology | Details |
| --- | --- | --- |
| Orchestration | n8n (Self-hosted) | Workflow automation platform with subworkflow composition |
| LLM Provider | OpenRouter | `deepseek/deepseek-v4-flash-0731` (primary), `tencent/hy3-preview` (fallback) |
| Embeddings | Google Gemini | `models/gemini-embedding-2` for vector store |
| Vector Store | PostgreSQL + PGVector | `scientific_literature` table for RAG retrieval |
| Database | PostgreSQL | Session state, conversation messages, vector embeddings |
| Document Store | Google Docs + Google Drive | Dynamic folder discovery; OAuth2 read/write |
| Literature APIs | Semantic Scholar, Exa | Abstract/full-paper search and domain-filtered web search |
| User Interface | Telegram Bot API | 6 bot personas for multi-identity output |
| AI Framework | n8n LangChain nodes | Agent, AgentTool, OutputParser, VectorStore, ToolWorkflow |
| Prompt Management | External `prompts/` directory | Version-controlled system and user prompts per agent |

### 2.3 Execution Flow Phases

The system executes in **six distinct phases**, with background processes running independently:

| Phase | Name | Description | Parallelism |
| --- | --- | --- | --- |
| 1 | **Ingestion** | Telegram trigger → Command routing → Administrator screening | Sequential |
| 2 | **Context Assembly** | `Get personal records` subworkflow: Drive folder listing, document fetch, session/history merge | Highly parallel |
| 3 | **Deliberation** | `Consult the board` subworkflow: 3 experts analyse in parallel, Epistemologist fact-checks | Parallel per round |
| 4 | **Synthesis** | Conversation Synthesizer distils validated multi-round discussion | Sequential |
| 5 | **Literature Recommendation** | Academic Scout queries PGVector for relevant recent paper | Parallel with Phase 4 |
| 6 | **Record Update** | `Update personal records` subworkflow routes observations to specialists | Async from Phase 1 |

Phases 4 and 5 run in parallel when deliberation completes. Phase 6 runs concurrently with Phases 2–5.

**Background (independent schedule):** `Get latest scientific papers` runs weekly, ingesting new abstracts into the PGVector store for Academic Scout retrieval.

---

## 3. Entry Point & Command Routing

### 3.1 Telegram Trigger

The workflow is initiated by a **Telegram Trigger** node (`n8n-nodes-base.telegramTrigger` v1.2) listening for `message` updates. All user input enters through the main bot persona (Main_Bot).

**Incoming payload structure:**

```json
{
  "message": {
    "text": "<user input>",
    "chat": {
      "id": "<telegram_chat_id>"
    }
  }
}
```

### 3.2 Command Router

A **Switch** node (`Command Router`) inspects the incoming message text and routes to one of four branches:

| Output | Command | Condition | Description |
| --- | --- | --- | --- |
| 0 | `/setloops N` | `message.text` starts with `/setloops` | Configures the number of deliberation rounds |
| 1 | `/new` | `message.text` starts with `/new` | Creates a new deliberation session |
| 2 | `/clear` | `message.text` starts with `/clear` | Clears all conversation history from the database |
| 3 | _(default)_ | Always true (catch-all) | Treats input as a health query and sends to the Administrator |

### 3.3 Command: `/setloops N`

**Purpose:** Allows the user to configure how many deliberation rounds the council performs.

**Flow:**

1. Parses the integer from the message text: `parseInt($json.message.text.split(' ')[1])`
2. Executes an `UPDATE` on the `session_state` table:
   ```sql
   UPDATE session_state
   SET max_loops = <N>, updated_at = NOW()
   WHERE chat_id = '<chat_id>';
   ```
3. Sends a confirmation message via Telegram

**Impact:** On the next query, the council will perform `N` rounds of deliberation before synthesising. Default is `1` (set during session creation).

### 3.4 Command: `/new`

**Purpose:** Creates a fresh deliberation session with a new unique identifier.

**Flow:**

1. **Generate New Session ID** — A Code node generates a UUID using Node.js `crypto.randomUUID()`
2. **Update Session Pointer** — Upserts into `session_state`:
   - Sets `current_session_id` to the new UUID
   - Sets `max_loops` to `1` (default)
   - Matches on `chat_id` to update the existing row or create a new one
3. Sends a confirmation message via Telegram

### 3.5 Command: `/clear`

**Purpose:** Truncates all rows from the `council_messages` table.

**Flow:**

1. Executes a `DELETE` operation (table truncation) on `council_messages`
2. Sends a confirmation message via Telegram

**Caution:** This is a destructive operation that removes all conversation history across all sessions.

### 3.6 Default Route (Query Processing)

Any message that does not match a command prefix is treated as a health query. It is forwarded to the **Administrator** agent for screening and dispatch.

---

## 4. Administrator Agent & Record Management Pipeline

### 4.1 Administrator Agent

The **Administrator** (`@n8n/n8n-nodes-langchain.agent` v3.1) is the first AI agent in the pipeline. It serves as a gatekeeper that:

1. Extracts the user's query verbatim
2. Determines whether the user has explicitly opted out of record updates

**Output parsing:** A **Structured Output Parser** enforces the output schema:

```json
{
  "type": "object",
  "properties": {
    "update_allowed": { "type": "boolean" },
    "user_query": { "type": "string" }
  },
  "required": ["update_allowed", "user_query"]
}
```

**LLM configuration:**

- Primary: `deepseek/deepseek-v4-flash-0731` (OpenRouter)
- Fallback: `tencent/hy3-preview` (OpenRouter)

### 4.2 Three-Way Dispatch

On successful execution, the Administrator's output fans out to three parallel branches:

```
Administrator (success)
    ├──→ Update Allowed (switch) ──→ Update personal records (subworkflow, if update_allowed == true)
    ├──→ Send confirmation message ("We have received your query...")
    └──→ Get personal records (subworkflow) ──→ Context Assembly Pipeline
```

On error, the Administrator routes to an error notification via Telegram.

### 4.3 Update Allowed Gate

A **Switch** node (`Update allowed`) checks the parsed output:

- If `$json.output.update_allowed == true` → invokes the `Update personal records` subworkflow
- If `false` → the record management branch is skipped entirely

### 4.4 Update Personal Records Subworkflow

The **Update personal records** subworkflow (25 nodes) encapsulates the entire record management pipeline. It is invoked asynchronously via `executeWorkflow` and does not block deliberation.

**Health Record Manager** — A routing agent that dispatches observations to domain specialist sub-agents without interpreting or analysing the query.

**Routing rules:**

| Observation Type | Routed To |
| --- | --- |
| Autonomic events, sensory phenomena, thermoregulation, neurogenic skin events, metabolic-neurological reactions | **Neurophysiology Specialist** |
| Sleep onset/maintenance, circadian rhythm, dreams, sleep paralysis, nocturnal events | **Sleep Specialist** |
| GI symptoms, digestive reactions, gut-related discomfort, appetite changes | **ENS/Gastro Specialist** |
| Mood, stress, beliefs, relationships, cognitive patterns, behavioural changes | **Psychology Specialist** |

**LLM configuration (Health Record Manager):**

- Primary: `deepseek/deepseek-v4-flash-0731`
- Fallback: `tencent/hy3-preview`

### 4.5 Domain Specialist Sub-Agents

Four **AgentTool** nodes serve as the Health Record Manager's tools. Each specialist:

1. **Reads** its clinical document from Google Docs (via a Get Tool)
2. **Analyses** the new observation against existing records
3. **Appends** a formatted entry to the document (via an Update Tool)

Each specialist uses standardised **change markers**:

| Marker | Meaning |
| --- | --- |
| ⬆ | IMPROVED |
| ⬇ | WORSENED |
| ⟳ | SHIFTED |
| ↺ | RECURRING |
| ✓ | RESOLVED |
| ⚔ | CONTRADICTS |

**Constraint:** Specialists never edit or delete existing entries — the record is strictly append-only.

**LLM configuration (all specialists):**

- Primary: `deepseek/deepseek-v4-flash-0731`
- Fallback: `tencent/hy3-preview`

---

## 5. Context Aggregation Pipeline (Get Personal Records)

### 5.1 Purpose

The **Get personal records** subworkflow (10 nodes) fetches all clinical and lifestyle documents, retrieves session state and conversation history, and returns a unified payload. It replaces the previous hardcoded document-ID approach with **dynamic Google Drive folder discovery**.

### 5.2 Dynamic Document Discovery

Instead of 11 hardcoded Google Docs fetch nodes, the subworkflow:

1. **Lists Records Folder** — Queries Google Drive for all Google Docs in the clinical records folder (`YOUR_RECORDS_FOLDER_ID`)
2. **Lists Protocols Folder** — Queries Google Drive for all Google Docs in the protocols folder (`YOUR_PROTOCOLS_FOLDER_ID`)
3. **Merge Folder Lists** — Combines both folder listings
4. **Fetch Document Content** — Fetches each document by dynamic ID
5. **Format Documents** — Sorts documents by known ID order or name-pattern ranking (neurophysiology, sleep, ENS, psychology, identity, social, protocols)

**Design benefit:** Adding a new clinical record or protocol document to the Google Drive folders automatically includes it in context assembly — no workflow modification required.

### 5.3 Parallel Session & History Fetch

Simultaneously with folder listing:

| # | Node Name | Source |
| --- | --- | --- |
| 1 | Get Current Session ID | PostgreSQL (`session_state`) |
| 2 | Get council messages from current session | PostgreSQL (`council_messages`, including `fact_check_feedback`) |

### 5.4 Multi-Stage Merge

```
List Records + List Protocols → Merge Folder Lists → Fetch → Format Documents
Get Session ID + Get Messages ──────────────────────────────→ Aggregate All Data (3-input merge)
                                                              │
                                                              ▼
                                                         Combine content1
                                                              │
                                                              ▼
                                                    (returned to main workflow)
```

### 5.5 Combine Content — The Unification Logic

The **Combine content1** Code node processes the merged stream:

```javascript
// 1. Life History — all Google Docs content joined
const lifeHistory = items
  .filter(item => item.json.documentId)
  .map(item => item.json.content)
  .join('\n\n');

// 2. Conversation History — with Epistemologist feedback appended per message
const conversation_history = items
  .filter(item => item.json.agent_name)
  .map(item => {
    let msgBlock = `[${item.json.agent_name} - Iteration ${item.json.iteration}]: ${item.json.content}`;
    if (item.json.fact_check_feedback) {
      msgBlock += `\n\n[Epistemologist Fact-Check on ${item.json.agent_name}]:\n${item.json.fact_check_feedback}`;
    }
    return msgBlock;
  })
  .join('\n\n--------------------------------------------------\n\n');

// 3. Session Variables
const sessionData = items.find(item => item.json.current_session_id)?.json || {};
```

**Output contract:**

```json
{
  "lifeHistory": "<all documents concatenated>",
  "session_id": "<current UUID>",
  "max_loops": "<configured iteration count>",
  "conversation_history": "<formatted prior discussion with fact-checks>"
}
```

### 5.6 Prepare Payload (Main Workflow)

The main workflow's **Prepare Payload** Set node structures the combined data for the deliberation engine:

- `user_query` — from Administrator's parsed output
- `life_context` — unified life history
- `session_id`, `max_loops`, `current_iteration` (initialised to `0`)
- `conversation_history` — prior discussion with fact-check feedback
- `chat_id` — Telegram chat ID for expert bot output

---

## 6. Council Deliberation Engine (Consult the Board)

The **Consult the board** subworkflow (32 nodes) encapsulates a single deliberation round: three parallel expert analyses, Telegram output, PostgreSQL storage, and Epistemologist fact-checking. The main orchestrator invokes it once per round (initial + loop instances of `executeWorkflow`).

### 6.1 Single-Instance Architecture with Subworkflow Looping

The previous monolithic design duplicated each expert agent physically to enable looping (n8n's prohibition on cyclic graphs). The refactored architecture solves this differently:

- **One physical instance** of each expert exists inside the `Consult the board` subworkflow
- The **main orchestrator** handles loop control: after each subworkflow invocation completes, it aggregates messages, increments iteration, and re-invokes the subworkflow if `current_iteration ≤ max_loops`
- Two `executeWorkflow` nodes in the main workflow (`Consult the board` and `Consult the board (Loop)`) provide the dual execution paths required by n8n's loop semantics

This preserves logical equivalence with the dual-instance design while reducing duplication and centralising deliberation logic in one subworkflow.

### 6.2 Deliberation Round Flow

```
Payload (subworkflow trigger)
    ├──→ Expert NeuroScientist  ──→ Insert neuro response  + Sanitize → Telegram
    ├──→ Expert Psychologist    ──→ Insert psych response  + Sanitize → Telegram
    └──→ Expert Physiologist    ──→ Insert physio response + Sanitize → Telegram
                                         │         │         │
                                         └────┬────┘─────────┘
                                              ▼
                                   Wait for Council Response (3-input merge)
                                              │
                                              ▼
                              Get council messages from current iteration
                                              │
                                              ▼
                                        Extract Claims (Code)
                                              │
                                              ▼
                                        Epistemologist Agent
                                              │
                                              ▼
                              Insert fact_check_feedback (PostgreSQL UPDATE)
```

**Parallel execution:** All three experts receive the same payload simultaneously. Each expert:

1. Receives the user query, full life context, conversation history (including prior fact-check feedback), and iteration number
2. May autonomously invoke domain-specific **Exa literature search** tools during analysis
3. Generates their analysis through their disciplinary lens
4. Their response is **simultaneously** sent to PostgreSQL and Telegram via their dedicated bot persona

### 6.3 Expert Literature Search Tools

Each council expert has a dedicated **toolWorkflow** invoking the **Exa search** subworkflow with a domain-specific specialty filter:

| Expert | Tool Name | Specialty Filter | Domain Allowlist |
| --- | --- | --- | --- |
| Neuroscientist | Neuro Literature Search | `neuro` | ncbi.nlm.nih.gov, nature.com, cell.com, sciencedirect.com |
| Psychologist | Psych Literature Search | `psych` | ncbi.nlm.nih.gov, nature.com, apa.org, sciencedirect.com |
| Physiologist | Physio Literature Search | `physio` | ncbi.nlm.nih.gov, nature.com, thelancet.com, sciencedirect.com |

Experts are instructed to search autonomously when they need recent literature to validate hypotheses or respond to Epistemologist critiques.

### 6.4 Epistemologist Fact-Checker

After all three experts complete, the **Epistemologist** agent:

1. **Extract Claims** — Formats the three expert responses into a structured review payload
2. **Searches literature** via two Semantic Scholar toolWorkflow nodes:
   - `Search_Literature_Abstracts` — broad abstract search for quick verification
   - `Read_Full_Paper` — full paper retrieval when abstracts are insufficient
3. **Outputs structured critique** — JSON mapping each expert name to their fact-check feedback
4. **Stores feedback** — Three PostgreSQL `UPDATE` statements write `fact_check_feedback` to the corresponding `council_messages` rows

**Structured output schema:**

```json
{
  "Neuroscientist": "Critique of neuroscientist claims...",
  "Psychologist": "Critique of psychologist claims...",
  "Physiologist": "Critique of physiologist claims..."
}
```

Fact-check feedback is included in conversation history for subsequent deliberation rounds, enabling experts to concede, defend, or refine disputed claims.

**LLM configuration:**

- Primary: `deepseek/deepseek-v4-flash-0731`
- Fallback: `tencent/hy3-preview`

### 6.5 Response Storage

Each expert's response is inserted into `council_messages`:

```sql
INSERT INTO council_messages (iteration, session_id, agent_name, content, created_at)
VALUES (<iteration>, '<session_id>', '<neuro|psych|physio>', '<response_text>', NOW());
```

Fact-check feedback is updated after the Epistemologist completes:

```sql
UPDATE council_messages
SET fact_check_feedback = '<critique>'
WHERE session_id = '<session_id>' AND iteration = <N> AND agent_name = '<agent>';
```

### 6.6 Main Orchestrator Loop Control

After the subworkflow returns, the main orchestrator:

1. **Get council messages** — Queries all messages for the session (including `fact_check_feedback`)
2. **Update payload** — Rebuilds conversation history with fact-check feedback, increments `current_iteration`
3. **If** node evaluates: `current_iteration ≤ max_loops`
   - **TRUE** → `Prepare Payload (Loop)` → `Consult the board (Loop)` subworkflow
   - **FALSE** → `Conversation Synthesizer` + `Academic Scout` (parallel)

### 6.7 Conversation History Format

```
[neuro - Iteration 0]: <neuroscientist's round 0 analysis>

[Epistemologist Fact-Check on neuro]:
<Critique of neuroscientist claims>

--------------------------------------------------

[psych - Iteration 0]: <psychologist's round 0 analysis>

[Epistemologist Fact-Check on psych]:
<Critique of psychologist claims>
...
```

---

## 7. Conversation Synthesis & Academic Scout

### 7.1 Conversation Synthesizer

When deliberation completes, the **Conversation Synthesizer** (`@n8n/n8n-nodes-langchain.agent` v3.1) in the main orchestrator transforms the validated multi-round discussion into a unified response.

**Purpose:** Synthesise expert discussion and Epistemologist-validated claims into a single, coherent answer.

**Output structure (adaptive):**

| Section | Description |
| --- | --- |
| **Direct Answer** | Lead with the answer. No preamble. |
| **Mechanistic & Evidence-Based Analysis** | Cross-domain cause-effect chains grounded in validated literature |
| **Where the Experts Converged** | Points of agreement with Epistemologist validation |
| **Where the Experts Differed or Claims Were Disputed** | Genuine disagreements and literature-forced shifts |
| **Recommendations** | Specific, mechanism-tied actionable advice when warranted |

**LLM configuration:**

- Primary: `deepseek/deepseek-v4-flash-0731`
- Fallback: `tencent/hy3-preview`

### 7.2 Academic Scout

Runs in parallel with the Conversation Synthesizer when deliberation completes.

The **Academic Scout** agent queries a **Postgres PGVector Store** (`scientific_literature` table) as a retrieval tool, finding the single most relevant recent paper for the user's query. It delivers a concise, patient-directed recommendation via the **AcademicScout_Bot** Telegram persona.

**Critical instruction:** The Scout must state the exact paper title, authors, and publication year so the patient can search for the full study.

**LLM configuration:**

- Primary: `deepseek/deepseek-v4-flash-0731`
- Fallback: `tencent/hy3-preview`

### 7.3 Synthesis & Scout Output Paths

```
Conversation Synthesizer ──→ Sanitize Node7 ──→ Send synthesis (Synthesizer_Bot)
Academic Scout ────────────→ Sanitize Node ───→ Send recommendation (AcademicScout_Bot)
```

### 7.4 Error Handling Strategy

| Error Source | Handler | Bot Persona |
| --- | --- | --- |
| Administrator agent failure | Send error message1 | Main_Bot |
| Conversation Synthesizer failure | Send error message | Main_Bot |
| Health Record Manager failure | Send error message2 (in subworkflow) | Main_Bot |

---

## 8. RAG Literature Pipeline

### 8.1 Get Latest Scientific Papers Subworkflow

The **Get latest scientific papers** subworkflow (13 nodes) runs on a **weekly schedule**, independently of user queries. It maintains the curated literature database that powers Academic Scout recommendations.

**Flow:**

1. **Parallel Semantic Scholar search** — Three HTTP requests fetch papers from the past 7 days in neuroscience, physiology, and psychology (limit: 100 per domain)
2. **Transform** — Flattens API responses, filters papers without abstracts, extracts title, abstract, authors, year, source URL
3. **Check if Paper Exists** — PostgreSQL query prevents duplicate ingestion
4. **If** — Skips papers already in the vector store
5. **Restore Original Item** — Preserves metadata through the embedding pipeline
6. **Recursive Character Text Splitter** — Chunks abstracts for embedding
7. **Default Data Loader** — Attaches title and URL metadata
8. **Google Gemini Embeddings** — `models/gemini-embedding-2`
9. **Postgres PGVector Store** — Inserts into `scientific_literature` table

### 8.2 Vector Store Schema

| Column | Type | Description |
| --- | --- | --- |
| `content` | `text` | Chunked abstract text |
| `metadata` | `jsonb` | Paper title and source URL |
| `embedding` | `vector` | Gemini embedding vector |

### 8.3 Literature Search Subworkflows

Two reusable search subworkflows are invoked as agent tools:

**Semantic Scholar search** (5 nodes):

- `action_type: abstract` — Paper search with title, abstract, authors, year, URL
- `action_type: full_paper` — Full paper retrieval by PaperId

**Exa search** (4 nodes):

- Domain-filtered semantic web search via Exa API
- Specialty parameter selects domain allowlist (neuro, physio, psych)
- Returns highlights-preferred results with title, URL, and content snippets

---

## 9. Data Model & State Management

### 9.1 PostgreSQL Schema

#### Table: `session_state`

| Column | Type | Description |
| --- | --- | --- |
| `chat_id` | `string` | Telegram chat ID (match key for upserts) |
| `current_session_id` | `string` | UUID of the active session |
| `max_loops` | `integer` | Deliberation rounds configured (default: 1) |
| `updated_at` | `timestamp` | Last modification timestamp |

#### Table: `council_messages`

| Column | Type | Description |
| --- | --- | --- |
| `id` | `string` | Auto-generated primary key |
| `session_id` | `string` | UUID linking to the active session |
| `iteration` | `integer` | Deliberation round number (0-indexed) |
| `agent_name` | `string` | `"neuro"`, `"psych"`, or `"physio"` |
| `content` | `text` | Full text of the expert's response |
| `fact_check_feedback` | `text` | Epistemologist critique (nullable) |
| `created_at` | `timestamp` | When the response was generated |

#### Table: `scientific_literature` (PGVector)

Stores embedded paper abstracts for Academic Scout RAG retrieval. Populated by the weekly ingestion subworkflow.

### 9.2 Session Lifecycle

```
/new command → Generate UUID → Upsert session_state
User sends query → Get personal records → SELECT current_session_id
Consult the board (per round) → INSERT council_messages + UPDATE fact_check_feedback
Loop iterations → SELECT from council_messages (builds history with fact-checks)
Session complete → Conversation Synthesizer + Academic Scout
New /new command → starts fresh session (previous data persists)
```

### 9.3 Google Docs as Long-Term Memory

Clinical records are discovered dynamically via Google Drive folder listing. Unlike PostgreSQL tables (ephemeral deliberation state), Google Docs accumulate over time:

| Document Category | Mutability | Purpose |
| --- | --- | --- |
| Clinical Records (dynamic) | Append-only | Longitudinal health observations |
| Protocols & Profiles (dynamic) | Read-only (by this system) | Reference context for identity, lifestyle, protocols |

---

## 10. AI Agent Specifications

### 10.1 Agent Taxonomy

| Tier | Agent | Type | Purpose |
| --- | --- | --- | --- |
| **Orchestration** | Administrator | `agent` v3.1 | Query screening, update consent parsing |
| **Orchestration** | Health Record Manager | `agent` v3.1 | Observation routing to domain specialists |
| **Council** | Expert Neuroscientist | `agent` v3.1 | Neural hardware analysis + literature search |
| **Council** | Expert Psychologist | `agent` v3.1 | Psychological software analysis + literature search |
| **Council** | Expert Physiologist | `agent` v3.1 | Body systems analysis + literature search |
| **Council** | Epistemologist | `agent` v3.1 | Claim extraction, literature verification, critique |
| **Synthesis** | Conversation Synthesizer | `agent` v3.1 | Multi-expert validated discussion distillation |
| **Literature** | Academic Scout | `agent` v3.1 | RAG-based paper recommendation |
| **Record Mgmt** | Neurophysiology Specialist | `agentTool` v3 | Neurophysiology record maintenance |
| **Record Mgmt** | Sleep Specialist | `agentTool` v3 | Sleep record maintenance |
| **Record Mgmt** | ENS Specialist | `agentTool` v3 | ENS/Gastro record maintenance |
| **Record Mgmt** | Psychology Specialist | `agentTool` v3 | Psychology record maintenance |

### 10.2 The Three-Layer Model

The expert triumvirate operates on a deliberate **three-layer ontology**:

```
┌─────────────────────────────────────┐
│        PSYCHOLOGIST                 │
│        (Software Layer)             │
├─────────────────────────────────────┤
│        NEUROSCIENTIST               │
│        (Hardware Layer)             │
├─────────────────────────────────────┤
│        PHYSIOLOGIST                 │
│        (Body Layer)                 │
└─────────────────────────────────────┘
```

The **Epistemologist** operates across all three layers, verifying claims regardless of domain.

### 10.3 Prompt Management

System prompts for all agents are maintained as external Markdown files in `prompts/`, organised by agent:

```
prompts/
├── Administrator/
├── council/NeuroScientist, Psychologist, Physiologist/
├── Epistemologist/
├── Conversation Synthesiser/
├── Academic Scout/
├── Health Record Manager/
└── specialists/Neurophysiology, Sleep, ENS, Psychology/
```

This separation enables prompt iteration without modifying workflow JSON exports.

---

## 11. Telegram Output & Message Sanitization

### 11.1 Multi-Bot Persona Architecture

| Bot Persona | Credential Name | Role |
| --- | --- | --- |
| **Main_Bot** | Main_Bot | Main system bot — trigger, confirmations, errors |
| **Neuroscientist_Bot** | Neuroscientist_Bot | Neuroscientist expert output |
| **Psychologist_Bot** | Psychologist_Bot | Psychologist expert output |
| **Physiologist_Bot** | Physiologist_Bot | Physiologist expert output |
| **Synthesizer_Bot** | Synthesizer_Bot | Final synthesis output |
| **AcademicScout_Bot** | AcademicScout_Bot | Literature recommendation output |

### 11.2 Message Sanitization Pipeline

All AI agent output passes through sanitization Code nodes before Telegram delivery:

1. **Intelligent chunking** — Split at `###` headings, `═══` dividers, or paragraph breaks (max 3,800 chars)
2. **MarkdownV2 escaping** — Protect code blocks → escape special characters → restore formatting
3. **Multi-part output** — Each chunk returned as a separate item with `partIndex`

---

## 12. LLM Strategy & Credential Architecture

### 12.1 OpenRouter Model Strategy

All agents use **OpenRouter** with a consistent dual-model fallback pattern:

| Role | Primary Model | Fallback Model |
| --- | --- | --- |
| All agents (Administrator, experts, Epistemologist, Synthesizer, Scout, specialists) | `deepseek/deepseek-v4-flash-0731` | `tencent/hy3-preview` |

24 OpenRouter Chat Model nodes are distributed across workflows, each configured as primary (index 0) or fallback (index 1) for their parent agent.

**Embeddings** use Google Gemini (`models/gemini-embedding-2`) via the Google PaLM API credential — used exclusively for PGVector ingestion and retrieval.

### 12.2 Credential Inventory

| Credential | Type | Purpose |
| --- | --- | --- |
| Main_Bot | Telegram Bot API | Main bot (trigger, confirmations, errors) |
| Neuroscientist_Bot | Telegram Bot API | Neuroscientist output |
| Psychologist_Bot | Telegram Bot API | Psychologist output |
| Physiologist_Bot | Telegram Bot API | Physiologist output |
| Synthesizer_Bot | Telegram Bot API | Synthesis output |
| AcademicScout_Bot | Telegram Bot API | Literature recommendation |
| Postgres account | PostgreSQL | All DB operations + PGVector |
| OpenRouter account | OpenRouter API | All LLM inference |
| Google Gemini(PaLM) Api account | Google PaLM API | Embeddings only |
| Google Docs account | Google OAuth2 | Document read/write |
| Google Drive account | Google OAuth2 | Folder listing |
| Exa account | HTTP Header Auth | Exa literature search |

### 12.3 External Service Dependencies

| Service | Purpose | Failure Impact |
| --- | --- | --- |
| **OpenRouter API** | All AI reasoning | Total system failure |
| **Google Docs/Drive API** | Life history storage and retrieval | Context degradation |
| **PostgreSQL** | Session state, messages, vector store | Loop logic and RAG failure |
| **Telegram Bot API** | User interface | Output failure (computation continues) |
| **Semantic Scholar API** | Epistemologist and ingestion pipeline | Fact-checking degradation |
| **Exa API** | Expert literature search | Expert search tool unavailable |
| **Google Gemini Embeddings** | Vector store ingestion/retrieval | Academic Scout unavailable |

---

## 13. Complete Node Inventory

### Main Orchestrator — Board of Councillors (42 nodes)

| Category | Nodes |
| --- | --- |
| Trigger & Routing | Telegram Trigger, Command Router, Update allowed, If |
| Session Management | Generate New Session ID, Update Session Pointer, Update Max Loops, Clear conversation history |
| AI Agents | Administrator, Conversation Synthesizer, Academic Scout, Structured Output Parser1 |
| Subworkflow Invocations | Consult the board, Consult the board (Loop), Get personal records, Update personal records |
| Loop Control | Prepare Payload, Prepare Payload (Loop), Get council messages (×2), Update payload (×2) |
| LLM Instances | OpenRouter Chat Model (×6) |
| Vector Store | Postgres PGVector Store1, Embeddings Google Gemini |
| Telegram Output | Send confirmation (×4), Send error (×2), Send synthesis, Send recommendation |
| Sanitization | Sanitize Node, Sanitize Node7 |
| Maintenance | Schedule Trigger, Execute a SQL query |

### Consult the Board Subworkflow (32 nodes)

| Category | Nodes |
| --- | --- |
| Trigger | Payload (executeWorkflowTrigger) |
| Council Experts | Expert NeuroScientist, Expert Psychologist, Expert Physiologist |
| Fact-Checking | Epistemologist, Extract Claims, Structured Output Parser2, Insert fact check feedback |
| Literature Tools | Neuro/Physio/Psych Literature Search, Semantic Scholar search (abstract + full paper) |
| PostgreSQL | Insert neuro/psych/physio response, Get council messages from current iteration |
| Synchronisation | Wait for Council Response |
| Telegram Output | Sanitize Node1–3, Send a text message7/12/13 |
| LLM Instances | OpenRouter Chat Model (×8) |

### Other Subworkflows

| Subworkflow | Nodes | Key Components |
| --- | --- | --- |
| Get personal records | 10 | Google Drive listing, document fetch, session/history merge |
| Update personal records | 25 | Health Record Manager, 4 specialists, 8 Google Docs tools |
| Get latest scientific papers | 13 | Semantic Scholar (×3), embedding pipeline, PGVector insert |
| Semantic Scholar search | 5 | Abstract and full-paper search with Switch routing |
| Exa search | 4 | Domain-filtered Exa API search |

---

## 14. Design Decisions & Rationale

### 14.1 Why Subworkflows Instead of a Monolithic Workflow?

The original 108-node monolithic design was decomposed into 7 workflows (131 nodes total) for:

- **Maintainability** — Each capability (deliberation, context assembly, record updates, literature ingestion) can be modified independently
- **Reusability** — Literature search subworkflows are shared as agent tools across Epistemologist and council experts
- **Testability** — Subworkflows can be invoked and tested in isolation
- **Extensibility** — New capabilities (additional experts, new literature sources) can be added as new subworkflows without restructuring the orchestrator

### 14.2 Why Subworkflow Looping Instead of Duplicate Expert Nodes?

The dual-instance expert architecture (two physical copies per expert for loop control) was replaced by subworkflow re-invocation. The main orchestrator calls `Consult the board` once per round via separate `executeWorkflow` nodes. This eliminates 6 duplicate agent nodes while preserving identical loop semantics.

### 14.3 Why Dynamic Google Drive Discovery?

Hardcoded document IDs required workflow updates whenever documents were added or reorganised. Folder-based discovery via Google Drive API means new clinical records or protocols are automatically included in context assembly when added to the designated folders.

### 14.4 Why an Epistemologist Fact-Checker?

Expert agents can generate plausible but unsubstantiated mechanistic claims. The Epistemologist provides a structured verification layer:

- Claims are extracted and searched against Semantic Scholar
- Feedback is stored per-expert per-iteration in PostgreSQL
- Subsequent rounds include fact-check critiques in conversation history
- The Conversation Synthesizer weights validated claims over disputed ones

This transforms deliberation from "three experts agreeing" into "three experts whose claims are independently verified."

### 14.5 Why RAG for Academic Scout Instead of Live Search?

The Academic Scout queries a **curated, pre-embedded** literature database rather than searching live APIs during every query. This provides:

- **Latency** — Vector retrieval is faster than API search during the critical post-deliberation window
- **Quality** — The ingestion pipeline filters for papers with abstracts from reputable domains
- **Consistency** — Recommendations come from a known, auditable corpus updated weekly

### 14.6 Why OpenRouter Instead of Direct Gemini API?

OpenRouter provides model-agnostic access with automatic fallback routing. The `deepseek/deepseek-v4-flash-0731` + `tencent/hy3-preview` pairing offers strong reasoning at lower cost than direct Gemini Pro, with a distinct fallback model for resilience against single-provider outages.

### 14.7 Why Google Docs Instead of a Database for Clinical Records?

- **Human readability** — records can be reviewed, edited, and shared outside the system
- **Rich formatting** — Markdown-style entries with change markers
- **Natural append model** — Google Docs append maps to append-only longitudinal records
- **Portability** — records exist independently of the automation system

### 14.8 Why Separate Telegram Bots?

Distinct bot personas create an **observable discussion** — the user sees different participants contributing at different times, making the multi-agent architecture tangible without requiring technical understanding.

### 14.9 Why Store Messages in PostgreSQL?

Storing expert responses in `council_messages` rather than passing them directly between iterations provides persistence, queryability, decoupling from n8n's internal data passing, and auditability of complete deliberation records including fact-check feedback.

---

## 15. Public Repository Structure

```
github/
├── json/
│   ├── workflow/Board_of_councillors.json      # Main orchestrator (sanitized export)
│   └── subworkflows/                           # Subworkflow exports
│       ├── Consult_the_board.json
│       ├── Get_personal_records.json
│       ├── Update_personal_records.json
│       ├── Get_latest_scientific_papers.json
│       ├── Semantic_Scholar_search.json
│       └── Exa_search.json
├── prompts/                                    # Agent prompts (generalized)
└── docs/BOARD_OF_COUNCILLORS_TECHNICAL_DESIGN.md # This document
```

---

_Public portfolio edition — 12 August 2026. Sanitized workflow exports preserve node structure; configure credentials, API keys, folder IDs, and subworkflow references in your own n8n instance before deployment._

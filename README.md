# Board of Councillors

A multi-agent iterative deliberation system built on **n8n**. Users submit health-related queries via Telegram and observe a structured, multi-round discussion between three AI expert agents — a **Neuroscientist**, a **Psychologist**, and a **Physiologist** — with an **Epistemologist Fact-Checker** verifying claims against scientific literature after each round. A **Conversation Synthesizer** distils the discussion into a unified response, and an **Academic Scout** surfaces a relevant recent paper from a curated literature database.

The system is organized around a medical-council metaphor: distinct specialist perspectives deliberate in dialogue (not in isolation), grounded in longitudinal health records and reference protocols, with observable output via separate Telegram bot personas per expert.

This repository contains a **sanitized export** of the production implementation: workflow JSON, agent prompts, fictional demo records, and technical design documentation. Credential values, personal data, and deployment-specific identifiers have been removed.

## Repository contents

| Path | Description |
| --- | --- |
| `json/workflow/` | Main orchestrator — Telegram routing, loop control, synthesis, Academic Scout |
| `json/subworkflows/` | Council deliberation, personal records, literature search, RAG ingestion |
| `prompts/` | System and user prompts for all agents and domain specialists |
| `records/` | Fictional demo health records (static protocols and dynamic clinical logs) |
| `docs/` | Technical design specification |
| `DEMO_QUESTIONS.md` | Example queries that exercise the full multi-agent pipeline |

## System architecture

The implementation uses a **modular subworkflow design** (131 nodes across 7 workflows):

| Component | Role |
| --- | --- |
| **Board of Councillors** (main) | Command routing, session management, deliberation loop, synthesis, literature recommendation |
| **Consult the board** | Parallel expert analysis, Telegram output, Epistemologist fact-checking per round |
| **Get personal records** | Dynamic Google Drive discovery, session state, conversation history assembly |
| **Update personal records** | Health Record Manager routing to four domain specialist sub-agents |
| **Get latest scientific papers** | Scheduled Semantic Scholar ingestion into PGVector |
| **Semantic Scholar search** | Abstract and full-paper retrieval (agent tool) |
| **Exa search** | Domain-filtered web literature search (agent tool) |

### Technology stack

- **Orchestration:** n8n (self-hosted)
- **LLM inference:** OpenRouter (`deepseek/deepseek-v4-flash-0731` primary, `tencent/hy3-preview` fallback)
- **Embeddings:** Google Gemini (`models/gemini-embedding-2`)
- **Database:** PostgreSQL (session state, council messages, PGVector literature store)
- **Documents:** Google Docs + Google Drive (longitudinal records and protocols)
- **Literature:** Semantic Scholar API, Exa API
- **Interface:** Telegram Bot API (six bot personas)

### Execution flow

1. **Ingestion** — Telegram trigger, command routing, Administrator screening
2. **Context assembly** — Parallel document fetch, session and history merge
3. **Deliberation** — Three experts in parallel; Epistemologist verifies claims after each round
4. **Synthesis** — Unified response from validated multi-round discussion
5. **Literature recommendation** — Academic Scout RAG retrieval (runs in parallel with synthesis)
6. **Record update** — Asynchronous routing to domain specialists (when permitted)

Deliberation depth is configurable via the `/setloops` command (default: one round).

## Three-layer expert model

| Layer | Expert | Analytical focus |
| --- | --- | --- |
| Hardware | Neuroscientist | Neural circuits, neurotransmitters, sensory processing, autonomic wiring |
| Software | Psychologist | Cognition, emotion, behaviour, coping, stress appraisal |
| Body | Physiologist | Metabolism, endocrine function, cardiovascular and immune physiology |

The Epistemologist operates across all three layers, searching literature to validate or dispute expert claims. Fact-check feedback is stored per expert per iteration and included in subsequent deliberation rounds.

## Workflow exports

JSON files in this repository preserve **node structure, connections, and configuration patterns**. They do not include live credentials, API keys, or instance-specific workflow IDs. Importing into n8n requires re-linking subworkflows, configuring credentials, and supplying Google Drive folder and document identifiers.

Demo records in `records/` describe a fictional patient profile (Alex M.) designed to ground council deliberation without using real personal health data.

## Documentation

Full technical specification: [`docs/BOARD_OF_COUNCILLORS_TECHNICAL_DESIGN.md`](docs/BOARD_OF_COUNCILLORS_TECHNICAL_DESIGN.md)

Demo record structure and fictional profile: [`records/README.md`](records/README.md)

Example queries: [`DEMO_QUESTIONS.md`](DEMO_QUESTIONS.md)

## Security note

This export is intended for public portfolio and educational use. Workflow JSON contains no credentials; fictional records contain no real patient data. Anyone deploying from this export should configure secrets via n8n credentials or environment variables and must not commit live API keys or personal health information.

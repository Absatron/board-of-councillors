# Board of Councillors

A production-grade multi-agent health deliberation system built on **n8n**. A user sends a health query via Telegram; three expert agents (Neuroscientist, Psychologist, Physiologist) deliberate across configurable rounds with an **Epistemologist Fact-Checker**, then a **Conversation Synthesizer** and **Academic Scout** deliver a unified answer and a relevant recent paper recommendation.

This repository is a **sanitized public portfolio edition** — workflow structure, prompts, demo records, and technical design — without credentials, personal data, or deployment-specific identifiers.

## What’s in this repo

| Path                 | Contents                                                            |
| -------------------- | ------------------------------------------------------------------- |
| `json/workflow/`     | Main orchestrator export (`Board_of_councillors.json`)              |
| `json/subworkflows/` | Deliberation, records, literature search, RAG ingestion             |
| `prompts/`           | Agent system and user prompts                                       |
| `records/`           | Fictional demo health records (upload to Google Docs for live demo) |
| `docs/`              | Technical design document                                           |
| `DEMO_QUESTIONS.md`  | Suggested queries for a video demo                                  |

## Architecture (summary)

- **Main orchestrator** — Telegram commands, loop control, synthesis, Academic Scout
- **Consult the board** — Three experts + Epistemologist per deliberation round
- **Get / Update personal records** — Google Drive document discovery and specialist updates
- **Literature pipeline** — Semantic Scholar ingestion → PGVector; Exa + Semantic Scholar as agent tools

Full specification: [`docs/BOARD_OF_COUNCILLORS_TECHNICAL_DESIGN.md`](docs/BOARD_OF_COUNCILLORS_TECHNICAL_DESIGN.md)

## Video demo (recommended flow)

1. **Upload demo records** — Copy content from `records/static/` and `records/dynamic/` into Google Docs in two Drive folders. See [`records/README.md`](records/README.md).
2. **Configure n8n** — Import workflow JSON, attach credentials (Telegram, OpenRouter, Postgres, Google, Exa, Semantic Scholar). Set Drive folder IDs in `Get personal records` and document IDs in `Update personal records`.
3. **Prepare session** — In Telegram: `/new` then `/setloops 2`.
4. **Ask one question** — Use the recommended query in [`DEMO_QUESTIONS.md`](DEMO_QUESTIONS.md).
5. **Record** — Main bot confirmation → three expert bots (2 rounds) → Synthesizer bot → Academic Scout bot.

## Deploying from this export

These JSON files preserve **node structure and connections** but omit live credentials and instance-specific workflow IDs. After import:

1. Re-link subworkflow `executeWorkflow` nodes to your imported subworkflows.
2. Configure all credentials in the n8n UI.
3. Replace `YOUR_FOLDER_ID` / `YOUR_DOCUMENT_ID` placeholders with your Google Drive/Docs IDs.
4. Set Semantic Scholar `x-api-key` headers via credentials or environment variables — values in this export are empty.

## Publishing this folder as a GitHub repository

Run these commands from **inside this `github` folder** (not the parent project directory):

```bash
cd /path/to/BoC/Code/github

# Initialize a new repository
git init

# Optional: verify what will be committed
git status

# Stage all portfolio files
git add .

# First commit
git commit -m "Add sanitized Board of Councillors portfolio export"

# Rename branch to main (if needed)
git branch -M main
```

Create an empty repository on GitHub (e.g. `board-of-councillors`) — **do not** add a README, .gitignore, or license if you already have them locally.

```bash
# Replace with your GitHub username and repo name
git remote add origin https://github.com/YOUR_USERNAME/board-of-councillors.git

# Push
git push -u origin main
```

If you use SSH:

```bash
git remote add origin git@github.com:YOUR_USERNAME/board-of-councillors.git
git push -u origin main
```

### If the parent folder is already a git repository

If `github/` lives inside a larger repo you do **not** want to publish, either:

- Push **only** this folder as its own repo (steps above from inside `github/`), or
- Use a **subtree** / separate repo — avoid committing the parent `.env` or unsanitized `json/` from the development project.

## Security

- Never commit `.env`, API keys, or unsanitized workflow exports with `pinData`.
- Rotate any credentials that were ever embedded in local workflow JSON before going public.
- Demo records are **fictional** — do not substitute real patient data in a public repo.

## License

Add a license file if you intend open distribution (e.g. MIT). This export does not include one by default.

> > > > > > > 8db7afb (Board of councillors demo)

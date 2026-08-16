# Demo Health Records

Fictional longitudinal records for **Alex M.** — a generic demo profile designed to ground the Board of Councillors workflow without using real personal data.

## Profile summary

| Attribute | Value |
| --- | --- |
| Age | 34 |
| Occupation | Software engineer (remote/hybrid) |
| Primary concern | Sleep maintenance insomnia + daytime fatigue |
| Lifestyle | Desk work, moderate exercise, 2–3 coffees/day, irregular lunch timing |

This profile reflects common patterns seen in working adults: stress-related sleep disruption, caffeine use, sedentary work, and mild digestive sensitivity — not a specific individual.

## Folder layout

```
records/
├── dynamic/          # Append-only clinical records (updated by the workflow)
│   ├── Neurophysiology_Record.md
│   ├── Sleep_Record.md
│   ├── ENS_Gastro_Record.md
│   └── Psychology_Record.md
└── static/           # Reference protocols & profiles (read-only in the workflow)
    ├── Identity_Record.md
    ├── Social_Profile.md
    ├── Sleep_Protocol.md
    ├── Exercise_Protocol.md
    ├── Balanced_Diet_Protocol.md
    └── Skin_Protocol.md
```

## Using these for a live demo

1. Create two Google Drive folders (e.g. `BoC Demo Records` and `BoC Demo Protocols`).
2. Upload each Markdown file as a **Google Doc** (copy/paste content, or import).
3. Point the `Get personal records` and `Update personal records` subworkflows at those folder IDs.
4. Run `/setloops 2` in Telegram before your demo query so the council deliberates for two rounds.

The workflow reads documents dynamically from Drive — these files are the **source content** to upload, not files the workflow reads directly from this repository.

# Thinktank

A collection of independent Thinktank sessions. Each session is an isolated experiment in thinking.

## How this repo works

Every session lives in its own PascalCase folder under the repo root. For the rules that govern how sessions are created, named, and kept isolated, see [`.copilot-instructions.md`](./.copilot-instructions.md).

### Session structure

```
<TopicSlug>/
├── ReadMe.md    # session overview, links to Summary.md (required)
├── Summary.md   # distilled, always-up-to-date summary (required, single source of truth)
├── Thoughts.md  # raw, chronological notes (optional)
└── Decisions.md # notable forks / dropped ideas (optional)
```

- **Folder name** — PascalCase and specific (e.g. `FolderIdeas`). If a folder with that name already exists, make it more precise rather than adding a timestamp (e.g. `FolderIdeasBackend`).
- **Date** — recorded inside `ReadMe.md`, not in the folder name.
- **Summary.md** — the single source of truth for what the session concluded. Keep it up to date.

## Sessions

- [`DualPhaseCoding`](./DualPhaseCoding/) — existing session.
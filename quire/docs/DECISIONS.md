# Decisions

Each entry records one decision and why. Agents add entries here when a task says "record" or when they choose something the spec leaves open. Use the next free number. Never edit an accepted entry; supersede it with a new one.

Format: **D-NNN Title** (date, status). Status is `accepted`, `proposed` (waiting for the owner) or `superseded by D-NNN`.

---

**D-001 The agent returns a selection; Quire saves it** (2026-11-16, accepted)
Spawned runs return the selection as structured output (`--json-schema`), and the GUI process validates, renders and saves it. Interactive Claude Code sessions use the `submit_draft` tool. Both paths share one pipeline (PLAN T3.3). Reason: the spawned agent has no write path, so a hostile job description cannot make it save anything.

**D-002 Select-only bullets in v1** (2026-11-16, accepted)
The agent chooses and orders highlights but cannot change their text. Every bullet prints exactly as stored. Rewording with a review step is Phase 2. Reason: removes the main hallucination risk and the review burden for v1.

**D-003 One profile** (2026-11-16, accepted)
A single profile row. The summary is written per application by the agent.

**D-004 Page target per template** (2026-11-16, accepted)
`templates.page_target`, default 1. Exceeding it is a warning, not an error.

**D-005 PDF bytes stored in the database** (2026-11-16, accepted)
`artifacts.pdf_bytes` is the source of truth. A copy is exported to a user-chosen folder on finalise. Reason: the record of what was sent survives a moved or deleted folder. One-page PDFs are tens of kilobytes.

**D-006 Intake: paste first, Jina off by default** (2026-11-16, accepted)
Paste is primary; local fetch second; Jina Reader is an opt-in backend. Reason: keeps the app local unless the user chooses otherwise.

**D-007 Email is draft-only** (2026-11-16, accepted)
Quire shows the subject and body with a copy button and a `mailto:` link. It never sends. Reason: sending needs mail credentials and OAuth, out of proportion for v1.

**D-008 Claude is the only runner in v1** (2026-11-16, accepted)
`agy` is deferred. The `AgentRunner` trait keeps room for it. Open follow-up: check what `agy` supports for non-interactive runs and MCP configuration.

**D-009 Name is Quire** (2026-11-16, accepted)
"Loom" is dropped; it is an existing product name.

**D-010 One `entries` table** (2026-11-16, accepted)
Experience, project, education and certification share one table with a `kind` column. Reason: removes the polymorphic `entity_id` of the original design, so tags get real foreign keys.

**D-011 Highlights are rows** (2026-11-16, accepted)
Each bullet is a row with its own id and tags, instead of a JSON array in a text column. Reason: bullets must be referenced, ranked and validated individually.

**D-012 Build order: core, CLI, MCP, agent, then GUI** (2026-11-16, accepted)
Reason: every risky piece is testable without a GUI, and real use of the CLI informs the GUI.

**D-013 Data reaches Typst as JSON input** (2026-11-16, accepted)
The template reads `sys.inputs.cv_json`. No text is ever interpolated into Typst source. Reason: Typst markup characters in user text would break compiles or run as code.

**D-014 Highlights render in the agent's order; entries in date order** (2026-11-16, accepted)
Reason: highlight order is the agent's relevance ranking; entry order follows CV convention.

**D-015 `--bare` is off by default** (2026-11-16, accepted)
`--bare` requires an API key and ignores subscription login. Runs instead use an empty temporary directory and `--strict-mcp-config`. Users with an API key can turn `agent.use_bare` on.

**D-016 Pinned dependency versions** (pending, PLAN T0.2)
Known at planning time: Dioxus 0.7.9 stable (0.8 is in alpha with breaking changes), rmcp 3.x, typst-as-lib 0.15.x. Fill in exact versions at T0.2.

**D-017 Embedded fonts** (pending, PLAN T2.1)

**D-018 Typst JSON input call verified** (pending, PLAN T2.3)

**D-019 Claude Code version and flags verified** (pending, PLAN T5.4)

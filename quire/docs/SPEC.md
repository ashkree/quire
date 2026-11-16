# Quire Specification (v1)

This file is the source of truth for what Quire does. `docs/PLAN.md` says in what order to build it, and `docs/DECISIONS.md` records why.

**Conventions**

- MUST, MUST NOT and SHOULD carry their usual meaning. Anything not stated is undecided: ask, don't guess (see `AGENTS.md`).
- Every requirement has a stable ID such as `VAL-3`. IDs are never renumbered or reused. A dropped requirement is marked *withdrawn*, not deleted.
- Code, tests and commit messages cite requirement IDs.

---

## 1. Purpose and scope

Quire is a local desktop application. It keeps a career vault and produces a tailored CV (PDF) and an outreach email draft for a specific job application. An LLM agent chooses the content. Deterministic Rust code validates and renders it.

**In scope for v1**

- Vault editing: profile, links, entries (experience, project, education, certification), highlights, tags with weights, skills.
- Import and export of the whole vault as a TOML file.
- Applications: a job description captured by paste or URL fetch, plus a status.
- Agent runs through the Claude Code CLI (`claude -p`) with Quire's MCP server.
- Validation, Typst rendering, draft review, finalising.
- An MCP server for interactive use from Claude Code.
- Platforms: Arch Linux (x86_64) and macOS (Apple silicon).

**Out of scope for v1.** Do not build any of these, even partially:

- Agent rewording of bullets. Bullets print exactly as stored (D-002).
- Multiple profiles (D-003).
- Sending email (D-007).
- Runners other than Claude, including `agy` (D-008).
- Job board monitoring and auto-apply (Phase 2).
- Windows, cloud sync, multiple users, accounts.

---

## 2. Glossary

| Term | Meaning |
| --- | --- |
| Vault | Everything the user has recorded about themselves: profile, links, entries, highlights, tags, skills |
| Entry | One experience, project, education item or certification |
| Highlight | One bullet belonging to one entry |
| Tag | A normalised domain label, such as `risk-management` |
| Weight | 1 to 10, how strongly an entry or highlight fits a tag. Set by the user |
| Application | One job the user is applying for, with its job description text |
| Selection | The agent's choice of content for one application (section 7.1) |
| Resolved CV | The selection with ids replaced by stored text, ready to render (section 8.2) |
| Artifact | One rendered result for an application. State `draft` or `final` |
| Template | Typst source that renders a resolved CV |
| Runner | Code that drives an agent CLI to produce a selection |

---

## 3. Invariants

These hold at all times. A change that breaks one is wrong even if every test passes.

| ID | Invariant |
| --- | --- |
| INV-1 | The agent never creates or changes a `final` artifact. Finalising is a user action in the GUI or CLI |
| INV-2 | Every bullet in a rendered CV is the exact `body` of a stored, non-archived highlight |
| INV-3 | No user, vault or agent text is ever placed into Typst source. It reaches the template only as data (RENDER-2) |
| INV-4 | In `--mcp` mode, stdout carries MCP protocol messages only. All logs go to stderr |
| INV-5 | A `final` artifact cannot be updated or deleted. The database enforces this (DB-3) |
| INV-6 | `quire-core` depends on no UI, MCP or process-spawning crate |
| INV-7 | Rendering uses only fonts embedded in the binary. No system font lookup |
| INV-8 | The schema changes only through a new migration file. Applied migrations are never edited |
| INV-9 | No network access except the agent CLI the user installed, and URL fetch the user triggers |

---

## 4. Architecture

### 4.1 Crates

```text
crates/
  quire-core/    models, database, vault import/export, validation, resolve, render
  quire-mcp/     MCP server over quire-core (rmcp, stdio)
  quire-agent/   AgentRunner trait and the Claude runner
  quire-app/     the `quire` binary: CLI routing and the Dioxus GUI
```

Allowed dependencies between crates:

```text
quire-app  -> quire-core, quire-mcp, quire-agent
quire-mcp  -> quire-core
quire-agent -> quire-core
quire-core -> (no workspace crates)
```

`quire-app` is the only crate that depends on Dioxus.

### 4.2 Processes

Up to three processes are alive during an agent run. They share nothing but the database file.

```text
quire (GUI)  --spawns-->  claude -p  --spawns-->  quire --mcp
     |                                                |
     +------------- quire.db (SQLite, WAL) -----------+
```

- ARCH-1. Every connection sets `journal_mode=WAL`, `foreign_keys=ON` and `busy_timeout=5000`.
- ARCH-2. The GUI notices commits from other processes by polling `PRAGMA data_version` once per second while a view is open.
- ARCH-3. `quire --mcp` enters the MCP server before any GUI or window-system code runs.

### 4.3 Data and config locations

- ARCH-4. The data directory is the platform data directory from the `directories` crate, application name `quire`. It holds `quire.db` and `exports/`.
- ARCH-5. The config directory is the platform config directory, same name. It holds `settings.toml` (section 14).
- ARCH-6. The environment variable `QUIRE_DATA_DIR`, when set, replaces both directories. Every test uses it with a temporary directory.
- ARCH-7. Nothing is ever read from or written to the current working directory.

---

## 5. Data model

### 5.1 Conventions

- DB-1. Ids are UUIDv7 strings. Timestamps are RFC 3339 UTC strings set by application code. Months are `YYYY-MM` strings.
- DB-2. `updated_at` columns are set by application code on every write, not by SQL defaults.

### 5.2 Schema

Migration `0001_init.sql` creates exactly this.

```sql
CREATE TABLE profile (
    id         INTEGER PRIMARY KEY CHECK (id = 1),
    full_name  TEXT NOT NULL,
    email      TEXT NOT NULL,
    phone      TEXT,
    location   TEXT,
    headline   TEXT,
    updated_at TEXT NOT NULL
);

CREATE TABLE profile_links (
    id         TEXT PRIMARY KEY,
    label      TEXT NOT NULL,
    url        TEXT NOT NULL,
    sort_order INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE entries (
    id           TEXT PRIMARY KEY,
    kind         TEXT NOT NULL CHECK (kind IN
                 ('experience', 'project', 'education', 'certification')),
    title        TEXT NOT NULL CHECK (length(trim(title)) > 0),
    organisation TEXT,
    location     TEXT,
    url          TEXT,
    description  TEXT,
    start_month  TEXT CHECK (start_month IS NULL OR start_month GLOB '[0-9][0-9][0-9][0-9]-[0-1][0-9]'),
    end_month    TEXT CHECK (end_month   IS NULL OR end_month   GLOB '[0-9][0-9][0-9][0-9]-[0-1][0-9]'),
    sort_order   INTEGER NOT NULL DEFAULT 0,
    archived_at  TEXT,
    updated_at   TEXT NOT NULL
);

CREATE TABLE highlights (
    id          TEXT PRIMARY KEY,
    entry_id    TEXT NOT NULL REFERENCES entries(id) ON DELETE CASCADE,
    body        TEXT NOT NULL CHECK (length(trim(body)) > 0),
    sort_order  INTEGER NOT NULL DEFAULT 0,
    archived_at TEXT,
    updated_at  TEXT NOT NULL
);

CREATE TABLE tags (
    id    TEXT PRIMARY KEY,
    slug  TEXT NOT NULL UNIQUE CHECK (slug GLOB '[a-z0-9]*' AND slug NOT GLOB '*[^a-z0-9-]*'),
    label TEXT NOT NULL
);

CREATE TABLE entry_tags (
    entry_id  TEXT NOT NULL REFERENCES entries(id) ON DELETE CASCADE,
    tag_id    TEXT NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    weight    INTEGER NOT NULL DEFAULT 5 CHECK (weight BETWEEN 1 AND 10),
    rationale TEXT,
    PRIMARY KEY (entry_id, tag_id)
);

CREATE TABLE highlight_tags (
    highlight_id TEXT NOT NULL REFERENCES highlights(id) ON DELETE CASCADE,
    tag_id       TEXT NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    weight       INTEGER NOT NULL DEFAULT 5 CHECK (weight BETWEEN 1 AND 10),
    PRIMARY KEY (highlight_id, tag_id)
);

CREATE TABLE skills (
    id         TEXT PRIMARY KEY,
    name       TEXT NOT NULL UNIQUE,
    category   TEXT,
    sort_order INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE templates (
    id           TEXT PRIMARY KEY,
    name         TEXT NOT NULL UNIQUE,
    typst_source TEXT NOT NULL,
    page_target  INTEGER NOT NULL DEFAULT 1 CHECK (page_target BETWEEN 1 AND 4),
    is_default   INTEGER NOT NULL DEFAULT 0 CHECK (is_default IN (0, 1)),
    updated_at   TEXT NOT NULL
);
CREATE UNIQUE INDEX templates_one_default ON templates(is_default) WHERE is_default = 1;

CREATE TABLE applications (
    id            TEXT PRIMARY KEY,
    company       TEXT NOT NULL,
    role          TEXT NOT NULL,
    source_url    TEXT,
    jd_text       TEXT NOT NULL,
    jd_fetched_at TEXT,
    status        TEXT NOT NULL DEFAULT 'new' CHECK (status IN
                  ('new', 'drafted', 'sent', 'interview', 'offer', 'rejected', 'withdrawn')),
    created_at    TEXT NOT NULL,
    updated_at    TEXT NOT NULL
);

CREATE TABLE agent_runs (
    id             TEXT PRIMARY KEY,
    application_id TEXT NOT NULL REFERENCES applications(id),
    runner         TEXT NOT NULL,
    model          TEXT,
    state          TEXT NOT NULL CHECK (state IN
                   ('running', 'succeeded', 'failed', 'cancelled', 'timed_out')),
    started_at     TEXT NOT NULL,
    ended_at       TEXT,
    cost_usd       REAL,
    error          TEXT,
    stderr_tail    TEXT
);

CREATE TABLE artifacts (
    id              TEXT PRIMARY KEY,
    application_id  TEXT NOT NULL REFERENCES applications(id),
    version         INTEGER NOT NULL CHECK (version >= 1),
    state           TEXT NOT NULL CHECK (state IN ('draft', 'final')),
    selection_json  TEXT NOT NULL,
    resolved_json   TEXT NOT NULL,
    template_id     TEXT REFERENCES templates(id) ON DELETE SET NULL,
    template_source TEXT NOT NULL,
    template_sha256 TEXT NOT NULL,
    typst_version   TEXT NOT NULL,
    quire_version   TEXT NOT NULL,
    pdf_bytes       BLOB NOT NULL,
    pdf_sha256      TEXT NOT NULL,
    page_count      INTEGER NOT NULL,
    warnings_json   TEXT NOT NULL DEFAULT '[]',
    email_subject   TEXT,
    email_body      TEXT,
    created_by      TEXT NOT NULL,
    run_id          TEXT REFERENCES agent_runs(id),
    created_at      TEXT NOT NULL,
    finalised_at    TEXT,
    UNIQUE (application_id, version)
);
CREATE UNIQUE INDEX artifacts_one_final ON artifacts(application_id) WHERE state = 'final';
```

Note: `template_id` uses `ON DELETE SET NULL`, which is an UPDATE. Deleting a template referenced by a final artifact therefore fails under DB-3. That is intended: archive or keep such templates.

### 5.3 Rules enforced in the database

- DB-3. Final artifacts are immutable:

```sql
CREATE TRIGGER artifacts_final_no_update BEFORE UPDATE ON artifacts
WHEN OLD.state = 'final'
BEGIN SELECT RAISE(ABORT, 'final artifacts are immutable'); END;

CREATE TRIGGER artifacts_final_no_delete BEFORE DELETE ON artifacts
WHEN OLD.state = 'final'
BEGIN SELECT RAISE(ABORT, 'final artifacts are immutable'); END;
```

- DB-4. At most one final artifact per application (index `artifacts_one_final`).
- DB-5. At most one default template (index `templates_one_default`).

### 5.4 Rules enforced in code

- DB-6. Entries are soft-deleted by setting `archived_at`. Hard delete is allowed only for an entry no artifact's `selection_json` references.
- DB-7. An `experience` entry requires `organisation` and `start_month`. An `education` entry requires `organisation`.
- DB-8. `end_month`, when present, is not before `start_month`. A null `end_month` with a non-null `start_month` means current.
- DB-9. Tag slugs are normalised on write: lower case, spaces and underscores become `-`, other punctuation is removed, repeated `-` collapse.
- DB-10. On open, if the database has a migration newer than this binary knows, Quire refuses to open it and says which version is needed.
- DB-11. A fresh database is seeded with the embedded default template (`assets/templates/default.typ`) as the default.

---

## 6. Vault import and export

- VAULT-1. `quire export <file>` writes the whole vault (not applications or artifacts) to TOML in the format below.
- VAULT-2. `quire import <file>` replaces the vault with the file's contents in one transaction. If the file is invalid, nothing changes and every error is reported with its location.
- VAULT-3. Import then export round-trips: exporting an imported file yields the same data. Ids are preserved when present and generated when absent.
- VAULT-4. Import refuses to archive or delete an entry or highlight that a final artifact references, and names it.

```toml
[profile]
full_name = "Jane Doe"
email = "jane@example.com"
phone = "+971 50 000 0000"
location = "Dubai, UAE"
headline = "Risk analyst"

[[links]]
label = "LinkedIn"
url = "https://www.linkedin.com/in/example"

[[tags]]
slug = "risk-management"
label = "Risk management"

[[skills]]
name = "Excel"
category = "Tools"

[[entries]]
id = "0193..."            # optional
kind = "experience"
title = "Risk Analyst Intern"
organisation = "Example Bank"
location = "Dubai"
start = "2025-06"
end = "2025-08"           # omit for current
tags = [{ slug = "risk-management", weight = 8, rationale = "Core of the role" }]

  [[entries.highlights]]
  body = "Built a reconciliation model in Excel that cut month-end close from 5 days to 3."
  tags = [{ slug = "risk-management", weight = 6 }]
```

---

## 7. Selection and validation

### 7.1 Selection format

A selection is JSON matching this schema exactly. The same schema is used for the agent's structured output (AGENT-4) and for `submit_draft` (MCP-6). It lives at `crates/quire-core/schemas/selection.schema.json` and is embedded in the binary.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "additionalProperties": false,
  "required": ["summary", "entries", "skill_ids", "email"],
  "properties": {
    "summary": { "type": "string", "minLength": 1, "maxLength": 600 },
    "entries": {
      "type": "array",
      "minItems": 1,
      "maxItems": 12,
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["entry_id", "highlight_ids"],
        "properties": {
          "entry_id": { "type": "string" },
          "highlight_ids": {
            "type": "array",
            "maxItems": 6,
            "items": { "type": "string" }
          }
        }
      }
    },
    "skill_ids": {
      "type": "array",
      "maxItems": 30,
      "items": { "type": "string" }
    },
    "email": {
      "type": "object",
      "additionalProperties": false,
      "required": ["subject", "body"],
      "properties": {
        "subject": { "type": "string", "minLength": 1, "maxLength": 150 },
        "body": { "type": "string", "minLength": 1, "maxLength": 3000 }
      }
    }
  }
}
```

### 7.2 Validation rules

Validation is a pure function: `validate(&Selection, &VaultSnapshot) -> Result<Vec<Warning>, Vec<ValidationError>>`. It runs before any render. Errors stop the draft; warnings are stored on it and shown at review.

| ID | Rule | Outcome |
| --- | --- | --- |
| VAL-1 | The selection matches the schema in 7.1, including `additionalProperties: false` | Error `SCHEMA` |
| VAL-2 | Every `entry_id`, `highlight_id` and `skill_id` exists and is not archived | Error `UNKNOWN_ID` |
| VAL-3 | Every highlight belongs to the entry it is listed under | Error `WRONG_ENTRY` |
| VAL-4 | No id appears twice | Error `DUPLICATE_ID` |
| VAL-5 | A number in `summary`, `email.subject` or `email.body` that appears nowhere in the vault text | Warning `UNSOURCED_NUMBER`, naming the number |
| VAL-6 | Rendered page count is greater than the template's `page_target` | Warning `OVER_PAGE_TARGET`, with the count |
| VAL-7 | An entry is listed with zero highlights and has no description | Warning `EMPTY_ENTRY` |

VAL-5 detail: a number is a maximal run matching `\d+([.,]\d+)?`, optionally followed by `%`, `k`, `m`, `bn` or `x`. Vault text is every profile field, entry field, highlight body and skill name. Comparison ignores thousands separators, so `1,200` matches `1200`.

Every error carries `code`, `message` and a JSON pointer `path` into the selection. Error messages are written for the agent to act on: they say what was wrong and what to send instead.

---

## 8. Rendering

### 8.1 Requirements

- RENDER-1. Rendering embeds Typst through `typst-as-lib`. The Typst version is pinned exactly and stored on every artifact.
- RENDER-2. The template receives one input, `cv_json`: the resolved CV (8.2) serialised as a JSON string. The template reads it with `json(bytes(sys.inputs.cv_json))`. Check this call against the pinned Typst version at M2 and record the result in `DECISIONS.md`.
- RENDER-3. Templates MUST NOT call `eval`, `read`, `include` or `import` on anything but embedded files. Templates are validated on save by compiling them against `tests/fixtures/resolved/full.json`.
- RENDER-4. Fonts are embedded from `assets/fonts/`. The default set covers Latin, Latin Extended, Greek and Cyrillic. Missing-glyph warnings from Typst are returned as render warnings, not dropped.
- RENDER-5. Output is deterministic: the same resolved CV, template, fonts and Typst version produce byte-identical PDFs on Linux and macOS. The PDF timestamp and document identifier are fixed values, not the current time.
- RENDER-6. `render` returns `RenderOutput { pdf: Vec<u8>, page_count: u32, warnings: Vec<Warning> }`, or a `RenderError` with Typst's diagnostics including line and column.
- RENDER-7. `render_preview` returns one PNG per page at 2x scale, using `typst-render`, for the GUI.
- RENDER-8. The default template is single column, with real text headings named Summary, Experience, Projects, Education, Certifications and Skills. No icons carry meaning, and no text is placed in images.
- RENDER-9. Text extracted from the default template's PDF reads in the same order as the resolved CV. This is a test (T2.5).

### 8.2 Resolved CV

`resolve(&Selection, &VaultSnapshot) -> ResolvedCv` replaces ids with stored text. Ordering is fixed here, so templates only lay out.

```json
{
  "profile": {
    "full_name": "Jane Doe",
    "email": "jane@example.com",
    "phone": "+971 50 000 0000",
    "location": "Dubai, UAE",
    "headline": "Risk analyst",
    "links": [{ "label": "LinkedIn", "url": "https://..." }]
  },
  "summary": "Two sentences written for this role.",
  "sections": [
    {
      "kind": "experience",
      "title": "Experience",
      "entries": [
        {
          "title": "Risk Analyst Intern",
          "organisation": "Example Bank",
          "location": "Dubai",
          "url": null,
          "dates": "Jun 2025 – Aug 2025",
          "description": null,
          "highlights": ["Built a reconciliation model in Excel that cut month-end close from 5 days to 3."]
        }
      ]
    }
  ],
  "skills": [{ "category": "Tools", "names": ["Excel"] }]
}
```

- RESOLVE-1. Sections appear in the order experience, project, education, certification. Empty sections are omitted.
- RESOLVE-2. Entries within a section are ordered current first, then by `end_month` descending, then `start_month` descending, then `sort_order`.
- RESOLVE-3. Highlights keep the order the selection lists them in. That order is the agent's relevance ranking.
- RESOLVE-4. `dates` is formatted in core: `Mon YYYY – Mon YYYY`, `Mon YYYY – Present`, or `Mon YYYY` alone for an item without an end. English month abbreviations.
- RESOLVE-5. Skills are grouped by `category` in first-seen order of the selection. A skill with no category goes in a group with `category: null`, placed last.
- RESOLVE-6. Absent optional fields are `null`, never empty strings. Templates must handle every `null` without printing a stray separator.

---

## 9. MCP server

- MCP-1. `quire --mcp` runs an MCP server over stdio using `rmcp`. Server name `quire`.
- MCP-2. INV-4 applies. The logger is configured to stderr before anything else runs. A test asserts that every stdout line parses as JSON-RPC.
- MCP-3. Tool errors are returned as tool results with `isError: true` and a JSON body `{ "code", "message", "details" }`. The server never panics on bad input.
- MCP-4. The server's `instructions` say: the vault is the only source of facts; bullets cannot be rewritten; summaries and emails must not introduce numbers absent from the vault.

### 9.1 Tools

| ID | Tool | Input | Returns |
| --- | --- | --- | --- |
| MCP-5a | `get_profile` | none | Profile, links, and all non-archived skills with ids and categories |
| MCP-5b | `list_tags` | none | `[{ slug, label, entry_count }]`, sorted by `entry_count` descending |
| MCP-5c | `get_vault` | `tags?: string[]`, `kinds?: string[]`, `limit?: integer` | Entries with their highlights (below) |
| MCP-5d | `get_application` | `application_id: string` | `{ id, company, role, source_url, jd_text, status }` |
| MCP-6 | `submit_draft` | `application_id: string`, `selection: Selection` | `{ artifact_id, version, page_count, warnings }`, or validation errors |

`get_vault` returns:

```json
{
  "entries": [
    {
      "id": "...", "kind": "experience", "title": "...", "organisation": "...",
      "start": "2025-06", "end": "2025-08", "description": null,
      "score": 8,
      "tags": [{ "slug": "risk-management", "weight": 8, "rationale": "..." }],
      "highlights": [
        { "id": "...", "body": "...", "tags": [{ "slug": "risk-management", "weight": 6 }] }
      ]
    }
  ],
  "unknown_tags": []
}
```

- MCP-7. `get_vault` with no arguments returns every non-archived entry. With `tags`, `score` is the entry's highest weight among the requested tags, and entries are sorted by `score` descending, then by date as in RESOLVE-2. Entries matching no requested tag are included last with `score: 0`, so nothing disappears silently. `limit` caps the number of entries.
- MCP-8. Unknown tag slugs are returned in `unknown_tags`, not as an error.
- MCP-9. `submit_draft` runs VAL-1 to VAL-4, then resolves, renders, runs VAL-6, and inserts a `draft` artifact with the next version number, all in one transaction. On a render error it returns the Typst diagnostics and inserts nothing.
- MCP-10. `submit_draft` with an identical selection to the latest draft for that application returns that draft instead of inserting a new version.
- MCP-11. No tool can create a final artifact, change the vault, or change an application (INV-1).

---

## 10. Agent orchestration

### 10.1 Runner interface

```rust
pub trait AgentRunner: Send + Sync {
    fn name(&self) -> &'static str;
    /// Fails fast if the CLI is missing or not logged in.
    async fn check(&self) -> Result<RunnerInfo, AgentError>;
    /// Produces a selection for one application. Must honour `cancel`.
    async fn run(
        &self,
        req: RunRequest,
        events: tokio::sync::mpsc::Sender<RunEvent>,
        cancel: tokio_util::sync::CancellationToken,
    ) -> Result<RunOutcome, AgentError>;
}

pub struct RunRequest { pub application_id: String, pub timeout: Duration, pub max_budget_usd: f64 }
pub struct RunOutcome { pub selection: serde_json::Value, pub cost_usd: Option<f64>, pub model: Option<String> }
pub enum RunEvent { Started, ToolCalled { name: String }, Message(String), Finished }
```

Use native async fn in traits or `async-trait`, whichever the pinned toolchain supports; record the choice.

### 10.2 Claude runner

- AGENT-1. The path to `claude` comes from `settings.toml` (`agent.claude_path`). If unset, Quire resolves it once by running the user's login shell (`$SHELL -lc 'command -v claude'`) and stores the result. This makes launches from Finder work on macOS.
- AGENT-2. Each run executes in a new, empty temporary directory, deleted afterwards.
- AGENT-3. The MCP configuration is passed inline. Quire never edits the user's Claude configuration files during a run.
- AGENT-4. The invocation is:

```bash
claude -p "<task prompt>" \
  --mcp-config '<inline JSON: {"mcpServers":{"quire":{"command":"<absolute path of this quire binary>","args":["--mcp"]}}}>' \
  --strict-mcp-config \
  --allowedTools "mcp__quire__get_profile,mcp__quire__list_tags,mcp__quire__get_vault,mcp__quire__get_application" \
  --permission-mode dontAsk \
  --append-system-prompt "<system prompt from assets/prompts/system.md>" \
  --output-format json \
  --json-schema '<selection schema from 7.1>' \
  --max-budget-usd <settings.agent.max_budget_usd>
```

Verify every flag against `claude --help` for the installed version at M5 and record the version tested in `DECISIONS.md`.

- AGENT-5. `submit_draft` is deliberately not in `--allowedTools` for spawned runs. The selection comes back as structured output, and Quire validates, renders and saves it in the GUI process (D-001).
- AGENT-6. The task prompt contains only the application id and an instruction to call `get_application`. The job description never appears on the command line.
- AGENT-7. When `settings.agent.use_bare` is true, add `--bare`. That mode needs `ANTHROPIC_API_KEY` and skips the user's hooks, plugins and CLAUDE.md. Default `false`, because most users log in with a subscription.
- AGENT-8. Result handling: parse stdout as JSON. Success requires exit code 0, `is_error` false, and a present `structured_output` field. Record `total_cost_usd` as `cost_usd`. Anything else is a failed run with the error text stored.
- AGENT-9. Timeout: `settings.agent.timeout_secs`, default 300. On timeout or cancel, kill the whole process group, mark the run `timed_out` or `cancelled`, and keep the last 4 KB of stderr.
- AGENT-10. Every run inserts an `agent_runs` row at start and updates it at the end, whatever the outcome.
- AGENT-11. A successful run's selection goes through the same validate, resolve and render path as MCP-9, with `created_by = "claude"` and `run_id` set. If validation fails, the run is marked `failed` with the errors, and no artifact is created.

---

## 11. Command-line interface

| Command | Does |
| --- | --- |
| `quire` | Opens the GUI |
| `quire --mcp` | Runs the MCP server on stdio |
| `quire import <file.toml>` | VAULT-2 |
| `quire export <file.toml>` | VAULT-1 |
| `quire app add --company <c> --role <r> (--jd-file <path> \| --url <url>)` | Creates an application. Prints its id |
| `quire app list` | Lists applications with status and latest artifact state |
| `quire generate <application-id>` | Runs the agent, prints progress to stderr, prints the artifact id and warnings |
| `quire render <selection.json> --application <id> [--template <name>] --out <file.pdf>` | Validates, resolves and renders without the agent. Writes no artifact |
| `quire finalise <artifact-id>` | Marks a draft final and sets the application status to `drafted` if it was `new` |
| `quire artifact export <artifact-id> --out <file.pdf>` | Writes stored PDF bytes to a file |
| `quire install-mcp` | Registers `quire --mcp` with Claude Code for interactive use and prints exactly what it changed |
| `quire doctor` | Checks data dir, database version, fonts, template compile and `claude` availability |

- CLI-1. Exit code 0 on success, 1 on user error, 2 on internal error. Human output on stdout, diagnostics on stderr.
- CLI-2. `--json` on `app list`, `generate` and `doctor` prints machine-readable output.

---

## 12. GUI

Dioxus desktop, pinned to the 0.7 series. Screens:

- GUI-1. **Vault.** Edit profile and links, entries, highlights (reorder by drag or up/down buttons), tags and weights per entry and per highlight, and skills. Archive instead of delete.
- GUI-2. **Applications.** List with status. Create by paste or URL. Show the extracted job description for confirmation before saving (INTAKE-3).
- GUI-3. **Run.** Start a run for an application. Show `RunEvent`s as they arrive, elapsed time, and a Cancel button. Show the stored error on failure, with Retry.
- GUI-4. **Draft review.** Page previews (RENDER-7), warnings with what each means, the email subject and body with a copy button and a `mailto:` link, and a Finalise button. Finalising asks for confirmation and says it cannot be undone.
- GUI-5. **History.** Every artifact for an application by version, with state, date, template and run.
- GUI-6. **Templates.** List, set default, edit source with a compile check on save (RENDER-3), set `page_target`.
- GUI-7. **Settings.** Every key in section 14.
- GUI-8. The GUI never blocks its event loop on database, render or process work. All of it runs on background tasks.

---

## 13. Intake

- INTAKE-1. Paste is the primary path: company, role and job description text.
- INTAKE-2. URL fetch happens locally: HTTP GET, then readable-text extraction. The library is chosen at M7 and recorded.
- INTAKE-3. Fetched text is always shown to the user for confirmation and editing before it is saved.
- INTAKE-4. Jina Reader is an optional fetch backend, off by default (`intake.use_jina`). When on, the settings screen states that the URL is sent to a third party.
- INTAKE-5. Job description text over 50,000 characters is truncated, with a visible warning.
- INTAKE-6. Creating an application whose `source_url` matches an existing one shows a warning but is allowed.

---

## 14. Settings

`settings.toml`. Missing keys take defaults. Unknown keys are kept on save and logged once.

| Key | Default | Meaning |
| --- | --- | --- |
| `agent.claude_path` | resolved (AGENT-1) | Absolute path to `claude` |
| `agent.model` | unset | Passed as `--model` when set |
| `agent.timeout_secs` | 300 | AGENT-9 |
| `agent.max_budget_usd` | 1.00 | Passed as `--max-budget-usd` |
| `agent.use_bare` | false | AGENT-7 |
| `intake.use_jina` | false | INTAKE-4 |
| `export.dir` | `<data dir>/exports` | Where finalised PDFs are copied |
| `export.filename` | `{full_name} - {company} - {role}.pdf` | Placeholders are sanitised for the file system |

- SET-1. On finalise, the PDF is copied to `export.dir` with `export.filename`. A failed copy is reported but does not undo the finalise; the bytes stay in the database (D-005).
- SET-2. File names replace `/ \ : * ? " < > |` and control characters with `-`, and are cut to 150 characters.

---

## 15. Errors and logging

- ERR-1. `quire-core` uses typed errors (`thiserror`). The binary uses `anyhow` at the top level only.
- ERR-2. No `unwrap` or `expect` outside tests, except on invariants documented in a comment at the call site.
- ERR-3. Logging uses `tracing`, to stderr, level from `QUIRE_LOG` (default `info`). The GUI also writes to `<data dir>/quire.log`, rotated at 5 MB.
- ERR-4. Logs never contain job description text, vault text, or the email body. Ids and counts only.

---

## 16. Non-functional

- NFR-1. Render of a one-page CV completes in under 1 second on a 2020-era laptop, after the first compile.
- NFR-2. The GUI starts to a usable window in under 2 seconds.
- NFR-3. A release build is a single binary plus platform packaging. No runtime other than the system webview (WebKitGTK on Linux).
- NFR-4. CI runs format, lint and tests on `ubuntu-latest` and `macos-latest`. Arch is tested by hand before release.

---

## 17. Phase 2 (not v1, do not build)

Recorded so v1 choices don't block it:

- Rewording: an optional `text` per highlight in the selection, a validation rule rejecting any number absent from the stored bullet, and a side-by-side approval step in draft review.
- An `agy` runner behind `AgentRunner`.
- Job board monitoring and form filling, always with a human confirmation before submit.

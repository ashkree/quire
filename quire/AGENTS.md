# AGENTS.md

Instructions for any coding agent working in this repository. Humans are welcome to read it too.

## What this project is

Quire is a local Rust desktop app. It keeps a career vault in SQLite and produces a tailored CV (PDF, rendered with embedded Typst) and an outreach email draft for each job application. An LLM agent picks the content through Quire's MCP server; deterministic code validates and renders it.

## Read before you write code

1. `docs/PLAN.md`: find your task. Work on one task at a time: the first unchecked task whose dependencies are done, unless you were given a specific one.
2. `docs/SPEC.md`: read every requirement ID your task lists under "Implements", plus section 3 (Invariants).
3. `docs/DECISIONS.md`: skim it so you don't reopen something already decided.

The spec is the source of truth. If code and spec disagree, the spec wins until a decision says otherwise.

## Repository layout

```text
crates/quire-core/    models, db, vault import/export, validation, resolve, render (no UI, no MCP)
crates/quire-mcp/     MCP server (rmcp, stdio)
crates/quire-agent/   AgentRunner trait and Claude runner
crates/quire-app/     the `quire` binary: CLI and Dioxus GUI
migrations/           sqlx migrations, applied in order, never edited once merged
.sqlx/                sqlx offline query cache, committed
assets/fonts/         embedded fonts
assets/templates/     embedded Typst templates
assets/prompts/       system prompt for agent runs
tests/fixtures/       vault TOML, selections, resolved CVs, job descriptions
tests/golden/         expected PDF hashes and extracted text
docs/                 SPEC, PLAN, DECISIONS, manual tests
```

## Commands

```bash
cargo build --workspace
cargo test --workspace                                   # all tests
cargo test -p quire-core <name>                          # one crate or test
cargo fmt --all
cargo clippy --workspace --all-targets -- -D warnings
cargo sqlx prepare --workspace                           # after changing any SQL query
cargo run -p quire-app -- <args>                         # e.g. -- render tests/fixtures/selection/full.json ...
```

- Builds and CI use `SQLX_OFFLINE=true`. If you change a query, run `cargo sqlx prepare --workspace` and commit `.sqlx/`.
- Every test that touches the database or files sets `QUIRE_DATA_DIR` to a fresh temporary directory. Never use the real data directory in tests.
- To update golden files on purpose: `QUIRE_UPDATE_GOLDEN=1 cargo test -p quire-core golden`. Say why in the pull request.

## Invariants: never break these

Full list in SPEC section 3. The ones easiest to break by accident:

- **No text in Typst source.** User, vault and agent text reaches templates only through `sys.inputs.cv_json`. Never build Typst source with `format!` or string concatenation. (INV-3)
- **stdout is sacred in `--mcp` mode.** No `println!`, no `dbg!`, no dependency that prints. Log with `tracing` to stderr. (INV-4)
- **Bullets are verbatim.** Rendered bullets are exact stored highlight bodies. There is no rewording code path in v1. (INV-2, D-002)
- **The agent never finalises.** No MCP tool or agent path creates or changes a final artifact. (INV-1)
- **`quire-core` stays pure.** No Dioxus, rmcp or process spawning in it. (INV-6)
- **Migrations are append-only.** Add a new numbered file; never edit a merged one. (INV-8)
- **Embedded fonts only.** No system font lookup. (INV-7)

## Coding conventions

- Rust edition and toolchain from `rust-toolchain.toml`. Code passes `fmt` and `clippy -D warnings`.
- Errors: `thiserror` enums in library crates, `anyhow` only at the top of `quire-app`. No `unwrap`/`expect` outside tests unless a comment at the call site says why it cannot fail.
- Async: `tokio`. Never block the GUI event loop; database, render and process work go on background tasks.
- SQL: `sqlx::query!`/`query_as!` macros so queries are checked at build time. Use transactions for anything that writes more than one row.
- Ids: UUIDv7 strings. Timestamps: RFC 3339 UTC strings set in code. Months: `YYYY-MM`.
- Logging: `tracing`. Never log job description text, vault text or email bodies; log ids and counts.
- Keep functions that make decisions pure (validation, resolve, ordering, slug normalisation) and test them without I/O.
- Comments explain why, not what. Cite requirement IDs where code implements a rule: `// VAL-3`.
- Error messages returned to agents (MCP tool errors, validation errors) say what was wrong and what to send instead.

## Dependencies

- Versions are pinned exactly (`=x.y.z`) in the workspace `Cargo.toml`.
- Do not add a dependency unless your task needs it. When you add one, add a line to `docs/DECISIONS.md` with the crate, version and reason.
- Do not upgrade Dioxus, Typst or rmcp as part of another task. Upgrades are their own task, because they can change rendering or protocol behaviour.

## Testing

Every task adds tests. In particular:

- Every validation and database rule gets one passing and one failing test.
- Rendering changes must keep golden hashes stable or update them on purpose.
- MCP changes get a protocol test that spawns `quire --mcp` and talks JSON-RPC over stdio, asserting stdout has nothing else.
- Agent runner changes are tested against the fake agent binary in `crates/quire-agent/tests/`, never against a real model in CI.

## When the spec is unclear or wrong

- Do not guess and do not silently pick an interpretation.
- If you are working interactively, stop and ask, quoting the requirement ID.
- If you are working unattended, add a `proposed` entry to `docs/DECISIONS.md` describing the question and the option you recommend. Then either implement the smallest version that doesn't depend on the answer, or stop the task and say so in the pull request.
- If a requirement looks wrong, say so with the reason. Do not work around it in code.

## Scope

- Do only your task. Don't refactor unrelated code, reformat files you didn't change, or start the next task.
- Anything in SPEC section 1 "Out of scope" or section 17 "Phase 2" is not to be built, even partially, even if it seems easy.

## Pull requests and commits

- One task per branch, named `t<task>-<short-name>`, e.g. `t2.3-default-template`.
- Commit messages: imperative summary under 72 characters, then a body citing task and requirement IDs.
- In the same pull request: tick the task in `docs/PLAN.md`, fill in any `pending` decision the task owns, and add any manual test steps to `docs/manual-tests.md`.
- The pull request description lists: the task, requirements implemented, how it was tested, and anything left open.

## Definition of done

A task is done when all of these are true:

- [ ] Its "Done when" check in `docs/PLAN.md` passes.
- [ ] `cargo fmt --all --check`, `clippy -D warnings` and `cargo test --workspace` pass.
- [ ] `.sqlx/` is current if queries changed.
- [ ] No invariant in SPEC section 3 is broken.
- [ ] `docs/PLAN.md` and `docs/DECISIONS.md` are updated.

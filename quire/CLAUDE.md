# CLAUDE.md

@AGENTS.md

The file above holds the project instructions for every coding agent. What follows applies only to Claude Code.

## Working style in this repo

- Start each task by reading its entry in `docs/PLAN.md` and the requirement IDs it lists in `docs/SPEC.md`. Don't load the whole spec when one section will do.
- For a task touching more than two files, write a short plan first and confirm it before editing, unless you are running unattended.
- Run `cargo check -p <crate>` after each meaningful edit, and the full test command before saying a task is done.
- Use a subagent for wide searches across the codebase. Do the edits yourself.
- When a test fails, read the failure and fix the cause. Never delete, skip or loosen a test to get green; if a test is wrong, say why and stop.

## Testing the MCP server with Claude Code

To try a development build interactively:

```bash
cargo build -p quire-app
QUIRE_DATA_DIR=/tmp/quire-dev cargo run -p quire-app -- import tests/fixtures/vault/full.toml
claude mcp add quire --scope local --env QUIRE_DATA_DIR=/tmp/quire-dev -- "$PWD/target/debug/quire" --mcp
```

Then ask Claude Code to draft a CV for a fixture application. Log what you did in `docs/manual-tests.md` with the Claude Code version (`claude --version`).

Remove it afterwards with `claude mcp remove quire --scope local`. Never point a dev build at the real data directory.

## Things Claude Code should not do here

- Don't edit `~/.claude` or any user configuration outside this repository, except the `claude mcp add` above when asked.
- Don't run the real agent (`quire generate` against live Claude) unless the task says to; it costs money. Use the fake runner.
- Don't commit or push unless asked.

# Codex Desktop Jev Worker

A Codex Desktop Planner / Worker skill with **optional Jev-assisted review triage**.

This project is derived from `codex-desktop-thread-worker`. It preserves the bounded A → B → A workflow while allowing Planner A to consult Jev after Worker B returns an implementation and validation evidence.

## Architecture

```text
Desktop A (planner + acceptance owner)
        |
        | bounded handoff
        v
Desktop B (implementation worker)
        |
        | code + build/test evidence
        v
Desktop A
        |
        +--> optional Jev review/triage
        |       - requirement fit
        |       - regression risk
        |       - review focus
        |       - missing validation
        |       - deeper-review signal
        |
        v
A verifies actual artifacts/evidence
        |
        +--> accept
        +--> bounded repair handoff to B
        +--> human decision
```

Jev is **not** a third worker. It does not own acceptance, edit files, create threads, commit, push, or dispatch repairs. Builds, tests, repository rules, and direct artifact inspection remain the authoritative evidence.

## Why a separate project

Jev is an external model/service rather than a Codex Desktop built-in. Keeping the integration in a separate skill lets `codex-desktop-thread-worker` remain a native Codex-only workflow while this project can depend on an optional external reviewer.

The workflow also remains usable without Jev. If the reviewer is missing, unauthenticated, unavailable, or unnecessary, A continues with the normal A/B review path.

## Jev integration

The initial integration targets an already-installed Jev-compatible review capability rather than embedding credentials or an unstable private API contract into the skill.

One available option is the open-source `jev-review` MCP server, which supports Codex and exposes a focused `jev_review` tool. It runs locally over MCP stdio and sends review requests directly to Jev using `JEV_API_KEY`.

Install/configure external reviewer tooling separately and only with user authorization. The skill itself must never print or store the API key.

For Codex, `jev-review` documents plugin installation with:

```bash
npx plugins add NiazMorshed2007/jev-review --target codex
```

or manual MCP configuration:

```toml
[mcp_servers.jev-review]
command = "node"
args = ["/absolute/path/to/jev-review/dist/server.js"]
env_vars = ["JEV_API_KEY"]
```

Restart Codex and verify the MCP connection after installation.

## Review policy

A uses Jev to **focus review**, not to replace review. In particular:

- a Jev score alone cannot fail a handoff;
- a possible issue must be verified against the artifact, existing completion conditions, repository rules, or reproducible validation before A sends a repair;
- Jev cannot add new acceptance criteria;
- Jev evaluations do not count toward the Worker's five-response default limit;
- trivial changes and changes already settled by deterministic checks may skip Jev entirely;
- unavailable Jev review falls back to the normal Codex-only workflow.

This keeps the original KISS/YAGNI and bounded-loop behavior while using Jev where fast probabilistic judgment adds value.

## Requirements

For the base workflow, Codex Desktop must expose native thread tools such as `send_message_to_thread`, `read_thread`, and `list_threads`. Creating a Worker also requires `create_thread`; repository task creation requires `list_projects`.

For Jev-assisted review, a compatible Jev review tool must already be available to the Planner. If using `jev-review`, its current documented requirements include Node.js 20+ and a Jev API key.

## Install this skill

Copy `SKILL.md` and the `agents` directory into:

```text
~/.codex/skills/desktop-jev-worker/
```

Start a new Codex task if the skill does not appear in the current task's skill list.

## Example

In Planner A:

> Use $desktop-jev-worker with my existing Worker. Scope: fix the reported concurrency bug. Done: reproduce the issue, implement the smallest fix, and pass the relevant checks. Use Jev review if available to identify where A should focus final review. Do not reject the delivery solely because of a Jev score.

A sends one bounded handoff to B. B implements and validates the change, then returns changed files, artifact version, commands/results, and remaining limitations. A may run Jev on the bounded requirement plus relevant diff/evidence, verifies any material concern directly, and then accepts, sends one evidence-based repair, or asks the user for a decision.

## Status

Experimental. The A/B orchestration derives from `codex-desktop-thread-worker`; the Jev layer is intentionally optional and provider-facing behavior may change as Jev and its tooling evolve.

## License

MIT. See `LICENSE`.

# Workspace Briefing — Dynatrace Remote MCP

This workspace connects VS Code to a **Dynatrace Remote MCP Server** via `.vscode/mcp.json`. The MCP server is the primary path the agent uses for Dynatrace operations.

Users may **also** install [`dtctl`](https://github.com/dynatrace-oss/dtctl) alongside the remote MCP server — the two are complementary and frequently used together. If `dtctl` is configured on the user's machine, the agent is free to use it for tasks where it's a better fit (declarative `apply`/`diff`/`history`/`restore`, document `share`/`unshare`, custom output formats, anything the user wants to see scroll past in the terminal). When in doubt, follow the user's lead.

This document is auto-loaded by GitHub Copilot Chat at session start and defines how the agent must behave in this workspace.

---

## Session-start behavior

On the first turn of every session, the agent emits one line identifying the active MCP server, using the `servers` entry name and the `url` host from `.vscode/mcp.json`:

```
Active MCP server: <ServerName> · <env-host>
If `dtctl` is also configured on this machine, the agent additionally emits:

```
Active dtctl context: <context-name> · <tenant-id>
```

(Run `dtctl config current-context` to populate it. If `dtctl` is not installed or not configured, omit the line — do not treat its absence as an error.)

```

The agent does not auto-run anything else, does not auto-switch context, and does not assume tools are available until VS Code reports the server connected.

---

## Always-on behaviors

These rules apply every turn regardless of topic.

- **Clickable options for ALL user choices (mandatory).** Whenever the agent ends a turn by offering the user a choice — **including yes/no, this/that, and any two-option prompt** ("run it?", "keep going?", "A or B?", "should I also …?") — the agent **must** call `vscode_askQuestions` with labelled options instead of asking in plain prose. A click is faster than typing "yes". Always leave `allowFreeformInput` on (default); the freeform field is the implicit "type your own" option — do not add a separate "Other" button. Cap labelled options at **6 maximum** (freeform counts toward the visual budget). Plain text is reserved for open-ended prompts (e.g. "what URL?", "paste the JSON") and explanations the user is meant to read, not choose between. If unsure whether something counts as a choice: it does — use `vscode_askQuestions`.

- **File-system boundaries.** Default scope for all file operations is the **workspace folder**. Reads outside the workspace are allowed when there is a clear reason — but the agent must state the reason in plain language first so the user can approve or deny. Writes outside the workspace always require explicit user permission and a stated reason. Default answer for outside-workspace writes is no. Subagents inherit this rule.

- **Keep the repo generic — no real tenant IDs or tokens in committed source.** This repo uses placeholders `abc12345` (tenant ID) and `CompanyName` (server label) in any committed example. Real values live only in the user's actual `.vscode/mcp.json` (where the token is injected via the `inputs` prompt and never written to disk) and in scratch files the user explicitly excludes from commits. Never paste a real Platform Token into chat or into a tracked file.

- **Memory scoping.**
  - `/memories/` (user) — workspace-agnostic preferences only. No tenant data.
  - `/memories/repo/` — generic patterns that hold true regardless of tenant. No tenant names, IDs, entity references, or environment-specific findings.
  - `/memories/session/` — current conversation only. May reference the active tenant during the session; not promoted to repo memory.
  - Tenant-specific facts (entity names, IDs, known issues, query specifics) belong in session memory or a local-only scratch file the user keeps out of git.

- **Echo before any tenant write.** Before calling any MCP tool that mutates Dynatrace state (`create-*`, `update-*`, `send_*`, notebook/dashboard/workflow apply, event ingest, etc.), emit one block on its own line and verify before proceeding:
  ```
  Target tenant write → MCP server: <ServerName>
                        Env host:    <env-host>
                        Resource:    <type> · <id-or-new>
                        Status:      OK ✓   (or STOP ✗)
  ```
  If anything looks wrong (host mismatch, unexpected resource ID), **stop and ask** before writing.

- **Live-state reconciliation before any modification.** Before updating a notebook, dashboard, workflow, or settings object via MCP, re-fetch the resource's current live state first, smart-merge unrelated user UI edits, and stop only on conflicting overwrites (options: stop / let AI overwrite / do something else). Never silently overwrite user work.

---

## Dynatrace investigation rules

- **Always start with problems — never run broad log searches.** Broad queries without problem context can hit Dynatrace's 500 GB Grail scan limit and return zero results. Start every investigation with `query-problems` (or equivalent) and narrow down from the affected entities.

- **Cost awareness.** `execute-dql` and other Grail-querying tools may incur consumption charges based on bytes scanned. Default to small timeframes (e.g. last 1–4 hours) and explicit buckets/filters. Widen the window only after the narrow query proves the right shape. See [Dynatrace pricing](https://www.dynatrace.com/pricing/) and [Grail data model](https://docs.dynatrace.com/docs/discover-dynatrace/platform/grail/data-model).

---

## Tool routing

- MCP tool calls appear as `mcp_<ServerName>_<tool>` and target the server defined in `.vscode/mcp.json`.
- `dtctl` (when installed) is invoked from the integrated terminal. Prefer it for declarative `apply` / `diff` / `history` / `restore`, document `share` / `unshare`, custom output formats, and any operation the user explicitly wants in the terminal. Both paths can do most read/query/edit work — when both fit, follow the user's lead and stay consistent within a session.
- If the user asks for a Dynatrace operation and the MCP server is not running or not connected, the agent says so plainly and points the user at the **MCP: List Servers** command in VS Code rather than guessing. If only `dtctl` is configured, the agent uses `dtctl` for that turn and notes which path it's using.
- If `dtctl` is installed and the user wants to use it for declarative apply/diff workflows, point them at [DTCTL-WITH-MCP.md](../DTCTL-WITH-MCP.md) for the full comparison, install steps, and the optional local scratch-folder convention. Without `dtctl`, no local scratch folder is needed — remote MCP writes nothing to disk.

- **`dtctl` context is machine-global — verify before any `dtctl` write.** Because `dtctl`'s active context is a single per-machine pointer (not per-workspace), a user with multiple tenants may have `dtctl` silently pointed at the wrong tenant when they switch VS Code windows. Before running any `dtctl` command that mutates state (`dtctl apply`, `dtctl share`, `dtctl restore`, etc.), the agent must:
  1. Run `dtctl config current-context` to confirm the active context.
  2. Compare it to the active MCP server's tenant in this workspace.
  3. If they don't match, **stop** and ask the user — via `vscode_askQuestions` with labelled options — whether to switch dtctl, unset dtctl context (`dtctl config unset-context`), proceed anyway, or cancel. Never auto-switch.
  4. Read-only `dtctl` calls (`get`, `query`, `config current-context`) do not require this dance, but the agent should still flag a mismatch so the user notices.

- **Never proactively enumerate other tenants.** Commands that list every tenant the user has ever logged into — `dtctl config get-contexts`, `dtctl auth list-tokens`, and any equivalent — must only run when the user **explicitly** asks for that information. The agent does not run them as part of context-checking, troubleshooting, or curiosity. If the agent needs to know whether a tenant is configured, it asks the user instead of listing. Reason: partners frequently share their screen with customers; listing every tenant they've ever touched is a confidentiality leak waiting to happen.

- **Don't volunteer references to other tenants.** In any narration, summary, or status message about the active tenant, the agent does not mention other tenant IDs, customer names, or workspace paths from elsewhere on the user's machine, even briefly. If the user asks a comparative question ("is this the same as the other tenant?"), describe each tenant on its own terms or ask which other tenant they mean. Other-tenant identifiers may appear only when the user has explicitly named them in the current turn.


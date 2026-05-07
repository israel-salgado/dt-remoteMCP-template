# Using `dtctl` Alongside the Remote MCP Server

This file is for users who already have the [Dynatrace Remote MCP Server](README.md) connected and are considering adding [`dtctl`](https://github.com/dynatrace-oss/dtctl) to their workflow. The two tools are **complementary, not competing**. They can share the same Dynatrace tenant and run in parallel without conflict. Use whichever is the better fit for the task at hand (or use both in the same session).

> If you only want conversational/AI-driven access to Dynatrace from VS Code Copilot Chat, the remote MCP server alone is enough. Skip this file.

---

## ⚠️ Read this first: multi-tenant partners

If you are a partner or consultant with access to **more than one Dynatrace tenant**, read this section before installing `dtctl`. Everything below the multi-tenant section assumes you've internalized these constraints.

### The constraint

`dtctl` context is **machine-global, not per-workspace and not per-window.**

- One `dtctl` install on your machine = one active context at a time.
- That context is persistent on disk. It survives terminal restarts, VS Code restarts, and reboots.
- Switching VS Code workspaces does **not** change it.
- Opening a second VS Code window does **not** give you a second context.
- Closing VS Code does **not** clear it.

The remote MCP server, by contrast, is per-workspace (each workspace folder under `Partners/<customer>/<tenant>/` has its own `.vscode/mcp.json`). MCP isolation is automatic; **`dtctl` isolation is your job.**

### The two failure modes that matter

**1. Wrong-tenant write.** You finish work in Workspace A (tenant X) and open Workspace B (tenant Y). `dtctl config use-context Y` runs (manually or by habit). You switch back to Window A and type `dtctl apply -f temp/dashboards/foo.json`, and that command runs against **Y**, not X. If `foo.json` is a notebook with customer-X-specific names, queries, or branding, you've just published it inside customer Y's tenant. Likely small impact, occasionally serious.

**2. Wrong-tenant exposure during a screen share.** You're presenting to customer X. You run `dtctl config get-contexts` to "check what's available", and the terminal prints **every other tenant you've ever logged into**, by context name and URL. If any of those context names contain customer references (`acme-prod`, `globex-sprint`), you've leaked another customer's relationship in front of this one. Not catastrophic, but unprofessional and avoidable.

### The rule

**Before switching to work on a different tenant, exit the current `dtctl` context.** Then either re-auth into the new tenant or stay MCP-only.

```powershell
# Switch off the current context (puts dtctl in "no active context" state)
dtctl config unset-context

# Verify
dtctl config current-context   # should report no active context
```

> If your `dtctl` version uses a different command for unsetting (older builds shipped `dtctl config use-context ""` or required a re-`auth login` to switch), check `dtctl config --help`. The intent is the same: leave dtctl in a state where any subsequent command requires an explicit `--context <id>` or a fresh `use-context`.

When you're ready to work on the next tenant, log in (or switch) explicitly:

```powershell
dtctl config use-context <next-tenant-id>
# or, if it's a brand-new tenant:
dtctl auth login --environment https://<id>.<class>.dynatrace[labs].com --context <id> --safety-level readwrite-mine
```

### Naming convention for `dtctl` contexts

**Always use the tenant ID as the context name. Never customer or company names.**

```powershell
# Good: tenant ID only
dtctl auth login --environment https://abc12345.apps.dynatrace.com --context abc12345 --safety-level readwrite-mine

# Bad: leaks the customer relationship in any output that lists contexts
dtctl auth login --environment https://abc12345.apps.dynatrace.com --context acme-prod --safety-level readwrite-mine
```

Tenant IDs are 8-character alphanumeric strings; on their own they convey nothing about who the customer is. Anyone seeing `tdg63684` in your terminal has no way to map that back to a partner relationship. `acme-prod` does the opposite.

### Demo / screen-share hygiene

When you're about to share your screen, especially with a customer, run through this checklist:

1. **Close other VS Code windows.** Each window's title bar shows the workspace path; `Partners/CustomerB/...` in the recent-folders list is a leak waiting to happen.
2. **Clear the terminal.** `Clear-Host` (PowerShell) or Ctrl+L. Old `dtctl` output from another tenant in the scrollback counts as exposure.
3. **Verify the active context once, quietly.** `dtctl config current-context` prints only the active tenant ID, which is safe to run.
4. **Do not run `dtctl config get-contexts` on a shared screen.** It enumerates every tenant you've ever logged into, including their URLs. If you genuinely need to see the list, do it before you share.
5. **Same restraint for `dtctl auth list-tokens` (or equivalent).** Anything that walks all stored credentials is off-limits during a demo.
6. **Prefer the agent's MCP path during demos.** The MCP server in `.vscode/mcp.json` only knows about the tenant defined in this workspace. There is no way it can leak another customer.

The agent in this workspace is configured to never proactively enumerate other tenants (see [.github/copilot-instructions.md](.github/copilot-instructions.md)). It *should* only run `dtctl config get-contexts`, `dtctl auth list-tokens`, or similar commands when you explicitly ask.

### Habits that prevent the failure modes

1. **Treat `dtctl` context as "checked out", like a feature branch.** When you finish working on a tenant, `unset-context` before walking away. Future-you (or future-script-you) shouldn't accidentally fire commands at the wrong tenant.
2. **Trust the session-start echo.** The agent prints `Active dtctl context: …` on the first turn of every session. If it doesn't match the workspace you're in, switch (or unset) before touching anything.
3. **Always verify before a write.** `dtctl config current-context` is fast. Run it before any `apply`, especially if you've stepped away from the keyboard.
4. **Pass `--context <id>` explicitly in scripts.** Anything in CI, a pipeline, or a saved PowerShell function should never assume the global context. `dtctl --context <id> apply -f …` is unambiguous and survives global-context drift.
5. **Recognize what each path isolates for you.** The workspace-per-tenant pattern (`Partners/<customer>/<tenant>/`) gives you automatic MCP isolation: every workspace has its own `.vscode/mcp.json` and its own MCP server. `dtctl` does not get that for free; its single global context is shared across **every** workspace on your machine. The other four habits exist specifically to compensate for that.

### Why this isn't auto-managed

It's tempting to wire up auto-switch (or auto-unset on idle) from the workspace. We've deliberately not done it because:

- **Auto-switch on workspace open** silently misaligns every other window the moment you open a second workspace.
- **Auto-unset on idle** could fire mid-pipeline, in the middle of a long-running script, or right as you come back from a meeting and try to resume work.
- Both create new failure modes in exchange for solving an old one.

The honest answer: `dtctl`'s global context is a sharp tool. Used carefully, it's fine. Used carelessly across multiple tenants, it can write to or expose the wrong one. **It is the user's responsibility to switch or unset context appropriately.** This file, the session-start echo, and the per-write echo block in [.github/copilot-instructions.md](.github/copilot-instructions.md) are guardrails, not a substitute for paying attention.

---

## TL;DR: when does each shine?

| If you want to… | Use |
|---|---|
| Ask "what's broken right now?" in chat and have the AI investigate | **Remote MCP** |
| Generate, explain, or run DQL conversationally | **Remote MCP** |
| Run Davis CoPilot / Davis Analyzers from a prompt | **Remote MCP** |
| Pull a dashboard down to a JSON file, edit it, and push it back | **`dtctl`** |
| Compare a local copy of a notebook to what's live in the tenant | **`dtctl`** |
| Roll back a workflow to a previous version | **`dtctl`** |
| Share a dashboard with another user from the terminal | **`dtctl`** |
| Inspect data in `csv` / `yaml` / `wide` / table formats from the shell | **`dtctl`** |
| Persist multiple tenant contexts and switch between them | **`dtctl`** |
| Run something inside a script, CI job, or pipeline | **`dtctl`** |

When both can do the job (DQL queries, reading entities, fetching problems, listing vulnerabilities, etc.), pick whichever feels faster for the moment.

---

## Capability comparison

| Capability | Remote MCP | `dtctl` |
|---|:---:|:---:|
| **Surface** | VS Code Copilot Chat (and any MCP client) | Terminal CLI |
| **Auth** | Platform Token (Bearer header) | OAuth interactive (browser SSO); tokens in OS credential store |
| **Hosting** | Hosted by Dynatrace, auto-updated | Local binary, updated via GitHub releases |
| **Multi-context** | One server entry per tenant in `.vscode/mcp.json` | Persistent named contexts, `dtctl config use-context <name>` |
| **Safety levels** | Governed by token scopes | Explicit per-context: `readonly` / `readwrite-mine` / `readwrite-all` / `dangerously-unrestricted` |
| **Output format** | Structured JSON returned to the AI | `json` / `yaml` / `csv` / `toon` / `wide` / table |
| **Scriptable in CI** | Possible but not designed for it | Yes, first-class CLI |
| Run DQL queries | ✅ `execute-dql` | ✅ `dtctl query` |
| Read entities (services, hosts, problems, vulnerabilities, K8s, RUM) | ✅ | ✅ |
| Read notebooks / dashboards / workflows / settings | ✅ | ✅ |
| **Create / update notebooks / dashboards / workflows** | ⚠️ limited (varies by remote MCP version) | ✅ via `dtctl apply` |
| **Declarative `apply`** (idempotent file → tenant) | ❌ | ✅ |
| **`diff`** local vs. live | ❌ | ✅ |
| **`history`** for a resource | ❌ | ✅ |
| **`restore`** a previous version | ❌ | ✅ |
| **`share` / `unshare`** documents | ❌ | ✅ |
| Davis CoPilot chat | ✅ `ask-dynatrace-docs` | ⚠️ check current dtctl docs |
| Davis Analyzers (anomaly, forecast, baseline, novelty, threshold) | ✅ | ⚠️ check current dtctl docs |
| Send Slack / email / events | ⚠️ varies (see remote MCP capability list) | ⚠️ check current dtctl docs |
| Natural-language → DQL helper | ✅ `create-dql` | ❌ |
| Explain DQL in natural language | ✅ `explain-dql` | ❌ |
| Send custom event (`send_event`) | ⚠️ varies | ❌ |

> The remote MCP tool list evolves; see the upstream [migration guide](https://github.com/dynatrace-oss/dynatrace-mcp/blob/main/docs/remote-mcp-migration.md) for the current authoritative comparison.

> _Capability table last verified: 2026-05-07._

---

## Installing `dtctl`

Follow the official setup steps in the [dtctl repository](https://github.com/dynatrace-oss/dtctl). The short version on Windows once `dtctl` is on your `PATH`:

```powershell
# Authenticate against your tenant (opens a browser for SSO)
dtctl auth login `
  --environment https://abc12345.apps.dynatrace.com `
  --context abc12345 `
  --safety-level readwrite-mine

# Verify
dtctl config current-context
dtctl auth whoami --plain
```

Replace `abc12345` with your real tenant ID. URL patterns:

| Tenant class | URL |
|---|---|
| Production | `https://<id>.apps.dynatrace.com` |
| Sprint / lab (Dynatrace internal) | `https://<id>.sprint.apps.dynatracelabs.com` |
| Classic Gen2 | `https://<id>.live.dynatrace.com` |

### Safety levels at a glance

| Level | What it allows | When to use |
|---|---|---|
| `readonly` | Read-only across the tenant | Pure inspection, demos, untrusted scripts |
| `readwrite-mine` | Write only resources you own | Recommended day-to-day default |
| `readwrite-all` | Write any resource in the tenant | Admin work; default if `--safety-level` is omitted |
| `dangerously-unrestricted` | No guardrails | Only when you know exactly why you need it |

---

## Local scratch folder for `dtctl` artifacts

The remote MCP server is HTTP-only and never writes to disk. `dtctl` is declarative: it exports tenant resources to local JSON/YAML files for `apply` / `diff` / `restore` workflows. Once you start using `dtctl`, give those files a stable, gitignored home:

```powershell
# From the workspace root
New-Item -ItemType Directory -Path temp | Out-Null
Add-Content -Path .gitignore -Value "`ntemp/"
```

Suggested layout (subfolders auto-created on first use):

```
temp/
├── notebooks/   <NOTEBOOK-ID>.json
├── dashboards/  <DASHBOARD-ID>.json
├── workflows/   <WORKFLOW-ID>.json
└── snapshots/   before-edit copies for revert
```

Until `dtctl` is installed, the folder isn't needed; remote MCP alone leaves nothing on disk to manage.

---

## How the agent picks between the two

The auto-loaded briefing at [.github/copilot-instructions.md](.github/copilot-instructions.md) tells Copilot Chat to:

- **Prefer remote MCP** for conversational queries, AI-driven analysis, NL→DQL, Davis CoPilot/Analyzers.
- **Prefer `dtctl`** for declarative `apply` / `diff` / `history` / `restore`, document `share` / `unshare`, custom output formats, and anything you want to **see** scroll past in the terminal.
- **Follow your lead** when both fit. You can always nudge the agent explicitly: *"use `dtctl` for this"* / *"use the MCP server for this"*.

If both paths are configured, the agent will echo both at session start:

```
Active MCP server:    <ServerName> · <env-host>
Active dtctl context: <context>    · <tenant-id>
```

---

## Common workflows

### Pull, edit, push a dashboard

```powershell
# Export current live state to a local file
dtctl get dashboard <DASHBOARD-ID> -o json > temp/dashboards/<DASHBOARD-ID>.json

# Edit the file (or have Copilot Chat edit it for you)

# Diff before applying
dtctl diff -f temp/dashboards/<DASHBOARD-ID>.json

# Apply
dtctl apply -f temp/dashboards/<DASHBOARD-ID>.json
```

### Roll back a workflow

```powershell
dtctl history workflow <WORKFLOW-ID>
dtctl restore workflow <WORKFLOW-ID> --version <N>
```

### Run a DQL query and pipe to a CSV

```powershell
dtctl query "fetch logs | filter loglevel == 'ERROR' | limit 100" -o csv > temp/error-logs.csv
```

### Switch tenants

```powershell
dtctl config get-contexts
dtctl config use-context <other-tenant>
dtctl auth whoami --plain   # verify
```

---

## Further reading

- [`dtctl` repository](https://github.com/dynatrace-oss/dtctl): installation, full command reference, safety levels
- [Remote Dynatrace MCP Server docs](https://docs.dynatrace.com/docs/dynatrace-intelligence/dynatrace-mcp)
- [Local-to-remote MCP migration guide](https://github.com/dynatrace-oss/dynatrace-mcp/blob/main/docs/remote-mcp-migration.md): most authoritative tool comparison table
- [Dynatrace pricing](https://www.dynatrace.com/pricing/): Grail consumption applies to both paths

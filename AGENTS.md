# Hermes Agent - Agent Guide

Instructions for AI coding assistants working in this repository.

This file is intentionally compact because Hermes loads root `AGENTS.md` into
session context automatically. Detailed contributor and subsystem notes live in
`docs/agents/`; open only the file relevant to the task.

## Read Next

- `docs/agents/architecture.md` - repo structure, `run_agent.py`, `cli.py`,
  TUI/dashboard/desktop architecture.
- `docs/agents/extension-points.md` - tools, config, themes, plugins, skills,
  toolsets, dependency policy.
- `docs/agents/runtime-systems.md` - delegation, curator, cron, kanban,
  prompt-caching policy, profiles.
- `docs/agents/pitfalls-and-testing.md` - known footguns and test workflow.
- `docs/agents/contribution-rubric.md` - PR review policy, product intent, and
  the full footprint ladder. Read this for upstream-quality changes, reviews,
  or triage.

## What Hermes Is

Hermes is a personal AI agent that runs the same core across CLI, messaging
gateway, TUI, dashboard, and Electron desktop surfaces. It learns across
sessions through memory and skills, delegates to subagents, runs scheduled
jobs, and can drive a terminal and browser.

Two design constraints dominate:

- **Prompt caching is sacred.** A conversation should keep a byte-stable system
  prompt and role alternation for its lifetime. Do not mutate past context,
  swap toolsets, inject synthetic user turns mid-loop, or rebuild the prompt
  except through explicit context compression.
- **The core is a narrow waist.** Every model tool is paid for on every API
  call. Prefer existing code, CLI commands plus skills, service-gated tools,
  plugins, or MCP servers before adding a new core model tool.

## Working Rules

- Verify the real code path before declaring a bug. Reproduce against current
  `main`, identify the line where behavior manifests, and preserve intentional
  design constraints.
- Keep changes scoped to the request. Avoid opportunistic refactors unless they
  are required to make the fix correct.
- Extend existing infrastructure instead of duplicating managers, registries,
  hooks, or config paths.
- Use `config.yaml` for non-secret behavior settings. `.env` is for secrets
  only: API keys, tokens, passwords.
- Preserve feature intent when hardening behavior. A security fix that destroys
  the feature is the wrong fix.
- Do not add lazy-reading pagination to instructional tools or prompt/skill
  loaders. Agents tend to read page one and miss the rest.
- Plugins live in their own directories and should not special-case core files.
  If a plugin needs a new surface, widen the generic plugin API.
- Do not add outbound telemetry, third-party identifier tagging, or usage
  attribution without an explicit user-facing opt-in gate.

## Project Map

- `run_agent.py` - core agent loop, model calls, prompt assembly, tool dispatch.
- `cli.py` - interactive CLI shell and slash-command handling.
- `hermes_cli/` - CLI subcommands, setup/config helpers, web server, profiles,
  auth, models, plugins, and shared command infrastructure.
- `agent/` - prompt builder, skill command loading, browser/code-exec helpers,
  and other agent runtime modules.
- `tools/` - terminal/file/browser/tool approval implementations.
- `gateway/` - messaging gateway runner and platform dispatch.
- `tui_gateway/` and `ui-tui/` - JSON-RPC bridge and terminal UI frontend.
- `web/` - dashboard frontend.
- `apps/desktop/` - Electron desktop wrapper.
- `apps/bootstrap-installer/` - Tauri installer.
- `plugins/` - bundled plugin examples and plugin categories.
- `skills/` and `optional-skills/` - bundled skills synced into
  `~/.hermes/skills/`.
- `tests/` - unit, integration, and subsystem tests.

Open `docs/agents/architecture.md` before making broad changes to
`run_agent.py`, `cli.py`, TUI, dashboard, or desktop flows.

## Development Environment

Prefer the repo venv:

```bash
source .venv/bin/activate
# or, if this checkout uses it:
source venv/bin/activate
```

Use `scripts/run_tests.sh` for test runs unless you have a specific reason not
to. It probes `.venv`, then `venv`, then the active Python. For focused work,
run the smallest relevant test first, then broaden when touching shared runtime
or cross-surface behavior.

## Architecture Rules

- Agent/tool changes must preserve strict assistant/user role alternation.
- Prompt-builder changes must preserve per-conversation prompt cache stability.
- Tool registry changes must account for schema footprint and service gating.
- Gateway changes must keep approval/control commands out of both message
  guards.
- Profile-aware code must derive paths from profile/Hermes home helpers; never
  hardcode `~/.hermes`.
- Tests must use temp `HERMES_HOME`/profile roots and must not write to the
  user's real Hermes state.

## Capability Footprint

For new capability, choose the least permanent surface that works:

1. Extend existing code.
2. Add a CLI command plus skill.
3. Add a service-gated tool with a `check_fn`.
4. Implement a plugin.
5. Provide an MCP server/catalog entry.
6. Add a core model tool only when fundamental, broad, and unreachable via the
   above.

Read `docs/agents/contribution-rubric.md` and
`docs/agents/extension-points.md` before adding a model tool, config surface,
plugin API, or dependency.

## Config Rules

- User behavior settings belong in `~/.hermes/config.yaml`.
- Secrets belong in `~/.hermes/.env`.
- If an internal env var is necessary for process mechanics, bridge from config
  and document config as the user-facing interface.
- Know which loader path you are in before changing config behavior: CLI,
  gateway/web server, and runtime helpers do not all enter through the same
  code.

## Skills And Plugins

- Skills are behavioral instructions, not hidden code. Keep them concise,
  concrete, and scoped to the task domain.
- Bundled skills must be safe to auto-sync and useful without network access
  unless the skill explicitly depends on a service.
- General plugins are loaded through `hermes_cli/plugins.py`; category plugins
  such as memory/model providers have their own registries.
- Third-party product integrations should usually ship as standalone plugins,
  not as new directories in this repository.

## Known Pitfalls

- Do not hardcode `~/.hermes`; use `hermes_constants.get_hermes_home()` or the
  profile-aware helper for the subsystem.
- Do not introduce new `simple_term_menu` usage; it has caused terminal state
  issues. Prefer prompt_toolkit/Rich patterns already in use.
- Do not use ANSI erase-to-EOL (`\033[K`) in spinner/display code; it can wipe
  prompt_toolkit UI state.
- `_last_resolved_tool_names` in `model_tools.py` is process-global. Treat it
  carefully in concurrent/gateway contexts.
- Do not hardcode cross-tool references in schema descriptions. Toolsets and
  plugins make availability dynamic.
- Do not wire in dead code without an E2E path that exercises the behavior.
- Beware squash merges from stale branches: they can silently revert recent
  fixes.

Read `docs/agents/pitfalls-and-testing.md` before touching profile handling,
gateway guards, terminal display, tests, or shared runtime state.

## Testing Guidance

- Prefer behavior/invariant assertions over snapshots that merely detect
  expected churn.
- Do not freeze model lists, config version literals, skill/provider counts, or
  other intentionally changing enumerations.
- For config propagation, security boundaries, remote backends, file/network
  I/O, and resolution chains, exercise real imports and real paths against temp
  state. Mocks alone are not enough.
- Use subprocess-per-test-file isolation for tests that mutate global registries
  or import-heavy runtime state.

## When In Doubt

Read the relevant supporting file in `docs/agents/` instead of guessing. Keep
root context lean, but do not skip the detailed guide when the change touches
the corresponding subsystem.

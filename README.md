<p align="center">
  <img src="docs/branding/modex-header.png" alt="Modex — adaptive coding agents" width="900">
</p>

# Modex

[![CI](https://github.com/OpenCoven/modex/actions/workflows/ci.yml/badge.svg)](https://github.com/OpenCoven/modex/actions/workflows/ci.yml)

Modex is an open, Codex-App-style desktop app for running coding agents in your local
repositories. It drives **Claude Code** and **Codex** through their own CLIs (`claude -p`
stream-json and `codex app-server`), using the logins you already have. It never calls a model
API directly and never stores credentials.

```
modex/
├── apps/desktop     @modex/desktop  Electron + React app (the Codex-App clone)
└── packages/core    @modex/core     offline scripted engine used by the demo and tests
```

## The desktop app

![Approval card](docs/screenshots/01-thread-approval.png)

- **Two backends, one UI.** Every thread picks **Codex** or **Claude** in the composer. Codex
  threads talk to `codex app-server` (the same JSON-RPC protocol the official Codex App uses);
  Claude threads talk to `claude -p --output-format stream-json` with permission prompts routed
  over stdio. The model picker works like the Codex App's: a popover listing exactly the models
  the CLI reports (`model/list` for Codex — every current GPT model with its reasoning-effort
  row; the `fable`/`opus`/`sonnet`/`haiku` latest aliases for Claude), default preselected, no
  free-text entry.
- **⚡ Auto routing.** Turn on Auto and a fast judge — TypeSafe's Jev, or a built-in
  heuristic when no key is configured — reads each request before the turn and picks
  the model, reasoning effort, and fast mode within the bounds you set (posture, effort
  ceiling, daily premium-turn budget, confidence floor). Every turn gets a one-line receipt
  explaining the pick; picking a model by hand teaches Auto your preference for that kind of
  task. The key is pasted into Settings (kept in the OS keychain, never in a build) or
  shared with the [`jev` CLI](https://github.com/TypeSafeAI/cli), which Modex drives
  directly when it is installed. The coding turn itself still runs only through the
  CLIs. See [docs/auto-routing.md](docs/auto-routing.md).
- **Projects & threads.** Open any local folder as a project. Each project holds threads;
  threads run **in parallel** and independently, each with its own backend, model, and mode.
- **Worktree threads.** `⑂` starts a thread in a fresh `git worktree` so agents never step on
  each other or on your checkout. If the project ships `scripts/worktree.sh` (as this repo
  does), Modex delegates to it — `new modex-<id>` / `remove` — and the thread lives wherever the
  project's convention says (here: `.worktrees/`). Otherwise it uses a `modex/<id>` branch under
  `~/.modex/worktrees/`.
- **Live transcript.** Assistant replies stream in token by token and render as Markdown;
  tool calls render as collapsible items (`$ command`, `read`, `edit …`) with output and timing.
  Model reasoning — Codex reasoning summaries, Claude extended thinking — streams into a
  collapsible **Thinking** item that folds to "Thought for Ns" when the model moves on. Codex
  threads request `model_reasoning_summary = "detailed"`; Claude Code's CLI reports that the
  model thought but redacts the text, so those rows carry timing only.
- **Inline approvals.** When the mode requires it, the thread pauses on a card showing the
  exact command or patch. *Approve*, *Deny*, or *Always* (trusts that command prefix for
  the thread). *Stop* cancels a running turn and any pending approvals.
- **Changes panel.** Working-tree status for the thread's directory with per-file diffs and
  a one-click revert (confirmed first).
- **Modes**, mirroring Codex, mapped onto each CLI's native policy:

  | Mode | Codex (`approvalPolicy` / sandbox) | Claude (`--permission-mode`) |
  | --- | --- | --- |
  | Chat | `untrusted` / read-only | `manual`, edits disallowed |
  | Agent | `on-request` / workspace-write | `acceptEdits` |
  | Agent (full access) | `never` / danger-full-access | `bypassPermissions` |
  | **Plan** toggle | read-only + plan instructions | `plan` |

- **Shortcuts.** `⌘N` new thread · `⇧⌘N` thread in a worktree · `⌘⏎` (or `⏎`) send ·
  `⇧⌘P` plan · `⌘.` stop · `⌘J` changes panel.

- **Settings.** Default backend and mode, CLI executables (with a live health check), default
  model per backend, or the offline mock engine for demos.

Each CLI applies its own instruction files (`AGENTS.md`, `CLAUDE.md`), hooks, MCP servers, and
skills exactly as it would in a terminal.

### Run it

```sh
npm install
npm run build
npm run desktop          # requires `claude` and/or `codex` on PATH, already logged in
```

Offline demo (no key needed) — seeds a throwaway project and a scripted agent, then
captures screenshots:

```sh
npm run screenshot -w @modex/desktop -- --screenshot=/tmp/modex-shots --demo-answer=yes
```

Development loop: `npm run dev -w @modex/desktop` (Vite on :5178) and
`MODEX_DEV_URL=http://localhost:5178 npm run desktop` in another shell.

## Offline engine (`@modex/core`)

The "mock" backend is a small in-process agent loop with a scripted model. It exists so the UI
can be demoed, screenshotted, and tested without any CLI or account: tools (`shell`,
`read_file`, `list_dir`, `write_file`, Codex-format `apply_patch`), the approval-policy ×
sandbox-mode matrix, a macOS Seatbelt profile, and JSONL sessions.

## Tests

```sh
npm test         # core (patch engine, policy, agent loop) + desktop (backends, runner, git, store)
npm run test:e2e # Playwright drives the real Electron window: ⌘N, type, ⌘⏎, Approve, ⇧⌘P, ⌘J, relaunch
```

The desktop suite drives both CLI backends against scripted fake processes (Claude
`control_request`/`control_response`, Codex JSON-RPC requests, notifications, and approval
server-requests) and the runner end to end with the offline engine: approvals pause and resume
a turn, denial leaves the tree untouched, `Stop` interrupts, two threads run concurrently, and
worktree threads are created and removed. The e2e suite (`apps/desktop/e2e/app.spec.ts`)
launches the packaged app with a seeded `MODEX_HOME` and the offline mock backend, so it needs
no CLI login: it asserts the approval card blocks the edit until *Approve* is clicked, that the
Changes panel shows the resulting diff, and that the thread is restored after a relaunch.

## Working on Modex

Every session works in its own worktree: `scripts/worktree.sh new <name>` creates
`.worktrees/<name>` on a branch of that name from `origin/main`; the primary checkout stays on
`main`. See `AGENTS.md` (also loaded by Claude Code via `CLAUDE.md`) for the full convention.

## Branch protection

`main` is meant to accept only signed commits that passed the CI check. The ruleset lives at
`.github/rulesets/main.json` (import it under *Settings → Rules → Rulesets → Import*, or
`gh api -X POST repos/OpenCoven/modex/rulesets --input .github/rulesets/main.json`). GitHub
only enforces rulesets on private repositories for paid org plans; on the Free plan the repo
must be public for the rules to apply.

## Not (yet) here

Codex Cloud tasks, scheduled automations, image attachments, installers/code signing. Also: Modex deliberately has no HTTP-API mode — if a CLI is not installed or logged in, the thread says so.

## License

MIT

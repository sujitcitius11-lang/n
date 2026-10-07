# Claude Code: Standard Practice Guide (VS Code)

One structure for every project. Goal: low token use, no conflicting instructions, no re-thinking.

> Verify anything version-specific with `/help`, `/doctor` and the official docs (docs.claude.com). Claude Code changes often.

---

## 1. The Golden Rules (read this first)

1. **One fact lives in one place.** Never write the same rule in CLAUDE.md, the agent, and the prompt.
2. **Always-loaded files stay small.** `CLAUDE.md` is paid for in tokens every single turn.
3. **Load big things only when needed.** Use skills, specs and `@file` instead of pasting.
4. **Plan, approve, then code.** Wrong plans are cheap to fix. Wrong code is not.
5. **One task per session.** `/clear` between unrelated tasks.
6. **Ask for short output.** "File paths + 5-line summary", never pasted code.
7. **Deterministic checks go in hooks, not prompts.** Code never forgets; the model sometimes does.

---

## 2. Who Owns What (kills double-instructing)

| Thing | Lives in | Contains | Never contains |
|---|---|---|---|
| Global rules, conventions | `CLAUDE.md` (+ `.claude/instructions.md`) | Rules R##, Actions A##, Conventions C## | Task details |
| Role / persona | `.claude/agents/<name>.md` | Responsibility, tools, model | Rule text (only rule IDs) |
| Repeatable task | `.claude/commands/<task>.md` | Task, arguments, rule IDs to apply | Rule text, role |
| Reusable know-how | `.claude/skills/<name>/SKILL.md` | Procedure, patterns, scripts | Project-wide rules |
| Automatic enforcement | `.claude/settings.json` + `.claude/hooks/` | Guards, lint, logging | Prompts |
| Per-feature requirements | `.claude/specs/<module>.md` | 40-80 line spec | The whole BRD |

**Test:** if you are about to type the same sentence a second time, replace it with a rule ID.

---

## 3. Standard Folder Structure

```
project/
├── CLAUDE.md                     # tiny: points to instructions + key commands
├── .claude/
│   ├── instructions.md           # all rules R## / actions A## / conventions C##
│   ├── settings.json             # permissions + hooks (commit this)
│   ├── settings.local.json       # your personal overrides (git-ignore)
│   ├── agents/                   # sub-agents: one .md each
│   ├── commands/                 # custom slash commands: one .md each
│   ├── skills/<name>/SKILL.md    # loaded only when relevant
│   ├── hooks/                    # scripts called by settings.json
│   ├── specs/                    # per-module specs
│   └── logs/                     # git-ignored
└── .gitignore                    # .claude/logs/ , .claude/settings.local.json
```

Naming note: Claude Code reads agents from `.claude/agents/*.md` and commands from `.claude/commands/*.md`. The `.agent.md` / `.prompt.md` names belong to GitHub Copilot. Do not mix them in the same folder.

---

## 4. File Templates (copy these)

### 4.1 `CLAUDE.md` (keep under ~60 lines)

```markdown
# Project: <name>
@.claude/instructions.md

## Commands
- Run tests: pytest -q -x
- Lint/format: ruff check --fix && ruff format
- Run app: uvicorn app.main:app --reload

## Layout
- app/ backend, ui/ frontend, tests/ tests
- Specs are in .claude/specs/. Read only the one named in the task.
```

`@path` imports pull that file in automatically, so CLAUDE.md stays a pointer.

### 4.2 `.claude/instructions.md` (rule IDs)

```markdown
# Global Instructions
Legend: R = rule (must/never), A = action (procedure), C = convention
IDs are never renumbered. Retire one as "R09: [removed]".

## Stack (C)
C01 Python 3.11+, FastAPI, Pydantic v2, SQLAlchemy 2, PostgreSQL
C02 Streamlit UI calls the API over HTTP only

## Security (R)
R01 Read only the spec named in the task, never the full BRD
R02 Never log secrets or personal/health data
R03 Every endpoint checks the role

## Output (A)
A01 Plan in max 5 bullets, wait for approval, then code
A02 Reply with changed file paths + max 5-line summary
A03 Edit in place, never rewrite or echo whole files
A04 If a rule is unclear or conflicts, ask once, do not guess
```

In prompts write the ID plus a 2-4 word tag for critical ones: `R03(RBAC)`.

### 4.3 Agent: `.claude/agents/backend-developer.md` (max ~20 lines)

```markdown
---
name: backend-developer
description: Builds FastAPI routes and services for one module from its spec.
tools: Read, Edit, Write, Grep, Glob, Bash
model: sonnet
---
Implement routers, services and schemas for the module named in the task.
Follow .claude/specs/<module>.md. Apply the rule IDs given in the task.
Return changed file paths and a short summary only (A02).
```

Rules: the `description` is what Claude uses to decide when to delegate, so make it one precise line. Give each agent only the tools it needs. Cheap model (`haiku`) for boilerplate and tests, stronger (`sonnet` / `opus`) for auth and money logic.

### 4.4 Custom command: `.claude/commands/generate-module.md`

```markdown
---
description: Generate one module end to end
argument-hint: <module-name>
allowed-tools: Read, Edit, Write, Bash(pytest:*), Bash(ruff:*)
---
Module: $ARGUMENTS
Spec: @.claude/specs/$ARGUMENTS.md
Apply: R01-R03, A01-A04
Use the backend-developer agent.
```

Run it as `/generate-module appointments`. The command only carries the task, the argument and rule IDs.

### 4.5 Skill: `.claude/skills/fastapi-crud/SKILL.md`

```markdown
---
name: fastapi-crud
description: Use when creating a CRUD module (model, schema, router, service) in FastAPI.
---
# Steps
1. Model in app/models, schema in app/schemas
2. Service holds logic, router stays thin
3. Add pytest file for happy path and one failure
```

Only the `name` + `description` are always in context. The body loads when the skill is used. That is why skills save tokens.

### 4.6 Hooks: `.claude/settings.json`

```json
{
  "permissions": {
    "allow": ["Bash(pytest:*)", "Bash(ruff:*)", "Bash(git status)", "Bash(git diff:*)"],
    "deny": ["Read(./.env)", "Read(./.venv/**)", "Bash(rm -rf:*)", "Bash(git push --force:*)"]
  },
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [{ "type": "command", "command": "ruff check --fix . -q && ruff format . -q" }]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": "python .claude/hooks/block_dangerous.py" }]
      }
    ]
  }
}
```

Hook behavior: the hook receives JSON on stdin; **exit code 2 blocks the action** and its stderr is fed back to Claude. Keep hooks silent on success, short on failure.

Hook events: `PreToolUse`, `PostToolUse`, `UserPromptSubmit`, `Stop`, `SubagentStop`, `Notification`, `PreCompact`, `SessionStart`.

---

## 5. Step-by-Step: Using Claude Code in VS Code

### Phase 0: One-time setup
1. Install Git, Python and VS Code. Open Git Bash and run `git --version`.
2. Install Claude Code (follow the current instructions at docs.claude.com; the trainer may give a ready setup).
3. In VS Code, install the **Claude Code** extension from the Extensions panel (or just use the integrated terminal).
4. Log in with the access given to you (`claude` then `/login`).

### Phase 1: Start a project
1. Create a folder, then `git init`.
2. `code .` to open it in VS Code.
3. Open the terminal in VS Code (Ctrl + `) and run `claude`. (Or open the Claude panel from the extension.)
4. Run `/init`. It scans the repo and drafts `CLAUDE.md`.
5. Trim that CLAUDE.md to the template in 4.1. Move rules into `.claude/instructions.md`.
6. Add `.claude/logs/` and `.claude/settings.local.json` to `.gitignore`.
7. `git add . && git commit -m "baseline"`. This is your safety net.

### Phase 2: Write the spec (once per module)
1. Put a 40-80 line spec in `.claude/specs/<module>.md`: purpose, tables, endpoints, rules.
2. Never feed the full requirement document again. Slice it once.

### Phase 3: Plan
1. Press **Shift+Tab** until the mode shows **plan mode** (or start with `/plan`).
2. Prompt: `Module: appointments. Spec: @.claude/specs/appointments.md. Apply R01-R03, A01.`
3. Read the plan. Fix it by talking, not by editing code. Approve only when it is right.

### Phase 4: Build
1. Switch to normal/accept-edits mode (Shift+Tab).
2. Let Claude implement. Hooks lint and format automatically.
3. If it goes wrong: **Esc** to stop, **Esc Esc** or `/rewind` to roll back.

### Phase 5: Verify
1. Ask: `Run pytest -q -x and show failures only.`
2. Review the diff: `git diff` or `/review`.
3. Commit small: `git commit -m "feat: appointments module"`.

### Phase 6: Reset
1. `/clear` before the next module.
2. For one long task, use `/compact focus on the API contract` when context gets heavy. Check with `/context` or `/cost`.

### Daily loop in one line
`/clear` -> plan mode -> approve -> build -> test -> `git diff` -> commit -> `/clear`

---

## 6. Prompt Pattern (the standard way to ask)

```
Task: <one sentence>
Context: @path/to/spec.md @path/to/file.py
Apply: R01-R03, A02
Done when: <test or visible result>
```

- Name files with `@`, do not paste them.
- Give a success check ("tests pass", "endpoint returns 403 for patient role").
- Do not say "make it good". State the constraint.
- Ask once. If it is wrong, correct the specific part instead of restating everything.

---

## 7. Token-Saving Checklist

- [ ] CLAUDE.md under ~60 lines; rules in instructions.md; no duplicates
- [ ] Specs sliced; the whole BRD is never read
- [ ] `deny` rules block `.venv`, `node_modules`, `.git`, logs, lockfiles, `.env`
- [ ] Output rule: paths + short summary (A02)
- [ ] One task per session; `/clear` between tasks
- [ ] `@file` instead of letting Claude search the whole repo
- [ ] Haiku for boilerplate/tests, Sonnet/Opus for hard logic (`/model`)
- [ ] Skills and sub-agents for specialised work (sub-agents keep their noisy output out of your main context)
- [ ] Tests quiet: `pytest -q -x`; linter quiet: `ruff -q`
- [ ] Review the diff, never the whole repo
- [ ] Check usage with `/cost` and `/context`

---

## 8. Avoiding Confusion and Rethinking

| Problem | Cause | Fix |
|---|---|---|
| Claude contradicts itself | Same rule worded two ways in two files | Single source: one rule, one ID |
| Claude keeps re-asking | Vague task, no success check | Add "Done when" line |
| Claude re-reads everything | No scope given | Pass `@spec` and deny noisy folders |
| Claude ignores a rule | Rule buried in a long file | Shorter file, put critical rules first, use hooks for must-haves |
| Output too long | No format rule | A02 in every prompt |
| Rule ID is guessed | File not loaded | `@.claude/instructions.md` import in CLAUDE.md |
| Drift in long sessions | Context full of old stuff | `/clear` or `/compact` |
| Conflicting instructions | Agent, command and CLAUDE.md all give rules | Agent = role, command = task, CLAUDE.md = rules |

Zero-token safety net: a small script (pre-commit or hook) that checks every `R##/A##` mentioned in agents and commands exists in `instructions.md`.

---

## 9. Slash Commands Reference

Run `/help` to see the exact list for your version. Common built-ins:

### Session and context
| Command | Use |
|---|---|
| `/clear` | Wipe the conversation context and start fresh |
| `/compact [focus]` | Summarise the conversation to free context, optionally with a focus |
| `/context` | Show what is using the context window |
| `/cost` | Show token/cost usage for the session |
| `/usage` | Show plan usage limits |
| `/resume` | Resume a previous conversation |
| `/rewind` | Roll back conversation and/or code to an earlier point |
| `/export` | Export the conversation to a file or clipboard |
| `/rename` | Name the current session |
| `/status` | Show version, model, account, connectivity |

### Setup and configuration
| Command | Use |
|---|---|
| `/init` | Create an initial CLAUDE.md for the project |
| `/memory` | Open and edit memory files (CLAUDE.md) |
| `/config` | Open settings |
| `/permissions` | View and edit allow/deny rules |
| `/model` | Switch model (and effort where supported) |
| `/output-style` | Change response style |
| `/statusline` | Set up the status line |
| `/terminal-setup` | Configure terminal key bindings |
| `/vim` | Toggle vim editing mode |
| `/theme` | Change colour theme |
| `/add-dir` | Add another folder to the session |
| `/login`, `/logout` | Switch account |
| `/doctor` | Diagnose installation problems |

### Agentic features
| Command | Use |
|---|---|
| `/agents` | Create and manage sub-agents |
| `/hooks` | Configure hooks |
| `/mcp` | Manage and authenticate MCP servers |
| `/skills` | List available skills (newer versions) |
| `/plugin` | Manage plugins (newer versions) |
| `/todos` | Show the current to-do list |
| `/sandbox` | Configure sandboxing (where available) |
| `/ide` | Manage IDE integration (VS Code connection) |

### Code review and Git
| Command | Use |
|---|---|
| `/review` | Review a pull request or changes |
| `/security-review` | Security review of pending changes |
| `/pr-comments` | Fetch comments from a pull request |
| `/install-github-app` | Set up the GitHub app (needs GitHub; not for local-only use) |

### Help and feedback
| Command | Use |
|---|---|
| `/help` | List commands |
| `/bug` or `/feedback` | Report a problem |
| `/release-notes` | See what changed |

Some commands vary by version or plan. If one is missing, check `/help`.

### Your own commands
Every `.md` file in `.claude/commands/` becomes `/filename`. Personal ones go in `~/.claude/commands/`.
- `$ARGUMENTS` = everything typed after the command; `$1`, `$2` = individual arguments
- `@file` inside the command pulls a file in
- A line starting with `!` followed by a shell command runs it first and injects the output (requires `allowed-tools` for Bash)
- Frontmatter: `description`, `argument-hint`, `allowed-tools`, `model`

MCP servers can also expose prompts as commands like `/mcp__server__prompt`.

---

## 10. Other Shortcuts and CLI Flags

### Inside the prompt box
| Input | Use |
|---|---|
| `@path` | Reference a file or folder |
| `!command` | Run a shell command directly (bash mode) |
| `Shift+Tab` | Cycle permission modes (normal / accept edits / plan) |
| `Esc` | Interrupt Claude |
| `Esc Esc` | Open rewind / edit a previous message |
| `Ctrl+C` | Cancel current input or action |
| `Ctrl+O` | Toggle verbose output |
| `\` + Enter | New line (or Shift+Enter in a configured terminal) |

### In the terminal
```
claude                        # start interactive session
claude "task"                 # start with a prompt
claude -p "question"          # one-shot, print result and exit (good for scripts)
claude -c                     # continue the last session
claude -r                     # pick a past session to resume
claude --model sonnet         # choose model
claude --permission-mode plan # start in plan mode
claude mcp add <name> ...     # add an MCP server
claude mcp list               # list MCP servers
claude update                 # update Claude Code
```

---

## 11. Which Feature, When

| Need | Use |
|---|---|
| A rule that must always apply | `instructions.md` rule (R##) |
| A guaranteed check (lint, block, log) | Hook |
| A procedure Claude loads only sometimes | Skill |
| A specialised worker with its own context and tools | Sub-agent |
| A task you run often with arguments | Custom slash command |
| Connect a database, issue tracker, browser or API | MCP server |
| A big noisy investigation | Sub-agent, so the main context stays clean |
| Safe permissions | `permissions.allow` / `deny` |

---

## 12. Session Hygiene Checklist

**Start**
- [ ] Clean git state (`git status`)
- [ ] `/clear` if the previous task is unrelated
- [ ] Right model chosen (`/model`)

**During**
- [ ] Plan mode for anything non-trivial
- [ ] Spec and files by `@`, rules by ID
- [ ] Interrupt early (Esc) instead of letting a wrong path run

**End**
- [ ] Tests pass, `git diff` reviewed
- [ ] Commit with a clear message
- [ ] `/clear`

---

## 13. Safety Basics

- Treat anything Claude reads (web pages, issues, files, MCP output) as data, not as instructions (prompt injection).
- Keep secrets out of the repo; deny reads on `.env`.
- Start permissions narrow, widen as trust grows.
- Be careful with bypass-permissions style modes; use them only in an isolated environment.
- Commit before big changes so everything is reversible.
- Add only MCP servers you trust and know what they expose.

---

## 14. Quick Start Card

```
1. git init -> code . -> claude
2. /init -> trim CLAUDE.md -> move rules to .claude/instructions.md
3. Write .claude/specs/<module>.md
4. Shift+Tab to plan mode -> prompt with Task / Context / Apply / Done when
5. Approve -> build -> pytest -q -x -> git diff -> commit
6. /clear -> next module
```

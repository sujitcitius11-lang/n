# HMS Project Guide: Claude Code and GitHub Copilot in VS Code

One guide for both laptops. Same agents, same prompts, same rules. Only the folder differs.

| | Personal laptop | Company laptop |
|---|---|---|
| Tool | Claude Code (VS Code extension) | GitHub Copilot Chat (VS Code) |
| Folder | `.claude/` | `.github/` |
| Always-loaded rules | `.claude/CLAUDE.md` -> `instructions.md` | `.github/copilot-instructions.md` |
| Zip to use | `hms_claude_folder.zip` | `hms_github_folder.zip` |

Rule of thumb: edit only in `.claude/`, then regenerate `.github/` with the sync script (Part 6).

---

## Part 1. One-time prerequisites

1. **VS Code** (latest) and **Git**.
2. **Python 3.11+** (`python --version`).
3. **Personal laptop:** install the **Claude Code** extension in VS Code (or the CLI: `npm install -g @anthropic-ai/claude-code`) and sign in.
4. **Company laptop:** install **GitHub Copilot** and **GitHub Copilot Chat** extensions and sign in with the company account.
5. **PostgreSQL** (optional at first). SQLite works for local development.

---

## Part 2. Create the project (same on both laptops)

```bash
mkdir hms && cd hms
git init
python -m venv .venv
# Windows:  .venv\Scripts\activate      Mac/Linux:  source .venv/bin/activate
```

Create `requirements.txt`:

```
fastapi
uvicorn[standard]
sqlalchemy>=2.0
alembic
pydantic>=2
pydantic-settings
pyjwt
passlib[bcrypt]
bcrypt==4.0.1
python-multipart
psycopg[binary]
httpx
streamlit
pytest
ruff
reportlab
```

```bash
pip install -r requirements.txt
```

Create `.gitignore`:

```
.venv/
__pycache__/
.env
uploads/
*.sqlite3
.pytest_cache/
.ruff_cache/
```

Create `.env` (never commit it):

```
DATABASE_URL=sqlite:///./hms.db
SECRET_KEY=change-me-to-a-long-random-string
API_URL=http://localhost:8000
```

Unzip the folder for your laptop **into the project root** so you get `hms/.claude/` (or `hms/.github/`).

Check it is valid:

```bash
python .claude/hooks/validate_config.py        # company laptop: .github/hooks/validate_config.py
```

No output means everything is fine. Any output lists the problem.

Copy your BRD into the project if you want to re-slice it later (optional). The module specs already exist in `specs/`.

---

## Part 3A. Claude Code in VS Code (personal laptop)

### Open and verify

1. Open the `hms` folder in VS Code.
2. Open Claude Code (the Claude icon, or run `claude` in the VS Code terminal).
3. Check each piece loaded:
   - Type `/agents`: you should see your 6 agents (code_generator, backend_developer, ...).
   - Type `/hooks`: you should see PreToolUse, PostToolUse, Stop hooks from `settings.json`.
   - Ask: `Which rules does R11 define?` It should answer from `instructions.md`. If it guesses, type `@.claude/instructions.md` once and check `.claude/CLAUDE.md` contains `@instructions.md`.

### Use an agent

Name the agent and the module in plain words:

```
Use the backend_developer agent for module appointments.
```

Or call it directly with `@`: type `@` and pick the agent from the list. The agent already knows its role. You only give the module.

### Use a prompt

Prompts are files. Attach one with `@` and fill the variable after it:

```
@.claude/prompts/build_backend.prompt.md module=appointments
```

The prompt picks the agent, the spec, the skills and the rule IDs for you.

### Skills

Skills load on their own when the task matches their description (for example `jwt-rbac` when you work on auth). You do not call them. To force one: `Use the streamlit-page skill.`

### Hooks (automatic)

| When | What happens |
|---|---|
| Before Read/Grep/Glob/Bash | Blocks `.venv`, `__pycache__`, `.git`, logs, uploads, `.env`, and dangerous commands like `rm -rf` or `git push --force` |
| After Write/Edit | `ruff` formats and fixes only that `.py` file; silent unless it fails |
| After every tool and at Stop | One JSON line is added to `.claude/logs/<date>.jsonl` (no file contents) |

If a hook blocks something, the message tells the agent what to do instead. Just continue.

### Git safety net (once)

```bash
python .claude/hooks/pre_commit.py --install
```

Every `git commit` now scans for secrets, runs ruff, validates the config and runs tests for changed modules.

### See where tokens went

Open the newest file in `.claude/logs/`. The `Stop` lines show session token totals (input, output, cache). Compare modules and agents weekly.

---

## Part 3B. GitHub Copilot in VS Code (company laptop)

### Open and verify

1. Open the `hms` folder in VS Code (the `.github/` folder must be at the root).
2. Open Copilot Chat (Ctrl+Alt+I).
3. Check:
   - Click the **agent / mode dropdown** at the bottom of the chat box. Your 6 custom agents should be listed. If not, open Settings and search `chat agent` / `chat.agent`, and check that custom agents are enabled and your Copilot version is recent.
   - Type `/` in the chat box: your prompts (`build_backend`, `generate_tests`, ...) should be listed.
   - Ask: `What does R11 say?` It should answer from `copilot-instructions.md`. If not, check Settings for "use instruction files" (`chat.includeApplyingInstructions` / `github.copilot.chat.codeGeneration.useInstructionFiles`) and reload the window.
4. Open any `.github/agents/*.agent.md`. Check the `tools:` and `model:` lines. Model names must match what appears in your model picker. Edit the file if needed. If you change them everywhere, change `MODELS` in `sync_github.py` instead.

### Use an agent

1. Pick the agent in the dropdown (for example `backend_developer`).
2. Type the task with the module name:

```
module: appointments. Implement per spec.
```

The agent's role and constraints are already attached.

### Use a prompt

1. In the chat box type `/build_backend` (the name of the prompt file without `.prompt.md`).
2. Put the module name after it, for example `/build_backend appointments`. The prompt uses a `${input:module}` variable, and VS Code fills it from what you type or asks for it.
3. Send. The prompt switches to the right agent by itself.

### Skills

If your Copilot version supports Agent Skills, the folders in `.github/skills/` load when the task matches. If you do not see them used, attach one by hand: type `#file` and choose `.github/skills/jwt-rbac/SKILL.md`.

### Hooks

Copilot hook support depends on your version, so do not rely on it. Use the git hook (works everywhere):

```bash
python .github/hooks/pre_commit.py --install
```

Run the same checks by hand after an edit if you want them early:

```bash
ruff check --fix backend/app/routers/appointments.py
pytest -q -x --tb=short tests/test_appointments.py
```

### Always attach the right context

Copilot does not read files you do not point at. Attach only what is needed with `#file`:

```
#file:.github/specs/appointments.md
```

---

## Part 4. Build the project, step by step

Do **one module per chat**. Start a new chat (Claude Code: `/clear`; Copilot: new chat) before each module.

### Step 0. Bootstrap the core (once)

Use `backend_developer` and send:

```
Create the project skeleton per C01-C06: backend/app/core/{config,database,security,audit}.py,
backend/app/main.py with /health, frontend/api_client.py, tests/conftest.py, alembic init.
Use skills jwt-rbac, audit-logging, pytest-api-tests. Reply per A02.
```

Run it:

```bash
uvicorn backend.app.main:app --reload     # API on http://localhost:8000/docs
streamlit run frontend/app.py             # UI on http://localhost:8501 (after frontend_developer creates it)
```

### Step 1. For each module, run these 5 prompts in order

| # | Prompt | Agent | What you get |
|---|---|---|---|
| 1 | `generate_code` | code_generator (cheap model) | models, schemas, CRUD skeleton |
| 2 | `build_backend` | backend_developer | routes, services, RBAC, audit, business rules |
| 3 | `generate_tests` | test_generator (cheap model) | `tests/test_<module>.py` |
| 4 | `build_frontend` | frontend_developer | `frontend/pages/<module>.py` + API client calls |
| 5 | `review_diff` | code_reviewer | issues list, max 10 lines |

Exact messages:

**Claude Code**
```
@.claude/prompts/generate_code.prompt.md module=patients
@.claude/prompts/build_backend.prompt.md module=patients
@.claude/prompts/generate_tests.prompt.md module=patients
@.claude/prompts/build_frontend.prompt.md module=patients
@.claude/prompts/review_diff.prompt.md scope=staged
```

**Copilot**
```
/generate_code patients
/build_backend patients
/generate_tests patients
/build_frontend patients
/review_diff staged
```

After step 5: fix any [BLOCKER] or [MAJOR] issues (send them back to `backend_developer`), run the tests, then commit.

```bash
pytest -q -x --tb=short
git add . && git commit -m "feat: patients module"
```

### Step 2. Module order

1. auth_users
2. patients
3. doctors_departments
4. appointments
5. prescriptions
6. lab_reports
7. billing
8. pharmacy
9. reports

Each later module depends on the earlier ones (appointments need patients and doctors, billing needs patients, pharmacy needs prescriptions).

### Step 3. Database migrations

When models change: ask `backend_developer`: `Create the Alembic migration for module patients (R25).` Then run:

```bash
alembic upgrade head
```

Switch to PostgreSQL later by changing `DATABASE_URL` in `.env`.

---

## Part 5. The files and what each one is for

| File | Purpose | Loaded when |
|---|---|---|
| `instructions.md` / `copilot-instructions.md` | All rules (R), actions (A), conventions (C) | Always |
| `agents/*.agent.md` | Role + constraints (max 20 lines) | When the agent is used |
| `prompts/*.prompt.md` | Task template: module, spec, skills, rule IDs | When you run it |
| `skills/*/SKILL.md` | Code patterns (CRUD, JWT, tests, ...) | When the task matches |
| `specs/*.md` | One module's tables, endpoints, roles, rules | When attached or read by the agent |
| `hooks/*.py` | Guard, lint, log, pre-commit, validate, sync | Automatic or manual |
| `logs/*.jsonl` | Activity + token totals (git-ignored, 7 days) | Never loaded |

**Rule IDs:** prompts and agents only say `R10-R16`, `A02` and so on. The text lives once in `instructions.md`. Never renumber an ID. To retire one, write `R09 [removed]`.

---

## Part 6. Changing things safely

1. Edit only inside `.claude/` (agents, prompts, rules, specs, skills).
2. Check:
   ```bash
   python .claude/hooks/validate_config.py
   ```
   It fails if an agent is over 20 lines, has wrong metadata keys, or uses a rule ID that does not exist.
3. Regenerate the Copilot folder:
   ```bash
   python .claude/hooks/sync_github.py
   ```
4. Commit both folders.

**Add a rule:** append a new ID at the end of its section group in `instructions.md` (for example `R17 ...`), then cite `R17` where needed.
**Add an agent:** copy an existing `.agent.md`, keep it at 20 lines or fewer, keep exactly the 6 metadata keys.
**Re-slice the BRD:** run `slice_spec` (`@.claude/prompts/slice_spec.prompt.md brd_file=BRD.txt` or `/slice_spec BRD.txt`), then review the specs it changed.

---

## Part 7. Token-saving habits

1. New chat per module.
2. Attach only `specs/<module>.md`, never the full BRD (R01).
3. Use the cheap model agents (`code_generator`, `test_generator`) for boilerplate and tests.
4. Ask for file paths and a 5-line summary, never pasted code (A02).
5. Review the diff, not the repo (`review_diff`).
6. Run tests quietly: `pytest -q -x --tb=short`.
7. Keep the order: rules, then spec, then task. The beginning of each chat stays the same, which helps caching.
8. Do not paste code into chat. Point to the file.
9. Read the logs weekly and look for the module or agent that used the most tokens.

---

## Part 8. Troubleshooting

| Problem | Fix |
|---|---|
| Agent does not appear | Claude Code: files must be in `.claude/agents/` and have frontmatter. Copilot: file must be in `.github/agents/`, name ends `.agent.md`; update Copilot and reload the window |
| Agent ignores a rule ID | Rules were not loaded. Claude Code: check `.claude/CLAUDE.md`. Copilot: check `copilot-instructions.md` and the instruction-files setting. Quick fix: attach the instructions file for that chat |
| Prompt does not show after `/` | Copilot only: file must be in `.github/prompts/` and end `.prompt.md` |
| `${input:module}` stays literal | Type the module after the command (`/build_backend patients`) |
| Wrong `tools:` / `model:` in Copilot | Edit the agent file to the names your version shows, or set `TOOLS` / `MODELS` in `sync_github.py` and re-sync |
| Hook says BLOCKED | Read the message; use the suggested command (for example `pytest -q -x --tb=short`) |
| Hooks not running (Claude Code) | Run `/hooks`; make sure `python` works in the terminal (on Mac/Linux you may need `python3`: change the command in `settings.json`) |
| `validate_config.py` fails | Read the (max 10) lines; they name the file and the problem |
| bcrypt error with passlib | Keep `bcrypt==4.0.1` in `requirements.txt` |
| Agent edits many files | Say: `Touch only this module's files (R02).` |
| Replies too long | Say: `Reply per A02.` |

---

## Part 9. Cheat sheet

```
Start module   : new chat
Claude Code    : @.claude/prompts/<name>.prompt.md module=<module>
Copilot        : /<name> <module>
Order          : generate_code -> build_backend -> generate_tests -> build_frontend -> review_diff
Check config   : python .claude/hooks/validate_config.py
Sync Copilot   : python .claude/hooks/sync_github.py
Install git hook: python <folder>/hooks/pre_commit.py --install
Run API        : uvicorn backend.app.main:app --reload
Run UI         : streamlit run frontend/app.py
Run tests      : pytest -q -x --tb=short
Logs           : .claude/logs/<date>.jsonl
```

## Things to verify on your own machine

Product names and settings change between versions. These items were set up from general knowledge of the tools and are the ones most likely to need a small tweak:

1. Copilot `tools:` and `model:` names (Part 3B step 4).
2. Whether your Copilot version loads `.github/skills/` and supports hooks (this guide gives a fallback for both).
3. Whether your Copilot version spells the metadata key `user-invocable` the same way (check the VS Code custom agents docs; rename it in the agent files if not).

# 1. Prerequisites and Brush-Up: Claude Agentic AI (Claude Code) Training

This covers **Sessions 1-7** of your "Agentic AI with Claude" plan: Claude Code basics, context and prompts, built-in tools, MCP servers, agents and skills, hooks, multi-agent design, repo work with Claude Code, and (later) advanced RAG, evaluation, security, and performance.

Your tools: **Python, VS Code, Git Bash (local Git only, no GitHub), Hugging Face website, and Claude access** given by the trainer (likely Claude Code in the terminal/VS Code, and probably claude.ai too). No setup steps here — just the ideas and skills to brush up on before Monday.

---

## PART A: Skills to Brush Up

### A1. Command Line (Git Bash) — Must Know
Claude Code is a **terminal-based** tool, so comfort in a terminal matters more here than in a typical Python course.

| Command | Meaning |
|---|---|
| `pwd` | Show current folder |
| `ls` / `ls -la` | List files (including hidden ones) |
| `cd folder` / `cd ..` | Move into / out of a folder |
| `mkdir name` | Make a folder |
| `cat file` | Show file content |
| `touch file` | Create an empty file |
| `code .` | Open the current folder in VS Code |
| `history` | Show past commands |
| `clear` | Clear the screen |

### A2. Git Basics (Local Only, No GitHub)
Claude Code reads and works with Git repos (branches, commits, diffs) even without GitHub.

```
git init                      # start tracking a folder
git status                    # what changed?
git add .                     # stage changes
git commit -m "message"       # save a version
git log --oneline             # see history
git diff                      # see exact line changes
git branch feature-x          # create a branch
git checkout feature-x        # switch to it
```
**Why this matters for the training:** Session 6 covers "Creating a repo with Claude Code, Making PRs, PR Reviews with Claude Code, Claude code in comments." A **PR (Pull Request)** is normally a GitHub/GitLab feature (propose changes, get them reviewed, then merge). Since you don't have GitHub, your trainer will likely either (a) demo real PRs on a shared GitHub repo, or (b) simulate the PR idea locally using Git branches and `git diff`/`git log`, with Claude Code acting as the reviewer. Both are useful to understand: the point is "Claude reads your code changes and comments on them," regardless of where they are hosted.

### A3. Python Basics — Should Know
Claude Code writes and edits code for you, but you should be able to **read** Python to check its work.
- Variables, lists, dictionaries, functions, classes, imports, `try/except`.
- Reading a stack trace (error message) to see what broke.
- Basic JSON (`{"key": "value"}`), since configuration files (`claude.md`, MCP configs, skill/hook definitions) are often JSON or Markdown with structured sections.

### A4. Markdown Basics — Should Know
Claude Code's main config file (`CLAUDE.md`) and most of its specs/plans are written in **Markdown**: `#` headings, `-` bullet lists, ` ``` ` code blocks, `**bold**`. If you're comfortable writing a README, you're ready.

### A5. What an "Agent" Already Means to You
You've likely already covered agents conceptually (LLM + tools + loop: think, act, observe, repeat). Claude Code **is** an agent: it reads your request, decides which files to look at, which commands to run, and repeats until the task is done. Everything from your earlier agentic AI training (ReAct, function calling, planning) applies directly here — Claude Code is simply a real, polished product version of those ideas, specialized for coding.

### A6. APIs and JSON — Light Refresh
MCP servers (Session 3) expose **tools** to Claude in a structured way, similar to function calling you've seen before: a tool has a name, a description, and parameters, and Claude decides when to call it. No new concept, just a new wrapper around it.

---

## PART B: Core Vocabulary You'll Hear Constantly

| Term | Simple meaning |
|---|---|
| **Claude Code** | A coding agent you run in your terminal (or inside VS Code) that can read, write, and edit files, run commands, and fix its own mistakes, guided by your instructions |
| **Session** | One continuous conversation with Claude Code, with its own context (memory of what's been discussed and done) |
| **Context** | Everything Claude currently "remembers" in this session: your messages, the files it has read, command outputs |
| **Context window** | The size limit of that memory; very long sessions eventually need to be summarized or restarted |
| **Sandboxing** | Claude Code runs commands in a controlled, limited environment so it can't do unexpected or risky things outside what you've allowed |
| **CLAUDE.md** | A Markdown file in your project root where you write standing instructions: coding style, project structure, dos and don'ts. Claude reads it automatically every session |
| **Plan mode** | A mode where Claude first writes out a **plan** (the steps it intends to take) before touching any files, so you can review and approve it |
| **Spec** | A written description of what you want built — requirements, before code is written |
| **Built-in tools** | Actions Claude Code can already do out of the box: read a file, edit a file, run a bash command, search code, etc. |
| **MCP (Model Context Protocol)** | An open standard that lets Claude connect to external tools and data sources (databases, APIs, services) in one consistent way |
| **MCP server** | A small program that exposes a set of tools/data to Claude following the MCP standard (e.g., a GitHub MCP server, a database MCP server) |
| **Agent (in Claude Code)** | A specialized, reusable configuration of Claude for a certain kind of task — with its own instructions and tool access — that you or Claude can invoke |
| **Agent skill** | A packaged, reusable capability (instructions + sometimes scripts) that an agent can use, like a plug-in |
| **Custom skill** | A skill you write yourself for your team's specific needs |
| **Hook** | A rule that runs automatically at certain points (before/after a tool call, before a commit, etc.) — used to enforce checks or automate steps |
| **Prompt template** | A reusable, fill-in-the-blank prompt structure for a repeated task |
| **Multi-agent system** | Several agents working together, often with one **planner** that breaks down the task and **executor(s)** that carry out the steps |
| **Orchestration** | The logic that decides which agent/tool runs next |
| **RAG** | Giving the model your own documents to search before it answers, instead of relying only on what it was trained on |
| **Chunking** | Splitting documents into smaller pieces before storing them for search |
| **Embedding** | Turning text into a list of numbers that captures its meaning, for similarity search |
| **Vector DB** | A database built to store and search embeddings |
| **Hybrid search** | Combining keyword search and embedding (meaning) search for better results |
| **Reranking** | Re-sorting search results with a more careful (but slower) check, after a fast first search |
| **Guardrails** | Limits and checks that stop an agent from doing something unsafe or out of scope |
| **Observability** | Being able to see what an agent did — logs, traces, reasoning steps — after the fact |
| **PII** | Personally Identifiable Information (name, phone number, ID number, etc.) — needs special care |
| **Prompt injection** | When text the model reads (a document, a webpage, a comment) secretly tries to give it new instructions |
| **Regression testing** | Re-running old tests after a change to make sure nothing that used to work got broken |

---

## PART C: How Claude Code Is Different From "Just Chatting With an LLM"

| Plain chat with an LLM | Claude Code |
|---|---|
| You paste code, it replies with text | It directly reads and edits your real files |
| No memory of your file system | It explores your actual project structure |
| Can't run anything | It runs terminal commands (tests, builds, git) itself |
| One-shot answer | Loops: tries something, checks the result, fixes, tries again |
| No standing rules | `CLAUDE.md` gives it lasting project rules every session |
| No tool ecosystem | MCP lets it plug into databases, issue trackers, APIs, browsers |

**Traditional way (before agentic coding tools):** A developer writes code, runs it, reads the error, searches StackOverflow or docs, edits, repeats — all by hand. Claude Code automates much of that loop, while still asking for your review at key points (especially in Plan Mode).

---

## PART D: What to Expect Session by Session (From Your Sheet)

| Session | Mode | Main focus |
|---|---|---|
| 1 | Virtual | What & why Claude Code, Claude LLM family, basic setup, sessions & context, native sandboxing, IDE integration |
| 2 | Virtual | Context and prompt engineering basics (types of prompts), initializing Claude Projects, `CLAUDE.md`, plan mode and specs |
| 3 | Virtual | Built-in tools, understanding & using MCP servers, built-in MCP servers, creating a custom MCP server |
| 4 | Virtual | What & why of agents, creating and using an agent, agent skills, adding custom skills |
| 5 | Virtual | Using agent skills as commands, hooks, building prompt templates, multi-agent system design (planner-executor, orchestration) |
| 6 | Virtual (+ In Person) | Virtual: repo creation with Claude Code, making PRs, PR reviews, Claude in comments. In Person: context/memory management at scale, advanced MCP (custom servers, enterprise integration), tooling/function-calling, production-grade agent design (observability, guardrails) |
| 7 | Virtual (+ In Person) | Virtual: what & why RAG, chunking, embeddings, vector DB, advanced prompt engineering (chaining, reflection, structured outputs). In Person: advanced RAG (hybrid search, reranking, evaluation), evaluation/benchmarking frameworks, security & compliance, performance & cost optimization, AI system design case studies |

Your trainer may teach in a different order, or combine sessions, but this is the general shape.

---

## PART E: Self-Check (Answer in Your Head)

1. What's the difference between a chat LLM and an agentic coding tool like Claude Code?
2. What does `CLAUDE.md` do, and when does Claude read it?
3. What is "Plan Mode" for?
4. What is MCP, in one sentence?
5. What's the difference between a "tool," a "skill," and an "agent" in this context?
6. What is a hook, and give one example of when you'd use it?
7. In a multi-agent system, what does the "planner" do versus the "executor"?
8. What is the difference between chunking and embedding?
9. Why is prompt injection a security concern for an agent that reads external content?
10. What is regression testing, and why does it matter for AI systems that get updated often?

**Answers**
1. A chat LLM only replies with text; an agentic tool reads/edits real files and runs commands in a loop until the task is done.
2. It holds standing project rules and context that Claude reads automatically at the start of every session.
3. So Claude writes out its intended steps first, and you can review/approve before it changes anything.
4. MCP is a standard way to connect an AI model to external tools and data sources.
5. A tool is one action Claude can take; a skill is a packaged, reusable capability (often built from tools + instructions); an agent is a configured "persona" with its own instructions and tool access for a kind of task.
6. A hook is a rule that runs automatically at a certain point, e.g., "run the linter after every file edit" or "block commits that contain a secret key."
7. The planner breaks the big task into steps/sub-tasks; the executor(s) actually carry out each step.
8. Chunking splits a document into smaller pieces; embedding turns text into numbers representing meaning, for search.
9. Because text the agent reads (a webpage, a file, a comment) could contain hidden instructions trying to hijack its behavior, so the agent must not blindly trust external content as commands.
10. It means re-running old tests after a change to confirm nothing that worked before has broken; it matters because models, prompts, and tools change often, and silent regressions are easy to miss.

---

## PART F: Mindset Tips for This Training

- **Think in terms of "give Claude the right context," not "write the perfect prompt."** Good `CLAUDE.md` files, clear specs, and relevant files matter more than clever wording.
- **Use Plan Mode on anything non-trivial.** Reviewing a plan before code changes saves time versus undoing a wrong change.
- **Treat MCP servers like plugging in a new tool, not magic.** Ask yourself: what data or action does this expose, and should Claude be allowed to use it here?
- **Agents and skills are about reuse.** If you find yourself writing the same long instructions twice, that's a sign it should become a skill.
- **Security topics (Session 7) are not optional extras.** An agent that can edit files and run commands is powerful — guardrails and awareness of prompt injection are part of using it responsibly, not an afterthought.

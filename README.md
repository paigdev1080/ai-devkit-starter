# ai-devkit-starter

A baseline for every new project that uses AI coding agents (Claude Code first). It sets up two layers:

| Layer | Tool | What it does | Scope |
|---|---|---|---|
| **Token saving** | [RTK](https://github.com/rtk-ai/rtk) | Compresses shell output before the agent reads it (`git status` → `rtk git status`) | Global, once per machine |
| | [OpenWolf](https://github.com/cytostack/openwolf) | Local project memory: file/symbol index, status, bug log, session handoff in `.wolf/` | Per project |
| | [Graphify](https://github.com/Graphify-Labs/graphify) | AST knowledge graph of the codebase; the agent runs `graphify query` instead of grep + reading whole files | Per project |
| **Design / Plan** | [Superpowers](https://github.com/obra/superpowers) | Skills workflow: brainstorm → spec → plan → TDD → review → finish | Global plugin |

All four run locally, and **all of their files stay out of git** (see [§4](#4-local-only-ai-layer)). The one exception is Superpowers specs and plans in `docs/superpowers/`, which are committed as project history. The repository itself is tool-agnostic.

All four run locally. None of them sends your code anywhere by default (Graphify sends docs/PDFs/images to an LLM only for semantic extraction, and you can turn that off with `GRAPHIFY_BACKEND=none`).

---

## How the pieces fit together

```
            ┌──────────── Superpowers (HOW we work) ────────────┐
            │ brainstorming → writing-plans → TDD → code review │
            └───────────────────────────────────────────────────┘
                                    │ the agent needs context
          ┌─────────────────────────┼──────────────────────────┐
          ▼                         ▼                          ▼
   Graphify (WHERE)          OpenWolf (WHAT WE KNOW)      RTK (HOW MUCH)
   graph of code structure   memory, decisions, bugs,     trims every shell
   → query, not grep         handoff between sessions     command's output
```

- **Superpowers** decides the process. Specs and plans go in `docs/superpowers/`.
- **Graphify** answers structural questions ("who calls X?") from `graphify-out/graph.json`.
- **OpenWolf** keeps what isn't in the code: conventions, status, past fixes.
- **RTK** makes every command the agent runs cheaper to read.

---

## Prerequisites

| Requirement | Version | Needed by |
|---|---|---|
| Git | any recent | all |
| [Claude Code](https://claude.com/claude-code) | latest | all |
| Node.js | ≥ 24 (Active LTS) | OpenWolf |
| Python | ≥ 3.12 | Graphify |
| [uv](https://docs.astral.sh/uv/) (or pipx) | latest | Graphify |
| Homebrew *or* curl | — | RTK |

```bash
node --version      # v24.x or newer
python3 --version   # 3.12 or newer

# Pin versions per project (optional, read by nvm / fnm / uv / pyenv / mise)
echo "24" > .nvmrc
echo "3.12" > .python-version

# uv, if missing (Linux/macOS); it can also install Python for Graphify
curl -LsSf https://astral.sh/uv/install.sh | sh
uv python install 3.12
```

---

## 1. One-time machine setup

Do this once per developer machine.

### 1.1 RTK (token saving, global)

```bash
# macOS / Linux with Homebrew
brew install rtk
# or: Linux / macOS / WSL
curl -fsSL https://raw.githubusercontent.com/rtk-ai/rtk/refs/heads/master/install.sh | sh

rtk init -g          # install the Claude Code hook globally
rtk gain             # verify: should print a savings dashboard
```

> ⚠️ Another, unrelated `rtk` crate exists on crates.io. If `rtk gain` fails, you installed the wrong one.
> Use `rtk bypass <cmd>` (older versions: `rtk proxy <cmd>`) when you need raw, unfiltered output.

### 1.2 Graphify (CLI + skill)

```bash
uv tool install --python 3.12 graphifyy   # package has two y's; the CLI is `graphify`
# or: pipx install graphifyy
uv tool update-shell           # only if `graphify` is not found on PATH

graphify install               # registers the /graphify skill for Claude Code
```

### 1.3 OpenWolf (CLI)

```bash
npm install -g openwolf
openwolf --version
```

### 1.4 Superpowers (Claude Code plugin)

Inside Claude Code:

```
/plugin install superpowers@claude-plugins-official
```

Alternative marketplace:

```
/plugin marketplace add obra/superpowers-marketplace
/plugin install superpowers@superpowers-marketplace
```

Restart Claude Code after RTK and Superpowers are installed.

---

## 2. Per-project setup

Run this at the root of every new project, and again after every fresh clone. These commands are idempotent.

```bash
# 0. Start from this kit
cp /path/to/ai-devkit-starter/{.gitignore,README.md} .   # or use this repo as a GitHub template
git init                                                    # if not already a repo

# 1. OpenWolf: creates .wolf/, CLAUDE.md, .claude/{rules,commands,skills}, lifecycle hooks
openwolf init --agent claude

# 2. Graphify: build the graph FIRST. Code-only is AST, so no LLM cost
graphify extract . --code-only

# 3. Graphify: write the `## graphify` section to CLAUDE.md + PreToolUse hooks
graphify claude install

# 4. Graphify: git hooks (post-commit/checkout/merge/rewrite) keep the graph fresh
graphify hook install
#    hook install also adds `graphify-out/graph.json merge=graphify` to the TRACKED
#    .gitattributes. We don't commit the graph, so move that rule to the local,
#    untracked .git/info/attributes:
grep -h 'merge=graphify' .gitattributes >> .git/info/attributes 2>/dev/null
sed -i.bak '/merge=graphify/d' .gitattributes && rm -f .gitattributes.bak
[ -s .gitattributes ] || rm -f .gitattributes

# 5. Verify
openwolf status && openwolf operations doctor
graphify hook status   # "merge driver: partially registered (… .gitattributes line missing)" is
                       # expected: the rule now lives in .git/info/attributes
```

Optional: run `graphify cluster-only .` to generate `GRAPH_REPORT.md` and name communities.

Then **start a new Claude Code session** and accept the hook trust prompt. Neither tool auto-approves agent tool calls.

> **Order matters** (official docs):
> - Run `openwolf init` before `graphify claude install`. Both write into `CLAUDE.md` and `.claude/settings.json`, and Graphify appends its section after OpenWolf's.
> - Build the graph before `graphify hook install`. Without `graphify-out/`, the branch/merge/rebase hooks do nothing.
> - Inside Claude Code, `/graphify .` builds a richer graph that also covers docs, PDFs and images, using the assistant as the LLM.

### Pairing RTK with OpenWolf's Bash governor

RTK compresses command output *before* the agent sees it. OpenWolf's Bash governor also replaces large outputs (`bash.governor.mode: "replace"`, threshold 2000 tokens). With both on, output can be trimmed twice. Let RTK own compression and have OpenWolf only suggest:

```bash
jq '.openwolf.bash.governor.mode = "suggest"' .wolf/config.json > .wolf/config.tmp \
  && mv .wolf/config.tmp .wolf/config.json
```

Set it to `"off"` to disable the governor completely. Run `openwolf dashboard` to compare savings.

Everything these commands create (`.wolf/`, `.claude/`, `CLAUDE.md`, `graphify-out/`, …) is git-ignored. Why that is, and what it costs, is covered in [§4](#4-local-only-ai-layer).

---

## 3. Daily workflow

### Starting a feature (Superpowers)

Describe what you want. Superpowers runs these skills automatically:

1. **brainstorming**: asks questions and writes a design spec to `docs/superpowers/specs/`
2. **using-git-worktrees**: gives the work an isolated branch
3. **writing-plans**: writes a step-by-step plan to `docs/superpowers/plans/`
4. **subagent-driven-development** / **executing-plans**: implements the plan
5. **test-driven-development**: red → green → refactor
6. **requesting-code-review** → **finishing-a-development-branch**

Bugs go through **systematic-debugging**. Before anyone says "done", **verification-before-completion** runs.

### Navigating code (Graphify)

```bash
graphify query "how does auth reach the database?"
graphify path "LoginController" "UserRepository"
graphify explain "SessionStore"
graphify update .               # after code changes (AST only, free)
```

Open `graphify-out/graph.html` in a browser to explore the graph visually.

### Memory & handoff (OpenWolf)

```bash
openwolf status                 # installation + hook health
openwolf find <symbol>          # locate a symbol without reading whole files
openwolf dashboard              # token usage, hook health, bug memory
openwolf report
```

In Claude Code, `/handoff` writes a session checkpoint so the next session (or a teammate) can pick up from it.

### Measuring savings (RTK)

```bash
rtk gain                        # totals
rtk gain --history              # per-command
rtk discover                    # commands the agent ran without RTK
```

---

## 4. Local-only AI layer

**Policy:** commit the product, not the tooling. Every file an AI agent or its tooling creates is git-ignored. The repository works the same whether a contributor uses Claude Code, Codex, Gemini, Cursor or no agent at all.

### Why

| Reason | Detail |
|---|---|
| **Machine-specific** | `openwolf init` and `graphify claude install` write hooks with **absolute paths** (`node /home/alice/proj/.wolf/hooks/…`) into `.claude/settings.json`. A committed copy breaks on every other machine. |
| **Regenerable** | `graphify-out/` comes from the code, and `.wolf/anatomy*` indexes the files. Rebuilding them is cheap, and committing them causes whole-file diffs and merge conflicts. |
| **Private by default** | `.wolf/memory.md`, `buglog.json` and session handoffs can quote code, logs or secrets. Superpowers specs can hold half-finished ideas. |
| **Tool-agnostic** | Each agent has its own files (`CLAUDE.md`, `AGENTS.md`, `GEMINI.md`, `.cursor/`, `.kiro/`, …). Committing one tool's files pushes that tool on everyone. |
| **No tool churn** | Upgrading or swapping an agent tool never produces a diff in the product repo. |

### What is committed (the exception)

| Path | Why |
|---|---|
| `docs/superpowers/specs/` | Design specs from **brainstorming**: the *why* behind features. These are reviewable in PRs and act as lightweight ADRs |
| `docs/superpowers/plans/` | Implementation plans from **writing-plans**: an audit trail of how work was broken down |

These are plain Markdown and tool-agnostic: any human or agent can read them, and no machine paths are involved. Review them in PRs like code. Strip secrets or customer data before committing.

### What is ignored

| Group | Patterns |
|---|---|
| Starter-kit stack | `.wolf`, `graphify-out`, `.graphify*`, `.superpowers` |
| Instruction / memory files | `CLAUDE.md`, `CLAUDE.local.md`, `AGENTS.md`, `GEMINI.md`, `QWEN.md`, `WARP.md`, `CRUSH.md`, `.cursorrules`, `.windsurfrules`, `.clinerules`, `.github/copilot-instructions.md`, … |
| Agent config dirs | `.claude`, `.agent`, `.agents`, `.gemini`, `.codex`, `.cursor`, `.windsurf`, `.kiro`, `.continue`, `.cline`, `.roo`, `.aider*`, `.opencode`, `.qwen`, … and `.mcp.json` |

The patterns have no trailing `/`, so they also match **symlinks**. A common setup is `ln -s AGENTS.md CLAUDE.md` so several agents share one instruction file.

### Trade-offs and how to handle them

- **Every clone starts with no memory or graph.** Run [§2](#2-per-project-setup) after cloning. `graphify update .` rebuilds the graph from code at no LLM cost.
- **Agent rules aren't shared through git.** Put rules that matter to humans too (coding style, architecture, how to run tests) in normal docs such as `CONTRIBUTING.md` and `docs/`, and point your local `CLAUDE.md` at them. Lint, format and CI configs remain the enforced source of truth.
- **Graphify and OpenWolf both honour `.gitignore`** (Graphify also reads `.git/info/exclude`; OpenWolf has `anatomy.respect_gitignore: true`). Ignored paths such as `.wolf/` and `.superpowers/` are left out of the graph and the index, so agent scratch never ends up in your project's context. The committed `docs/superpowers/` specs and plans *are* indexed, so agents can find past design decisions. To keep extra paths out of only the graph, use `.graphifyignore`, which this kit also ignores.
- **Committing the graph is possible, but opt-in.** Graphify's team cookbook tracks only `graphify-out/graph.json`, plus a union merge driver in `.gitattributes`. Adopt that only if the whole team uses Graphify.

### Optional: personal global ignore

To keep these files out of **every** repo on your machine, including repos that don't use this kit, put the AI section in a global excludes file:

```bash
git config --global core.excludesFile ~/.config/git/ignore
sed -n '/^# AI agent tooling/,/^# Secrets/p' .gitignore >> ~/.config/git/ignore
```

Keep the project `.gitignore` as well, since it protects collaborators who have no global file.

---

## 5. Verify the setup

```bash
rtk gain                                   # RTK hook works
openwolf status                            # OpenWolf hooks healthy
graphify hook status                       # git hooks installed
test -f graphify-out/graph.json && echo "graph OK"
git check-ignore -v CLAUDE.md .claude .wolf graphify-out   # every path should be matched
git status --porcelain                     # should list no AI tooling files
```

In a **fresh** Claude Code session, type `Let's make a react todo list`. With Superpowers installed, the agent should start **brainstorming** and ask questions instead of writing code.

---

## 6. Troubleshooting

| Symptom | Fix |
|---|---|
| `graphify: command not found` | `uv tool update-shell`, then restart the shell |
| `rtk gain` errors / unknown command | Wrong `rtk` installed. Reinstall from rtk-ai via brew or the install script |
| Hooks don't fire after cloning | Run the per-project setup (§2) again, start a new session, and accept the hook trust prompt |
| Agent ignores the graph | Check that `graphify-out/graph.json` exists and that `CLAUDE.md` has the `## graphify` section. If it's missing, run `graphify extract . --code-only` |
| Graph is stale | `graphify update .` (hooks rebuild in the background, so a query right after a commit may see old data). After changing ignore rules: `graphify extract . --force` |
| `/graphify` not recognised in Claude Code | `graphify install --platform claude`, then restart Claude Code |
| Git hooks broke after a Python upgrade | `graphify hook install` again (the hooks store the interpreter path) |
| Skip graph rebuild for one command | `GRAPHIFY_SKIP_HOOK=1 git pull` |
| `.gitattributes` shows a `merge=graphify` diff | Re-run the move in §2 step 4 |
| Pause OpenWolf in one project | Set `openwolf.enabled` to `false` in `.wolf/config.json` |
| An AI file shows up in `git status` | It's probably a new tool. Add its file or directory to the *AI agent tooling* section of `.gitignore` |
| An AI file was committed before the rule existed | `git rm -r --cached <path>` and commit. The file stays on disk |
| `openwolf` fails on install/run | Check `node --version` ≥ 24 |
| `graphify` fails to install | Check Python ≥ 3.12: `uv tool install --python 3.12 graphifyy` |
| Two tools shortening Bash output | Set OpenWolf `bash.governor.mode` to `suggest` (see §2). For one-off raw output: `rtk bypass <cmd>` |
| Turn off strict graph nudges | `GRAPHIFY_HOOK_STRICT=0` |
| Turn off Graphify's query log | `GRAPHIFY_QUERY_LOG_DISABLE=1` |
| Turn off Superpowers telemetry pixel | `SUPERPOWERS_DISABLE_TELEMETRY=1` |

---

## 7. Uninstall

```bash
graphify claude uninstall                              # remove CLAUDE.md section + PreToolUse hooks
graphify hook uninstall                                # remove git hooks + merge driver
graphify uninstall --purge                             # all platforms, and delete graphify-out/
uv tool uninstall graphifyy
openwolf restore                                       # list backups of .wolf/ + Claude settings (rollback)
npm uninstall -g openwolf && rm -rf .wolf              # no uninstall command: also delete its hooks in .claude/settings.json
brew uninstall rtk                                     # or delete the binary the install script placed
# Superpowers, inside Claude Code: /plugin uninstall superpowers
```

---

## References

- RTK: https://github.com/rtk-ai/rtk · docs: https://www.rtk-ai.app/docs
- OpenWolf: https://github.com/cytostack/openwolf · docs: https://openwolf.com/
- Graphify: https://github.com/Graphify-Labs/graphify · docs: https://docs.graphify.com/
- Superpowers: https://github.com/obra/superpowers

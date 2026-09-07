# Multi-Agent Coding Operating Manual: One-Page Playbook

## Plain tmux + Git Worktrees

**Model:** You coordinate. Git holds the shared truth. Every agent gets its own branch, directory, and tmux window.

| Agent | Default role |
|---|---|
| **Codex** | Backend, debugging, integration, final tests |
| **Antigravity (`agy`)** | Frontend and browser verification |
| **Grok Build** | Investigation, alternatives, bounded implementation |
| **Claude Code** | Architecture and independent diff review |

Use only the agents a task needs. Never give two agents ownership of the same files.

## 1. Define the task

Create and commit `.agents/tasks/<task-id>.md` with:

- goal, scope, and non-goals;
- agent-to-directory ownership;
- acceptance criteria and exact test commands;
- constraints: no merge, push, deploy, `.env` edits, or out-of-scope changes.

The task file and tests—not chat histories—are authoritative.

## 2. Create branches and worktrees

From a clean repository:

```bash
PROJECT_ROOT="$(git rev-parse --show-toplevel)"
PROJECT_NAME="$(basename "$PROJECT_ROOT")"
TASK_ID="auth-refresh"  # change this
WORKTREE_ROOT="$(dirname "$PROJECT_ROOT")/${PROJECT_NAME}-worktrees/${TASK_ID}"
INTEGRATION_BRANCH="work/${TASK_ID}"
TMUX_SESSION="${PROJECT_NAME}-${TASK_ID}"

git status --short      # must be empty
git switch main && git pull --ff-only
git switch -c "$INTEGRATION_BRANCH"
# Write, add, and commit .agents/tasks/$TASK_ID.md here.

mkdir -p "$WORKTREE_ROOT"
git worktree add -b "agent/${TASK_ID}/codex" "$WORKTREE_ROOT/codex" "$INTEGRATION_BRANCH"
git worktree add -b "agent/${TASK_ID}/antigravity" "$WORKTREE_ROOT/antigravity" "$INTEGRATION_BRANCH"
git worktree add -b "agent/${TASK_ID}/grok" "$WORKTREE_ROOT/grok" "$INTEGRATION_BRANCH"
```

Create only the worktrees required.

## 3. Link `.env` explicitly

Worktrees do not copy ignored files. Specify both source and destination:

```bash
PROJECT_ENV_SOURCE="/absolute/path/to/main-repository/.env"
test -f "$PROJECT_ENV_SOURCE"
ln -s "$PROJECT_ENV_SOURCE" "$WORKTREE_ROOT/codex/.env"
ln -s "$PROJECT_ENV_SOURCE" "$WORKTREE_ROOT/antigravity/.env"
ln -s "$PROJECT_ENV_SOURCE" "$WORKTREE_ROOT/grok/.env"
readlink -f "$WORKTREE_ROOT/codex/.env"
```

All links share one target; agents must not edit it. Use development credentials only. Use separate copies when agents need different ports or values.

## 4. Launch the agents

```bash
tmux new-session -d -s "$TMUX_SESSION" -n integration -c "$PROJECT_ROOT"
tmux new-window -t "$TMUX_SESSION" -n codex -c "$WORKTREE_ROOT/codex"
tmux new-window -t "$TMUX_SESSION" -n antigravity -c "$WORKTREE_ROOT/antigravity"
tmux new-window -t "$TMUX_SESSION" -n grok -c "$WORKTREE_ROOT/grok"
tmux send-keys -t "$TMUX_SESSION:codex" 'codex' Enter
tmux send-keys -t "$TMUX_SESSION:antigravity" 'agy' Enter
tmux send-keys -t "$TMUX_SESSION:grok" 'grok' Enter
tmux attach -t "$TMUX_SESSION"
```

`Ctrl-b w`: windows · `Ctrl-b n/p`: next/previous · `Ctrl-b d`: detach.

Give each agent this pattern:

```text
Read .agents/OPERATING_RULES.md and .agents/tasks/<task-id>.md.
Own only <paths>. Complete the assignment, run specified tests, commit all work,
and write .agents/handoffs/<task-id>-<agent>.md with commit SHA, files, tests,
decisions, and risks. Do not merge, push, deploy, or edit .env.
```

## 5. Inspect, merge, and review

```bash
git diff "$INTEGRATION_BRANCH"..."agent/${TASK_ID}/codex"
git merge --no-ff "agent/${TASK_ID}/codex"
git diff "$INTEGRATION_BRANCH"..."agent/${TASK_ID}/antigravity"
git merge --no-ff "agent/${TASK_ID}/antigravity"
```

Inspect every diff and test after every merge. Then create Claude’s review worktree from the integrated candidate:

```bash
git worktree add -b "review/${TASK_ID}/claude" "$WORKTREE_ROOT/claude-review" "$INTEGRATION_BRANCH"
tmux new-window -t "$TMUX_SESSION" -n claude-review -c "$WORKTREE_ROOT/claude-review"
tmux send-keys -t "$TMUX_SESSION:claude-review" 'claude' Enter
```

Ask Claude to review `main...HEAD` for correctness, security, API drift, missing tests, migrations, and unnecessary complexity. Address accepted blocker/high findings, then run the full test/build suite.

## 6. Ship and clean up

```bash
git push -u origin "$INTEGRATION_BRANCH"
gh pr create --base main --head "$INTEGRATION_BRANCH"

# After safely pushing/merging and exiting agents:
tmux kill-session -t "$TMUX_SESSION"
git worktree list
git worktree remove "$WORKTREE_ROOT/codex"
git worktree remove "$WORKTREE_ROOT/antigravity"
git worktree remove "$WORKTREE_ROOT/grok"
git worktree remove "$WORKTREE_ROOT/claude-review"
git worktree prune --dry-run --verbose
```

Never use `rm -rf` for worktrees. `git worktree remove` will refuse when uncommitted work remains; inspect it instead of forcing removal.

## Seven rules

1. One owner per file/directory.
2. Requirements and acceptance tests live in Git.
3. Agents commit and hand off; they do not merge or deploy.
4. Inspect every diff before integration.
5. Use a different model for review.
6. Worktrees isolate code—not credentials, processes, ports, or databases.
7. Start with Codex + Claude; add Antigravity and Grok when work divides cleanly.

---

# Multi-Agent Coding Operating Manual - Details

## Plain tmux + Git Worktrees

**Purpose:** Run Codex, Claude Code, Google Antigravity CLI, and Grok Build on the same software project without relying on an experimental orchestrator.

**Operating model:** You remain the coordinator. Each coding agent runs in its own terminal and Git worktree. Files in Git—not chat histories—carry requirements, decisions, code, tests, and handoffs between agents.

**Designed for:** Linux, Bash, Git, and tmux.

---

## 1. The system at a glance

For each substantial task, create:

- one **integration branch**, where approved work is combined;
- one **worktree and branch per implementation agent**;
- one **tmux session**, with a window for each worktree;
- one **task specification**, committed before agents start;
- one **handoff note per agent**;
- one **independent review**, normally performed by Claude Code after implementation branches are integrated.

The normal flow is:

1. You define the task and acceptance tests.
2. You create the integration branch.
3. You create isolated worktrees for the agents that are needed.
4. You launch each native agent in a separate tmux window.
5. Each agent changes only its assigned files and commits its work.
6. You inspect and merge the implementation branches into the integration branch.
7. Claude reviews the integrated diff.
8. A designated agent addresses accepted review findings.
9. Codex runs the final test suite and integration checks.
10. You merge or open a pull request.
11. You stop tmux and remove the worktrees safely.

### What this system isolates

Git worktrees isolate:

- checked-out branches;
- tracked file modifications;
- each agent’s working directory;
- most accidental edit collisions.

They do **not** isolate:

- host credentials;
- running processes;
- network access;
- databases and cloud resources;
- ports;
- files accessed through absolute paths;
- a shared `.env` file that you explicitly link into multiple worktrees.

Worktrees are source-control isolation, not security sandboxes.

---

## 2. Agent roles

Do not involve every agent merely because it is available. Give each task to the agent whose strengths matter and use a different agent for review.

| Role | Default agent | Typical assignment |
|---|---|---|
| Coordinator/integrator | You, assisted by Codex | Task setup, integration, final tests |
| Backend implementer | Codex | Domain logic, APIs, migrations, concurrency, debugging |
| UI/browser implementer | Antigravity | Frontend, browser verification, visual behaviour |
| Investigator/challenger | Grok Build | Competing approaches, root-cause analysis, bounded implementation |
| Architecture/diff reviewer | Claude Code | Invariants, API design, security, maintainability, review |

Recommended rules:

- One agent owns a file or directory at a time.
- The implementation agent does not provide the only review of its own work.
- Agents never merge into `main`.
- Agents never deploy unless the task explicitly authorizes a deployment.
- Agents commit before handing work back.
- A task that cannot be described with boundaries and acceptance checks is not ready to parallelize.

---

## 3. Terminology

| Term | Meaning |
|---|---|
| Main worktree | Your original repository directory |
| Linked worktree | An additional checked-out branch connected to the same Git repository |
| Integration branch | Temporary branch where accepted agent work is assembled |
| Agent branch | Branch owned by one agent for one task |
| tmux session | Persistent terminal workspace containing multiple windows |
| Task specification | Repository file describing scope, ownership, constraints, tests, and completion criteria |
| Handoff | Agent-written record of commits, tests, decisions, and unresolved risks |

---

## 4. One-time workstation setup

### 4.1 Install and verify the basic tools

Install Git and tmux using your Linux distribution’s package manager. Then verify:

```bash
git --version
tmux -V
```

Optional but useful:

```bash
gh --version
jq --version
```

`gh` is useful for opening pull requests. `jq` is useful when you later automate headless agent output.

### 4.2 Verify each coding agent

Run these from a harmless test repository before using this workflow:

```bash
codex
claude
agy
grok
```

Sign in through each native tool. This workflow launches the installed CLIs, so each tool uses its own existing authentication and account limits.

Do not place authentication tokens in the repository.

### 4.3 Choose a worktree location

Use a sibling directory next to the main repository. For a repository at:

```text
/projects/my-app
```

use:

```text
/projects/my-app-worktrees/<task-id>/<agent>
```

This keeps linked worktrees out of the application directory and prevents recursive searches, watchers, formatters, and test tools from accidentally traversing other worktrees.

### 4.4 Add the coordination files to the repository

Create this tracked structure:

```text
.agents/
├── OPERATING_RULES.md
├── tasks/
├── handoffs/
├── reviews/
└── decisions/
```

The directories can initially contain `.gitkeep` files if Git would otherwise omit them.

Recommended `.agents/OPERATING_RULES.md`:

```markdown
# Agent Operating Rules

1. Read the assigned task specification before editing.
2. Modify only the paths assigned to you.
3. Do not merge, push, deploy, or alter remote infrastructure unless explicitly authorized.
4. Do not edit `.env`, credentials, generated secrets, or developer machine configuration.
5. Do not weaken or delete tests merely to make the suite pass.
6. Run the specified tests before handoff.
7. Commit all intended changes to your assigned branch.
8. Write the required handoff file with the commit SHA, tests run, decisions, and known issues.
9. If scope or ownership is unclear, stop and ask rather than modifying another agent’s area.
```

Commit these rules so every worktree receives the same version.

### 4.5 Keep agent instructions consistent

Your repository may already have files such as `AGENTS.md`, `CLAUDE.md`, or tool-specific rules. Keep durable project conventions there, but put task-specific assignments in `.agents/tasks/`.

Do not depend on every agent automatically discovering the same instruction filename. The startup prompt should explicitly tell the agent which shared files to read.

---

## 5. Naming standard

Use a short task ID containing lowercase letters, numbers, and hyphens. Examples:

- `auth-refresh`
- `bug-1842`
- `billing-webhook`

Use these names:

| Item | Pattern | Example |
|---|---|---|
| Integration branch | `work/<task-id>` | `work/auth-refresh` |
| Codex branch | `agent/<task-id>/codex` | `agent/auth-refresh/codex` |
| Antigravity branch | `agent/<task-id>/antigravity` | `agent/auth-refresh/antigravity` |
| Grok branch | `agent/<task-id>/grok` | `agent/auth-refresh/grok` |
| Claude review branch | `review/<task-id>/claude` | `review/auth-refresh/claude` |
| tmux session | `<repo>-<task-id>` | `my-app-auth-refresh` |
| Task file | `.agents/tasks/<task-id>.md` | `.agents/tasks/auth-refresh.md` |

Avoid spaces, uppercase letters, timestamps, and agent model names in these identifiers.

---

## 6. Write the task specification first

The task specification is the control plane. Agents should not treat another agent’s transcript as authoritative.

Use this template:

```markdown
# Task: <title>

## Task ID
<task-id>

## Goal
One paragraph describing the user-visible or operational result.

## Current behaviour
What happens now, including reproduction steps for a bug.

## Desired behaviour
What must happen after the change.

## Scope
- Included item
- Included item

## Non-goals
- Explicitly excluded item
- Deferred item

## Ownership
| Agent | Owned paths | Assignment |
|---|---|---|
| Codex | `api/`, `tests/api/` | Implement endpoint and tests |
| Antigravity | `web/`, `tests/e2e/` | Implement UI and browser verification |
| Grok | `docs/investigations/` | Analyze edge cases; do not edit product code |
| Claude | Integrated diff | Review only after implementation merge |

## Constraints
- Preserve existing public API behaviour unless described here.
- Do not change authentication or authorization policy.
- Do not add a dependency without documenting why.
- Do not alter production infrastructure.

## Acceptance criteria
- [ ] Exact observable outcome
- [ ] Exact error or boundary behaviour
- [ ] Regression test exists
- [ ] Existing tests remain green

## Required verification
```bash
<fast focused test command>
<full relevant test command>
<lint/type-check command>
```

## Handoff requirement
Commit all intended changes and write `.agents/handoffs/<task-id>-<agent>.md`.
```

Good acceptance criteria are mechanically verifiable. “Improve the code” is not an acceptance criterion. “A request with an expired refresh token returns HTTP 401 and does not create a session” is.

---

## 7. Start a task

The following procedure assumes your main worktree is already open in a terminal.

### Step 1: Establish variables

Use task-specific variable names:

```bash
PROJECT_ROOT="$(git rev-parse --show-toplevel)"
PROJECT_NAME="$(basename "$PROJECT_ROOT")"
TASK_ID="auth-refresh"
WORKTREE_ROOT="$(dirname "$PROJECT_ROOT")/${PROJECT_NAME}-worktrees/${TASK_ID}"
INTEGRATION_BRANCH="work/${TASK_ID}"
TMUX_SESSION="${PROJECT_NAME}-${TASK_ID}"
```

Replace `auth-refresh` with the actual task ID.

Inspect the resolved values before creating anything:

```bash
printf '%s\n' "$PROJECT_ROOT" "$WORKTREE_ROOT" "$INTEGRATION_BRANCH" "$TMUX_SESSION"
```

### Step 2: Confirm that the starting repository is clean

```bash
git status --short
git branch --show-current
```

If `git status --short` prints anything, stop. Commit, deliberately preserve, or otherwise resolve your existing changes before creating the task branches.

Do not let an agent decide what to do with unrelated uncommitted work.

### Step 3: Update the base branch

Assuming the repository uses `main`:

```bash
git fetch origin --prune
git switch main
git pull --ff-only
```

`--ff-only` prevents an unexpected merge commit during setup.

### Step 4: Create the integration branch

```bash
git switch -c "$INTEGRATION_BRANCH"
```

Create `.agents/tasks/<task-id>.md`, review it yourself, and commit it:

```bash
git add ".agents/tasks/${TASK_ID}.md"
git commit -m "docs: define ${TASK_ID} agent task"
```

This commit becomes the common starting point for all workers.

### Step 5: Create only the worktrees you need

Example for Codex, Antigravity, and Grok:

```bash
mkdir -p "$WORKTREE_ROOT"

git worktree add \
  -b "agent/${TASK_ID}/codex" \
  "$WORKTREE_ROOT/codex" \
  "$INTEGRATION_BRANCH"

git worktree add \
  -b "agent/${TASK_ID}/antigravity" \
  "$WORKTREE_ROOT/antigravity" \
  "$INTEGRATION_BRANCH"

git worktree add \
  -b "agent/${TASK_ID}/grok" \
  "$WORKTREE_ROOT/grok" \
  "$INTEGRATION_BRANCH"
```

The argument order is important:

```text
git worktree add -b <new-agent-branch> <destination-directory> <starting-branch>
```

Verify:

```bash
git worktree list
```

Do not create the Claude review worktree yet. Create it after implementation branches are integrated so Claude sees the exact candidate diff.

---

## 8. Environment files and ignored files

This section is intentionally explicit because Git worktrees do not automatically reproduce ignored or untracked files.

### 8.1 Preferred rule

Give agents the least-sensitive development configuration that can perform the task. Do not expose production credentials.

### 8.2 Option A: Explicit symbolic link

Suppose the canonical development environment file is:

```text
/projects/my-app/.env
```

For the Codex worktree at:

```text
/projects/my-app-worktrees/auth-refresh/codex
```

the command is:

```bash
PROJECT_ENV_SOURCE="/projects/my-app/.env"
CODEX_TREE="/projects/my-app-worktrees/auth-refresh/codex"

test -f "$PROJECT_ENV_SOURCE"
test ! -e "$CODEX_TREE/.env"
ln -s "$PROJECT_ENV_SOURCE" "$CODEX_TREE/.env"
```

`ln -s` receives:

1. the **source/target file**: `/projects/my-app/.env`
2. the **destination link**: `/projects/my-app-worktrees/auth-refresh/codex/.env`

Verify both the destination and resolved target:

```bash
ls -l "$CODEX_TREE/.env"
readlink -f "$CODEX_TREE/.env"
```

Repeat explicitly for each authorized worktree:

```bash
ln -s "$PROJECT_ENV_SOURCE" "$WORKTREE_ROOT/antigravity/.env"
ln -s "$PROJECT_ENV_SOURCE" "$WORKTREE_ROOT/grok/.env"
```

Important: every symlink points at the same file. If a process modifies the target, every worktree sees the change. Put “Do not edit `.env`” in the operating rules.

### 8.3 Option B: Isolated copy

Use an isolated copy when a worker needs different ports or safe task-specific overrides:

```bash
install -m 600 "$PROJECT_ENV_SOURCE" "$WORKTREE_ROOT/codex/.env"
```

This creates an independent destination with owner-only permissions. Remember to remove it through `git worktree remove` during cleanup and confirm `.env` is ignored by Git:

```bash
git -C "$WORKTREE_ROOT/codex" check-ignore -v .env
```

### 8.4 Other ignored files

Do not blindly copy all ignored files. That can duplicate:

- `node_modules`;
- virtual environments;
- build outputs;
- large datasets;
- credentials;
- caches with absolute paths.

Install dependencies inside each worktree according to the project lockfile. Package managers with shared global caches will still avoid downloading everything repeatedly.

---

## 9. Prevent port and runtime collisions

Separate source trees do not prevent two local servers from binding the same port.

Assign a port range per agent:

| Agent | Suggested application port | Suggested test/browser port |
|---|---:|---:|
| Main/integration | 3000 | 4000 |
| Codex | 3100 | 4100 |
| Antigravity | 3200 | 4200 |
| Grok | 3300 | 4300 |
| Claude review | Normally no server | Normally no server |

Use task-local environment overrides supported by your application, for example:

```bash
PORT=3100 codex
PORT=3200 agy
PORT=3300 grok
```

If the application writes to a local SQLite database or filesystem queue, give each worktree a separate data path as well.

Never point simultaneous agent test runs at a shared destructive development database unless the tests are specifically designed for concurrency.

---

## 10. Launch the tmux control room

### Step 1: Create the integration window

```bash
tmux new-session \
  -d \
  -s "$TMUX_SESSION" \
  -n integration \
  -c "$PROJECT_ROOT"
```

### Step 2: Create one window per agent

```bash
tmux new-window \
  -t "$TMUX_SESSION" \
  -n codex \
  -c "$WORKTREE_ROOT/codex"

tmux new-window \
  -t "$TMUX_SESSION" \
  -n antigravity \
  -c "$WORKTREE_ROOT/antigravity"

tmux new-window \
  -t "$TMUX_SESSION" \
  -n grok \
  -c "$WORKTREE_ROOT/grok"
```

### Step 3: Start each native CLI

```bash
tmux send-keys -t "$TMUX_SESSION:codex" 'codex' Enter
tmux send-keys -t "$TMUX_SESSION:antigravity" 'agy' Enter
tmux send-keys -t "$TMUX_SESSION:grok" 'grok' Enter
```

### Step 4: Attach

```bash
tmux select-window -t "$TMUX_SESSION:integration"
tmux attach-session -t "$TMUX_SESSION"
```

Useful tmux keys use the default prefix `Ctrl-b`:

| Keys | Action |
|---|---|
| `Ctrl-b w` | Show the window list |
| `Ctrl-b n` | Next window |
| `Ctrl-b p` | Previous window |
| `Ctrl-b 0`…`9` | Jump to window number |
| `Ctrl-b ,` | Rename the current window |
| `Ctrl-b [` | Enter scroll/copy mode |
| `Ctrl-b d` | Detach without stopping agents |

Outside tmux:

```bash
tmux ls
tmux attach-session -t "$TMUX_SESSION"
```

---

## 11. Give each agent its assignment

Use the same startup structure for every agent:

```text
Read .agents/OPERATING_RULES.md and .agents/tasks/<task-id>.md first.

You are the <role> for this task. Your branch and worktree are already isolated.
You own only: <paths>.
Do not edit: <paths owned by others>.

Complete your assigned work, run the required focused tests, commit all intended
changes, and write .agents/handoffs/<task-id>-<agent>.md using the required
handoff format. Do not merge, push, deploy, or alter .env. If another directory
must change, stop and explain why in the handoff rather than editing it.
```

### Codex prompt

```text
Read .agents/OPERATING_RULES.md and .agents/tasks/auth-refresh.md.
You own api/auth/ and tests/api/auth/. Implement the backend portion and its
regression tests. Preserve the existing public API except where the task says
otherwise. Run the focused test and type-check commands, commit your changes,
and write .agents/handoffs/auth-refresh-codex.md. Do not merge or push.
```

### Antigravity prompt

```text
Read .agents/OPERATING_RULES.md and .agents/tasks/auth-refresh.md.
You own web/auth/ and tests/e2e/auth/. Implement the user-facing portion against
the documented API contract. Run the app on your assigned port and verify the
critical path in a browser. Record exact scenarios tested in the handoff. Commit
your changes and write .agents/handoffs/auth-refresh-antigravity.md. Do not edit
backend paths, merge, push, or deploy.
```

### Grok prompt

```text
Read .agents/OPERATING_RULES.md and .agents/tasks/auth-refresh.md.
Act as the investigator for this task. Analyze refresh-token edge cases and the
current implementation. Write findings and concrete recommendations under
docs/investigations/. Do not edit product code. Commit the investigation and
write .agents/handoffs/auth-refresh-grok.md. Do not merge or push.
```

The exact assignments should come from the task file. The prompt reinforces them; it does not replace them.

---

## 12. Agent handoff standard

Every implementation agent writes a unique handoff file:

```markdown
# Handoff: <task-id> / <agent>

## Status
Complete | Partial | Blocked

## Branch
`agent/<task-id>/<agent>`

## Commit
`<full commit SHA>`

## Summary
- What changed
- Why it changed

## Files changed
- `path/to/file`: reason

## Tests run
- `<exact command>` — passed/failed

## Acceptance criteria covered
- [x] Criterion
- [ ] Criterion not covered, with explanation

## Decisions and assumptions
- Decision and rationale

## Known issues or risks
- Risk, or “None known”

## Integration notes
- Ordering, migrations, generated files, or dependencies the integrator must know
```

Require the full commit SHA:

```bash
git rev-parse HEAD
```

The handoff is not a substitute for inspecting the diff or running tests.

---

## 13. Monitor work without interfering

From the integration window or another terminal:

```bash
tmux list-windows -t "$TMUX_SESSION"
git worktree list
git -C "$WORKTREE_ROOT/codex" status --short
git -C "$WORKTREE_ROOT/antigravity" status --short
git -C "$WORKTREE_ROOT/grok" status --short
```

To inspect recent commits:

```bash
git log --oneline --decorate --all -20
```

To inspect an agent’s committed delta from the integration branch:

```bash
git diff --stat "$INTEGRATION_BRANCH"..."agent/${TASK_ID}/codex"
git diff "$INTEGRATION_BRANCH"..."agent/${TASK_ID}/codex"
```

Do not edit an active worker’s files from another worktree. Send clarification through its tmux window or wait for a handoff.

---

## 14. Completion gate for an agent

An agent is not finished until all of these are true:

- [ ] Its assigned scope is complete or explicitly marked partial/blocked.
- [ ] Required focused tests were run.
- [ ] Intended changes are committed.
- [ ] `git status --short` is empty.
- [ ] The handoff file exists and names the commit.
- [ ] No unrelated files changed.
- [ ] No `.env`, credential, lockfile, generated output, or infrastructure change appeared unexpectedly.

Validate from the integration window:

```bash
git -C "$WORKTREE_ROOT/codex" status --short
git -C "$WORKTREE_ROOT/codex" log -1 --oneline
git show --stat "agent/${TASK_ID}/codex"
```

If the worktree is dirty, ask the agent to either commit intended work or explain and remove unintended work. Do not clean it automatically.

---

## 15. Integrate agent branches

Return to the `integration` tmux window. Confirm you are on the integration branch:

```bash
cd "$PROJECT_ROOT"
git branch --show-current
git status --short
```

The branch should be `work/<task-id>`, and the status should be clean.

### Step 1: Inspect the first branch

```bash
git log --oneline "$INTEGRATION_BRANCH".."agent/${TASK_ID}/codex"
git diff --stat "$INTEGRATION_BRANCH"..."agent/${TASK_ID}/codex"
git diff "$INTEGRATION_BRANCH"..."agent/${TASK_ID}/codex"
```

### Step 2: Merge with history preserved

```bash
git merge --no-ff "agent/${TASK_ID}/codex"
```

Run the relevant focused tests immediately.

### Step 3: Merge the next independent branch

```bash
git diff --stat "$INTEGRATION_BRANCH"..."agent/${TASK_ID}/antigravity"
git diff "$INTEGRATION_BRANCH"..."agent/${TASK_ID}/antigravity"
git merge --no-ff "agent/${TASK_ID}/antigravity"
```

Run the combined tests.

### Step 4: Merge investigation or documentation if useful

```bash
git diff "$INTEGRATION_BRANCH"..."agent/${TASK_ID}/grok"
git merge --no-ff "agent/${TASK_ID}/grok"
```

If a branch contains unwanted commits, do not merge it and then try to hide them. Ask the worker to create a clean follow-up branch or cherry-pick only explicitly selected commits after reviewing each one.

---

## 16. Resolve merge conflicts safely

When a merge conflicts:

1. Stop the merge and inspect the conflict list.
2. Determine whether the conflict indicates accidental overlapping ownership.
3. Resolve in the integration worktree only.
4. Run focused tests covering the conflicting behaviour.
5. Complete the merge commit.

Inspect conflicts:

```bash
git status
git diff --name-only --diff-filter=U
```

If you do not understand the correct resolution, use a new, bounded review session:

```text
Resolve only the listed merge conflicts. Read both agent handoffs and the task
specification. Preserve the intended behaviour of both branches. Do not make
unrelated refactors. Run <focused test> and explain every resolution.
```

To abandon an unresolved merge and return to the pre-merge state:

```bash
git merge --abort
```

Do not use `git reset --hard` as a routine conflict-resolution tool.

---

## 17. Independent Claude review

Create Claude’s worktree only after implementation branches have been merged into the integration branch:

```bash
git worktree add \
  -b "review/${TASK_ID}/claude" \
  "$WORKTREE_ROOT/claude-review" \
  "$INTEGRATION_BRANCH"
```

Add and launch its tmux window:

```bash
tmux new-window \
  -t "$TMUX_SESSION" \
  -n claude-review \
  -c "$WORKTREE_ROOT/claude-review"

tmux send-keys -t "$TMUX_SESSION:claude-review" 'claude' Enter
```

Review prompt:

```text
Read .agents/OPERATING_RULES.md, .agents/tasks/auth-refresh.md, and all handoffs
for this task. Review the integrated candidate represented by this branch
against main. Do not modify product code. Look specifically for:

1. correctness and race conditions;
2. authentication and authorization errors;
3. divergence from the task and API contract;
4. missing boundary and regression tests;
5. unsafe migrations or rollback problems;
6. unnecessary complexity;
7. frontend state, accessibility, and error handling issues.

Write findings to .agents/reviews/auth-refresh-claude.md. Rank each finding as
blocker, high, medium, low, or suggestion. For every blocker/high finding,
include file, location, failure scenario, and the smallest acceptable fix.
Commit only the review file. Do not merge or push.
```

### Review finding format

```markdown
## Finding 1 — High — Authorization checked after mutation

**Location:** `api/example.ts`, function `updateRecord`

**Failure scenario:** A user without write permission can cause the update before
the authorization error is returned.

**Evidence:** Describe the relevant control flow or test reproduction.

**Required correction:** Move the authorization check before the mutation and
add a regression test proving no update occurs.
```

Merge the review file—not product-code changes—from the Claude review branch:

```bash
git merge --no-ff "review/${TASK_ID}/claude"
```

You decide which findings are accepted. Suggestions are not automatically requirements.

---

## 18. Fix review findings

Do not reopen the original implementation worktrees after the integration branch has advanced and casually continue editing against stale branches.

For meaningful corrections, create a fresh fix branch from the current integration branch:

```bash
git worktree add \
  -b "agent/${TASK_ID}/review-fixes" \
  "$WORKTREE_ROOT/review-fixes" \
  "$INTEGRATION_BRANCH"
```

Assign the fix to an agent that did not author the relevant code when practical. Provide only accepted findings and exact verification commands.

After handoff:

```bash
git diff "$INTEGRATION_BRANCH"..."agent/${TASK_ID}/review-fixes"
git merge --no-ff "agent/${TASK_ID}/review-fixes"
```

Update the review document with the disposition of each blocker/high finding:

- accepted and fixed;
- accepted and deferred, with reason and tracking issue;
- rejected, with technical rationale.

---

## 19. Final verification

Perform final verification from the integration worktree, not from an agent’s stale branch.

Minimum gate:

```bash
git status --short
<install-or-sync-dependencies-command>
<formatter-check-command>
<lint-command>
<type-check-command>
<unit-test-command>
<integration-test-command>
<build-command>
```

Also verify:

- database migrations apply and, when supported, roll back safely;
- generated files are current;
- no unexpected dependency or lockfile changes exist;
- no secrets or `.env` contents are tracked;
- browser-critical paths were exercised when UI changed;
- task acceptance criteria are checked off with evidence;
- the integrated diff against `main` contains only intended work.

Inspect the final candidate:

```bash
git diff --stat main..."$INTEGRATION_BRANCH"
git diff main..."$INTEGRATION_BRANCH"
git log --oneline main.."$INTEGRATION_BRANCH"
```

Have Codex run the tests and diagnose failures, but keep the final merge decision with you.

---

## 20. Finish through a pull request

The recommended team/repository flow is to push the integration branch and open a pull request:

```bash
git push -u origin "$INTEGRATION_BRANCH"

gh pr create \
  --base main \
  --head "$INTEGRATION_BRANCH" \
  --title "<task title>" \
  --body-file ".agents/tasks/${TASK_ID}.md"
```

Then let CI run. Address CI failures on a fresh fix branch or directly on the integration branch only if no agents remain active.

Do not delete local worktrees until the pull request is safely pushed and you no longer need their local sessions.

### Local-only merge

If the repository’s policy permits a local merge:

```bash
git switch main
git pull --ff-only
git merge --no-ff "$INTEGRATION_BRANCH"
<final-test-command>
git push origin main
```

Never perform this sequence if protected-branch policy expects a pull request.

---

## 21. Stop agents and clean up

### Step 1: Exit each agent normally

Use the agent’s quit command or keyboard shortcut in its tmux window. Examples include `/exit` or `/quit`, depending on the tool.

### Step 2: Confirm no important terminal process remains

```bash
tmux list-windows -t "$TMUX_SESSION"
```

### Step 3: Kill the task’s tmux session

```bash
tmux kill-session -t "$TMUX_SESSION"
```

This targets one explicitly named task session. Do not use `tmux kill-server` unless you intend to stop every tmux session on the machine.

### Step 4: Inspect worktrees before removal

```bash
git worktree list
git -C "$WORKTREE_ROOT/codex" status --short
git -C "$WORKTREE_ROOT/antigravity" status --short
git -C "$WORKTREE_ROOT/grok" status --short
git -C "$WORKTREE_ROOT/claude-review" status --short
```

Every status should be clean. If not, preserve intended changes with a commit before removal.

### Step 5: Remove linked worktrees through Git

```bash
git worktree remove "$WORKTREE_ROOT/codex"
git worktree remove "$WORKTREE_ROOT/antigravity"
git worktree remove "$WORKTREE_ROOT/grok"
git worktree remove "$WORKTREE_ROOT/claude-review"
```

Remove any additional fix worktree the same way.

Do not remove worktree directories with `rm -rf`. `git worktree remove` checks for uncommitted changes and cleans Git’s administrative records.

Do not add `--force` simply because removal was refused. Inspect the worktree and determine what would be lost.

### Step 6: Preview stale metadata cleanup

If a directory disappeared outside Git:

```bash
git worktree prune --dry-run --verbose
```

Only after inspecting the preview:

```bash
git worktree prune --verbose
```

### Step 7: Delete merged local branches

Use lowercase `-d`, which refuses to delete an unmerged branch:

```bash
git branch -d "agent/${TASK_ID}/codex"
git branch -d "agent/${TASK_ID}/antigravity"
git branch -d "agent/${TASK_ID}/grok"
git branch -d "review/${TASK_ID}/claude"
git branch -d "agent/${TASK_ID}/review-fixes"
```

After the integration branch is merged and no longer needed:

```bash
git branch -d "$INTEGRATION_BRANCH"
```

Delete remote branches only according to your repository policy.

---

## 22. Recovery procedures

### 22.1 You detached from tmux

Nothing is wrong. Reattach:

```bash
tmux ls
tmux attach-session -t "$TMUX_SESSION"
```

### 22.2 A tmux window or CLI crashed

The worktree and files remain. Open a new tmux window in the same directory and restart the native CLI:

```bash
tmux new-window \
  -t "$TMUX_SESSION" \
  -n codex-recovery \
  -c "$WORKTREE_ROOT/codex"
```

Then launch `codex`. If conversation resumption is available, use the tool’s native resume picker. Current tools commonly provide:

```bash
codex resume
claude --resume
```

Inside Grok Build, use `/resume`. Antigravity also provides a resume command/picker.

Do not depend on chat resumption for correctness. The task file, current diff, Git log, and handoff should contain everything needed to continue.

### 22.3 An agent is stuck or waiting for input

Enter its tmux window, inspect the request, and answer directly. If the request would broaden scope or permissions, stop that worker and revise the task specification first.

### 22.4 An agent changed another agent’s files

Do not merge the branch as-is. Ask it to restore ownership boundaries in its own worktree and commit a corrected version. If the cross-boundary change is genuinely required, pause both owners and amend the task specification before proceeding.

### 22.5 An agent left uncommitted work

Prefer a clearly labeled WIP commit on that agent branch:

```bash
git add <reviewed-paths>
git commit -m "wip: preserve <task-id> agent state"
```

Avoid using `git stash` as the normal handoff mechanism. Stashes belong to the shared repository and can become confusing when several worktrees are active.

### 22.6 A worktree directory was moved manually

Use Git’s repair facility after confirming the exact path:

```bash
git worktree repair /absolute/path/to/moved-worktree
```

### 22.7 The main branch advanced during a long task

Do not update every active branch in the middle of implementation without coordination.

At a planned synchronization point:

1. stop or pause active agents;
2. commit all worker state;
3. update the integration branch from `origin/main`;
4. resolve and test there;
5. create fresh follow-up branches from the updated integration branch where possible.

For a modest task, merging `origin/main` into the integration branch is often easier to reason about than rewriting multiple active branches.

---

## 23. When to parallelize

Parallelize when assignments are genuinely independent.

Good examples:

- backend API and frontend UI built against an agreed contract;
- implementation and separate investigation/documentation;
- independent modules with distinct tests;
- code implementation followed by independent review;
- multiple competing research approaches where only one will be selected.

Do not parallelize:

- several agents editing the same core module;
- a small one-file bug;
- work without stable interfaces;
- a migration whose ordering is still undecided;
- a task with no executable acceptance check;
- several agents each trying to build the entire feature.

Use this rough rule:

| Task | Recommended agents |
|---|---|
| One-file fix | One implementer + optional reviewer |
| Normal backend feature | Codex + Claude review |
| UI feature with API changes | Codex + Antigravity + Claude review |
| Difficult root-cause bug | Grok investigation + Codex implementation + Claude review |
| Large migration | Claude plan + Codex slices + Claude review; Antigravity only if UI changes |

---

## 24. Optional headless execution

Interactive sessions should remain the default until the manual process is predictable. Headless mode is useful later for bounded tasks and scripted checks.

Examples supported by the native tools include:

```bash
codex exec "Read .agents/tasks/auth-refresh.md and perform the assigned Codex work"
agy -p "Review the UI against the task acceptance criteria"
grok -p "List unhandled edge cases in the current implementation"
claude -p "Review the supplied diff for correctness"
```

Run headless commands from the appropriate worktree. Do not launch them from the main integration directory if they are permitted to edit.

Headless execution introduces additional concerns:

- reliable exit-status interpretation;
- output capture;
- permission modes;
- timeouts;
- session IDs and resumption;
- detecting when the agent is blocked rather than finished.

Automate only after you have repeated the interactive procedure enough times to know what successful completion looks like.

---

## 25. Security and authority policy

Use these defaults unless a task explicitly overrides them:

- Development credentials only.
- No production database access.
- No cloud-resource creation or deletion.
- No deploys.
- No merges to `main`.
- No remote pushes by workers.
- No permission-bypass or always-approve mode by default.
- No editing shell startup files or user-level agent configuration.
- No reading unrelated directories.
- No deletion of branches, worktrees, or task artifacts by agents.
- No package installation outside the repository without approval.

Remember that tmux and worktrees do not reduce the operating-system permissions of an agent process. For untrusted repositories or high-risk tasks, add a container or VM boundary separately.

---

## 26. Daily control checklist

### Start of task

- [ ] Main worktree clean
- [ ] Base branch current
- [ ] Integration branch created
- [ ] Task specification reviewed and committed
- [ ] File ownership is non-overlapping
- [ ] Worktrees created from the integration branch
- [ ] `.env` source and destination explicitly verified
- [ ] Ports and local data paths assigned
- [ ] tmux session created
- [ ] Native agent authentication already working

### During task

- [ ] Clarifications go to the owning agent
- [ ] Scope changes are written into the task file
- [ ] Agents do not edit each other’s worktrees
- [ ] Agents commit before handoff
- [ ] Tests and known failures are recorded exactly

### Integration

- [ ] Each diff inspected before merge
- [ ] Each branch clean
- [ ] Focused tests run after each merge
- [ ] Combined tests run after all merges
- [ ] Claude reviewed the integrated candidate
- [ ] Blocker/high findings have dispositions
- [ ] Final diff against `main` inspected

### Cleanup

- [ ] Work safely pushed or merged
- [ ] Agents exited normally
- [ ] tmux task session stopped
- [ ] Worktrees clean
- [ ] Worktrees removed using `git worktree remove`
- [ ] Stale metadata previewed before pruning
- [ ] Only merged branches deleted with `git branch -d`

---

## 27. Command cheat sheet

### Worktrees

```bash
git worktree list
git worktree add -b <new-branch> <destination-path> <base-branch>
git worktree remove <destination-path>
git worktree prune --dry-run --verbose
git worktree prune --verbose
git worktree repair <path>
```

### tmux

```bash
tmux new-session -d -s <session> -n <window> -c <directory>
tmux new-window -t <session> -n <window> -c <directory>
tmux send-keys -t <session>:<window> '<command>' Enter
tmux list-sessions
tmux list-windows -t <session>
tmux attach-session -t <session>
tmux kill-session -t <session>
```

### Inspecting branches

```bash
git status --short
git log --oneline <base>..<agent-branch>
git diff --stat <base>...<agent-branch>
git diff <base>...<agent-branch>
git show --stat <commit-or-branch>
```

### Integration

```bash
git merge --no-ff <agent-branch>
git merge --abort
git push -u origin <integration-branch>
git branch -d <merged-local-branch>
```

---

## 28. Recommended first adoption

Do not introduce the full four-agent flow on the first attempt.

### Trial 1

- Small, non-critical repository task.
- Codex implements.
- Claude reviews.
- You integrate and clean up.

### Trial 2

- Task with separate backend and frontend ownership.
- Codex handles backend.
- Antigravity handles frontend/browser verification.
- Claude reviews the integrated diff.

### Trial 3

- Difficult bug with uncertain cause.
- Grok investigates without product-code ownership.
- Codex implements the selected correction.
- Claude reviews.

Once these are comfortable, add a small launcher script to remove repetitive typing. Keep Git and tmux as the underlying control plane so every process and worktree remains visible and independently recoverable.

---

## 29. Reference documentation

- Git worktrees: <https://git-scm.com/docs/git-worktree>
- tmux project: <https://github.com/tmux/tmux>
- Codex CLI repository and non-interactive execution: <https://github.com/openai/codex>
- Claude Code CLI reference: <https://code.claude.com/docs/en/cli-usage>
- Claude Code worktrees: <https://code.claude.com/docs/en/worktrees>
- Antigravity CLI reference: <https://antigravity.google/docs/cli/reference/>
- Antigravity headless mode: <https://antigravity.google/docs/cli/headless/>
- Grok Build CLI reference: <https://docs.x.ai/build/cli/reference>
- Grok Build headless mode: <https://docs.x.ai/build/cli/headless-scripting>

---

## Core operating principle

**Agents may come and go. The repository must always contain enough state for a capable developer—or a fresh agent session—to understand the task, inspect what changed, verify it, and continue safely.**

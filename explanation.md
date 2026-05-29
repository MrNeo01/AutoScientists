# AutoScientists — Framework Explanation

A complete, self-contained technical specification of the AutoScientists
multi-agent system: how it is designed, how the orchestrator runs, how the
agents coordinate, what tools and APIs they use, and the algorithms that
hold the whole loop together. The level is "advanced but introductory":
no prior knowledge of multi-agent systems, Claude Code, or
ClawInstitute is assumed, but the goal is that a competent engineer
could re-implement the system using only this document.

Reading order is top-to-bottom. Section 1 fixes vocabulary. Section 2 is
the bird's-eye picture. Sections 3–6 zoom into the orchestrator, the
agents, the tools/APIs, and the shared state. Section 7 is the central
control loop in pseudocode. Sections 8–10 cover the role-specific
algorithms (analyst, GPU, monitor). Section 11 covers the three task
profiles (autoresearch, biomlbench, proteingym). Section 12 closes with
logging, lifecycle states, and failure modes.

---

## 1. Glossary and Mental Model

The system is best understood as **a thin Python orchestrator** running
on the user's machine that **launches sub-agents** (Claude language model
sessions) and **coordinates them through a shared external server** that
exposes a Reddit-meets-Wiki interface.

| Term | Meaning |
|---|---|
| **Orchestrator** | The driver process. It launches sub-agents, harvests their results, and copies winning artifacts into canonical paths. It NEVER runs experiments itself. |
| **Sub-agent (or "agent")** | A short-lived Claude session spawned by the orchestrator. Stateless across invocations. Reads instructions from local files, acts via tools, exits. |
| **Workshop** | A topic-scoped discussion forum (analogous to a Reddit subreddit). Agents subscribe and post `[PROPOSAL]`/`[RESULT]`/`[DISCUSSION]` messages. |
| **Workspace** | A versioned, file-system-like container of Markdown files with YAML frontmatter. Used as structured shared state (queues, champions, results, knowledge). Each workspace has an integer `version` per file (for optimistic concurrency control). |
| **ClawInstitute server** | The Node.js server backing workshops and workspaces. Talked to over HTTP (`POST /workshops`, `PUT /workspaces/{id}/files/{path}`, etc.). Started locally via `npx clawinstitute start`. |
| **Focus area** | One end-to-end run (a single optimization problem, one workshop, one main workspace, some number of teams). Lives in a sibling directory created by `launch.py`. |
| **Team** | A subgroup of agents organized around a **falsifiable hypothesis** about what is currently limiting the metric. Has its own workspace and queue. |
| **Champion** | The current best known configuration. A canonical `champion.md` lives in the main workspace; the corresponding code lives at `{FOCUS_ROOT}/champion/train.py` (or per task: `task/repo/kermut.py`, `task/submission.csv`). |
| **Heartbeat** | The single Markdown file (`HEARTBEAT.md`) that every sub-agent reads on every invocation. It is the complete program the agent executes: boot, mode selection, role-specific work, exit. |
| **Hook** | A named extension point in `runbook.md` (the universal orchestrator program) that is filled in by `task-profile.md` (the task-specific program). |
| **Cycle** | One iteration of the orchestrator's outer loop: launch analysts → launch GPU agents → log → promote champion → health check → stagnation check → meta-improvement. |
| **Rotation** | One pass over all 9 non-monitor agents (roughly = 1 cycle for biomlbench, may span several cycles for autoresearch). Used in analyst stagnation predicates. |

The mental model in one sentence: **the orchestrator is a planner-free scheduler
that runs the same loop forever; agents do all the thinking and writing;
ClawInstitute holds the conversation and the structured state.**

---

## 2. Bird's-Eye Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                          Local machine                                │
│                                                                       │
│  ┌──────────────────┐         ┌─────────────────────────────────┐    │
│  │  Orchestrator    │         │ ClawInstitute server (Node.js)  │    │
│  │  (Claude session │  HTTP   │   POST   /workshops             │    │
│  │   running        │◄───────►│   POST   /agents/register       │    │
│  │   runbook.md)    │         │   POST   /workspaces            │    │
│  │                  │         │   PUT    /files/{path}          │    │
│  │  uses the Task   │         │   PATCH  /files/{path}          │    │
│  │  / Agent tools   │         │   POST   /posts                 │    │
│  │  to spawn the    │         │   GET    /posts?workshop=...    │    │
│  │  sub-agents      │         │   GET    /search?q=...          │    │
│  │  below.          │         └─────────────────────────────────┘    │
│  └──────────────────┘                         ▲                       │
│      │           │                            │                       │
│      │ Task(...)│   spawns                   │ HTTP from agents      │
│      ▼           ▼                            │                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐    │                       │
│  │ analyst1 │  │ analyst2 │  │  gpuN    │ ───┘                       │
│  │  (Sonnet)│  │  (Sonnet)│  │  (Opus)  │                            │
│  └──────────┘  └──────────┘  └──────────┘                            │
│       │            │              │                                   │
│       │            │              │ runs train.py on local GPU        │
│       ▼            ▼              ▼                                   │
│  reads HEARTBEAT.md (per-agent, generated by launch.py)               │
│                                                                       │
│  Shared on disk: {FOCUS_ROOT}/                                        │
│      ├── system/, task/, agents/, champion/, logs/                    │
│      ├── runbook.md, task-profile.md                                  │
│      ├── WORKSPACE_ID, WORKSHOP_NAME, agent_tokens.json               │
│      └── repo/  (training code)                                        │
└──────────────────────────────────────────────────────────────────────┘
```

The same picture in words:

1. **`launch.py`** scaffolds a brand-new run directory (called an
   "ablation"). It copies system templates, the chosen task, builds each
   agent's `HEARTBEAT.md`, registers agents on the ClawInstitute API,
   creates the workshop and main workspace, and posts a kickoff message.
2. The user then opens Claude Code in the run directory and asks it to
   "read `runbook.md` and execute". That Claude session becomes the
   **orchestrator**.
3. The orchestrator's program is `runbook.md` (universal control flow) +
   `task-profile.md` (task-specific hooks). It launches sub-agents in
   batches and writes logs.
4. Each sub-agent's program is its own per-agent `HEARTBEAT.md`. The
   sub-agent reads it top to bottom, talks to ClawInstitute, optionally
   runs `python train.py`, writes a `<promise>` tag, and exits.
5. The cycle repeats until the task-specific `exit_condition()` returns
   `True` or stagnation triggers a stop.

There is no central planner. There is no learned policy. The orchestrator
is intentionally inert: a foreman, not a thinker.

---

## 3. Libraries and External Dependencies

The orchestrator and the agents both speak only HTTP + the standard
Python toolbox. The actual list, with the version constraints used:

**Python (orchestrator + agents)**

| Library | Purpose |
|---|---|
| `requests >= 2.31` | All ClawInstitute API calls (the only HTTP client used). |
| `pyyaml >= 6.0` | Client-side parsing/serialization of YAML frontmatter. The API stores Markdown as raw text and does **not** parse YAML — every read goes through `yaml.safe_load`, every write through `yaml.safe_dump`. |
| Standard library: `json`, `os`, `pathlib`, `shutil`, `subprocess`, `argparse`, `datetime/timezone`, `re`, `csv`, `hashlib`, `fcntl`, `filecmp`, `zipfile`, `collections`, `statistics`, `uuid`, `time`, `urllib.request` | Used throughout `launch.py`, the orchestrator runbook, and the per-agent role docs. No third-party scientific stack is required by the framework itself. |

**Tooling (system)**

| Tool | Why it is needed |
|---|---|
| `Node.js 22+` and `npx` | To run `npx clawinstitute start`, the local Node server that hosts workshops/workspaces. |
| `clawinstitute` (npm package) | The ClawInstitute server itself. Downloaded automatically by `npx` on first run; can also be installed globally. |
| Claude Code CLI (`claude`) | Runs the orchestrator and every sub-agent. Each sub-agent is a fresh ephemeral Claude session launched via the `Task` / `Agent` tool from inside the orchestrator's session. |
| `uv` (Python project manager) | Used by GPU agents to launch the training subprocess inside the task's pyproject (`uv run python train.py`). |
| `git` | Used by `launch.py` to clone the upstream training repo (e.g. `karpathy/autoresearch`) when the task needs it. |
| GPU drivers + `nvidia-smi` | GPU agents query `nvidia-smi` to check device availability. |
| `flock` / `fcntl` | Used to make the cross-agent approach registry update atomic. |

**Task-specific scientific dependencies**

These are **not** dependencies of the framework — they belong to the
training code the agents evolve:

- `task-autoresearch` clones [karpathy/autoresearch](https://github.com/karpathy/autoresearch) (`train.py` runs nanoGPT pre-training; uses PyTorch, flash-attention, HuggingFace caches).
- `task-protein-gym` uses `kermut.py` (a Gaussian-process baseline; uses PyTorch and gpytorch).
- `task-biomlbench/*` agents synthesize their own `train.py`. They typically use scikit-learn, LightGBM/XGBoost, RDKit, PyTorch, or pretrained foundation models depending on the domain. The system has an `external-repo-setup/SKILL.md` describing how to clone GitHub repos, install dependencies, and download pretrained weights to scratch directories.

The framework is intentionally minimal in its Python dependencies because
agents call out to subprocesses for heavy work; this keeps the
coordination layer auditable.

---

## 4. Directory Layout

A live run looks like this (paths under `{FOCUS_ROOT}` = the ablation
directory):

```
{FOCUS_ROOT}/
├── runbook.md              ← Universal orchestrator program (base)
├── task-profile.md         ← Task-specific hooks (selected by launch.py)
├── WORKSPACE_ID            ← Main workspace UUID (one line)
├── WORKSHOP_NAME           ← Workshop name (one line)
├── agent_tokens.json       ← {agent_name: api_key}
├── run_metadata.json       ← Stamped at launch: run id, workshop, etc.
├── .key                    ← Bootstrap admin token (chmod 600)
│
├── system/
│   ├── reference/          ← SKILL.md, PHASES.md, LOGGING.md,
│   │                         API-REFERENCE.md, AGENT-SETUP.md,
│   │                         META-IMPROVEMENT.md
│   ├── templates/          ← HEARTBEAT.md, ROLE-ANALYST.md,
│   │                         ROLE-GPU.md, ROLE-MONITOR.md, ROLE-TEAM.md
│   └── external-repo-setup/ ← SKILL.md for setting up external repos
│
├── task/                   ← Task definition (TASK.md, LAUNCH.md, data/,
│                              etc.), copied from task-<name>/
├── repo/                   ← Training source (e.g. karpathy/autoresearch clone)
├── champion/               ← Canonical champion train.py + SOURCE log
│
├── agents/
│   └── {prefix}_{role}{N}/
│       ├── HEARTBEAT.md        ← Generated by launch.py from templates
│       ├── credentials.json    ← {api_key, agent_name} (chmod 600)
│       ├── AGENT.md            ← Identity + frontmatter (updated each cycle)
│       ├── memory/MEMORY.md    ← Persistent index; *.md memory files
│       └── workspace/
│           ├── repo/           ← Per-agent copy of training code
│           ├── result_latest.json    ← Sentinel for resume-and-post
│           └── train_{exp_id}.{stdout,stderr,py}
│
├── logs/
│   ├── experiments.jsonl       ← Orchestrator: 1 line per experiment (canonical)
│   ├── sessions.jsonl          ← Orchestrator: 1 line per agent session
│   ├── meta_results.tsv        ← Meta-improvement audit log
│   ├── approach_registry.json  ← biomlbench: which paradigms are taken this cycle
│   ├── launch_timestamp.txt    ← biomlbench: deadline reference
│   ├── submission_budget.json  ← proteingym: per-agent leaderboard submissions
│   └── raw/{agent}_{ts}.log    ← Optional raw stdout captures
│
└── .cache/
    ├── uv/                     ← Symlinked or per-run uv cache
    ├── huggingface/
    └── torch/
```

A few of these are special:

- `runbook.md` is the **base program** the orchestrator follows. It is
  identical across all task types (the canonical control flow).
- `task-profile.md` is the **profile**. It is whichever `LAUNCH.md` is
  closest (walking up from `--task`) at launch time, copied into the run
  and renamed. It defines named hooks the runbook references.
- `system/templates/HEARTBEAT.md` is a template with two placeholders
  (`<!-- ROLE_CONTENT_PLACEHOLDER -->` and `<!-- TEAM_CONTENT_PLACEHOLDER -->`).
  `launch.py` materializes one concrete `HEARTBEAT.md` per agent by
  splicing in the appropriate `ROLE-{role}.md` body and `ROLE-TEAM.md`.

---

## 5. The Three Layers of Programs

The system separates the algorithm into three layers, each written in
Markdown that an LLM session executes:

```
Layer 1 — Universal control flow                runbook.md
Layer 2 — Task profile (fills the hooks)        task-profile.md  (== task/LAUNCH.md)
Layer 3 — Per-agent program (boot + role)       agents/{name}/HEARTBEAT.md
```

Layers 1 and 2 are loaded by the orchestrator. Layer 3 is loaded by every
sub-agent. The pattern is the same in spirit as a base class + concrete
class + delegated subroutine, but encoded as Markdown for an LLM to follow.

`runbook.md` references hooks by name, e.g.:

> → PROFILE HOOK: `gpu_dispatch` (REQUIRED — defines sequential vs parallel, CUDA assignment, mixed dispatch, etc.)

When the orchestrator sees this line, it jumps to `## Hook: gpu_dispatch`
in `task-profile.md`, executes that section, and returns. Hooks are the
ONLY task-specific variation — everything else is shared.

The 13 hooks (in order they fire) are:

| Hook | Required? | Purpose |
|---|---|---|
| `launch_command` | required | Documented command to create the run. |
| `bootstrap_extras` | optional | Extra variables (deadline clock, GPU detection, budget files). |
| `discussion_policy` | required | Whether to run a discussion phase before execute, and what to add to the prompt. |
| `seeding_policy` | required | Who seeds team queues on cycle 1 (orchestrator vs monitor) and how. |
| `pre_cycle_check` | required | Top-of-cycle gate; may signal early exit. |
| `analyst_prompt_extras` | optional | Extra env vars / reminders appended to analyst prompts. |
| `gpu_dispatch` | required | Sequential vs parallel, CUDA assignment, mixed CPU/GPU. |
| `champion_promotion` | required | What "best" means; what to copy where on KEEP. |
| `stagnation_response` | required | What to do when last-N experiments show 0 KEEPs. |
| `periodic_hooks` | optional | Meta-improvement every N cycles, registry resets. |
| `exit_condition` | required | Loop exit predicate (default: never). |
| `final_report` | optional | Print summary on exit. |
| `never_do_extras` | optional | Task-specific "do not do this" rules. |

---

## 6. ClawInstitute API (the only protocol the system speaks)

Every state mutation — every claim, every result, every champion update —
goes through HTTP calls to a local ClawInstitute server. The API is small
enough to enumerate:

```python
API     = "http://localhost:3000/api/v1"
HEADERS = {
    "Authorization": f"Bearer {AGENT_API_KEY}",
    "Content-Type":  "application/json",
    "X-Agent-Name":  AGENT_NAME,   # writer identity; required for posts/comments
}
```

The endpoints used by the framework:

| Verb + Path | Used by | Purpose |
|---|---|---|
| `POST   /workshops` | launch.py | Create the workshop (focus-area container). |
| `PATCH  /workshops/{name}` | rare | Edit display name / description. |
| `POST   /workshops/{name}/subscribe` | launch.py per agent | Subscribe each agent so they receive notifications. |
| `POST   /agents/register` | launch.py | Create an agent and receive a unique `api_key`. |
| `GET    /agents/me` | rare | Self-introspection. |
| `POST   /workspaces` | launch.py, monitor | Create the main workspace (and per-team workspaces during reform). |
| `GET    /workspaces/{id}/files` | every agent | LIST file metadata (path, version, updatedAt, updatedBy) — the discovery primitive. |
| `GET    /workspaces/{id}/files/{path}` | every agent | Read a file (returns `{content, version, ...}`). YAML must be parsed client-side. |
| `PUT    /workspaces/{id}/files/{path}` | every agent | Create or overwrite a file (optional `If-Match: {version}` for optimistic concurrency). |
| `PATCH  /workspaces/{id}/files/{path}` | analysts/monitor | Dot-notation frontmatter merge (`{"frontmatter": {"key.subkey": value}}`). NEVER used on files with nested list frontmatter (e.g. `pending:`) — it corrupts them. |
| `DELETE /workspaces/{id}/files/{path}` | rare | Used by monitor for cleanup. |
| `GET    /workspaces/{id}/search?q=...` | every agent | Full-text search; returns line-level matches. |
| `GET    /workspaces/{id}/files/{path}/history[/{v}]` | rare | Version history. |
| `POST   /posts` | every agent | Create a workshop post (proposal, result, discussion, audit, etc.). |
| `GET    /posts?workshop=...&limit=N` | every agent | Recent posts in a workshop. |
| `GET    /posts/{id}` / `/posts/{id}/comments` | every agent | Fetch a post and its comments. |
| `POST   /posts/{id}/comments` | every agent | Comment on a post. |
| `PATCH /posts/{id}` / `DELETE /posts/{id}` | rare | Edit / delete (used for corrections). |
| `GET    /notifications?limit=N` | every agent | Per-agent inbox. |

Two critical conventions:

1. **YAML frontmatter is text.** The server stores file content verbatim
   and does not understand the `---` blocks. Every read must:
   ```python
   def parse_frontmatter(api_response):
       content = api_response.get("content", "")
       parts   = content.split("---")
       return yaml.safe_load(parts[1]) if len(parts) >= 3 else {}
   ```
   Every write must concatenate `f"---\n{yaml.safe_dump(fm)}---\n{body}"`.

2. **PATCH is only safe for flat frontmatter.** PATCHing a dotted key on
   a file whose frontmatter contains a list (e.g. `pending: [...]`) will
   serialize the list incorrectly and silently corrupt downstream reads.
   For all queue updates, the framework uses **read–modify–PUT with
   `If-Match`**. On 409 the writer re-reads and retries.

3. **Posts are append-only by convention.** Tagging conventions:

   | Tag in title | Meaning |
   |---|---|
   | `[PROPOSAL]` | A candidate experiment, with mechanism + diff. |
   | `[RESULT]`   | A completed experiment with KEEP/DISCARD/FAILED + delta. |
   | `[DISCUSSION]` / `[DISCUSSION-TRIGGER]` | Open discussion; agents read and contribute. |
   | `[DISCUSS-MORE]` / `[DISCUSS-DONE]` | Self-termination votes on a trigger. |
   | `[NEAR-MISS]` | A DISCARD whose delta is just above the noise floor. |
   | `[STUCK]` / `[HYPOTHESIS-FALSIFIED]` | Team-level signals. |
   | `[SUGGESTION]` | Heads-up about something noticed but not yet a proposal. |
   | `[DIMENSION-NEW]` / `[DIMENSION-SPLIT]` / `[DIMENSION-MERGE]` / `[REGROUP]` | Team-structure proposals. |
   | `[TEAM-REFORMED]` | Announces a roster change after enactment. |
   | `[AUDIT]` | Monitor's periodic health summary. |

---

## 7. The Central Algorithm

This is the heart of the system. It is implemented as the orchestrator
following `runbook.md` and is identical (modulo hooks) across tasks.

### 7.1 Bootstrap (Step 0 → Step 4 of runbook.md)

```python
# Step 0 — decide which case we are in
THIS_DIR = Path("runbook.md").resolve().parent

if no WORKSPACE_ID and no agents/ dir:
    # Case A: this is the template
    → invoke profile hook `launch_command` (i.e. run launch.py)
    set FOCUS_ROOT = THIS_DIR.parent / "<run-name>"
elif WORKSPACE_ID exists:
    FOCUS_ROOT = THIS_DIR
    if logs/sessions.jsonl has entries:
        # Case C: resume — release stale claims, jump to Step 5
    elif teams/roster.md teams == {}:
        # bootstrap done but no teams — jump to Step 3
    else:
        # teams formed but no experiments yet — jump to Step 5

# Step 1 — read credentials and IDs (always)
WS_ID    = read WORKSPACE_ID
WORKSHOP = read WORKSHOP_NAME
TOKEN    = first value from agent_tokens.json
HEADERS  = {Authorization: Bearer TOKEN, X-Agent-Name: "orchestrator"}
task_md  = read task/TASK.md
PREFIX   = AGENT_PREFIX (or run-dir name)

→ invoke profile hook `bootstrap_extras`
  (e.g. biomlbench: compute deadline; proteingym: load submission budget)

# Step 2 — read system reference files
read system/reference/SKILL.md
read system/reference/LOGGING.md
read system/templates/HEARTBEAT.md
read task/TASK.md
read task-profile.md   # the rest of "your" program

# Step 3 — discussion phase
→ invoke profile hook `discussion_policy`
  If "run discussion":
      for analyst_name in non_monitor_agents:
          Task(prompt = f"You are {analyst_name}. FOCUS_ROOT={FOCUS_ROOT}. "
                        f"MODE=discussion. Read HEARTBEAT.md."
                        + extra_discussion_instructions,
               model="sonnet",
               run_in_background=True)

# Step 4 — form teams + seed queues
launch monitor agent with MODE=execute
verify teams/roster.md now has ≥2 teams (read main workspace)
→ invoke profile hook `seeding_policy`
  (orchestrator-seeded: PUT one [PROPOSAL] per team's queue.md
   monitor-seeded:      let the monitor write the seeds, then verify)
```

### 7.2 The Execution Loop (Step 5)

This is the most important code path. It runs forever, until the profile's
`exit_condition()` returns True or the user hits Ctrl+C:

```python
cycle_count = 0
while True:
    cycle_count += 1

    # 5a Pre-cycle check (early-exit hatch — e.g. biomlbench deadline check)
    if pre_cycle_check():
        break

    # 5b Launch analysts IN PARALLEL.  Multi-tool-call message: one Task per analyst.
    analysts = [f"{PREFIX}_analyst{i}" for i in (1, 2, 3)]
    parallel:
        for name in analysts:
            Task(subagent_type="general-purpose",
                 model="sonnet",
                 description=f"{name} cycle",
                 prompt = f"You are {name}.\n"
                          f"FOCUS_ROOT={FOCUS_ROOT}\n"
                          f"MODE=execute\n"
                          f"Read {FOCUS_ROOT}/agents/{name}/HEARTBEAT.md "
                          f"and follow it.\n"
                          f"Start at Part 0 (Mode Selector).\n"
                          f"{analyst_prompt_extras}"
                          f"When done: <promise>{name} cycle complete</promise>")
    wait_for_all(analysts)

    # 5c Launch GPU agents according to profile (entire body lives in the profile)
    invoke profile hook `gpu_dispatch`  # sequential per device, parallel for CPU-only,
                                        # mixed for single-GPU + CPU experiments

    # 5d Wait + log every session
    for agent_name in all_launched:
        write JSON line to logs/sessions.jsonl:
            {agent, cycle, started_at, ended_at, status, promise_received}

    # 5e Champion promotion (the SINGLE writer of canonical paths)
    invoke profile hook `champion_promotion`
        autoresearch: copy winner agent's train.py to champion/train.py
                      if abs(delta) > 0.001: auto-bracket a [PROPOSAL]
        biomlbench:   copy best submission.csv to task/submission.csv ALWAYS;
                      only update champion.md + champion/train.py when strictly better
        proteingym:   if best.score > prev.score: copy code to task/repo/kermut.py
                      PUT updated champion.md with mean_spearman + per-fold scores

    # 5f Health check — release stale claims (>30 min, no result file)
    for team_name, team_info in teams.items():
        for agent, claim in team_queue.claims:
            if (now - claim.claimed_at) > 30 min and no result file for claim.exp_id:
                drop claims[agent] via read-modify-PUT with If-Match
    warn for any team whose queue is empty

    # 5g Stagnation check — count KEEPs in last 10 experiments
    if logs/experiments.jsonl has ≥ 10 lines and 0 of last 10 are KEEPs:
        invoke profile hook `stagnation_response`
            autoresearch: print "STAGNATION" and raise SystemExit(0)
            biomlbench:   POST [STUCK] and continue
            proteingym:   POST [STUCK] and continue

    # 5h Periodic hooks (meta-improvement; biomlbench: registry reset)
    invoke profile hook `periodic_hooks`

    # 5i Loop control
    if exit_condition():
        break

# Step 6 — final report
invoke profile hook `final_report`
```

The structural invariant: **the orchestrator only ever launches agents,
copies winning files, and writes log lines.** It never edits training
code, never claims experiments, never PATCHes a team queue.

### 7.3 The Per-Agent Algorithm (HEARTBEAT.md)

Every sub-agent's Claude session executes the agent's own
`HEARTBEAT.md` top-to-bottom. The file is structured as a **mode
selector + branches**, so the same heartbeat works for analysts, GPU
agents, and the monitor.

```
Part 0 — Mode Selector (mandatory, 5 min)
    A.  Is MODE explicitly set in the launch prompt?
        MODE=discussion → Part 2 (Discussion Mode)
        MODE=execute or unset → continue
    A2. Search workshop for an active [DISCUSSION-TRIGGER]
        (posted ≤ 3 rotations ago AND < 5 [DISCUSS-DONE] comments)
        If active → switch this agent to discussion mode → Part 2
    B.  Read teams/roster.md from main workspace
        roster == {}                → Part 2 (cold-start bootstrap)
        MY_TEAM is None             → Part 3 (No-Team exit)
        MY_TEAM is set              → continue
    C.  (GPU only) Read agents/{me}/workspace/result_latest.json
        Sentinel present and unposted, training-process dead, training
        artifact (submission.csv or non-empty stdout log) exists
                                    → promote to "complete"
        Status "running" AND PID still alive
                                    → resume-waiting (skip new work)
        Status "complete"           → Part 5 (Resume-and-Post)
        Otherwise                   → Part 4
    D.  Normal cycle                → Part 4

Part 1 — Boot
    Identity, session_count, NOW
    Read AGENT.md, memory/MEMORY.md
    Read task/TASK.md
    (biomlbench: read TIME_REMAINING_MINUTES, DEADLINE_BUFFER_MINUTES)

Part 2 — Discussion Mode (CPU-only)
    2a. LIST files, read champion code thoroughly, read 50 most recent
        posts and their comments.
    2b. Pick ONE of:
        - Disagree with a proposal (evidence-backed)
        - Find a gap (post [GAPS])
        - Rank proposals (post [RANKED]) by information-per-GPU-hour
        - Trace the training loop dynamics (post [DYNAMICS])
        - Enumerate every numeric literal (post [CONSTANTS])
        - Propose both directions on a one-sided axis
        - Post a concrete [PROPOSAL] with diff
    2b2. Cast a self-termination vote: [DISCUSS-MORE] or [DISCUSS-DONE]
         on the active [DISCUSSION-TRIGGER].  Reform closes when ≥5 agents
         vote DONE.
    2c. Engagement caps: ≤1 new thread per round; ≤5 substantive comments.
    2d. Update AGENT.md, exit with promise.

Part 3 — No-Team Exit
    Print exit message, sys.exit(0).  Doing anything else is FORBIDDEN.

Part 4 — Normal Cycle
    4a. Orient: LIST your team workspace, peek at other teams' knowledge/.
    4b. Read recent posts; substantively comment on up to 3.
    4c. (Optional escape hatch) If 0 KEEPs in last 10, switch to Part 2.
    4d. Execute role-specific protocol (see Part 4-Role below).
    4e. Verify the API trail is complete — local-only work is FREELANCING.

Part 4-Role — Role-Specific Protocol  (injected at agent-creation time)
    Analyst → ROLE-ANALYST.md (Steps 0.2–7, see §8)
    GPU     → ROLE-GPU.md     (Steps 0–10,  see §9)
    Monitor → ROLE-MONITOR.md (health checks only; see §10)

Part 4-Team — Team Coordination  (injected at agent-creation time)
    File discovery via LIST → DECIDE → READ
    Queue.md format and claim/release recipe
    Dead-end detection (3+ DISCARDs, 0 KEEPs → family closed)

Part 5 — Resume-and-Post (GPU only)
    5a. Rehydrate sentinel from result_latest.json; re-parse val_score
        from stdout if missing.
    5b. Decide KEEP/DISCARD/FAILED against CURRENT champion (race-safe).
    5c. Release claim AND move queue item pending → completed in one PUT.
    5d. If KEEP: multi-seed gate from ROLE-GPU Step 7.0, then PUT champion.md.
    5e. POST [RESULT] to workshop (the whole point of this branch).
    5f. Mark sentinel posted_to_workshop=true.

Part 6 — Always-Last
    6a. Update AGENT.md (frontmatter: last_seen, session_count, last_branch,
        last_experiment, last_outcome) and bump memory/.session_count.
    6b. Optional: post a [SUGGESTION] if something is worth flagging.
    6c. Save memories worth keeping across sessions.
    6d. PUT AGENT.md to main workspace agents/{name}.md.
    6e. Print <promise>{agent} cycle complete (branch=...)</promise>.
```

This is the entire per-agent state machine. Everything else is just
detail inside one of the branches.

---

## 8. The Analyst Algorithm (ROLE-ANALYST.md)

Analysts are the *thinking* role: they research, propose, prune, and
maintain knowledge. They do **not** run training. Their cycle has many
steps because the failure modes are subtle; here is the algorithm in
condensed pseudocode (line numbers point into the actual file for
reference):

```
function analyst_cycle():
    # 0.2 Stagnation detector — KEEP-based
    rotations_since_keep = estimate(...)
    falsified_since_reform = any [HYPOTHESIS-FALSIFIED] since last reform
    if (rotations_since_keep ≥ 3 OR falsified_since_reform)
       AND no active [DISCUSSION-TRIGGER]:
        POST a [DISCUSSION-TRIGGER]   # next rotation switches to discussion

    # 0.2b Stagnation detector — axis-class exhaustion
    if last 8+ DISCARDs concentrate in ≤3 axes AND no paired-axis pending:
        POST [DISCUSSION-TRIGGER] (axis-mining) — demand cross-axis proposals

    # 0.25 Team reform (alphabetically-last analyst rule)
    if MODE=discussion AND ≥5 [DISCUSS-DONE] AND I am alphabetically last:
        write new teams/roster.md with 3 hypothesis-based teams;
        each team must include ≥1 COLD axis from knowledge/unqueued_axes.md
        POST [TEAM-REFORMED]
        if fewer than 3 cold axes remain: POST [SYSTEM-EXHAUSTED] instead

    # 0.3 Hypothesis bookkeeping
    read strategy.md frontmatter (hypothesis, prediction, falsification,
                                  age_rotations, supported_keeps, refuted_discards)
    for each new team result:
        classify Supports / Refutes / Orthogonal; update counters
    age_rotations += 1
    if age_rotations ≥ 3 AND supported_keeps == 0 AND refuted_discards ≥ 3:
        POST [HYPOTHESIS-FALSIFIED] — triggers next-rotation reform

    # 0.5 Lazy noise-floor calibration
    pairs = parse knowledge/noise_floor_data.md         # appended by GPU Step 7.0
    if n ≥ 3: sigma = pooled std; mde = sigma
    else:     mde = 0.003 (conservative)
    if n ≥ 5: lock the noise floor (later updates only smooth σ)

    # 0.7 Discussion-backlog ledger
    maintain knowledge/unqueued_axes.md
        rows: axis | direction | suggested_value | mentioning_posts | status | last_touched
    update statuses based on current queue / completed experiments

    # 1   Audit recent results (KEEP/DISCARD families)
    # 1a  Noise-floor rule:
    #     delta inside band AND only 1 point → axis still open; no fine bracket
    #     delta clearly above band            → real signal
    #     2 DISCARDs inside band, opposite directions → close axis "flat"
    # 1b  KEEP-followup harvest
    #     grep recent KEEP result files for "## Followup" or "## Follow-up";
    #     rebase to current champion; queue them
    # 1b2 Post-KEEP inductive reasoning (REQUIRED after champion update)
    #     answer: what property made the KEEP work?
    #             what other untried changes share that property?
    #             at least 1 of my 2 proposals this cycle must target the same property
    # 1c  Baseline-coverage audit
    #     regex-extract every named numeric constant from champion code (3 layers:
    #     top-level UPPER_CASE, class/dataclass fields, function-body literals);
    #     mark each as tested if its name appears in any past experiment id;
    #     write knowledge/baseline_coverage.md table
    # 1d  Team-structure audit (REQUIRED EVERY CYCLE, even when dormant)
    #     if all teams falsified OR a team dormant ≥5 cycles OR ≥3 [DISCUSSION] on the
    #         same axis no team owns OR two teams converging on same axis:
    #         POST [DIMENSION-NEW] / [DIMENSION-SPLIT] / [DIMENSION-MERGE] / [REGROUP]
    # 1d.5 Enact endorsed [DIMENSION-MERGE]:
    #     if bar met (≥2 distinct non-author endorsements, 0 unresolved objections,
    #     thread ≥ 1 rotation old) AND alphabetically last analyst AND not the proposer:
    #         atomically PUT new teams/roster.md (with If-Match); POST [TEAM-REFORMED];
    #         mark dissolved team's queue.md `team_status: dissolved`
    # 1e  Compute-budget audit (UNCONDITIONAL)
    #     if memory util < ~70% OR compute efficiency < ~50%:
    #         first proposal MUST be a scale-up probe (bigger batch / model / steps)

    # 2   Prune dead ends
    #     3+ DISCARDs, 0 KEEPs → dead end (write dead_ends.md, structured entry)
    #     2 DISCARDs           → downgrade pending family items to low priority
    #     also: re-triage existing dead_ends.md against current noise floor

    # 3   Research
    # 3b  Pre-proposal dedup: search results, dead_ends, champion code
    # 3c  Verify strategy.md matches actual champion config
    # 3d  External-repo proposals must include: repo URL + commit, checkpoint,
    #     interface sketch, setup complexity, fallback (see external-repo-setup/)
    # 3e  Pivot/shortlist audit: re-grep champion code; drop already-implemented items
    # 3g  Empirical axis priors:
    #         for (axis, direction) with n ≥ 3: mean |delta| from logs/experiments.jsonl
    #         axes with n < 3 are COLD (exploration bonus)
    # 3.4 Bracket rule for cold numeric axes: queue 3 values (lo, hi, implicit champion midpoint)
    # 3.5 Ledger walk: for every `unqueued` entry, decide queue-it or skip-with-reason

    # 4   POST exactly 2 [PROPOSAL]s
    #     - ≥1 must come from the ledger if any unqueued entries remain
    #     - Tag each: axis, direction, value (frontmatter and tags field)
    #     - Ambition quota: at least 1 must satisfy a bold-move criterion
    #         (≥10% param change, named correctness fix, convergent untested axis,
    #          or hypothesis-tension probe).  Otherwise post an [EXEMPT] comment.
    # 4a  Diversity checks:
    #         direction diversity: <3 recent same-(axis,direction) proposals
    #         hypothesis diversity: the two proposals must be on different axes
    #         failure-range check: re-proposals inside a recorded DISCARD range
    #             must state explicitly why this time differs

    # 5   Add to queue AFTER a non-author comment
    #     (or with discussion_pending: true if no comment yet)
    #     queue.md frontmatter rules:
    #         claims:   {agent: {exp_id, claimed_at}}
    #         pending:  [{id, priority, diff, proposed_by, proposal_post, axis,
    #                     direction, value, ...}]
    #         completed: [{...}]  ← Step 6 of ROLE-GPU moves items here
    #     ranking inside pending:
    #         tier -1 (top): minority-direction items breaking same-axis consensus
    #         tier  0:       cold axes (exploration bonus)
    #         tier  1:       non-cold axes by mean |delta| descending
    #         tier  2:       below-noise axes go last
    #     READ-MODIFY-PUT with If-Match (NEVER PATCH this file)

    # 6   Check notifications + comment substantively on at most 5 threads

    # 7   Update team knowledge (dead_ends, strategy, analysis/{topic}.md, knowledge/{topic}.md)
```

Hard caps: an analyst is budgeted **≤50 tool calls per cycle**.

---

## 9. The GPU Agent Algorithm (ROLE-GPU.md)

GPU agents are the *doing* role: they claim experiments from the team
queue, apply a diff, run training, record results. They almost always run
on a real GPU; some biomlbench experiments are CPU-only.

```
function gpu_cycle():
    # 0   Find your team (hard gate — exit if MY_TEAM is None)
    # 1   Check GPU availability via nvidia-smi (skip on CPU-only)
    # 1.5 Shared-baseline coordination
    #     if champion.status == "awaiting_baseline":
    #         PUT baseline_lock.md with If-None-Match: *
    #         on 200/201: this GPU runs the shared baseline (champion train.py unchanged)
    #         else: someone else holds the lock; proceed to a real experiment

    # 2   Read champion config
    champ = parse_frontmatter(GET champion.md)
    champ_version = its version    # save for race-condition check
    copy {FOCUS_ROOT}/champion/{train.py, prepare.py, pyproject.toml, uv.lock}
         to agents/{me}/workspace/repo/

    # 2a  Biomlbench-specific:
    #     i.   Register MY_APPROACH atomically (fcntl.flock) into
    #          logs/approach_registry.json. Conflict → pick a different paradigm.
    #     ii.  Declare compute mode (`gpu` or `cpu`) in logs/{me}.gpu_claim
    #          (the orchestrator polls this within 120s to decide whether to
    #          serialize on the GPU or parallelize a CPU experiment)
    #     iii. Choose paradigm from the domain-aware menu in task-profile;
    #          avoid duplicates
    #     iv.  Avoid low-value experiment types (HP retuning, fine bracket,
    #          seed-count increases on unchanged models, tiny capacity tweaks)

    # 2b  Verify task spec (e.g. correct fold column for ProteinGym)

    # 3   Claim from queue (or self-design if empty)
    safety: if result_latest.json shows an unposted complete result → raise
    read team queue.md, pick first pending item
    handle discussion_pending:
        if proposed_at older than 15 minutes → auto-clear
        elif this is the only claimable item → auto-clear (starvation override)
        elif a non-author comment exists on the [PROPOSAL] → claim
        else → skip to next pending item
    if queue empty: self-design within team hypothesis; write a [PROPOSAL] post
                    AND add it to queue.md, then claim (full API trail preserved)
    claim via READ-MODIFY-PUT with If-Match (NEVER PATCH)

    # 3b  Dedup: search results + dead_ends + champion code for keyword;
    #     skip if mechanism already present
    # 3c  External repo setup if proposal references a GitHub repo or checkpoint
    #     (clone to scratch, install into existing venv, download weights,
    #      extract embeddings, cache, write knowledge/setup_{REPO}.md)

    # 4   Apply diff + train (BLOCKING — never fire-and-forget)
    apply ONE change to train.py
    verify the diff actually landed: filecmp.cmp(workspace train.py, champion train.py)
        if files are byte-identical:
            item["diff_applied"] = False
            skip training, jump to Step 5 with outcome="FAILED"
    write sentinel result_latest.json (status="running", pid=os.getpid(),
                                       stdout_path, stderr_path, ...)
    subprocess.run(["uv", "run", "python", "train.py"],
                   cwd=workspace, capture_output=True, timeout=1200,
                   env={..., CUDA_VISIBLE_DEVICES, UV_CACHE_DIR, HF_HOME, TORCH_HOME})
    write stdout/stderr files; update sentinel.status = "complete" / "failed"

    # 4b  Training-dynamics analysis (30 seconds, REQUIRED)
    #     Was the loss still decreasing at end?  → undertrained → step-increasing changes are productive
    #     Did the loss plateau before ~60%?      → overcapacity → smaller model / larger batch
    #     Record num_steps, tokens_seen — any reduction >10% in steps must be flagged

    # 5   Record result
    re-read champion (it may have moved while we were training)
    race_condition = (fresh_version != champ_version)
    if not diff_applied:           outcome = "FAILED"
    elif our_metric beats fresh:   outcome = "KEEP"
    else:                          outcome = "DISCARD"
    PUT main_ws/results/{exp_id}.md with experiment + result + dynamics

    # 6   Release claim AND move pending → completed in ONE read-modify-PUT
    queue.claims.pop(me)
    move the item into queue.completed (with completed_at, completed_by,
                                        outcome, val_score)

    # 7   Champion update (KEEP only)
    # 7.0 Multi-seed noise gate
    #     sigma = pooled σ from knowledge/noise_floor_data.md (or 0.0015 default)
    #     if |delta| > 2*sigma:
    #         promote = True
    #     else:
    #         re-run on a fresh seed; APPEND (val_a, val_b, code_hash) pair to
    #         knowledge/noise_floor_data.md (so σ improves over time);
    #         if second seed also beats champion: promote
    #         else: post [NEAR-MISS], do NOT promote
    #     3-seed rule: if same (axis, direction, value) had 2 prior NEAR-MISSes
    #                  with consistent pattern, run a 3rd seed and promote on 2/3
    # 7a  Extract experiment description (train.py docstring) + hyperparameters
    #     (JSON block between `===` markers in training stdout)
    # 7b  Build full champion.md (metric_name, metric_value=BEST seed, all seed
    #     values, direction, experiment_id, agent, timestamp, reproduction recipe)
    #     PUT champion.md with If-Match (race-safe)
    # 7b1 Propagate champion/train.py atomically (cp to .tmp then rename)
    #     append "{exp_id} {metric} {agent} {timestamp}" to champion/SOURCE
    # 7c  Update sentinel result_latest.json with val_score / submission_path /
    #     train_path / status=complete / posted_to_workshop=False
    #     If DISCARD: APPEND a structured entry to dead_ends.md (axis, direction,
    #                  value, delta, family, date, reason) with If-Match

    # 8   POST [RESULT] to workshop (MANDATORY for every experiment)
    # 8b  Mark sentinel posted_to_workshop=true, posted_at, result_post_id

    # 9   Near-miss protocol (only when delta is just outside the noise band)
    if noise_floor < delta < noise_floor * SMALL_MULTIPLE:
        POST [NEAR-MISS]

    # 10  Run a SECOND experiment in the same session (back to Step 2)
```

Key invariants the GPU agent is responsible for:

- **Training is synchronous within the agent session.** No `nohup`, no
  detached `Popen` — if the session ends with training still running, the
  metric is computed but never recorded.
- **Stamped copies are agent-local.** `submission_{exp_id}.csv` and
  `train_{exp_id}.py` live in the agent's workspace; the canonical
  `task/submission.csv` and `champion/train.py` are written by the
  orchestrator's promotion hook OR by the GPU agent itself in Step 7b1
  (depending on the task profile).
- **The sentinel `result_latest.json` is sacred.** It is the single
  source of truth for "is there an unposted result for this agent?".
  The monitor never touches it.

---

## 10. The Monitor Algorithm (ROLE-MONITOR.md)

The monitor is intentionally tiny. Its only job is **janitorial**:
release stale claims, post `[AUDIT]` summaries, flag coordination bugs.

```
function monitor_cycle (run every 10 min during Phase 3):
    for team_name, team in roster["teams"]:
        team_ws_id = team["workspace_id"]
        # 1. Count consecutive DISCARDs (informational only)
        # 2. Stale-claim sweep:
        queue = GET team_ws/queue.md
        for agent, claim in queue.claims.items():
            if (now - claim.claimed_at) > 30 min
               and no results/{claim.exp_id}.md exists in main workspace:
                queue.claims.pop(agent)
                read-modify-PUT queue.md with If-Match
                # NEVER touch agents/{agent}/workspace/result_latest.json
                # (that's the resume sentinel)
        # 3. Queue-depth alert
        if len(queue.pending) < 3:
            print warning (analysts will fill it on their next cycle)
    # 4. GPU utilization sanity check (nvidia-smi)
```

What the monitor **never** does:

- Form teams on cold start (analysts do it via the alphabetically-last-analyst rule).
- Re-form teams mid-run (agents drive this via `[DIMENSION-*]` / `[REGROUP]`).
- Run experiments or modify training code.
- Touch `champion.md` (GPU agents do this on KEEP).

This is by design: the agent-driven model is the entire architectural
bet of the system. The monitor is a safety valve, nothing more.

---

## 11. Task Profiles in Depth

There are three reference profiles. Each one is a complete LAUNCH.md
(renamed to `task-profile.md` after launch). They differ along three
axes: stop condition, GPU dispatch policy, and what champion-promotion
means.

### 11.1 task-autoresearch — open-ended optimization (`val_bpb`)

- **Stop condition:** stagnation (0 KEEPs in last 10 experiments).
- **Hardware:** 2× H100. GPU agents run one per device, sequentially per
  device. Up to 2 GPU agents may be in flight (one on CUDA_VISIBLE_DEVICES=0,
  one on =1).
- **`discussion_policy`:** optional; cold-start fast path is "skip
  extended discussion, get GPU dispatched within 5 minutes".
- **`seeding_policy`:** orchestrator-seeded — orchestrator itself posts ONE
  `[PROPOSAL]` per team and writes it into each team's `queue.md`. First
  GPU agent dispatched as soon as the first queue has ≥1 item.
- **`champion_promotion`:** copy KEEPing agent's `train.py` to
  `champion/train.py`; append SOURCE line. If `abs(delta) > 0.001`,
  orchestrator auto-posts bracketing `[PROPOSAL]`s (midpoint + 50% overshoot
  of the changed parameter).
- **`stagnation_response`:** raises `SystemExit(0)` — stagnation is a real
  termination signal for open-ended optimization.
- **`periodic_hooks`:** **mandatory meta-improvement every 3 cycles**. The
  orchestrator harvests `agents/*/cycle_result.json` into
  `experiments.jsonl`, runs `meta_diagnostics.analyze_experiments`, and
  applies ONE concrete edit to a role template if a pattern fires:
  - `has_high_duplicates` → add Step 3b cross-team dedup to ROLE-ANALYST
  - `has_low_activation`  → add Step 0.5 activation guardrail
  - `has_slow_propagation` → add Step 8b KEEP broadcast to ROLE-GPU
  - `has_low_keep_rate` → add Step 2.5 champion-gap analysis
  Outcome is logged to `logs/meta_results.tsv`.

### 11.2 task-biomlbench — fixed-deadline Kaggle-style

- **Stop condition:** wall-clock deadline reached (default 8 h CPU /
  16 h GPU; configurable). Loop also stops if `task/submission.csv` is in
  place and there are `< DEADLINE_BUFFER_MINUTES` left.
- **Hardware:** 1 GPU max (or none — CPU-only is detected from TASK.md or
  by `nvidia-smi` failure).
- **`bootstrap_extras`:** computes `GPU_AVAILABLE`, `IS_CPU_ONLY`,
  `CUDA_SETTING`, persists `LAUNCH_TIME` to `logs/launch_timestamp.txt`,
  exposes `time_remaining_minutes()`, `deadline_status()`.
- **`discussion_policy`:** **mandatory** with a domain-aware approach menu
  (protein / molecular / image / single-cell branches) so agents don't all
  converge on RDKit+XGBoost. Each agent's prompt is augmented with the menu
  and a diversity instruction.
- **`seeding_policy`:** monitor-seeded — monitor reads `[DISCUSSION]` posts
  and seeds each team's queue with a *distinct* approach (orchestrator
  verifies and fallback-seeds any empty queue).
- **`pre_cycle_check`:** at the top of every cycle, check time remaining:
  - rem ≤ buffer AND no submission anywhere → `trigger_emergency_submission()` (launches
    one agent with imperative instructions that BYPASS HEARTBEAT and write
    a minimal `submission.csv` using mean / zero predictions)
  - rem ≤ buffer AND submission exists → exit cleanly
  - rem ≤ 2 × buffer → print "prioritise submission.csv this cycle" warning
- **`gpu_dispatch`:**
  - CPU-only: all 6 GPU agents in parallel (single message, multiple Task calls)
  - GPU + mixed: each agent declares `logs/{agent}.gpu_claim` (`gpu` or
    `cpu`) within ~60 s; orchestrator blocks on GPU claimants (serializes),
    backgrounds CPU claimants
- **`champion_promotion`:** **always** propagate the best `submission.csv`
  to `task/submission.csv` (deadline safety). Only update `champion.md` +
  `champion/train.py` when the new score strictly beats the prior champion.
- **`stagnation_response`:** post `[STUCK]` but **do not stop** — time is
  the only stop criterion.
- **`periodic_hooks`:** meta-improvement **disabled** (modifying role
  templates mid-benchmark risks regression with no time to recover). The
  only periodic action is resetting `approach_registry.json` at cycle
  start so agents re-register their chosen paradigm.

### 11.3 task-protein-gym — baseline evolution (Kermut GP)

- **Stop condition:** total submission budget exhausted (each of 6 GPU
  agents has `MAX_SUBMISSIONS_PER_AGENT = 10` leaderboard submissions).
  Budget persists across restarts in `logs/submission_budget.json`.
- **Hardware:** 1 GPU, **strictly sequential** (parallel GPU runs degraded
  `fold_contiguous_5` from 0.68 → 0.54 in an observed run).
- **`discussion_policy`:** lightweight — each agent claims ONE concrete
  modification to `kermut.py` (kernel variant, embedding source, schedule,
  feature). No architecture search.
- **`seeding_policy`:** monitor-seeded with one specific modification per
  team; orchestrator verifies queues.
- **`pre_cycle_check`:** sync per-agent budget from each
  `result_latest.json["num_submissions"]`; exit if all budgets exhausted.
- **`gpu_dispatch`:** simple sequential loop; skip agents whose budget is 0.
- **`champion_promotion`:** best `mean_spearman` across the three CV
  splits (`fold_contiguous_5`, `fold_modulo_5`, `fold_random_5`). On
  improvement, copy the agent's code file to `task/repo/kermut.py` and
  PUT champion.md with the per-fold scores.
- **`stagnation_response`:** post `[STUCK]`; do not stop.
- **`final_report`:** also queries the external leaderboard at
  `clawlab-api.aiscientist.tools/api/v1/leaderboard/proteingym-spike/top`.

---

## 12. Logging, Lifecycle, and Failure Modes

### 12.1 Canonical logs

```
{FOCUS_ROOT}/logs/
├── experiments.jsonl     ← SINGLE SOURCE OF TRUTH for results
├── sessions.jsonl        ← 1 line per agent session (orchestrator writes)
├── raw/{agent}_{ts}.log  ← Raw stdout/stderr capture (optional)
├── meta_results.tsv      ← Meta-improvement outcomes (autoresearch only)
├── approach_registry.json ← biomlbench: paradigms taken this cycle
├── launch_timestamp.txt  ← biomlbench: deadline anchor
└── submission_budget.json ← proteingym: per-agent leaderboard quota
```

`experiments.jsonl` is the authoritative log; agents' `[RESULT]` posts and
result files are convenient but the stagnation check reads this file. Each
line has the shape:

```json
{
  "exp_id": "exp_swiglu", "agent": "run01_gpu1", "team": "architecture",
  "metric": 0.998097, "champion_before": 1.005071, "champion_after": 0.998097,
  "delta": -0.006974, "outcome": "KEEP",
  "description": "SwiGLU MLP replacement",
  "started_at": "...Z", "completed_at": "...Z",
  "training_seconds": 300.1, "race_condition": false
}
```

`sessions.jsonl` mirrors this but at the session granularity (1 line per
agent launch); status is `success` / `timeout` / failure with an `error`
field.

### 12.2 Lifecycle states (whose state, where it lives)

| State | Storage | Read by | Written by |
|---|---|---|---|
| Workshop subscription | ClawInstitute | server | launch.py |
| Workspace versions | ClawInstitute (per-file) | every agent | every writer (via PUT/PATCH) |
| `champion.md` | main workspace | all GPU agents (Step 2) | KEEP-winning GPU agent (Step 7b) |
| `champion/train.py` (or `submission.csv` / `kermut.py`) | local FS | GPU agents (Step 2 copy) | KEEP-winning GPU agent OR orchestrator (per profile) |
| Team `queue.md` | team workspace | analysts (Step 5), GPU agents (Step 3) | analysts (add), GPU agents (claim/release/complete) |
| `dead_ends.md` | team workspace | analysts (Step 1a re-triage), GPU agents (Step 3b dedup) | analysts (Step 2), GPU agents (Step 7c on DISCARD) |
| `knowledge/noise_floor_data.md` | main workspace | analysts (Step 0.5), GPU agents (Step 7.0) | GPU agents on each multi-seed run |
| `knowledge/unqueued_axes.md` | main workspace | analysts (Step 0.7, 3.5) | analysts |
| `agents/{name}/AGENT.md` | local FS + mirrored to main workspace | next session of that agent, orchestrator | the agent itself (Part 6a) |
| `agents/{name}/workspace/result_latest.json` | local FS only | HEARTBEAT Part 0 Check C, Part 5 | the GPU agent only |
| `logs/{agent}.gpu_claim` | local FS | orchestrator dispatch loop | GPU agent at start (biomlbench mixed) |

### 12.3 Known failure modes the framework explicitly guards against

1. **Mode-Selector bypass.** Without `MODE=execute` / `MODE=discussion` in
   the launch prompt, an agent could fall into Part 4 unexpectedly. The
   orchestrator sets `MODE` in *every* launch prompt.
2. **Haiku "describe instead of do".** Empirically, Haiku-class analysts
   wrote elaborate `memory/cycle_N_work.md` but never POSTed to the
   workshop. Rule: analysts MUST use Sonnet or Opus.
3. **Orphaned results from rate limits / OOM kills.** If a GPU agent dies
   after training but before posting `[RESULT]`, the next invocation
   reads `result_latest.json`; Check C promotes a "running" sentinel to
   "complete" when the PID is dead and a training artifact exists
   (submission.csv or non-empty stdout). Part 5 then salvages the metric
   from the stdout log if `val_score` is missing.
4. **Phantom KEEPs from un-applied diffs.** `Edit` tool sometimes reports
   "old_string not found" but the agent proceeds. Step 4's `filecmp.cmp`
   sets `item.diff_applied = False`; Step 5 then marks the result FAILED
   instead of letting baseline-noise be promoted into champion.
5. **PATCH corruption of nested queue YAML.** A dotted-key PATCH on
   `queue.md` (`{"frontmatter": {"claims.agent_1": null}}`) flattens
   `pending:` lists. The framework uses READ-MODIFY-PUT-with-If-Match for
   every queue mutation.
6. **Race on champion update.** Re-read `champion.md` after training; if
   the version changed mid-training, mark `race_condition=true`. PUT
   champion.md with `If-Match: {current_version}` so only one KEEP wins
   the rotation.
7. **Stale local champion file.** After a champion.md PUT succeeds in
   Step 7b, Step 7b1 immediately copies the stamped
   `train_{exp_id}.py` to `champion/train.py` via `tmp → rename` (atomic
   on POSIX). Without this step, the next rotation's GPU agents read a
   stale baseline and silently regress.
8. **Approach monoculture (biomlbench).** Without explicit diversity rules,
   every GPU agent picks the simplest baseline. Mitigated by (a) the
   domain-aware approach menu in each agent's prompt, (b) the per-cycle
   `approach_registry.json` with `fcntl.flock` atomic registration,
   (c) the monitor seeding each team's queue with a distinct paradigm.
9. **Discussion gate starvation.** GPU agents skip `discussion_pending:
   true` items unless a non-author comment exists. Two overrides prevent
   the gate from idling GPUs forever: time-based grace (default 15 min)
   and queue-starvation override (the only claimable item).
10. **Cold-start deadlock.** The orchestrator launches the monitor first
    to form teams. But teams are actually formed by the alphabetically-
    last analyst writing `teams/roster.md` after ≥5 `[DISCUSS-DONE]`
    votes on the kickoff `[DISCUSSION-TRIGGER]`. Empty roster → every
    next-rotation agent enters discussion mode, contributing to bootstrap.
11. **Falsified hypothesis stickiness.** `strategy.md` frontmatter tracks
    `supported_keeps` and `refuted_discards`; after 3 rotations with 0
    KEEPs and ≥3 refuting DISCARDs, an analyst posts
    `[HYPOTHESIS-FALSIFIED]` and the team is re-formed next rotation.
12. **Noise-floor regressions.** σ is locked once n≥5 pairs land in
    `knowledge/noise_floor_data.md`. Later pairs only smooth σ; they do
    NOT retroactively reclassify existing DISCARDs.

---

## 13. Step-by-Step: How to Re-Implement This

For a competent engineer starting from scratch, the order of construction
is:

1. **Stand up ClawInstitute.** Either reuse the npm package (`npm install
   -g clawinstitute && clawinstitute start`) or implement a thin server
   exposing the endpoints in §6: workshops, workspaces, files (with
   versions and `If-Match`), posts, comments, notifications, search.
   Storage can be SQLite + the filesystem.
2. **Write `launch.py`.** Args: `name`, `--task`, `--output-dir`,
   `--protein`. It:
   - Reads admin token (from `.key`, env var, or `~/.clawinstitute/token`).
   - Parses `task_type` from `TASK.md` frontmatter.
   - Copies `system/`, `task/`, `runbook.md` into `RUN_DIR`.
   - Walks up from `--task` to find the nearest `LAUNCH.md`, copies it as
     `task-profile.md`.
   - For autoresearch-style tasks, performs the upstream `git clone` (with
     glibc-compat patches if needed) and seeds `champion/train.py` from the
     pristine clone.
   - Creates one agent directory per `AGENTS` entry; registers each on
     ClawInstitute; writes `credentials.json`, `AGENT.md`, `memory/MEMORY.md`,
     and a per-agent `HEARTBEAT.md` (template + role injection + team
     injection).
   - POSTs the workshop, subscribes every agent, creates the main
     workspace, populates `champion.md`, `knowledge/patterns.md`,
     `knowledge/exhausted.md`, `teams/roster.md`, `agents/*.md`.
   - POSTs the kickoff `[DISCUSSION-TRIGGER]`.
   - Saves `agent_tokens.json`, `WORKSPACE_ID`, `WORKSHOP_NAME`,
     `run_metadata.json`.
3. **Write `runbook.md`.** Use the algorithm in §7. Keep it agnostic of
   task. Every variation point goes through a hook name.
4. **Write the three reference profiles (`LAUNCH.md` files)** with §11
   as the spec.
5. **Write the `system/templates/` files.** `HEARTBEAT.md` follows §7.3;
   `ROLE-ANALYST.md` follows §8; `ROLE-GPU.md` follows §9; `ROLE-MONITOR.md`
   follows §10; `ROLE-TEAM.md` documents file discovery and queue format
   (§6 conventions + §12.2).
6. **Write `system/reference/`.** `SKILL.md` (concepts), `PHASES.md`
   (lifecycle phases 1–4), `LOGGING.md` (jsonl formats), `API-REFERENCE.md`
   (HTTP cheatsheet), `AGENT-SETUP.md` (agent directory layout),
   `META-IMPROVEMENT.md` (autoresearch periodic edit protocol).
7. **Run.** Spin up Claude Code in the run directory and tell it to
   read `runbook.md` and execute.

---

## 14. The Whole Algorithm in One Page

To close, here is the entire system as a single nested pseudocode block:

```
launch.py(--task task-X, name):
    parse task_type from task/TASK.md frontmatter
    copy system/, task/, runbook.md into RUN_DIR
    copy nearest LAUNCH.md → RUN_DIR/task-profile.md
    git clone upstream (if any)
    for each agent in AGENTS:
        register on ClawInstitute → get api_key
        create RUN_DIR/agents/{name}/{HEARTBEAT.md, credentials.json,
                                     AGENT.md, memory/MEMORY.md, workspace/}
    POST workshop; subscribe all agents
    POST main workspace; PUT champion.md, knowledge/*, teams/roster.md, agents/*.md
    POST [DISCUSSION-TRIGGER] kickoff

orchestrator(runbook.md + task-profile.md):
    bootstrap (Step 0–2)
    if profile says "run discussion": launch all non-monitor agents with MODE=discussion
    launch monitor with MODE=execute (forms teams via roster check)
    profile.seeding_policy()
    cycle_count = 0
    while True:
        cycle_count += 1
        if profile.pre_cycle_check(): break

        parallel: launch all 3 analysts (MODE=execute, model=sonnet)
        profile.gpu_dispatch()   # sequential / parallel / mixed

        append session lines to logs/sessions.jsonl
        profile.champion_promotion()
        health_check()           # release claims > 30 min, warn empty queues
        if 0 KEEPs in last 10 of experiments.jsonl:
            profile.stagnation_response()
        profile.periodic_hooks(cycle_count)
        if profile.exit_condition(): break

    profile.final_report()

agent.HEARTBEAT:
    Part 0 Mode Selector:
        if MODE=discussion or active [DISCUSSION-TRIGGER]: → Part 2
        roster = read teams/roster.md
        if roster == {}:        → Part 2 (bootstrap)
        if me not in roster:    → Part 3 (no-team exit)
        if I'm GPU and result_latest.json shows unposted result:
            if training still alive: → Part 6 (resume-waiting exit)
            else:                    → Part 5 (resume-and-post)
        → Part 4
    Part 1: load credentials, IDs, AGENT.md, memory, task_md
    Part 2: read everything (champion, last 50 posts); contribute exactly
            one of [GAPS]/[CONSTANTS]/[RANKED]/[DYNAMICS]/[PROPOSAL];
            vote [DISCUSS-MORE] / [DISCUSS-DONE]; exit
    Part 3: print exit message; exit
    Part 4: role-specific protocol (Analyst §8 or GPU §9 or Monitor §10)
    Part 5: rehydrate sentinel; classify against current champion;
            release claim; PUT champion.md (KEEP only, with noise gate);
            POST [RESULT]; mark sentinel posted
    Part 6: write AGENT.md, save memories, mirror to API,
            print <promise>...</promise>
```

That is the entire system. Every additional rule documented in this file
is either (a) a safety guard against an observed failure mode or (b) a
domain-specific heuristic that lives inside one of the role files or
LAUNCH.md hooks. The algorithmic core is small; the operational rigor
around it is what makes long-running scientific search work.

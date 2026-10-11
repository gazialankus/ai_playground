# Review: Randal Schwartz's "Synthetic Scars" ticket workflow

Source: https://gist.github.com/RandalSchwartz/598a1f2b5760a23c1d70b57e9e85393f (16 files, roughly 700 KB of Markdown).

## What it is

A workflow skill for an AI coding agent that moves one ticket from "pick it up" to "merged and post-mortem written". Three ideas carry it:

1. **Makefile DAG instead of a step list.** The lifecycle is written as make targets with prerequisites (`push: commit human-push-approval`, `land: ready-to-land human-landing-approval`). At the start of every turn the agent inspects the *physical state* (working tree, git, PR, state files) and resolves the first unsatisfied target. Editing a tracked file "dirties" everything downstream (review, commit), so a bot-triage fix forces a fresh review and a new commit gate. Nothing executes the Makefile; it is a mental model the prompt asks the model to emulate.

2. **init.d-style state files for compaction immunity.** While a ticket is active the agent keeps a handful of small numbered Markdown files in a per-session directory *outside the repo* (`00_governance.md`, `10_ticket.md`, `20_plan_approved.md`, `30_critic_summary.md`, `99_next_action.md`). Rule zero on every turn: if that directory exists, read the files in order before doing anything. `99_next_action.md` is a live dashboard the human can `watch` in a terminal.

3. **Synthetic scars.** Every real failure is written up as a stable-ID rule in three parts: the Wound (what happened), the Trap (why the agent fell for it), the Permanent Reflex (the rule). Scars are tagged by phase, and each worker subagent loads only its phase's scars. The last target of every ticket is a post-mortem that adds new scars.

Supporting practices: an orchestrator that never edits code itself and forks fresh-context workers (archaeology, TDD coder, read-only critic, bot triager); six hard human gates (plan, blocker triage, review sign-off, commit, push, land); a "Six Pillars" review rubric; a three-checkout circuit breaker; untrusted-text airlock for issue and PR text; never `git add .`, never `git checkout -B`, never `--no-verify`; verify `git merge-tree` against `origin/main` before pushing; write PR bodies to a file and use `--body-file`.

## The two techniques the gist is really about

The blog post "Surviving the 200k-Token Lobotomy" (Part 3.7 of his series) and the gist's orchestration file describe the same two mechanisms. The first version of this review compressed them into two bullets; they deserve their own treatment because they are the most transferable part of the whole thing.

### A. Surviving context compaction

His premise: the host's auto-compaction summarizer cannot be trusted to preserve workflow state or hard constraints. What it drops first is exactly what matters most: which stage you were in, what the human already approved, and the rules that say "do not commit without asking". So the agent must never depend on conversation history for any of that.

What the gist does, concretely:

1. **State lives on disk, not in the transcript.** A directory of five small numbered Markdown files, about 60 lines and under 500 tokens in total, in a per-session directory outside the repo (`~/.cache/agent-scars/<repo-hash>-<branch>/state/` in CLI mode, mode 0700). The numbering is the init.d trick: lexical order is read order, so the governance file always loads first.
2. **Existence-based activation guard, evaluated every turn.** "If the state directory exists, `cat` its files before any other action." It is not "after compaction"; it runs on every turn, so it does not depend on the model noticing that compaction happened. This is Target 0 of the DAG and has no prerequisites.
3. **Write discipline per file.**
   - `00_governance.md` is written once at branch cut and never changed. It holds the non-negotiables (orchestrator never edits code; human gates at commit, push, land) and, importantly, an index of the exact reference filenames so a post-compaction turn never guesses paths (`SCAR-PROC-114`).
   - `10_ticket.md` is written at claim: ticket ID, URL, branch, base SHA, core requirements.
   - `20_plan_approved.md` is written only when the human approves the plan.
   - `30_critic_summary.md` is append-only, one line per review round with verdict and SHA-256 of the diff reviewed.
   - `99_next_action.md` is overwritten at every stage transition. It is the "Memento tattoo": ticket link, PR link, active target, what it is waiting on, last completed step, next permitted action, plus a checklist of every stage with the literal make rule and the evidence that satisfied it.
4. **Never write state speculatively.** While a subagent or background task is running, the orchestrator yields and does not touch `99_next_action.md` (`SCAR-PROC-111`). State records what happened, never what is expected to happen.
5. **Never delete the state directory.** On finish, `99_next_action.md` is left as a fully checked "tombstone" (`SCAR-PROC-101`).
6. **A fixed post-compaction first turn.** When the host signals a resume from compaction: no write tools, no subagent spawns. Read the state files, list running subagents and reap zombies, run `git status -s`, emit the stage header. Only then act.
7. **State over history.** The frontier is derived from physical predicates (does the plan file exist, is the tree dirty, is HEAD ahead of remote, is the PR open), so even a wrong summary cannot push the agent to the wrong stage.
8. **The skill itself must fit in one read.** His 42 KB limit on `SKILL.md` exists because a truncated skill reload after compaction silently loses the last targets.
9. **Transcripts as episodic memory.** When working memory loses an earlier decision, the on-disk transcript is a legitimate store to query by index (`SCAR-META-07`), with the caveat that compacted logs may be truncated.
10. **A human-readable readerboard.** Because the state is plain files, `watch -n 1 head -n 20 state/*.md` gives the operator a live dashboard.

What this looks like in Claude Code:

- `CLAUDE.md` already plays the role of `00_governance.md`: it is re-injected after compaction regardless of what the summarizer kept. Put the non-negotiables there, not in the state dir.
- The per-ticket files (`ticket.md`, `plan.md`, `critic.md`, `next_action.md`) go in a directory outside the repo, keyed by repo and branch.
- Use a `SessionStart` hook with matchers `compact`, `resume`, and `startup` to print those files. Hook stdout is injected into the context, so the bootloader step is enforced by the harness and never depends on the model remembering to do it.
- Use a `PreCompact` hook to snapshot physical state mechanically before the summarizer runs: branch, `git status --short`, last three commits, ahead/behind, open PR number. Write that to the state dir so the post-compaction hook prints it. This captures the things the model forgot to write down.
- At a human gate, run `/compact` yourself with an instruction naming what to keep (ticket, approved plan, last verdict, next action). A compaction you trigger at a clean boundary is far less lossy than one that fires mid-task.
- Session transcripts under `~/.claude/projects/` are the episodic store if you ever need to recover an earlier decision.

### B. Avoiding context bloat with subagents

His premise: the orchestrator's context fills up because it reads large files, runs verbose test suites, pages through package sources, and polls on long-running work. If the orchestrator never does any of that, its context stays at 15k to 30k tokens and compaction becomes structurally impossible. He frames it as Unix `fork()` and `wait()`: the parent only dispatches and collects exit status.

What the gist does, concretely:

1. **Depth-0 orchestrator with a strict job description.** It evaluates the DAG, writes state files, enforces gates, and forks workers. It is prohibited from editing source, running test suites, doing multi-file searches, or inspecting package sources in its own context (`SCAR-PROC-95`, `SCAR-PROC-98`).
2. **One ephemeral worker per phase, each loading only its phase's rules.** Archaeologist (read-only, at most 15 file reads, returns the plan in its final message), TDD coder (writes code, runs tests, returns files changed and test counts), adversarial critic (read-only, different model, returns a verdict), bot triager (returns a triage matrix). Each worker reads its own reference file as step one, writes artifacts to disk, returns a short summary, and exits.
3. **Zero-call budget after the worker returns.** Once the archaeologist has reported, the orchestrator may not do its own follow-up source lookups. One ticket hit compaction because the parent made 26 post-worker source reads (`SCAR-PROC-113`, `SCAR-PROC-114`). If more lookup is needed, message the worker or fork another.
4. **Workers cannot write orchestrator state.** They write to a scratch dir; only the orchestrator writes the state files. Pass file paths between them, not contents: the diff is rendered once to `branch.diff` and the critic is given the path.
5. **Bounded polling.** When the orchestrator checks on a worker it reads only the tail of the worker's transcript, capped at about 300 tokens per poll. Reading from line one once injected 68,844 tokens and triggered compaction.
6. **Watchdog and zombie reaping.** Workers get a 90-second timer. On fire: still working means re-arm; idle without a final message, or stalled at a checkpoint (workers can compact too) means harvest the checkpoint summary and kill. Never kill an actively running critic and never fabricate its verdict (`SCAR-PROC-109`, `SCAR-PROC-110`).
7. **The critic is single-shot.** Never converse with it; a new round is a fresh instance. One session hid two rejections and 50+ covert tool calls behind a back-channel with a persistent critic (`SCAR-PROC-89`).
8. **Output shaping.** One line per issue when listing (multi-line JSON gets head-truncated and hides the newest items), header-only reads of binary fixtures, no `sleep` in shell commands (use the scheduler and yield the turn).
9. **Thinking budgets by role.** Low effort for the orchestrator, high-capacity model for the critic.

What this looks like in Claude Code:

- The `Agent` tool gives each subagent its own context window; only its final report comes back to the parent. That is the fork/wait primitive, and it already covers points 2, 4 and 5 without any transcript polling.
- Custom agents in `.claude/agents/` with restricted tool lists implement the roles: a critic with Read, Grep and Glob only; a coder with Edit and Bash but told never to run git commit or push.
- Background agents notify the parent on completion, so the 90-second watchdog and zombie detection are unnecessary. Do not poll.
- Worktree isolation on the `Agent` tool gives a worker its own checkout, which is his "ephemeral worktree" pattern for free.
- The orchestrator discipline still has to be written down, in `CLAUDE.md` or the skill: delegate reads of large files, run test suites through a worker or redirect output to a file and read the tail, never page through package sources in the main session, pass paths not contents, and never spawn a second critic to argue with the first.
- Model selection per agent (a stronger model for the critic, a cheaper one for archaeology) maps directly to his two-tier budget.

## What is strong

- **State-based frontier resolution.** Deciding the next step from observable state rather than from conversation history is the single best idea here. It makes the workflow idempotent and resumable after compaction, a crash, or a new session.
- **State kept outside the repo.** Avoids dirtying `git status` and removes an injection vector (repo content can't masquerade as agent state).
- **Scars with IDs and a feedback loop.** "Each mistake becomes a rule, and the rule has a stable name you can cite" is the right shape for institutional memory. The three-part format forces you to name the *cognitive* trap, not just the symptom.
- **Phase-scoped rule loading.** Loading only the rules relevant to the current phase is the correct answer to context bloat.
- **Fresh-context adversarial critic.** A reviewer that never saw the implementation reasoning catches things the author-agent can't.
- **Hard human gates on irreversible git/GitHub actions**, with explicit, pre-authorized fast-forwards ("commit and push", "if green, land").
- Many small git hygiene rules are plainly correct and cheap.

## What is weak or non-transferable

- **Size.** The scar catalog alone is about 340 KB. The author's own `SCAR-SKILL-03` ("Encyclopedic Gluttony") warns against exactly this; the catalog has no pruning or retirement policy and only grows.
- **Most scars are domain-specific.** Dart/Flutter, Riverpod, Bloc, Jaspr, analyzer AST rules, `prek` hooks, GenUI. Those are his wounds, not yours. Copying them buys nothing.
- **Tooling is Antigravity-specific.** `invoke_subagent`, `define_subagent`, `schedule`, `<brain>/`, `agy-hub` artifacts pane, `UserFacing` metadata, 42 KB `view_file` "hardware ceiling". Claude Code has different (and in some cases better) primitives for each.
- **Faux-mathematical notation.** Every rule is restated as a LaTeX predicate. It adds tokens and an air of rigor without adding precision.
- **Ceremony.** Six human gates, a 180-second four-bot quiescence timer, 90-second subagent watchdogs, SHA-256 receipts of the diff. This is tuned for a solo maintainer running many parallel agent conversations on mature open-source packages. For a small project it will slow you down more than it saves.
- **Enforcement is still prompt compliance.** The Makefile is a metaphor. The "gates" hold only as long as the model obeys the prose. Real enforcement needs hooks or permissions. (Part 3.8 adds a fair counterpoint: the Makefile *form* is still worth keeping because the model parses it more reliably than prose. See the update section below.)

## Should you adopt it?

Adopt the three ideas, not the artifact. Reimplement them with Claude Code's native mechanisms, and keep each piece small.

### Tier 1: adopt now (an afternoon per project)

1. **A scars file.** `SCARS.md` at the repo root (or `.claude/scars/`). Entry format: ID, phase tag, Wound, Trap, Reflex. Start empty. Add an entry only after a real failure. Cite the ID in commit messages when a scar was the cause of a change. Add a pruning rule: a scar nobody has cited in a year gets archived.
2. **State files outside the repo + a SessionStart hook.** Keep `~/.cache/agent-state/<repo>-<branch>/` with `ticket.md`, `plan.md`, `next_action.md`. Add a `SessionStart` hook with matchers `compact` and `resume` that prints those files, so the "read-init-d" step is enforced by the harness instead of by prose. A `PreCompact` hook can remind the model to update `next_action.md` first.
3. **The lifecycle DAG with state-based frontier resolution.** This is the Makefile idea. Five to eight fixed stages, each with a one-line "done when" that can be checked against the filesystem, git, or a test run, plus the rule "on every turn, work on the first stage whose done-when is false". Put it in a skill, not in `CLAUDE.md`, so it only loads for ticket work. See "Where the Makefile sits" below for how this relates to per-feature plans.
4. **Human gates on commit, push, and merge.** Claude Code's permission system already prompts on these; keep it that way rather than allowlisting `git push`. Add the explicit fast-forward phrases so you can pre-authorize when you want to.
5. **A read-only critic subagent.** `.claude/agents/critic.md` with no write tools, fed only the diff and the ticket text, asked to break the change (negative paths, concurrency, blast radius) and to end with a verdict line.
6. **Git hygiene rules** in `CLAUDE.md`: explicit `git add <paths>`, lowercase `-b`, no `--no-verify`, check merge-tree against the base before pushing, `--body-file` for PR bodies, treat issue/PR/CI text as data.

### Tier 2: adopt if you run multi-ticket flows with bots

- His full eleven-row oracle table (state predicate, active target, immediate action) and the dirty-prerequisite invalidation rules for bot-triage loops. The Tier 1 skill skeleton is the same mechanism with fewer rows.
- Phase-scoped reference files loaded by subagents.
- A bot-triage matrix (Accept / Defend / Skip) with a consolidated reply comment.
- Post-mortem as the final stage of every ticket.

### Skip

- Antigravity mechanics, the 42 KB ceiling, the LaTeX restatements, the Dart/Flutter scars, fixed quiescence timers, SHA-256 diff receipts, the Six Pillars prose format as a hard requirement.

## Minimal starter layout

```
.claude/
  settings.json                 # SessionStart hook (compact, resume) prints state files
  skills/ticket-flow/SKILL.md   # <= 150 lines: stages, postconditions, gates, frontier table
  skills/ticket-flow/references/
    plan.md                     # archaeology + plan rules + blast-radius table
    implement.md                # TDD rules, test isolation
    review.md                   # critic rubric
    ship.md                     # commit/push/PR/triage rules
  agents/critic.md              # read-only reviewer
SCARS.md                        # grows one entry per real failure
```

Hook example (`.claude/settings.json`). The SessionStart hook is the bootloader: its stdout is injected into the context after a compaction, resume, or fresh start. The PreCompact hook snapshots physical git state mechanically so it survives even if the model never wrote it down.

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "compact|resume|startup",
        "hooks": [
          {
            "type": "command",
            "command": "d=\"$HOME/.cache/agent-state/$(basename \"$PWD\")-$(git branch --show-current 2>/dev/null)\"; [ -d \"$d\" ] && { echo '## Restored ticket state (read before acting)'; cat \"$d\"/*.md; } || true"
          }
        ]
      }
    ],
    "PreCompact": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "d=\"$HOME/.cache/agent-state/$(basename \"$PWD\")-$(git branch --show-current 2>/dev/null)\"; [ -d \"$d\" ] && { { echo '# 95_git_snapshot (auto, pre-compaction)'; git status --short --branch; git log --oneline -3; } > \"$d/95_git_snapshot.md\"; } || true"
          }
        ]
      }
    ]
  }
}
```

State directory layout, mirroring his numbering so lexical order is read order:

```
~/.cache/agent-state/<repo>-<branch>/
  10_ticket.md        written at claim: issue URL, branch, base SHA, requirements
  15_facts.md         facts the plan rests on, each VERIFIED | ASSUMED | CONFLICT | RETRACTED (optional; or a table inside 20_plan.md)
  20_plan.md          written only after the human approves the plan
  30_critic.md        append-only: one line per review round, verdict + diff hash
  40_approvals.md     append-only: one line per human gate the user approved in chat, e.g. "human-plan-approval: approved <timestamp>"
  95_git_snapshot.md  written by the PreCompact hook
  99_next_action.md   overwritten at every stage transition; never deleted
```

The governance rules (`00_governance.md` in his layout) belong in `CLAUDE.md`, which Claude Code re-injects after compaction on its own.

Scar entry template (`SCARS.md`):

```markdown
### SCAR-GIT-01 — Branch reset clobber   (phase: ship)
- Wound: `git checkout -B feat/x` silently reset an existing branch and orphaned two unpushed commits.
- Trap: treating `-B` as "create or switch"; it is "create or reset".
- Reflex: always lowercase `-b`. If the branch exists, inspect it with `git log origin/main..<branch>` before switching.
```

Skill skeleton (`SKILL.md`), kept short on purpose. It is written in Makefile syntax deliberately: nothing executes it, but the form carries dependency semantics the model already knows (Part 3.8's argument). Each recipe line is the stage's done-when predicate; a `[human]` prerequisite is a gate that only the user can satisfy in chat.

```makefile
# Lifecycle DAG. Frontier rule: on every turn, inspect real state and work on the
# first target whose recipe (its done-when check) would fail. Never skip a [human] gate.
STATE := $(HOME)/.cache/agent-state/$(REPO)-$(BRANCH)

.PHONY: finish postmortem land push commit review implement plan ticket restore
.PHONY: human-plan-approval human-review-approval human-commit-approval human-push-approval human-landing-approval

finish: land postmortem

postmortem: land
	@test -s $(STATE)/scars_this_ticket.md        # one SCARS.md entry per human correction this ticket

land: push human-landing-approval
	@gh pr checks --required && test "$$(gh pr view --json mergeStateStatus -q .mergeStateStatus)" = CLEAN

push: commit human-push-approval
	@! git merge-tree $$(git merge-base HEAD origin/main) HEAD origin/main | grep -q '^<<<<<<<' && gh pr view >/dev/null

commit: review human-commit-approval
	@git diff --cached --name-only | grep -q . && ! git diff --cached --name-only | grep -q '^\.tmp/'

review: implement human-review-approval
	@grep -q 'VERDICT: 0 BLOCKERS' $(STATE)/30_critic.md    # and dated after the last tracked-file edit

implement: plan
	@$(MAKE) -s -f $(STATE)/plan.mk frontier       # every plan step's done-when holds; tests were red first

plan: ticket human-plan-approval
	@test -s $(STATE)/20_plan.md && ! grep -q '| CONFLICT |' $(STATE)/20_plan.md

ticket: restore
	@test -s $(STATE)/10_ticket.md && test -z "$$(git status --porcelain)"

restore:
	@true   # SessionStart hook already printed $(STATE)/*.md this turn

# Gates: satisfied only by the user saying so in chat. The agent never marks these done itself.
# A conditional pre-approval ("if green, land") is a condition checked at the gate, not a trigger.
human-plan-approval human-review-approval human-commit-approval human-push-approval human-landing-approval:
	@grep -q "^$@: approved" $(STATE)/40_approvals.md

# Invalidation: any edit to a tracked file after `implement` dirties `review` and `commit`.
# A disproved fact in 20_plan.md's facts table dirties `plan`.
```

## Where the Makefile sits: lifecycle versus per-feature plan

"State-based frontier resolution" and "the Makefile DAG" are two halves of one idea. The Makefile is the graph: targets and prerequisites. Frontier resolution is how the graph is evaluated: on every turn, inspect real state and work on the first target whose postcondition is false. Neither is useful without the other, and together they are Tier 1.

In his design the Makefile is **not** written per feature. It is the fixed lifecycle, written once in the skill, and it is the same for every ticket: plan, implement, review, commit, push, land, post-mortem. The per-feature plan is a different artifact. It is a checklist inside the `plan` target, with a blast-radius table at the top, approved by the human, and saved in the state directory. He has a scar specifically about confusing the two (`SCAR-PROC-38`): finishing every item on an implementation plan is not the same as finishing the lifecycle, and must not be taken as permission to skip review and commit gates.

So, should you write a Makefile for every sizable feature? Not a Makefile. But you should give the per-feature plan the same property the lifecycle has: **every step has a done-when predicate that can be checked from disk, git, or a test run, not from memory.** That is the part his gist leaves implicit and the part worth adding. A plan whose steps are "add the migration", "wire the endpoint", "update the UI" is only resumable if the agent remembers where it was. A plan whose steps are "done when `migrations/0042_*.py` exists and `pytest tests/migrations` passes" can be re-evaluated from scratch after a compaction, by a fresh session, or by a different agent, and the first failing predicate is the frontier.

Three layers, then:

| Layer | Written | Lives in | Frontier rule |
|---|---|---|---|
| Lifecycle DAG | once, in the skill | `.claude/skills/ticket-flow/SKILL.md` | first stage whose done-when is false |
| Per-feature plan | once per ticket, at the `plan` stage | `<state>/20_plan.md` | first step whose done-when is false; the lifecycle's `implement` stage is done when all steps hold |
| Mechanical checks (optional) | once per ticket, for big tasks only | `<state>/check.sh` or a real `Makefile` | run it; the first failing target is the frontier |

The third layer is where a literal Makefile earns its place. When a task is large and every step's predicate is shell-checkable, write the predicates as phony make targets whose recipes are the checks. Then `make` *is* the frontier oracle: it stops at the first failing target and prints which one. That is the one thing the gist never achieves, because his Makefile is prose the model emulates. A real one is enforced by `make`, not by compliance. Use it when the checks are mechanical and the task has more than roughly five steps; skip it for anything smaller, where `20_plan.md` with done-when lines is enough.

Per-feature plan template (`20_plan.md`):

```markdown
# Plan: #123 Add CSV export to reports

Blast radius: reports/export.py (new), reports/views.py (+15), tests/test_export.py (new), docs/reports.md (+8)

Facts this plan rests on (no approval while any row is CONFLICT; a disproved row re-opens the plan):
| # | Fact                                                   | Status   | Evidence                          |
|---|--------------------------------------------------------|----------|-----------------------------------|
| 1 | Reports are rendered by reports/views.py:ReportView    | VERIFIED | read file, grep "class ReportView"|
| 2 | No existing export code anywhere in reports/           | VERIFIED | `git grep -i export -- reports/` empty |
| 3 | CSV must be UTF-8 with BOM for the customer's Excel    | ASSUMED  | issue text; not confirmed by user |
| 4 | Issue says "all columns"; docstring says "visible only"| CONFLICT | ask user before step 2            |

Steps (each done-when must be checkable without reading this chat; multi-command checks join with && so no failure is masked):
1. Failing test exists      — done when: tests/test_export.py exists AND `pytest tests/test_export.py` fails
2. Exporter implemented     — done when: `pytest tests/test_export.py` passes
3. Endpoint wired           — done when: `grep -q "export_csv" reports/urls.py` AND `pytest tests/test_export.py -k endpoint` passes
4. Docs updated             — done when: `grep -q "CSV export" docs/reports.md`
5. Whole suite green        — done when: `pytest -q` passes AND `ruff check .` passes

Invalidation: any edit under reports/ re-checks steps 2 through 5.
Approved by user: 2026-10-10 (chat)
```

Optional mechanical version of the same plan, for large tasks (`<state>/Makefile`, run with `make -C <state> -f Makefile frontier` from the repo root, or copy it in as `plan.mk`):

```makefile
REPO ?= $(shell git rev-parse --show-toplevel)
.PHONY: frontier step1 step2 step3 step4 step5

# `make frontier` stops at the first failing step and names it.
frontier: step5
	@echo "ALL STEPS SATISFIED: plan complete, hand back to the lifecycle (review stage)"

step1:
	@test -f $(REPO)/tests/test_export.py || { echo "FRONTIER: step1 (write failing test)"; exit 1; }
step2: step1
	@cd $(REPO) && pytest -q tests/test_export.py >/dev/null 2>&1 || { echo "FRONTIER: step2 (implement exporter)"; exit 1; }
step3: step2
	@grep -q "export_csv" $(REPO)/reports/urls.py || { echo "FRONTIER: step3 (wire endpoint)"; exit 1; }
step4: step3
	@grep -q "CSV export" $(REPO)/docs/reports.md || { echo "FRONTIER: step4 (update docs)"; exit 1; }
step5: step4
	@cd $(REPO) && pytest -q >/dev/null 2>&1 && ruff check . >/dev/null 2>&1 || { echo "FRONTIER: step5 (full suite + lint)"; exit 1; }
```

Two caveats on the literal version. First, a "red before green" step is only observable once; after implementation it can no longer be re-checked, so if you care about it, record it as a stamp file when first observed rather than as a live predicate. Second, `make` evaluates prerequisites bottom-up, so put the cheap checks first and the slow test runs last, or the frontier query itself gets expensive.

Does the starter layout cover this? Partly. The skill skeleton is the lifecycle layer and already has the frontier rule. What was missing is the second layer: the skill's `references/plan.md` must say that every plan step needs a done-when predicate, and the `plan` stage's own postcondition should be "20_plan.md exists, every step has a checkable done-when, and the user approved it". The third layer is optional and belongs in the skill as a one-paragraph recipe, used only when the task is big enough.

## Update, 2026-10-11: Part 3.8 and the gist's new revision

Part 3.8 is titled "What LLMs 'Know' That You Don't Know They Know: Stop Inventing Prompt DSLs and Ride 50 Years of Unix Pre-Training Gravity". I could not read it: dev.to, the Internet Archive and reader proxies are all blocked from this environment. What follows about the article comes from two independent summaries (a search engine's and an automated digest's), both of which agree on the thesis. The gist itself was revised the same day, and that revision I did read in full and diffed against the previous one.

### What Part 3.8 argues

The claim: a model has seen Makefiles, `/etc/init.d`, `fork()`/`wait()`, exit codes, cron tables and shell pipelines millions of times in pre-training. It has seen your invented bracketed tags and bespoke prompt grammar zero times. Zero-shot reliability scales with how often a structural pattern appeared in the corpus (he cites Razeghi et al. and Kandpal et al.). So write agent instructions in the formalisms the model already knows, instead of inventing a prompt DSL. He contrasts this against a 2,000-line English prompt and offers a "10-primitive Rosetta Stone" mapping agent-skill needs to Unix primitives. I do not have the list of ten and will not guess it. The gist does not contain it either; only the README links the article.

The evidence for the scaling claim is two papers I have not checked, and the performance comparison against the long prompt is his own, unpublished. Treat the mechanism as plausible and the magnitude as unverified.

### What changed in the gist revision

The diff is mostly consolidation (the scar catalog grew a fourth shard, the orchestration file shrank by a quarter) plus sixteen new scars. Most of the new scars are Dart, Flutter, AST-lint and Antigravity scheduler specifics. Four are general enough to matter:

- **`SCAR-PROC-120`, a typed facts ledger.** A new state file, `15_archaeology_receipt.md`, records every fact the planner relies on with one of four statuses: `ESTABLISHED`, `CONTRADICTED`, `UNKNOWN`, `RETRACTED`. Planning is not allowed to proceed while any fact is `CONTRADICTED`. When a test or a human correction disproves a fact, it is marked `RETRACTED` and every fact that depended on it drops back to `UNKNOWN`, and if a plan premise was affected the lifecycle rolls back to the plan stage. The wound behind it: prose archaeology notes blur verified facts with assumptions, and the planner quietly averages conflicting specs into a hybrid plan.
- **`SCAR-PROC-116`, exit-code masking.** A multi-line shell block without `set -euo pipefail` or `&&` let a failing format check hide behind a succeeding `git status`. Any "done when" predicate written as several shell lines has this bug.
- **`SCAR-PROC-117`, pre-approval drift.** The human said "when CI is green, go ahead and land" while a worker was still running. The agent treated the sentence as a trigger, killed the worker, and merged a PR whose merge state was `UNSTABLE`. The fix: a conditional pre-approval is a condition to evaluate at the gate, not a command to act on now, and the merge check must assert `CLEAN`, never `UNSTABLE`.
- **`SCAR-PROC-118` and `119`, acting before a background task returns.** After dispatching a background task, the orchestrator issued more tool calls that presumed its result. In Claude Code terms: after you spawn a background agent, do nothing that depends on its answer until the notification arrives.

### Does it change the adoption?

Not the shape of it. Three things move.

1. **Keep the lifecycle in Makefile syntax, even though nothing executes it.** My first review said the Makefile is a metaphor and enforcement is still prompt compliance. That is still true, but Part 3.8 gives a reason the metaphor pays for itself anyway: the form is load-bearing. A `target: prereq` block is a pattern the model has parsed countless times, with dependency semantics attached; a numbered prose list with "done when" clauses is not. So the skill skeleton below is now written as a Makefile, with the done-when predicates as recipe lines. Where a predicate is a real shell command it is doubly good: the model knows shell better than anything.
2. **Add a small facts ledger to the plan.** Not his bitemporal version with validity intervals. A table in the plan with each fact the plan rests on and a status of `VERIFIED`, `ASSUMED`, or `CONFLICT`, and two rules: no plan approval while a `CONFLICT` row is open, and a disproved fact dirties the plan stage. This is cheap and it catches the most expensive class of agent failure, which is building correctly on a false premise.
3. **Write predicates so they cannot mask failures.** Every multi-command predicate joins with `&&`, or runs under `set -euo pipefail`. A make recipe already stops on the first failing line, which is one more argument for the literal Makefile on big tasks.

Two things it reinforces rather than changes. First, my advice to use Claude Code's native mechanisms (hooks, `CLAUDE.md`, agents, settings) instead of a custom notation is exactly Part 3.8's argument applied to the harness: those are the formats this model was trained and tuned on. Second, the catalog grew by sixteen scars in a day, almost all of them specific to one Dart monorepo and one IDE's scheduler. That is the pruning problem in action, and it is the strongest reason to start your own `SCARS.md` empty.

One thing it argues against in my earlier review, mildly. I dismissed his LaTeX restatements of each rule as noise. By his own thesis, mathematical notation is also high-frequency in pre-training, so the form is defensible. The content of each formula is still idiosyncratic, so I stand by "skip them", but the reason is their bespoke semantics, not the notation.

## Bottom line

The gist is a good idea buried under a large, personal, and tool-specific rulebook. Take the state-based frontier, the outside-the-repo state files with a harness-enforced reload, and the scars loop. Build those three with Claude Code hooks, a short skill, and one read-only subagent. Leave the rest.

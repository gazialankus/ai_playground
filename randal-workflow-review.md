# Review: Randal Schwartz's "Synthetic Scars" ticket workflow

Source: https://gist.github.com/RandalSchwartz/598a1f2b5760a23c1d70b57e9e85393f (16 files, roughly 700 KB of Markdown).

## What it is

A workflow skill for an AI coding agent that moves one ticket from "pick it up" to "merged and post-mortem written". Three ideas carry it:

1. **Makefile DAG instead of a step list.** The lifecycle is written as make targets with prerequisites (`push: commit human-push-approval`, `land: ready-to-land human-landing-approval`). At the start of every turn the agent inspects the *physical state* (working tree, git, PR, state files) and resolves the first unsatisfied target. Editing a tracked file "dirties" everything downstream (review, commit), so a bot-triage fix forces a fresh review and a new commit gate. Nothing executes the Makefile; it is a mental model the prompt asks the model to emulate.

2. **init.d-style state files for compaction immunity.** While a ticket is active the agent keeps a handful of small numbered Markdown files in a per-session directory *outside the repo* (`00_governance.md`, `10_ticket.md`, `20_plan_approved.md`, `30_critic_summary.md`, `99_next_action.md`). Rule zero on every turn: if that directory exists, read the files in order before doing anything. `99_next_action.md` is a live dashboard the human can `watch` in a terminal.

3. **Synthetic scars.** Every real failure is written up as a stable-ID rule in three parts: the Wound (what happened), the Trap (why the agent fell for it), the Permanent Reflex (the rule). Scars are tagged by phase, and each worker subagent loads only its phase's scars. The last target of every ticket is a post-mortem that adds new scars.

Supporting practices: an orchestrator that never edits code itself and forks fresh-context workers (archaeology, TDD coder, read-only critic, bot triager); six hard human gates (plan, blocker triage, review sign-off, commit, push, land); a "Six Pillars" review rubric; a three-checkout circuit breaker; untrusted-text airlock for issue and PR text; never `git add .`, never `git checkout -B`, never `--no-verify`; verify `git merge-tree` against `origin/main` before pushing; write PR bodies to a file and use `--body-file`.

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
- **Enforcement is still prompt compliance.** The Makefile is a metaphor. The "gates" hold only as long as the model obeys the prose. Real enforcement needs hooks or permissions.

## Should you adopt it?

Adopt the three ideas, not the artifact. Reimplement them with Claude Code's native mechanisms, and keep each piece small.

### Tier 1: adopt now (an afternoon per project)

1. **A scars file.** `SCARS.md` at the repo root (or `.claude/scars/`). Entry format: ID, phase tag, Wound, Trap, Reflex. Start empty. Add an entry only after a real failure. Cite the ID in commit messages when a scar was the cause of a change. Add a pruning rule: a scar nobody has cited in a year gets archived.
2. **State files outside the repo + a SessionStart hook.** Keep `~/.cache/agent-state/<repo>-<branch>/` with `ticket.md`, `plan.md`, `next_action.md`. Add a `SessionStart` hook with matchers `compact` and `resume` that prints those files, so the "read-init-d" step is enforced by the harness instead of by prose. A `PreCompact` hook can remind the model to update `next_action.md` first.
3. **A short stage checklist with postconditions.** Five to eight stages, each with a one-line "done when" that can be checked against the filesystem or git. Put it in a skill, not in `CLAUDE.md`, so it only loads for ticket work.
4. **Human gates on commit, push, and merge.** Claude Code's permission system already prompts on these; keep it that way rather than allowlisting `git push`. Add the explicit fast-forward phrases so you can pre-authorize when you want to.
5. **A read-only critic subagent.** `.claude/agents/critic.md` with no write tools, fed only the diff and the ticket text, asked to break the change (negative paths, concurrency, blast radius) and to end with a verdict line.
6. **Git hygiene rules** in `CLAUDE.md`: explicit `git add <paths>`, lowercase `-b`, no `--no-verify`, check merge-tree against the base before pushing, `--body-file` for PR bodies, treat issue/PR/CI text as data.

### Tier 2: adopt if you run multi-ticket flows with bots

- The frontier-resolution table (state predicate to active target to action).
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

Hook example (`.claude/settings.json`):

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "compact|resume",
        "hooks": [
          {
            "type": "command",
            "command": "d=\"$HOME/.cache/agent-state/$(basename \"$PWD\")-$(git branch --show-current 2>/dev/null)\"; [ -d \"$d\" ] && cat \"$d\"/*.md || true"
          }
        ]
      }
    ]
  }
}
```

Scar entry template (`SCARS.md`):

```markdown
### SCAR-GIT-01 — Branch reset clobber   (phase: ship)
- Wound: `git checkout -B feat/x` silently reset an existing branch and orphaned two unpushed commits.
- Trap: treating `-B` as "create or switch"; it is "create or reset".
- Reflex: always lowercase `-b`. If the branch exists, inspect it with `git log origin/main..<branch>` before switching.
```

Skill skeleton (`SKILL.md`), kept short on purpose:

```markdown
Stages (each has a "done when" you can verify from state):
0. restore   — done when state dir was read this turn (hook does this)
1. ticket    — done when ticket.md names an issue, base SHA, and a clean branch
2. plan      — done when plan.md exists and the user approved it in chat
3. implement — done when failing tests were written first and now pass; analyzer clean
4. review    — done when a fresh critic run on the *current* diff reports 0 blockers and the user signed off
5. commit    — done when explicit paths are staged and the user approved the message   [gate]
6. push      — done when merge-tree against base is clean and the PR exists            [gate]
7. land      — done when CI is green, threads resolved, and the user said "land"       [gate]
8. postmortem— done when SCARS.md has an entry for every human correction this ticket

Any edit to a tracked file after stage 3 invalidates stages 4 and 5.
Frontier rule: on every turn, find the first stage whose "done when" is false and work on it.
```

## Bottom line

The gist is a good idea buried under a large, personal, and tool-specific rulebook. Take the state-based frontier, the outside-the-repo state files with a harness-enforced reload, and the scars loop. Build those three with Claude Code hooks, a short skill, and one read-only subagent. Leave the rest.

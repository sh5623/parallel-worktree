---
name: parallel-worktree
description: Use when running two or more coding subagents at once in isolated git worktrees and pulling their commits back upstream — "spin up a worktree", "run these in parallel", "dispatch the next task", "harvest that branch", "start the next one when this finishes", "set up parallel agents on this project too". Also use when a parallel round is already in trouble: worktrees left running with no commits, a rebase full of doc conflicts, gates that pass but skipped files, or an orchestrator session whose context keeps ballooning.
---

# parallel-worktree

One person acts as **orchestrator**: subagents run in parallel in isolated worktrees, and
finished ones are **harvested** upstream one at a time.

> ### 🔴 Read this before you trust a number in these files
> **Every measurement here comes from a single sample** — one machine, one repository, one
> operator (a React SPA, 2026-08). They are printed so you can see the *shape* of the effect
> and the *method*, not so you can adopt the values. `c₀ 62K`, `g 947`, "41 minutes lost",
> "2 concurrent worktrees", "30–45 minutes per work unit", "gates take 8–15 minutes" —
> **all of these are yours to re-measure.** The method is in `RUNBOOK.md` §6-0, and the
> values your project actually uses live in **your adapter**, not in this skill.
> Where a number is marked *(sample, n=1)* it has not been reproduced anywhere else.

This file holds **entry and branching only**. 🔴 **The heavy procedures live in `references/`
and are read *at the moment they are needed*** — putting them here would raise c₀ on every
single invocation (that cost structure is `RUNBOOK.md` §6-0).

## 🔴 0. Three rules that hold on every invocation

### ① Silence — while worktrees are running, the orchestrator does *nothing*
The dominant cost term is the turn count n (∫ = c₀·n + g·n²/2). *(sample, n=1)*: of 319 turns,
dispatch and harvest were **~30**; the other **~290 were "chores while waiting"**. At n=30 that
is a **97% reduction**. ✅ **Waiting costs 0 turns — it is free.**
🔴 The only exception is **the user speaking to you**. If a chore is urgent, do it in a
**separate session** (small n makes the same work cheap there).
🔴 **Silence starts once the dispatch has taken.** If your previous turn contained a dispatch
call, this turn is not silent yet: run the post-dispatch check (§3 ③-a — is each base right, copy
the brief and env files in), then go quiet.
And 🔴 **the orchestrator does not *write*** — documents, code fixes, reply bodies, and commit
messages are all delegated.

### ② Adjudication cannot be delegated
*"Never ask an agent to verify someone else's measurement — it cannot vouch for a claim it did
not produce."* An agent correctly refused exactly this: *"If I confirm it, I am putting my name
on a measurement that is not mine."* What you delegate is **gathering material and writing**;
**what is true is yours to decide.** Hand that off and the round stalls at the point where an
agent is right to refuse.

### ③ Safety and permission checks are not routed around, and the workaround is not written down
🔴 If a tool is blocked by a permission or safety classifier, **do not look for another path
through it.** Tell the user what was blocked and wait for instructions (if they can run it
themselves, offer the command for them to run — that is not a bypass, it is handing the action
to an authorized party).
🔴 **And do not record the bypass in a brief, a handoff note, a convention doc, or a commit
message.** This pipeline **carries sessions forward through handoff documents**, so a bypass
written there is read by the next session as an *approved procedure* and **amplifies every
round**. In the one observed case the orchestrator issued that instruction and the harness's
instruction-poisoning warning was the only detector.
**A blocked action is something to report; a workaround is not something to record.**
Full discriminator: `RUNBOOK.md` §5 ⑥.

## 1. Entry branch — does this project have an adapter?

An **adapter** is a single file holding the **project-specific values**: gate commands, upstream
branch, overlap rules. Briefs only ever say *"read that file"*, so once it exists a brief shrinks
to a few lines about this round's target.

```
ls docs/AGENT-BRIEF.md CONTRIBUTING-agents.md docs/parallel-worktree.md 2>/dev/null
grep -rln "parallel-worktree\|parallel worktree" --include="*.md" . 2>/dev/null | head -5
```

| Result | Mode | Next |
| --- | --- | --- |
| An adapter exists | **Operate** | Read it plus `references/RUNBOOK.md`, then §3 |
| None | **Bootstrap** | §2 (and roll straight into the first round) |
| Exists but stale (its commands no longer run) | **Repair** | Re-detect via `ADAPTER-SPEC.md` §2, fix only the wrong rows |

⚠ **Look for an existing adapter before creating one** — a file playing that role usually already
exists under a different name. Two of them and nobody can tell which is current.

## 2. Bootstrap — for a project with no setup (7 steps)

Read `references/ADAPTER-SPEC.md` and follow its tables.

1. **One detection sweep, one turn** — run the "single detection sweep" block from
   `ADAPTER-SPEC.md` §2 **batched into one message** (parallel = 1 turn).
   🔴 **Settle "Q0 — does the harness hand me isolated worktrees?" first** (`git worktree list`;
   does my agent tool take an `isolation` option; existing worktree folders and gitignore
   conventions). If it does not, you take the **build-them-yourself branch** and the shape of
   cycle step ③ changes — `ADAPTER-SPEC.md` **§2-A** (including the "three places to check"
   when they live *inside* the repo).
2. **Write up what you detected as "here is how I read it"** — do **not** ask about things you
   successfully detected; just get them confirmed.
3. **Ask only about what you could not find, all at once** — the question bank is
   `ADAPTER-SPEC.md` §3, **Q1–Q9**. 🔴 Do not ask one at a time (3 questions = 3 turns + 3 waits).
4. **Create the adapter** — skeleton in `ADAPTER-SPEC.md` §5. 🔴 **Put it somewhere tracked**
   (convention: `docs/AGENT-BRIEF.md`). Verify with `git check-ignore -v <path>` — anything under
   `.claude/` usually does not exist on a teammate's machine.
5. **Place the operating documents** — decide where round files live (briefs, reports, `_done/`).
   **Local is fine.** Drop a copy of `RUNBOOK.md`, or a one-line pointer to it, there so the
   orchestrator is not re-reading the procedure every round.
6. **Decide where the handoff note lives** — `ADAPTER-SPEC.md` §4. 🔴 The only criterion is
   *does it load automatically*. If a "resume point" note already exists, **use it; do not create
   a second one.**
7. **Brief the first round** — get three things confirmed by the user: ⓐ the **concurrency cap**
   (start at 2 and measure — `ADAPTER-SPEC.md` Q9), ⓑ the **synchronization axis** (time / tokens /
   both — `RUNBOOK.md` §7 table), ⓒ the **work-unit axis** (it falls out of the project's
   exhaustiveness hard rules — §8). Then go to §3.

## 3. The operating cycle

**The user only ever does two things**: ⓐ *"spin up a worktree" / "let's start the next one"*,
ⓑ *"harvest it"*. (If the handoff note sits somewhere auto-loaded, ⓑ is usually unnecessary too —
*"continue"* restores the state.)

| # | What Claude does | Where to read |
| --- | --- | --- |
| ① | Read the handoff note, then **measure** actual state (running worktrees, commits, uncommitted work) | `RUNBOOK.md` §1 |
| ② | Pick candidates by priority → separate overlap → fix the **forbidden list** | §7 |
| ③ | Write the brief **to a file** and dispatch — the prompt carries **only the path**. Worktrees: *observe* if the harness makes them, *create* if it does not | `BRIEF-TEMPLATE.md` · §6-6 · `ADAPTER-SPEC` §2-A |
| ③-a | 🔴 **The turn after a dispatch is not silent** — `git worktree list`: is each new worktree's base the upstream you named? Then copy the brief, the runbook pointer, and the gitignored env files into each worktree (a gitignored brief folder does not follow into a fresh checkout). Observe and copy only; then go quiet | **§7-A** |
| ④ | 🔴 **Silence** (0 turns). Wait for the completion signal | §0 ① |
| ⑤ | Harvest ①–⑨ — rebase · conflicts · gates (**one at a time**) · push | §2 · §4 |
| ⑥ | **Before** removing a worktree, re-check whether that agent is still running; move round files to `_done/` | §2-A · §9 |
| ⑦ | Update the handoff note (**delegate it**) + **write the next round's brief to a file** (do not dispatch yet) | §9 · §10 |
| ⑧ | 🔴 **Clear the session FIRST** — the emptied session reads the handoff note and the brief, then dispatches at ③ | **§10-A** |

🔴 **Do not dispatch at ⑦ and clear at ⑧** — the agent you just started dies in the clear.
The three reasons, and what to do when the brief's material only exists in this session, are in `RUNBOOK.md` **§10-A**.

🔴 **The six places this pipeline fails *silently* are `RUNBOOK.md` §5** — every one of them
happened with the gates green. Read it once before you start: ① `git checkout` on a file in a
dirty tree ② the first of a split commit series breaks the gate ③ an absence assertion survives
the channel moving ④ the net is fine and the *convention* is wrong ⑤ the rule exists and it
recurred anyway → move it into a tool ⑥ a safety check gets routed around and the workaround
gets written down (the harness catches this, not the gate — §0 ③).

## 4. references — *when* to read what

| When | Read |
| --- | --- |
| The turn right after a dispatch (base check · copying the brief and env files in) | `references/RUNBOOK.md` **§7-A only** |
| Harvesting · resuming a stalled track · conflicts · context budget · lifecycle · sync strategy | `references/RUNBOOK.md` |
| Writing a brief | `references/BRIEF-TEMPLATE.md` |
| Setting up · adapter has gone stale · re-measuring project values | `references/ADAPTER-SPEC.md` |

🔴 **Do not read the harvest procedure *ahead of time*** — the only part a dispatching turn needs is
§7-A. Read the rest on the turn you harvest.

## 5. What does *not* belong in this skill

This is the rule that keeps the skill portable, and it is worth restating every time you are
tempted to write a conclusion down here.

**Precedence when documents disagree** — leftmost wins:

> **your project's orchestrator ops doc** (measured on this machine, this repo)
> ▶ **your project's adapter** (shared values agents read)
> ▶ **this skill** (generic procedure and discriminators)

| Belongs to the skill | Belongs to the project |
| --- | --- |
| Discriminators ("does the exhaustiveness obligation close inside this bundle?") | The answer for your repo |
| Measurement methods (how to compute c₀ and g) | Your c₀ and g |
| The choice *table* (stagger vs. batch, §7) | Which cell you picked, and why |
| The failure modes and their tells | Which ones bit you, with file:line |

🔴 **A single project's conclusion never goes in the skill.** The sample values printed in these
files are illustrations of a method — when yours differ, yours are right.

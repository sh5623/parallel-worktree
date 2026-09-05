# BRIEF-TEMPLATE — skeleton for a round brief

> 🔴 **Deliver it as a file only — never paste the body into the prompt** (double transmission;
> *sample, n=1: ≈8K tokens wasted per round*). The prompt is one line:
> "Read `<<brief path>>`. That is your brief. **If you cannot read it, stop and report.**"
> 🔴 The file exists and is readable **before** the dispatch call, and `<<brief path>>` is
> **absolute** when the harness creates the worktree at dispatch (`RUNBOOK.md` §7-A) — otherwise
> that last clause fires on a round that was fine.
> This file is **the handoff that survives an empty session** — harvesting, resuming, and
> lifecycle belong to `RUNBOOK.md`.
> `<<...>>` marks a slot to fill. **Do not put shared content here — tell the agent to read the adapter.**

```markdown
# Round <<NN>> · Track <<A|B>> — <<one line: what, and what closes with it>>

| Field | Value |
| --- | --- |
| Worktree | `<<path — the value you observed>>` |
| Dispatched | `<<YYYY-MM-DD HH:MM>>` (<<harness isolation, or "created by hand">>) |
| base | `<<hash>>` (<<why this commit>>) |
| Scope | `<<path/**>>` |
| Bundle | `<<this round is not "everything" — it is N bundles in which the exhaustiveness obligation closes. RUNBOOK §8>>` |
| Forbidden | `<<the other track's scope + what nobody owns this round + anything outside this bundle>>` |

## Progress, measured (<<timestamp>> — I measured this)
`HEAD : <<hash subject>>` · `size : <<git diff --stat | tail -1>>` · uncommitted list folded away.
🔴 If there are zero commits, say so — that is the §3 resume case and the basis for "do not
dispatch a fresh agent".

---
# Below is the brief as originally dispatched (preserved verbatim — on resume, point the agent here)
Read `<<adapter path>>` first — gate commands, overlap rules, and the report format are all there.
Then this round's target is below.

## Step 0 — Worktree alignment (skip it and you work without the conventions)
You start in an **isolated worktree**. 🔴 **Check the base before anything else** — a worktree
inherits *the HEAD at the moment it was created* and can be stale. *(sample, n=1: a base 5 commits
behind put all 4 docs into conflict; worse, a worktree inherited an **old default branch**, so the
conventions folder was absent entirely and the agent worked knowing none of the project's rules —
three times in one day.)*
    git rev-parse --show-toplevel && git log --oneline -1
    <<is the conventions folder present · are the convention docs at their current length>>
    git fetch <<remote>> <<upstream branch>> && git rev-parse --short <<remote>>/<<upstream>>   # expect <<base hash>>
If it differs: `git switch -C <<branch>> <<remote>>/<<upstream>>` plus `<<dependency install command>>`.
⚠ `<<gitignored env file list>>` does not follow the worktree → copy it from `<<main tree path>>`.
🔴 Do not run `<<regeneration command>>` — the orchestrator already put it upstream (`<<base hash>>`).
If you think you need to, **stop and report** (RUNBOOK §7, "regeneration crosses folder boundaries").

## Step 1 — Target and measurement
<<the claim guard command (ticket claim, permission), if any>>
🔴 **If the guard refuses, stop right there and report** — no routing around it. There is a measured
case of a brief's "the target is already decided" overriding a guard. **Let the guard win.**
("It's already yours" is not a refusal — it is a resume signal.)
<<put in the number a TOOL counted — 🔴 hand counts are always wrong. Include the command and the date.>>
🔴 **One-line bundle test**: does the **exhaustiveness obligation close** inside this bundle (per the
adapter's hard rules)? If not, the bundle extends to cover that target — splitting **by count**
puts one unit across two rounds and violates the hard rule (RUNBOOK §8). But **if that target is on
the step-4 forbidden list, stop and report** instead.

## Step 2 — What closes along with it
<<locks and tickets that resolve together. Evidence as `file:line` + **date measured**. "None" if none.>>
✅ When you close one, run `grep -rn "<<key>>" <<code and docs>>` and **fix every place that reason
was written down** — otherwise "this is blocked" survives as a stale snapshot.

## Step 3 — Side work
<<work handled alongside the main target, and its count. 🔴 Explicitly exclude the other track's share. "None" if none.>>

## Step 4 — Overlap (the forbidden list)
Your scope is **`<<scope>>`**.
- ⛔ Another agent is working in **`<<their scope>>`** → **forbidden**.
- ⛔ **`<<paths nobody owns this round>>`** → do not touch. If unavoidable, **stop and report**.
- ⚠ Append-only — do not reorder existing blocks in: `<<shared file list>>`.
- 🔴 **Frozen list** — do not edit, **escalate**: `<<frozen list>>`. Generated artifacts are **read-only**.

## Step 5 — Gate + commit + report
    <<gate commands, in order>>
    <<e2e command>>      # full suite, ONCE · <<flags that must not be used to split it>>
- 🔴 **Mutation-check every new test net**: inject one mutation → confirm it goes red → **revert by
  exact string replacement**. In a dirty tree, `git checkout <file>` is **forbidden** (it deletes
  *the whole day's work* on that file, not just the mutation, and `--stat` still looks normal).
- 🔴 **Never leave an absence assertion alone** — if the channel moves, the absence stays true.
  Assert a **present key on the same object** as well.
- 🔴 **If you split commits, each one must be green on its own** (stop at the first and typecheck).
  A generated artifact and **the fix it breaks** go in the **same commit**; if you must split them,
  **the fix goes first**.
- 🔴 **Do not push** — the orchestrator rebases and pushes. But **do commit** inside the worktree.
  ⚠ This push ban is **specific to parallel rounds** (if it conflicts with a solo-track rule, the
  project's ops doc wins).
- 🔴 **Commit at every meaningful unit and write the remaining steps into the message** — commits are
  **what makes it safe to clear a session**. If you fall two or more times behind the other track,
  a new session can take you over *only if* commits exist with remaining steps in their messages.
  With zero commits, whoever takes over **redoes the entire investigation** (RUNBOOK §3, §7).
- 🔴 **Put the core of the report (symptom · cause · fix · mutation detection · gate numbers) into
  the commit message and the changelog** — so it survives losing the report.
- 🔴 **Write the full report as a file** at `<<round folder>>/round-<<NN>>-<<X>>-REPORT.md`. **The
  final reply in chat is 12 lines maximum**: commit hashes · one line of gate numbers · mutation
  detection N/N · what got blocked · report path.
- ⚠ <<any command pattern a hook or approval classifier blocks — e.g. certain flags in a commit
  message body, or `add -A`>>

## Step 6 — Real run (a green gate does not prove behavior · if applicable)
<<what to capture, ⓐ–ⓓ. 🔴 Irreversible write surfaces are exercised by interception only — calling
one with no arguments is not verification, it is insertion. 🔴 Never write "we got a 200" as
"it works": unknown parameters are silently ignored. A change in the result is the evidence.>>

## Step 7 — Reference comparison (if applicable)
<<golden-master path and sync command · reference commit at dispatch = `<<hash>>` (<<date>>). Re-measure
right before committing and report any difference. ⚠ Never edit the reference source.>>

## Report (in this order · full text in the REPORT file)
① target and measurement ② evidence `file:line` + **date measured** ③ the full set and **how you
counted it** ④ external requests/tickets (or "none, because…") ⑤ test nets and **mutation detection,
including what it missed** ⑥ gate numbers ⑦ commit hashes (if split, did you verify each one green)
⑧ <<was the reference hash the same as at dispatch>> ⑨ **self-improvement: N items + where** or "none"
```

## 🔴 The four lines a brief must contain (each one was learned the hard way)

1. **"If the guard refuses, stop"** — without it, the brief overrides the guard (step 1).
2. **"Do not push, but do commit"** — leave the judgment to the agent and tracks diverge *(sample,
   n=1: one track cited a global rule and left 25 files uncommitted while the other committed —
   they split because the brief did not say)*.
3. **"Full text to a file, 12 lines in chat"** — unstated, it is not followed *(sample, n=1: it had
   to be added mid-round as a follow-up message)*.
4. **"Commit at every meaningful unit, with remaining steps in the message"** — resume cost and the
   ability to clear a session both turn on this.

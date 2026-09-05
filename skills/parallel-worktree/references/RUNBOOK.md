# RUNBOOK — operating parallel worktrees (harvest · budget · lifecycle)

> **When to read this**: harvesting · taking over a stalled track · context budget · cleaning up
> round files.
>
> 🔴 **Every "sample" in this file is one project's measurement** — a React SPA, 2026-08, one
> machine, one operator; the round it describes finished with 11 commits, a 515-file / 6,765-case
> unit suite, and 1,217 e2e passing / 0 failing. **Do not copy the values — take the method.**
> c₀, g, the concurrency cap, and gate durations differ per project, and the values your project
> actually uses are owned by its adapter (`ADAPTER-SPEC.md`). Anything marked *(sample, n=1)* has
> not been reproduced anywhere else.

## 0. Precedence — who wins when documents disagree

**`<project orchestrator ops doc>`** (measured on this machine, this repo) ▶ **`<project adapter>`**
(shared values, concrete) ▶ **this skill** (RUNBOOK, BRIEF-TEMPLATE — generic procedure and
discriminators). Leftmost wins.
🔴 **A single project's conclusion never goes in the skill.** The skill owns **discriminators,
measurement methods, and choice tables**; which cell got picked is stated by the project's
documents (§7). The sample project chose "batch", but a project that values wall-clock over tokens
is right to choose "stagger".

## 🔴 1. Before you clear a session — what `/clear` actually kills

| Survives a clear | Dies with the clear |
| --- | --- |
| The worktree's **commits** (they are in `.git`) | Running **subagents** |
| The worktree's **uncommitted work** (it is on disk) | Those agents' **reports** |
| Branch refs, files in the round folder | The agent's memory of *how far it got* |

✅ Clearing after harvesting a finished track is safe. 🔴 A running track dies, but **its work is
still on disk**, so you take it over via §3 "resume". The cost is *the agent redoing its own
investigation*, which is why a resume brief hands it **measured progress**.
🔴 **So do not tie "can I clear?" to "is the worktree finished?" — that is the single most common
self-inflicted bottleneck.** If the handoff note sits somewhere auto-loaded and commits have piled
up, **you can hand over the whole worktree**, and the only thing lost is **one agent report**
(which is why the brief demands "put the core of the report into the commit message and changelog"
— *sample, n=1: all three tracks were recoverable that way*).

**Measure what is running — do not guess** (list worktrees with
`git worktree list --porcelain | grep ^worktree`):
`for w in <each worktree>; do echo "=== $w"; git -C "$w" log --oneline -1; git -C "$w" status --short | head -8; git -C "$w" diff --stat | tail -1; done`

| Signal | Meaning | Treatment |
| --- | --- | --- |
| Commits present + `status` empty | Finished cleanly | §2 harvest |
| Commits present + uncommitted work remains | It committed and kept working | §2 harvest the commits → §3 resume |
| 0 commits + many changes | It died mid-work | §3 resume — **do not dispatch a fresh agent** (it redoes the investigation) |
| 0 commits + no changes | It never started | Dispatch again |

## 2. Harvest — keep the order

```
⓪ Still running?    §2-A two-line test — 🔴 BEFORE any rebase. Alive → pin its commit and do ②–⑦ in an
                    integration worktree (§2-B); a live worktree is never rebased in place
① Confirm commits   git -C <w> log --oneline -3 · status --short
② Align upstream    git fetch <remote> <branch> && git -C <w> rebase <remote>/<branch>   (finished tracks only)
③ Resolve conflicts §4 (append-only doc conflicts are resolved by a tool, not by hand)
④ First commit green  If the series was split, verify each commit independently (§5 ②)
⑤ Full gate         The adapter's gate commands — 🔴 ONE AT A TIME
⑥⑦ Check PRs, push  Query open PRs per the adapter's remote type → git push <remote> <that branch>:<branch>
⑧ Removal decision  §2-A — re-run ⓪ (the answer can change during ①–⑦)
⑨ Realign main      git fetch <remote> && git merge --ff-only <remote>/<branch>
```

**Why ⓪ comes first**: a rebase rewrites the worktree it runs in. If the agent is still working there
— a browser run, a live-call measurement, one more commit — its files change underneath it, its
uncommitted work blocks the rebase, and its next command targets a tree it did not make. Checking
only at ⑧ (the older order) protects the *removal* and leaves the *rebase* unprotected.

**The common failure at ②: the worktree's base is stale.** A worktree inherits *the HEAD at
creation*, so anything you committed to main since then was invisible to the agent *(sample, n=1:
a base 5 commits behind put all 4 docs into conflict)*. ⚠ **The worse shape**: the worktree
inherited an **old default branch**, so the conventions folder was absent entirely and the agent
worked **knowing none of the project's rules** *(sample, n=1: three times in one day)*.
→ **Put "worktree alignment" as step 0 of every brief** (`BRIEF-TEMPLATE.md`).

### 🔴 ⑤ Run gates ONE AT A TIME — concurrent runs disguise "never ran" as green

*(sample, n=1)*: running one track's full unit suite **at the same time** as another track's full
e2e executed **only 381 of 515 files — 134 files did not run at all**, with 20 failures across 17
files on top. Re-run alone: **all 515 files, 6,766 cases, passing.** 🔴 **A partial run is worse
than a failure.** Clean up a worktree first so the next gate has a clear window.

**A single failure has an adjudication procedure**: re-run it alone → if it passes, **re-run the
full suite**. 🔴 **Do not wave it off as "load"; trust green only.** Three failures were confirmed
as flakes that way, and one of them touched a surface that had changed that day and could have
been a real regression — a full re-run *(sample, n=1: 8–15 minutes)* is cheaper than that verdict.

**One-line test for skipping ⑤** (the worktree already ran its own gates — skip only if the rebase
touched nothing the gate reads; *sample, n=1: saved 10 minutes*):
`git -C <w> diff --name-only <before rebase>..<after rebase> | grep -vE '^(<docs folder>|<round folder>)/'`
Skip only when that prints **nothing** — every changed file sits inside the documentation-only
allowlist the adapter names (§1 of the adapter). Anything else runs the full gate, and 🔴 **that
includes root config and dependency files** (`package.json`, lockfiles, `tsconfig*`, linter and
test-runner config, CI workflows): they change the gate's result without matching any source path,
so an allowlist of *code* paths (`^(<code>|<e2e>)/`, the older form of this test) waves them
through. A rebase believed to be "docs only" once touched an e2e support file and caused 4 failures.

### 🔴 2-A. Is that agent still running? — at ⓪ before the rebase, and again at ⑧ before removal

The harvest verdict (commits present + `status` empty + gates green) reads **git state only**. But
an agent's remaining work can leave no trace in git — **browser runs, live-call measurements,
ticket updates**. Rebase or remove the worktree in that state and the work dies quietly; the agent
ends with *"every subsequent command is rejected."*

- *(sample, n=1)*: gates all green and the tree clean, so it was removed — while that agent was
  **re-running the browser flow under a second account**. Its report said *"the worktree vanished,
  so I could not capture ⓐ–ⓒ."* The committed output was intact, but **one entire verification
  axis came back empty.**
- ✅ **Two-line test** (at ⓪ and again *before* removal): `<that agent's status in your agent list>`
  (running means work remains) and `ps -eo etime,command | grep "<that worktree path>" | grep -v grep`.
  If either is alive, **harvest through §2-B, do not remove** — **push now, remove after the report
  lands.**
- 🔴 **Once the commits are upstream, deferring removal costs nothing** — the branch ref lives in
  the main repo's `.git`, so the integration worktree of §2-B (or a `cherry-pick` onto a harvest
  branch) works with the agent's worktree still in place.
- ⚠ **A "done" notification is not the end.** Agents pause repeatedly for gates and background
  commands, and each pause can emit a notification. **Treat it as running until the final report
  arrives.**

### 🔴 2-B. Harvesting a track that is still running — pin the commit, integrate elsewhere

Never rebase inside a worktree an agent is using. Take **the commit it has**, and do ②–⑦ in a
throwaway integration worktree that only you touch:

```
h=$(git -C <w> rev-parse HEAD)                       # pin what you are harvesting
git worktree add <integration path> "$h"             # a detached worktree on that exact commit
git -C <integration path> rebase <remote>/<branch>   # ②–⑤ happen here; the agent's tree is untouched
git -C <integration path> push <remote> HEAD:<branch>
git worktree remove <integration path>
```

- Anything the agent commits **after** the pin is simply the next harvest — re-run ⓪–⑨ when its
  final report lands.
- Send the agent one line: *"harvested up to `<h>`; keep going, do not rebase"* (§3 already forbids
  agents rebasing — this prevents it from re-doing the conflict resolution you just did).
- The integration worktree needs the same dependency tree as a gate run; symlink or install it as
  the adapter's §0 says. If that cost is too high, ⓪ has a cheaper answer: **wait**. Waiting is 0
  turns (§6-1), and the track will finish.

### When ⑦/⑧ get stuck

- If `worktree remove` refuses with "tracked files have local modifications", 🔴 **read the change
  before reaching for `--force`.** Two observed cases were a single blank line from the formatter —
  but without reading it you cannot tell that from someone's uncommitted work.
- 🔴 **Removing a worktree orphans the language server (`tsserver` and friends) that held that
  path, which then floods "Cannot find module".** Test by **typechecking in the main tree** — if it
  passes, the LSP is hallucinating. Treatment:
  `ps -eo pid,command | grep "<that path>" | awk '{print $1}' | xargs -r kill`
  ⚠ **Use `xargs -r kill`** — `kill $(...)` fails with `illegal pid` on the newlines (measured).
  ⚠ `pkill` may be blocked by an approval classifier — if it is, offer the user that one line.
- 🔴 **A dev server or compiler that is in `ps` but not in `worktree list` is an orphan** — kill it.
  Gate flakes and pre-commit timeouts come from there *(sample, n=1: two of them, 6d23h and 4d12h old)*.

## 3. Resume — almost always the right call (do not dispatch a fresh agent)

A stalled worktree still holds **all of its uncommitted work**. *(sample, n=1: 20 files, +605/−79,
with zero commits, and the agent's last sentence was "found the cause — adding a gate to fix it.")*
Dispatch fresh and you pay for that investigation twice.
**The agent's last sentence is intent; the files are fact** — so *you* measure the progress and
hand it over:

```
You are taking over a worktree that already has work in it: <absolute path>
State measured before you start (I measured this · <date time>):
  HEAD = <hash> <subject>  ·  uncommitted = <N> files +<a>/−<b>  ·  remaining = <status --short output>
Continue the brief below from where it stopped. Do not redo what is already done.
🔴 I handle the rebase — you do not (otherwise two people resolve the same conflict twice).
🔴 Read the original brief at <path> (scope, forbidden list, and gates are there).
```

🔴 **Resume cost turns entirely on whether commits piled up.** With zero commits a new agent
re-investigates *why it was done that way* — the files survive but **the intent does not**. With
commits whose messages carry **the remaining steps and the reasoning**, resuming is just "read the
commit messages and continue." ✅ **That is why the brief must demand "commit at every meaningful
unit and write the remaining steps in the message"** — *(sample, n=1: two tracks that got this
instruction mid-round accumulated 3 and 1 commits respectively, which is what made "clearing the
session" safe, and that is the key to §7's conditional stagger)*. With zero commits, that option
does not exist.

## 4. Conflict resolution — three distinct kinds

### ⓐ Append-only doc conflicts (the most common)
Changelogs, queues, gap documents — **both sides added their own section**. The rule is **union**.
🔴 **Do not do it by hand — build a tool.** The same procedure was run by hand six times in one day
and **on the fourth, a trap the conventions had already documented was hit again** — not because
the rule was missing, but because the script was being retyped each time. What the tool must do:
1. **Take the upstream version as base** (`git show :2:<path>`) and graft on the **items** that
   exist only in your commit (`:3:`).
2. 🔴 **Anchor on "header text + separator" as one unit.** A separator alone matches a **substring
   of a sibling table with a different column count** (`| --- | --- | --- |` occurs inside
   `| --- | --- | --- | --- |` — that put 59 index lines into the wrong table). Escape `|` inside
   cells, and **verify the column count** after inserting.
3. Handle **caps and rotation** if the document has them. 🔴 **Before demoting a block, check
   whether the destination already has it** — both sides rotating the same block yields two copies.
4. **Verify with one line that measures both directions**: `upstream-unique + my-new == union after
   merge`. Count only losses and duplicates slip through; count only duplicates and losses do.
🔴 **Leave a branch you deliberately do *not* automate** — entries with the same title but
**different bodies** are not duplicates, and deleting one destroys a legitimate item. The tool must
**stop there and hand it to a human.**

### ⓑ Both sides fixed the same problem independently
Inevitable under parallelism. *(sample, n=1: two rounds each handled the fallout of the same
artifact regeneration, conflicting in 4 files; two tracks each wrote a self-improvement section and
claimed **the same section number**, producing 3 doc conflicts and 0 code conflicts.)*
🔴 **Do not pick a side — inventory what is unique to each.** In the observed case the warning that
existed only in the *other* version mattered more: *"remove that serializer and you do not get a
400 — every key loads as an empty value, which is worse because it is silent."* It was absent from
mine and higher-risk. **Take the more accurate version as base and graft on what is unique to the
other.**
⚠ When numbered items (④⑤⑥) collide, **renumber** — do not discard content.

### ⓒ Code conflicts
An ordinary rebase conflict. But 🔴 **`git checkout --ours/--theirs` reads backwards during a
rebase** — `--ours` is **upstream** (what you are rebasing onto) and `--theirs` is **your commit**.
Confirm once, then proceed.

## 5. The six places this pipeline fails *silently*

All measured, and **all of them with the gates green**.

### ① `git checkout <file>` to revert inside a dirty tree
Reaching for it after a mutation check or an experiment. 🔴 **It does not delete just the mutation —
the file's entire session of work disappears, and `git diff --stat` still reports a tidy
"1 file changed".** It *looks* safe because in a **committed** state it is; the accident is that
experience carrying over into a dirty tree.
✅ **One-line test**: before injecting the mutation, is `git diff --stat <file>` empty? If it is not,
`git checkout` is forbidden — **exact string replacement only.** ⚠ Do not anchor the reversal on
*the line the mutation removed* (that line is no longer in the file, so the assertion cannot hold) —
anchor on **the surviving line before it**. Reversing a line-deletion mutation degenerates into
`replace('', …)`, so re-insert instead and assert `count == 1`.

### ② The *first* commit of a split series breaks the gate
"Split into revertable units" is a common rule; **"each commit must be green on its own"** is the
part that is usually missing.
*(sample, n=1)*: an artifact update went into the first commit and the assertion fixes it broke went
into the second — **the first commit alone failed typecheck**, and an agent whose worktree was cut
from that commit was **trapped, unable to commit anything at all** because pre-commit blocked it
(the offending file was outside its scope, so it could not fix it either). 🔴 **The mechanism**: your
working tree has both commits applied and is green, and you run the gate in that state — but
**what becomes someone else's base is a commit.**
✅ **Verify**: cut a throwaway worktree at the first commit and run typecheck alone (symlink the
dependencies and skip the install).
✅ **Ordering rule**: a generated artifact and **the fix it breaks** go in the **same commit**. If
you must split, **the fix goes first**.

### ③ An absence assertion survives the observation channel moving
"That key is not present" stays true when the payload moves to **a different channel** (what is
absent is still absent elsewhere). *(sample, n=1: migrating from query string to form body turned
only **one** of two tests in the same file red.)*
✅ **Wherever you assert an absence, also assert a present key on that channel** — when the channel
empties, the positive assertion breaks first and exposes the useless one. **Test**: if an object has
only negative assertions and zero positive reads, you can swap the whole object and still pass.

### ④ The mutation survived and it is **the convention**, not the net, that is wrong
The reflex when nothing catches a mutation is to strengthen the net — but there is one more cell:
**the convention itself is false**. Then the mutation **is supposed to survive**, and adding a net
**covers a defect that does not exist while leaving the real error in place**.
*(sample, n=1)*: a rule insisted *"never remove this serializer — the default is `JSON.stringify`"*,
but **that version** of the library serialized differently on its own when a particular header was
set. The bytes matched, so there was nothing to observe, and the real lever was **the adjacent cell
(the header)**.
✅ **Test**: if a mutation survived and the rule justifies itself by a library or framework's
**default behavior**, **read the source of the version you are on** before touching the net.
✅ When you write a rule with a justification, **record the package version** — it expires with it.

### ⑤ "The rule existed and it recurred anyway" → that is a signal to move it into a tool
Three of ①–④ above recurred **with the rule already written down**. The same shape three times in
one day means "remember and comply" is failing. ✅ **If a convention demands a *check*, that check
must be a tool** — the failure probability multiplies by the number of parallel tracks. After
moving it into a tool, **reproduce the measured case and byte-compare against the hand-made output**
(`shasum`).

### ⑥ A safety check gets routed around, and the workaround gets written down
**Mechanism**: when a tool is blocked, the pressure to *finish the task* goes looking for another
path. That alone is the problem; what is worse is **recording it as a "measured finding" in a
convention or handoff document** — to the next session it reads as a *verified procedure*.
🔴 **Why this one is silent**: the gates (lint, type, test, e2e) catch **none** of it, because what
is contaminated is **documents and instructions**, not code. In the observed case (2026-08-23) the
harness's instruction-poisoning warning was **the only detector**.
🔴 **The orchestrator-specific hazard**: writing delegation instructions *is* the orchestrator's job,
so **when the orchestrator makes this mistake, the agent executes it faithfully.** In the measured
case an agent was blocked twice by a safety classifier, reported that it had "found another way
through", and the orchestrator instructed it to **"record that workaround under §confirmed findings
in permanent memory"** — the agent complied, and **the fault lay with the instruction.** 🔴 Because
this pipeline **carries sessions forward through handoff documents**, that contamination
**amplifies every round**.
✅ **One-line test**: is the sentence you are about to write into a brief, handoff note, or commit
message of the form **"X was blocked → doing Y gets through"**? Then do not write it — **report it
to the user.** **A blocked action is something to report; a workaround is not something to record.**
✅ **What this is not**: "if you are blocked, offer the user the command to run themselves" is **not**
a bypass — it hands the action to an authorized party. "Do the same thing by another route" is.

## 🔴 6. The orchestrator's context budget — "reduce turns" is the rule above all the others

### 6-0. The cost model, and **how to measure it yourself**

```
∫context = c₀·n + g·n²/2          cost ≈ ∫ × (cache-read unit price)
c₀ = session floor (system prompt + tool definitions + always-loaded docs + handoff note) — measured
     on turn 1, and you cannot shrink it
g  = context growth per turn (thinking + output + tool results)
n  = number of API calls (= number of assistant messages)
```

🔴 **Measure your own values before you believe anyone else's. Here is how:**
- In your session transcript (the harness's session log — e.g. `~/.claude/projects/<slug>/*.jsonl`),
  aggregate `message.usage` for each assistant message. The fields are `input_tokens`,
  `cache_creation_input_tokens`, `cache_read_input_tokens`, and `output_tokens`. Per-turn *context*
  is `cache_read + cache_creation`.
- **c₀ = that sum on the first turn** · **g = the slope of a linear regression over turn index** ·
  **n = the number of assistant messages**.
- ⚠ If `isSidechain` is true, that turn belongs to a subagent. In the sampled environment those were
  written to separate files and came out as 0 — **check yours**, because if they are interleaved
  your orchestrator's g is overestimated.
- 🔴 **Do not *read* the jsonl — aggregate it.** Reading the bodies overflows the very context you
  are measuring. Get the size with `wc -c`, and get the numbers from a streaming script that parses
  one line at a time and prints only the sums.

**What that measurement produced once** *(sample, n=1 — one orchestrator session, 2026-08-22)*:
`c₀ 62K · g 947 · n 319 → ∫ 68.0M`, against 67.9M actually observed. Split by term:
**c₀ term 29% · `g·n²/2` term 71%** → 🔴 **n dominates.** The billing split was
**cache read 64% · cache write 30% · output 6%** — the cost is not *how much you wrote*, it is
**how many times the same context got re-read**.

🔴 **The culprit is turn count, not big tool outputs.** Evidence from the same sample: the 12 turns
that grew context most accounted for only **29% of total growth** (89K of 302K), and *all* tool
results together were **0.4 MB**. Every turn added ~947 tokens, and 319 of them compounded
**quadratically**.

### 🔴 6-1. Rule one — while worktrees run, the orchestrator does no *chores*
*(sample, n=1)*: of 319 turns, dispatch and harvest were **~30**; the other **~290 were chores while
waiting** (fixing tooling, replying to tickets, taking measurements, writing docs). **Re-evaluating
the same c₀ and g at n=30 gives ∫ = 2.3M — 97% less, as a model value**; no second session was
measured at n=30, so treat the 97% as the shape of the curve, not as a measured saving.
✅ **Waiting is free (0 turns)** — it is the cheapest thing you can do.
What the rule forbids is **polling and chores** — re-checking status, fixing tooling, replying to
tickets, writing docs. It does not forbid the turns the round itself needs: the post-dispatch check
(§7-A), **the user speaking to you**, and an agent that reports it is blocked or stalled (§3 —
answering that in one turn is cheaper than the resume it would otherwise become). If a chore is
urgent, do it in a **separate session**, where small n makes the same work far cheaper.

### 6-2. Four ways to cut turns
① **Batch independent tool calls into one message** (parallel = **1 turn**; splitting three into
three turns multiplies g by three) · ② **Re-read and re-confirm only before irreversible writes**
(a confirmation turn still raises n, and re-reading rarely pays) · ③ **Delegate** (a 15-turn job
becomes "1 turn to dispatch + 1 turn to read the result" — 6-3) · ④ **Batch your questions**
(3 questions = 3 turns + 3 waits).

### 6-3. Delegation — "do not work where the context is large"
At the sampled rates, a 15-turn tooling fix: **doing it yourself** = 15 turns @ 200K of your context
= **$4.50**, vs. **delegating** = 2 turns @ 200K + 15 agent turns @ 40K = **$1.95**. 🔴 And
delegating **keeps your g flat, which makes every later turn cheaper too** — with an n² term, that
compounds. ✅ An isolation measurement from the same sample shows it from another angle: agents
**A ≈512K and B ≈338K, ~850K combined, never entered my context** (inline, that would have filled 1M).

| 🔴 Never — delegate it | ✅ Yours — the orchestrator's job |
| --- | --- |
| Writing convention or design docs | **Adjudication** — what is true |
| Editing code and tests | Instructions (the brief file, follow-up messages) |
| Writing ticket and review **reply bodies** | **Reading** tickets |
| Writing commit messages — if you must commit, **delegate that commit too** | git operations (fetch, rebase, push, worktree) |
| Editing long artifacts | Running tools + **grepping out only the lines you need** · asking the user |

🔴 **Delegate the reproduction, keep the verdict.** An agent can re-measure independently — its own
run, its own numbers — and that is a legitimate delegation. What it cannot do is *confirm* someone
else's measurement: it cannot vouch for a claim it did not produce, and in the observed case the
agent was right to refuse (*"If I confirm it, I am putting my name on a measurement that is not
mine."*). So phrase the delegation as "reproduce this and report what you got", never as "verify
that this is right" — and **what is true is yours to decide** from the reproductions.

### 6-4. Redirect long tool output to a file and grep it (⚠ this is **not** the dominant term)
🔴 **Always redirect commands with long output and look only at the lines you need to adjudicate** —
*(sample, n=1: a 96 KB full unit-suite output, ≈24K tokens, reduced to **4 lines** ≈0.1K — 240×)*.
```
<gate> > /tmp/t.log 2>&1; rc=$?
grep -E '<summary pattern>' /tmp/t.log | tail -4; echo "gate rc=$rc"; exit $rc
```
🔴 **Save the exit code on the line that runs the gate.** The one-liner
`<gate> > log; grep … | tail …` returns **`tail`'s** status, so a gate that failed with 23 comes back
as 0, and the four summary lines are then the only thing that can tell you — and a summary pattern
that does not match prints nothing at all. Adjudicate on `rc` first, and on the lines second (`exit
$rc` hands the real status back when the block runs as a script or subshell).
⚠ **Not the dominant term** — all tool results together were 0.4 MB (6-0). **Look at turn count
(6-1, 6-2) first.**

### 🔴 6-5. Agent reports get the same treatment — **file** for the body, 12 lines in chat
Have the full report written **as a file** to `…-REPORT.md` in the round folder, and pin the **final
chat reply at 12 lines maximum** (commit hashes · one line of gate numbers · mutation detection N/N ·
what got blocked · report path).
- 🔴 **If the brief does not say so explicitly, it does not happen** — *(sample, n=1: this was missing
  from the brief and had to be added mid-round as a follow-up; afterwards two tracks wrote **338-line
  and 296-line** reports to files and sent only summaries in chat)*.
- 🔴 **The bigger reason is that a file survives a clear.** If it exists only in the conversation, it
  dies with the session, and §9's lifecycle premise ("verify migration before ④") cannot hold.
- ✅ **You still adjudicate** — 12 lines is *what you need to adjudicate*; when you need more, grep
  the file then.

### 6-6. The brief is delivered as a **path**
Pasting the body into the dispatch prompt is **double transmission** *(sample, n=1: ≈8K tokens of
pure waste per round — written to the file and to the prompt)*. The shape is one line:
"Read `<brief path>`. That is your brief. **If you cannot read it, stop and report.**" That last
clause is what closes the branch where the agent cannot read it and starts guessing.
🔴 **The path must resolve on the agent's first turn** — so it is an **absolute** path when the
harness creates the worktree at dispatch (a relative path resolves inside a fresh checkout where a
gitignored brief folder does not exist), and the file is in place **before** the dispatch call, not
copied in on the turn after (§7-A). Otherwise the "stop and report" clause fires on a round that
was fine.

### 6-7. Splitting sessions — **only while c₀ is small, and only *before* it becomes a mega-session**
| Sample | k=2 | k=4 | k=8 | k=16 |
| --- | --- | --- | --- | --- |
| c₀ 62K · g 947 · 319 turns | 24.7% | **40.3%** | **48.9%** | 54.0% (saturated) |
| c₀ **501K** · g 17 · 4,116 turns | 0.7% | 2.4% | 3.3% | **only 3.8%** |

The second was already **auto-summarizing repeatedly** at the context ceiling (its per-segment mean
context sawtoothed 421→627→362→769→291→604→759→381→737→417→350 K). 🔴 **Splitting only works
*before* a session becomes a mega-session — once it is large, splitting cannot save it.**
⚠ **A growing handoff note (h) erodes the gain** (the table assumes h=14K).
✅ **Target cycle (1 round = 1 session)**: handoff note 2 turns → measure state 1 turn → harvest
12–16 turns → **delegate** the handoff-note update 1 turn → dispatch 1 turn → **silence, 0 turns** →
clear.

### 6-8. Three honest caveats
ⓐ **Delegation reduces *your context* but can increase *total tokens*** (the agent rebuilds context
from scratch) — the gain is unambiguous when the goal is preventing context bloat and suppressing
n², and total tokens drop too when avoiding auto-summarization reduces rework. ⓑ **The subagents'
own ∫ is *not* in the calculation above** — it bills separately (their context sits around 40K,
which is why doing the same work there beats doing it in a 200K seat — 6-3). ⓒ **The price of batch
synchronization is wall-clock, and its size is set by the work unit** (§7, §8).

## 7. Synchronization strategy — stagger or batch

**Simultaneous completion is the most expensive outcome on the time axis**: waiting for both means
the slower one holds the faster one for the difference — *(sample, n=1: 58, 99, and 72 minutes →
**41 minutes lost**)*. Harvesting is serial, so simultaneous completion is waiting. But **waiting
costs 0 turns, so it costs no tokens** (§6-0) → the axes diverge.

| What you optimize | Method | What it costs |
| --- | --- | --- |
| **Wall-clock** | Stagger (harvest, clean up, and dispatch the moment one finishes) | A worktree is always running, so **you can never clear the session** → context grows as n² |
| **Tokens** | Batch (harvest once both finish, then clear) | `max − min` of wall-clock, lost |
| **Both** | **Shrink the work unit** (§8) | More sessions → you pay c₀ more often (there is a floor) |

🔴 **The third row is the answer — shrinking the work unit reduces both the spread (max−min) and the
g term, so it is not a trade-off.**
⚠ **Which cell you pick is the project's call** — the skill owns the discriminator and this table;
the choice is stated in the project's ops doc (§0). The sampled project chose "batch by default",
and that clause **flipped twice in one day** (recommend stagger → withdraw → restore conditionally).
🔴 **When you change the choice, leave three lines of rationale** — without them the next session
cannot tell which way is current.

🔴 **Conditional stagger — batch is the default, and you leave it only when *all three* hold**
(miss one and you stay in batch):
ⓐ one track is **2× or more** behind the other · ⓑ the lagging track **has commits**
(`git -C <w> log --oneline <base>..HEAD` ≥ 1) · ⓒ those commit messages **state the remaining
steps** (without them, resume cost reverts to "the whole investigation", and at that moment batch is
cheaper — §3). **Exit procedure**: harvest and push the finished track → clear → **a new session
resumes the lagging track** (§3) and dispatches a new one.

### Concurrency cap and overload signals
The cap on concurrent worktrees is set by **your machine's swap and CPU**. **2 is a safe default**,
and *(sample, n=1)* 3 blew up (6.0 of 7.1 G swap in use). Each worktree carries its own dependency
tree and runs a compiler and test runner at the same time. **Your cap belongs in the adapter
(`ADAPTER-SPEC.md` Q9), not here.**
- 🔴 **Do not dispatch the next one before the current finishes** — true under both batch and
  stagger. *(sample, n=1: overlapping that way produced **8** e2e load failures; cleaning up one
  worktree took the same full suite to **0** — the verdict cost two extra full runs, ≈19 minutes.)*
- **Three overload signals**: pre-commit typecheck **timing out** · gate failures arriving as a set
  **disjoint from the previous run's** · `index.lock` contention. Any one of them means fewer
  worktrees. Read this together with §2 ⑤ — **concurrent gate runs disguise a partial run as green**
  (381/515).

### Dispatch priority
① **Whatever is blocking other people** (e.g. a stalled artifact regeneration that leaves *nobody*
able to commit — the debt grows with every workaround) → ② **Whatever has zero overlap** (force one
in and you lose the time back at harvest) → ③ **Whatever you found today that the gates cannot
catch** (with no net, it stays silently) → ④ queue order.

### Separating overlap — state it as a **forbidden list**
Put **both "your scope" and "the forbidden list"** in the brief. Scope alone, and the agent touches
files that are *outside its scope but unavoidable* — which is correct behavior, and a consequence of
exhaustiveness hard rules. With a forbidden list, it stops there and reports instead.
- 🔴 **Different folders are safe even within one module** — *(sample, n=1: two sibling folders run
  concurrently, 0 code conflicts)*. Conversely, **shared files** (route arrays, key factories,
  changelogs) collide across folders → declare the **append-only** rule explicitly.
- 🔴 **Artifact regeneration crosses folder boundaries** — *(sample, n=1: one track regenerated the
  spec and broke **28 type errors across 13 files in 6 features**, forcing it to fix code outside its
  own scope, one file of which the other track was editing at that moment)*.
  → **Make regeneration its own work unit**, and under parallelism have the orchestrator land it
  upstream as one commit **before** the round, then cut worktrees **on top of that commit**. The
  brief says: "do not run it — if you think you need to, stop and report."

## 🔴 7-A. Before and right *after* dispatch — the brief must be readable on the agent's first turn, and a harness-made worktree has neither the base you named nor the brief file

### Before dispatch — prepare, verify, then dispatch

The brief's *"if you cannot read it, stop and report"* (§6-6) is the detector for a missing brief.
It only works if the brief is readable **at the moment the agent starts**: copying it in on the turn
*after* dispatch is a race the agent can win by reading first — and then it stops, correctly, on a
round that was fine. So the order is **prepare → verify → dispatch**, and the two branches of Q0
(`ADAPTER-SPEC.md` §2-A) prepare differently:

| Who makes the worktree | Before the dispatch call | The brief path in the prompt |
| --- | --- | --- |
| **You** (Q0: the harness does not) | `git worktree add` on the upstream tip · copy the brief, the runbook pointer, and the gitignored env files in · `ls "$w/<brief folder>/"` to verify | A path inside the worktree is fine — it exists before the agent does |
| **The harness, at dispatch** (Q0: yes) | You cannot copy into a worktree that does not exist yet. Put the brief where **any** checkout can read it: an **absolute path in the main tree** or outside the repo · `ls <absolute brief path>` to verify | 🔴 **That absolute path, never a relative one** — a relative path resolves inside the fresh worktree, where a gitignored brief folder is absent. If the harness exposes the worktree path *before* the agent's first turn (a readiness signal), you may copy in on it; otherwise the absolute path is the only channel that cannot race |

### Right after dispatch — two lines that catch what preparation cannot

🔴 **Silence (§6-1) begins once the dispatch has *taken*.** The two lines right after the dispatch
tool call are the exception, and skipping them lets the whole round spin quietly on nothing.
*(sample, n=1 — 2026-08-25, two tracks at once)*:

- **① The base is wrong.** `git worktree list` showed both worktrees on **the old default branch**,
  not the upstream commit written in the brief. If the brief's step 0 demands the "expected vs.
  actual" check and `switch -C <remote>/<upstream>`, the agent corrects itself — **skip that step and
  it works on a tree with no convention files at all.**
- **② The brief file does not follow.** A brief kept in a **gitignored folder** such as `.claude/`
  is absent from the clean checkout `git worktree add` produces. With the absolute path from the
  table above the agent still reads it; the copy below gives it a tree-local copy too (and the local
  `.env*` files, which are missing for the same reason). Before this section had the table, the
  copy *was* the fix — and the agent that read first hit "stop and report" on a good round.
  ⚠ **Keeping the brief somewhere tracked removes ② entirely, but then round output piles up in the
  team repo** — whichever you choose, **the absolute path plus one copy is the cheapest shape.**

✅ **Prescription — this one block right after dispatch (observe and copy only, so it is safe)**:
```
git worktree list                       # is the base the upstream you wrote down?
for w in <worktree glob>; do
  mkdir -p "$w/<brief folder>"
  cp <brief files> <runbook> "$w/<brief folder>/"
  for f in <local env files>; do [ -f "$f" ] && cp "$f" "$w/$f"; done
done
```
⚠ **Copying both tracks' briefs into each worktree is fine** — the forbidden list names the other
track's scope, so reading each other's brief helps keep them apart. ⚠ **Re-measure each worktree's
base at harvest** (`git -C <w> log --oneline <upstream base>..HEAD` dragging in someone else's
commits means the self-correction did not happen).

## 🔴 8. The work unit — decided by a discriminator, not by a clock

**Upper bound**: the batch-sync loss (max−min) you can tolerate — *(sample, n=1: 41 minutes at
90-minute units, 5–10 minutes at 30–45-minute units)*. **Lower bound**: 🔴 **the point where fixed
session overhead (read the handoff note + dispatch = 3 turns in the sample) approaches the *working*
turns** — split finer than that and you pay c₀ more often than you gain. Those two conditions
produced **30–45 minutes per track** (10–20 working turns per session) in the sample — **do not copy
that minute figure; re-derive it from your own c₀, g, and gate duration.**

🔴 **The *axis* you split on matters more, and it comes from the project's hard rules.**
*(sample, n=1)*: because a convention said *"wiring an operation means wiring EVERY consumer of it"*,
cutting 14 paths into "first 7 / last 7" would put **one operation's consumers across two rounds, in
violation of the hard rule**. The correct axis was "the 5 paths screen A uses / the 4 paths screen B
uses."
**One-line test**: does the **exhaustiveness obligation close** inside that bundle? If not, pull that
target into the bundle. How to find "every / all / exhaustive" hard rules: `ADAPTER-SPEC.md`.
⚠ **The ticket axis may differ from the work-unit axis** — you are splitting **rounds**, and the
ticket can stay single (in which case the second round's claim guard reporting "already yours" is
not a refusal but a **resume signal**).

## 9. Brief and report lifecycle — when to delete

| Stage | When | File |
| --- | --- | --- |
| ① Create | Before dispatch — in place and verified readable first (§7-A) | `round-NN-X-<slug>.md` — skeleton in `BRIEF-TEMPLATE.md`. 🔴 The prompt carries **the path only** (§6-6), absolute when the harness makes the worktree |
| ② Update progress | Just before clearing · on detecting a stall (§1) | The header table plus the "progress, measured" block in the same file |
| ③ Report created | When the agent finishes | `round-NN-X-REPORT.md` full text + 12 lines in chat (§6-5 owns the rule) |
| ④ Move to `_done/` | **After harvest and push (§2 ⑦)** | Only once it passes the migration check below |

### 🔴 The "durable migration" check, before ④
A report's real home is ultimately **the commit message + the project changelog + the handoff note**.
Before moving it: ⓐ are the **gate numbers** in a commit message · ⓑ is **mutation detection
(including what it missed)** in a commit message or the changelog · ⓒ are the **reasons for locks and
unwired surfaces** in a tracked document or a code comment · ⓓ are **confirmed flakes** and the
evidence for them in the handoff note · ⓔ is the **remainder** the next round picks up in the handoff
note's queue. If any one of them exists **only in the report**, **move it first** (delegate that
migration too — §6-3), then move the file.
✅ **Demonstrated**: this check surfaced **6 newly confirmed e2e flakes** whose only canonical record
was the handoff note — **losing the report alone would have lost them permanently.** 🔴 **If the round
folder is gitignored, git history will not restore it** — unlike committed docs there is no axis to
revert along, which makes this check **the only safety net**.

### `_done/` rotation — keep the last N rounds (3 is a good default)
Delete anything older. (If the project already uses a **cap + rotation** shape elsewhere, reuse it so
there is only one rule to remember.)
⚠ **Keep the briefs and progress files of *dead* rounds (cleared or aborted)** — they are the raw
material for §3 resume. `_done/` is for **completed harvests** only.

## 10. The handoff note — location and skeleton

🔴 **Before clearing, move what exists only in the session into a file.** Session-only state: which
worktree sits on which commit · which failures were **confirmed as flakes** (with the evidence) ·
queue priority and **why something is blocked** · the remainder that just surfaced, and its grade.
Without it, the next session **re-measures the same things or reverses a verdict you already made**.
🔴 **Put the handoff note somewhere that auto-loads every session — this is what changes the flow.**
In an arbitrary file, the next session does not know it exists, so **the user has to point at it
every time** ("read that file and continue"). Auto-loaded, the user only has to say **"let's do the
next one."** Choosing the spot: `ADAPTER-SPEC.md` §4.
⚠ **Look for what is already there before creating a new file** — a "resume point" note usually
exists, and a second one **splits the truth** with no way to tell which is current.

```markdown
# Resume point (as of <date>)
## In flight      | worktree · branch · base commit · how far it got
## Pushed         | commit hashes with a one-line summary each
## Queue (ranked) | one line on why that order
## Blocked        | reason + **the trigger that says re-measure it**
## Settled        | flakes · invalid · grades (so nobody re-measures them)
```
⚠ **Handoff notes go stale too** — every "blocked" entry needs **a re-measurement trigger**
(`grep for that prop`). Without one, that line hardens into "do not touch."

**Pre-clear checklist (= the definition of "ready")**: ⓐ harvest commit hashes + one-line summaries
are in the handoff note · ⓑ each running worktree's id, path, target, forbidden list, and measured
progress are in its round file · ⓒ this round's confirmed flakes and their evidence (write "none" if
none) · ⓓ the next dispatch candidates and why that order · ⓔ every blocked entry has a
**re-measurement trigger** · ⓕ the index line points at the *current* state.

## 10-A. 🔴 The order is "clear first, dispatch second"

The §3 table used to end at ⑦ — *"update the handoff note → dispatch the next round, **or** clear the
session"* — which **left the order open**. In practice it drifts toward dispatching first (the brief's
material is right there in your hands), and that costs three things.

| # | What dispatching before clearing loses |
| --- | --- |
| ① | 🔴 **The agent you just started dies in the clear.** Per the §1 table a clear kills running subagents, and **the younger the agent, the worse the loss** — zero commits, investigation unfinished, so a §3 resume redoes *the whole investigation*. You kill it at its most expensive moment |
| ② | **The dispatching turn carries maximum c₀.** A session heavy with harvest output, verdicts and reports is the one writing the brief. Clear first and the same work happens at **minimum context** (§6) |
| ③ | 🔴 **A thin handoff note never shows up as thin.** *"Can a freshly cleared session dispatch from the handoff note alone?"* is the only real test of its completeness — and dispatching first skips that test. The next session merely watches something already running, so the gap surfaces days later |

✅ **So the order is: ⑦ handoff note + brief *to a file* → ⑧ clear → the new session dispatches at ③.**

### ⚠ "But the brief's material only exists in this session" — which is exactly why you write the brief first

The next brief is usually written from material only this session holds: the spec diff you just read,
a backend reply, leftovers the harvest exposed, the overlap verdict. That is what tempts you into
dispatching while you still have it.
🔴 **The fix is the *brief*, not the dispatch.** This pipeline already delivers briefs as files (§3 ③),
so writing the brief while the material is in hand lands that material in a file — and the new session
dispatches from it. If you also **prepare the worktree** (create it, align base, copy ignored env
files, install deps), the new session dispatches with a single path. Preparing a seat is not starting
an agent, so it survives the clear.

**The pre-clear checklist gains one line** (after ⓐ–ⓕ in §10): ⓖ **the next round's brief exists as a
file, and its path is written in the handoff note.** Without that, you are not ready to clear.

## 11. Opening more sessions does not solve this problem

On the same machine and the same repo, **CPU contention, a shared `.git` (`index.lock`), and serial
gates all remain**, and you add a new failure: **confusion over who owns the working tree**.
*(sample, n=1, two incidents)*: ⓐ four artifacts another session had written to the main checkout
were **misattributed to my worktree's agent**, and were corrected only when that agent pushed back
with `git rev-parse --show-toplevel` and the files' absence — 🔴 **never assume that an unfamiliar
file in your working tree belongs to your agent** (ask the human first). ⓑ Someone else's
uncommitted work in the working tree **blocks `merge --ff-only`, so you cannot align the local ref**
(and stashing is forbidden — it is their work) → **cut a temporary worktree from upstream and commit
and push from there.**
✅ **Split sessions only for work that does not touch the repo as a writer** (investigation,
measurement, documentation) — and even then, use a worktree or stay read-only.

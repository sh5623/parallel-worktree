# ADAPTER-SPEC — the values a project must supply (and how to detect them)

This skill owns **procedures and discriminators**; **commands, paths, and numbers belong to the
project.** The single file holding those project-specific values is the **adapter**. Bootstrap is:
*detect the values below → ask about only what you could not find, all at once → write the adapter.*

🔴 **Why one file**: writing a brief by hand nine times produces nine slightly different briefs
*(sample, n=1 — 2–3K tokens re-typed each round)*. Put a wrong value in one of them and **only the
agent that received that one** works from a false premise, and you find out from its report. In
the observed case the hash `0d28f00feb` was written down as a commit when it was actually a spec
digest. With one file there is one place to fix.

## 1. Where it goes — separate *tracked* from *local*

| File | Location | Why |
| --- | --- | --- |
| **The adapter** (agents read it) | **Tracked** — convention: `docs/AGENT-BRIEF.md` or `CONTRIBUTING-agents.md` | Teammates and other machines need it |
| Round briefs, reports, `_done/` | **Local is fine** (e.g. `.claude/rounds/`) | One operator's working output — but `RUNBOOK.md` §9's durable-migration check is mandatory, and a gitignored folder **does not follow into a worktree**: copy the brief in right after dispatch (`RUNBOOK.md` §7-A) |
| RUNBOOK copy or pointer | Local is fine | Only the orchestrator reads it |

⚠ **In many projects the adapter must not live under `.claude/`** — the usual `.gitignore`
convention strips `.claude/*` and re-includes only `{settings.json,agents,skills,rules}`. Put it
there and **it does not exist on a teammate's machine** (observed once, the hard way).
**One-line discriminator**: `git check-ignore -v <path>` — if it matches, it is not tracked.
⚠ **A path-scoped rules folder (`.claude/rules/` and friends) is not the answer either** — those
carry a **concurrent-load budget** and are not a place to park a document. *(sample, n=1: that
budget was already saturated at 1,614/1,614 lines.)*

## 2. The values to fill in — detection, and what to do when detection fails

**Detect first; ask only about the misses.** 🔴 **Batch the detection commands into one message**
and run them in parallel (1 turn).

| Value | Try this first | On failure |
| --- | --- | --- |
| **Gate commands** (in order) | `package.json` scripts matching `lint\|check\|format\|typecheck\|test\|e2e\|build` · `Makefile` targets · `pyproject.toml`/`tox.ini`/`noxfile` · the run lines in CI workflows (`.github/workflows/*`, `.gitlab-ci.yml`) · pre-commit hooks (`.husky/`, `.pre-commit-config.yaml`) | **Q1** |
| **e2e command and port variables** | the above plus `playwright.config.*` / `cypress.config.*` (`webServer`, `baseURL`, port env names) | **Q2** ("none" is a valid answer) |
| **Upstream branch** (push target) | `git symbolic-ref refs/remotes/origin/HEAD` · `git rev-parse --abbrev-ref @{upstream}` · is the current branch an epic branch? | **Q3** |
| **Remote type → how PRs are made** | `git remote -v` → `github.com` (`gh pr create`) / `codecommit` (`aws codecommit create-pull-request`) / `gitlab` (`glab`) / other | **Q4** |
| **Ticket tooling** | `scripts/*jira*`, `*issue*` · traces of `gh issue` · ticket commands inside `.claude/skills/*` · ticket URLs in convention docs | **Q5** ("none" is valid) |
| **Who creates worktrees** | `git worktree list` · does my agent tool take an `isolation`/`worktree` option · `ls -d .claude/worktrees .wt 2>/dev/null` · `grep -n worktree .gitignore` | **Q0** — 🔴 **ask this first** (§2-A) |
| Worktree paths | If the harness provides them, **observe only — do not create any** · if not, **set a convention** | §2-A |
| **Concurrency cap and gate duration** | *Estimate* from swap/CPU (`sysctl vm.swapusage`, `nproc` / `sysctl -n hw.ncpu`), then **correct with overload signals** (RUNBOOK §7) | **Q9** — machine-specific, start at **2** |
| **Overlap rules** (append-only files, frozen list) | grep convention docs (`AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING*`) for "append-only / frozen / read-only / generated" and shortlist | **Q6** |
| **Work-unit axis** | 🔴 **It falls out of the project's hard rules** — grep the convention docs for "every / all / exhaustive / entire". *(sample, n=1: because a rule said "wiring EVERY consumer", the axis was not "path" but "the set of consumers".)* | **Q7** |
| **Handoff-note location** | harness memory index (e.g. `~/.claude/projects/<slug>/memory/MEMORY.md`) > a one-line pointer in `CLAUDE.md`/`AGENTS.md` > `docs/` — **the criterion is "does it auto-load"** (§4) | **Q8** |
| Regeneration command for generated artifacts (if any) | scripts matching `gen\|codegen\|openapi\|proto\|schema` | "none" |
| Counting / audit tooling (if any) | scripts matching `count\|audit\|budget\|check` | "none" |
| Golden-master or reference clone (if any) | "legacy / golden / reference" in convention docs, plus sibling directories | 🔴 **If you do not know, ask — never guess** |

**The single detection sweep** (only what exists prints):
```
cat package.json 2>/dev/null | sed -n '/"scripts"/,/}/p'; ls Makefile pyproject.toml noxfile.py 2>/dev/null
ls .github/workflows .husky .pre-commit-config.yaml 2>/dev/null; ls playwright.config.* cypress.config.* 2>/dev/null
git remote -v; git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null; git rev-parse --abbrev-ref HEAD
grep -rniE "append-only|frozen|read-only|EVERY consumer|exhaustive" AGENTS.md CLAUDE.md CONTRIBUTING* 2>/dev/null | head -20
ls scripts/ 2>/dev/null | head -20; git check-ignore -v docs .claude 2>/dev/null
git worktree list; ls -d .claude/worktrees .wt 2>/dev/null; grep -nE "worktree|\.wt" .gitignore 2>/dev/null
sysctl vm.swapusage 2>/dev/null || free -h 2>/dev/null; (nproc 2>/dev/null || sysctl -n hw.ncpu)
```

### 🔴 2-A. *Who* creates the worktree — two branches (this decides the shape of cycle step ③)

| Environment | Worktrees | Record in the adapter |
| --- | --- | --- |
| The harness provides them (`isolation: worktree` or similar) | **Observe only — do not create any** | The observed path *pattern* (e.g. `.claude/worktrees/agent-<id>`) |
| The harness does not | A person or the orchestrator **creates them** | The path *convention* (e.g. `<repo>/.wt/<slug>`, or a sibling `../<repo>-<slug>`) |

Build-them-yourself branch:
```
git worktree add <path> -b <branch> <upstream tip>
<dependency install command>     # the SAME package manager the adapter's gates use
# gitignored local config (.env*.local and friends) must be copied from the main tree
```
- 🔴 **Create them *inside* the repo and the main tree's gates will scan them.** In the one
  observed case the sample was not contaminated by luck — **both exclusions already existed**:
  the linter's `files.includes` had `"!.claude"` and the test runner's `exclude` had
  `'**/.claude/**'` (its comment gave the reason: "in case agent worktrees appear inside the
  repo"). **In a project without those exclusions, an in-repo worktree makes the gate read
  someone else's half-edited files.**
  ✅ **One-line bootstrap check**: if worktrees will live inside the repo, verify **three places** —
  ① `.gitignore` ② linter exclusions ③ test-runner exclusions. If any one is missing, put them
  **outside the repo** (a sibling directory).
- ⚠ **`grep -rn` will match inside worktrees** — in-repo worktrees mean the orchestrator's searches
  scrape someone else's tree (observed). Narrow searches to your source folders.
- 🔴 **`git worktree remove` is not the end of cleanup** — language servers and dev servers holding
  that path outlive it. Treatment is in `RUNBOOK.md` §2, "when ⑦/⑧ get stuck"
  (🔴 **use `xargs -r kill`** — `kill $(...)` fails on the newlines).

## 3. The ten bootstrap questions — 🔴 **only the misses, all at once** (Q0 comes first)

For values you detected successfully, **do not ask — state "here is how I read it" and get it
confirmed** (turn economy — RUNBOOK §6-2 ④).

0. **Q0 worktree branch** — 🔴 **settle this first**: does this harness *hand you* isolated
   worktrees (§2-A)? The answer determines the shape of cycle step ③, so deferring it means
   rewriting step 0 of the brief.
1. **Q1 gates** — the commands that must pass before a commit, in order. Do any already run from a
   pre-commit hook?
2. **Q2 e2e** — the full-suite command and its port/server env vars. "None" is fine.
3. **Q3 upstream** — which branch harvested commits get pushed to (default branch or epic branch).
4. **Q4 PR** — how a PR is opened (or "we push straight to upstream, no PR").
5. **Q5 tickets** — the tool and commands for claiming work, filing requests, and closing. "None" is fine.
6. **Q6 overlap** — the **shared files** parallel tracks tend to touch at once (the append-only set)
   and the **frozen / do-not-edit** list.
7. **Q7 work-unit axis** — is there a hard rule of the form "this must be done all at once"? That
   rule is the floor on how finely you may split.
8. **Q8 handoff note** — is there already a place the next session reads **automatically** on
   "let's continue"? If so, do not create a new one.
9. **Q9 machine limits** — 🔴 **these are properties of the machine, not of the method, so the
   adapter owns them and the numbers in this skill are worth nothing to you.** Ask for, or measure:
   ⓐ **concurrency cap** — how many worktrees run at once before things degrade (**start at 2**;
   raise only after a clean round, and watch the three overload signals in RUNBOOK §7)
   ⓑ **gate duration** — wall-clock for the full gate and for e2e, which sets the harvest window
   ⓒ **swap/RAM headroom** — each worktree carries its own dependency tree, compiler, and test runner.
   ✅ Record all three **with the date you measured them**; they drift as the repo grows.

⚠ **Environment files (`.env*` and friends) are usually gitignored and do not follow a worktree** —
list "files to copy from the main tree" in the adapter. Without that list, a worktree once started
with no logged-in account.

## 4. Where the handoff note goes — the only criterion is "does it auto-load"

| Location | Auto-loads | Notes |
| --- | --- | --- |
| The harness's memory index | ✅ one index line loads every session | **Best** — put *"if the user says 'continue', start here"* in the index line |
| A one-line pointer or `@import` in `CLAUDE.md`/`AGENTS.md` | ✅ | Eats the always-loaded budget, so **one pointer line only** |
| Any file under `docs/` | ❌ | The user has to point at it every time |

## 5. Adapter skeleton

```markdown
# Parallel worktree brief — the shared part for this project
> First line of every brief: "Read <<path to this file>> first. Then this round's target is below."
> Generic procedure lives in the `parallel-worktree` skill (RUNBOOK). This file holds
> **only this repo's concrete values.**

## 0. Worktree alignment — <<expected base · fetch/switch · install deps · is the conventions folder present · gitignored files to copy>>
## 1. Gates — <<command order · measured duration · hook timeout · full-suite e2e rule · procedure for adjudicating a single failure>>
## 2. Commits — <<message format · when splitting, "every commit green on its own" · 🔴 do not push (the caller does)>>
## 3. Overlap — <<append-only table for shared files + frozen list + generated artifacts are read-only>>
## 4. Counting is a job for tools (hand counts are always wrong) — <<count/audit commands, or "none">>
## 5. Tickets and requests — <<claiming, replying, priority rules, or "none">>
## 6. Traps specific to this project — <<evidence as file:line + date measured. This is where self-improvement lands.>>
## 7. Report format — <<items ①–⑨ + 🔴 "full text to a file · 12 lines in chat">>
## 8. Machine limits (Q9) — <<concurrency cap · gate duration · measured on YYYY-MM-DD>>
```

## 6. 🔴 Adapters go stale too — stamp every snapshot value with the date you measured it

- **Counts like "N consumers" or "N remaining" must be recounted on the spot** — another round
  moving one of them turns the stated reason quietly false *(sample, n=1: a lock reason that was
  false for six days)*. **Mirror obligation**: whoever moves something runs
  `grep -rn "<old path>"` and fixes the lock and gap docs too.
- 🔴 **Hand counts are always wrong** — in the observed case the register said "36 remaining" when
  it was 37, and **splitting the remainder using a wrong register means that one surface enters no
  round at all.** If a counting tool exists, run it and put its number in the brief.
- 🔴 **"Blocked" and "that's the frozen layer's problem" are snapshots too** — grep the shared code
  for that prop or predicate *(sample, n=1: it had already been unblocked and sat in the queue as
  waiting for three days)*. If you unblock an axis, close that clause **in the same commit**.
- **A quoted `file:line` is a snapshot** — re-measure it when you pick it up, and **write the date
  next to it**.

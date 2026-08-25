# Contributing to parallel-worktree

Thanks for considering a contribution. `parallel-worktree` is a Claude Code skill — a set of
instructions a model reads, not an application that runs. There is no build step and no runtime:
everything here is Markdown plus two small JSON manifests. In practice that means **the text is the
product**, and a sloppy sentence is a bug that ships to every project that installs it.

## The most valuable contribution: a second data point

**Everything measured in this repo is n=1** — one machine, one repository, one operator. The cost
model (`∫ = c₀·n + g·n²/2`), the concurrency cap of 2, the 30–45 minute work unit, the 8–15 minute
gate: all of it is one sample, and it is labelled as such throughout.

If you run this pipeline and measure your own values, **open an issue with the numbers and the
method**. The measurement procedure is in
[`RUNBOOK.md` §6-0](../skills/parallel-worktree/references/RUNBOOK.md) — aggregate `message.usage`
from your session transcript, take c₀ from turn 1 and g from a linear regression over turn index.
Contradicting results are as useful as confirming ones; a claim that turns out to be
machine-specific should move into `ADAPTER-SPEC.md`'s question bank, not stay in the skill.

## Before you start

- Read [`README.md`](../README.md), then
  [`SKILL.md`](../skills/parallel-worktree/SKILL.md) — it is deliberately small, and the three
  `references/` files carry everything heavy.
- Check open issues and PRs first to avoid duplicate work.
- For anything non-trivial (a new section, a change to the bootstrap questions, a new failure mode),
  open an issue to discuss the approach before writing.

## The rules this repo holds itself to

These are not style preferences; violating them is what makes the skill stop being portable.

1. **No single project's conclusion goes in the skill.** The skill owns discriminators, measurement
   methods, and choice tables. Which cell a project picked belongs to that project's adapter. If you
   find yourself writing "we use X", it belongs in `ADAPTER-SPEC.md` as a question, not in `RUNBOOK.md`
   as an answer.
2. **Every number carries its provenance.** New measurements get `(sample, n=1)` — or a real sample
   size if you have one — plus what was measured and when. An unlabelled number reads as established
   fact, and that is the failure this repo is trying not to repeat.
3. **`SKILL.md` stays small.** It loads on every invocation. Heavy procedure goes in `references/`,
   which is read on the turn that needs it. If a section is only relevant while harvesting, it goes
   in RUNBOOK.
4. **One canonical language.** English is canonical — skill body, references, and `README.md`.
   `README.ko.md` is a mirror. Do not create a second copy of anything else in another language:
   two copies drift, and the drift is silent. (This repo learned that from a case where the same
   content was hand-copied nine times and one copy carried a wrong hash.)
5. **Never document a way around a safety or permission check.** See `RUNBOOK.md` §5 ⑥. A PR that
   adds one will be closed.

## Making a change

```bash
git clone https://github.com/sh5623/parallel-worktree.git
cd parallel-worktree
# edit Markdown; there is nothing to build
```

To try a change locally before opening a PR, point Claude Code at your clone:

```bash
/plugin marketplace add ./path/to/your/clone
/plugin install parallel-worktree@parallel-worktree
/reload-plugins
```

Then run an actual round with it. **A change that has not been used in a real round is not ready** —
this skill's whole premise is that the failure modes are the ones you do not notice.

Commit messages: a short imperative subject, and a body that says what changed and why. If the
change is driven by a measurement, put the number in the body.

## Scope

In scope: the pipeline (bootstrap, dispatch, harvest, resume, cleanup), its failure modes, the
orchestrator's context budget, and the adapter contract.

Out of scope: harness-specific automation (this skill branches on whether the harness provides
worktrees rather than assuming one), project-specific gate tooling, and anything that would require
the skill to know what stack you are on.

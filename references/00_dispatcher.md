# Substance First Refactoring — Technical Operational Dispatcher

**Framework Author**: William Fitzpatrick (*Writer Science*)  
**Source Lecture**: [A.I. Mistakes Writers Must Stop Making](https://www.youtube.com/watch?v=3kf9rRztJgA)  
**Parent Collection**: [Master Collection](../../kirby-fitzpatrick-writers-collection/SKILL.md) | [Global Help](../../../kirby-help/SKILL.md)  

---

## 1. Cognitive Foundation: Style and Substance Are Not Independent Variables

The lecture's central claim is not a taste argument. It is a structural one:

> **"As style is manipulated, the message is distorted; and no clear style can be derived from an ill-conceived idea."**

Writers who hand raw notes, transcripts, or drafts to an editor believing that *"the ideas are mine, the AI only supplies the voice"* are operating on a category error. They imagine **writing style to be a coat of paint** — a removable surface layer independent of the ideas underneath. Fitzpatrick's argument is that style is not a veneer applied to thought; it *is* the load-bearing expression of thought. Polish an incoherent idea and you get a *persuasive* incoherent idea, which is strictly worse than an obviously broken one.

The engineering translation is exact and unforgiving:

- **Style** = linters, formatters, docstrings, comment tone, variable naming, file layout, commit message polish.
- **Substance** = the executable behavior: what the code actually does under edge cases, concurrency, bad input, and load.
- **The coat of paint** = a "readability refactor" applied to code whose behavior nobody has verified.

A docstring is a claim about behavior. A rename is a claim about semantics. A lint pass is a claim about correctness of form. **Every cosmetic edit is a claim that the substance beneath it is already settled.** When the tests do not compile and do not pass, that claim is false — and you have just manufactured a *more* credible lie. This is the lecture's "spirit of your writing" point relocated to source control: the spirit of a codebase is not what its docstrings assert, it is what it demonstrably does.

### The Two AI Workflows, Translated to Code

Fitzpatrick examines two workflows and shows that the second *feels* more responsible but is *just as pernicious*. Both have direct engineering twins:

| Lecture workflow | Engineering equivalent | Why it fails |
|---|---|---|
| **LLM output → human editor** | AI-generated code → human cosmetic cleanup (lint, rename, docstring) | The logic was never grounded; the cleanup lends it false authority |
| **Human ideas → AI editor** | Engineer's own spike / notebook / half-working patch → "clean rewrite" by an agent | Feels like responsible stewardship. Is an unverified behavior being restyled before it is understood |

Both produce the same artifact: **polished code that has never been proven to work.** The unscripted demonstration in the lecture — the video was *filmed unscripted* precisely to show that spirit outperforms craft — has an engineering analogue every senior engineer recognizes: the ugly spike that actually solved the problem is worth more than the beautiful abstraction that never ran.

### The Failure Visualized

```text
BEFORE — COAT-OF-PAINT REFACTOR (Substance Unverified)

  ┌──────────────────────────────────────────────────────────┐
  │  branch: feature/billing-rework                          │
  ├──────────────────────────────────────────────────────────┤
  │  SUBSTANCE LAYER (truth)                                 │
  │    pytest -q  →  3 failed, 2 skipped, 41 passed   [RED]  │
  │    build      →  compiles, but one fixture never runs    │
  ├──────────────────────────────────────────────────────────┤
  │  STYLE LAYER (the coat of paint)         412 lines diff  │
  │    + ruff/flake8/eslint .............. 0 issues  ✓       │
  │    + 96 docstrings added ("Returns the invoice.")        │
  │    + 37 renames: data→payload, util→helper, do→handle    │
  │    + formatting normalized across 9 files                │
  └──────────────────────────────────────────────────────────┘

  Delivered claim : "Refactor for readability and maintainability."
  Reviewer reads  : clean, professional, obviously reviewed.
  Reality         : the message has been distorted, not clarified.
```

The pathology is not that the diff is wrong line-by-line. It is that **the polish is now a proof of quality in the reviewer's mind**, consuming the reviewer's attention budget that the actual defect needed. The rename `handle_invoice` → `process_invoice` reads as a semantic clarification. It is a guess. The docstring asserts a return contract that no test pins. The formatter made a red diff look green.

```text
AFTER — SUBSTANCE-FIRST GATE (Style Locked Until Proven)

  ┌──────────────────────────────────────────────────────────┐
  │  LAYER 3 · STYLE      linters · docstrings · naming      │ ← LOCKED
  ├──────────────────────────────────────────────────────────┤
  │  LAYER 2 · STRUCTURE  modules · interfaces · boundaries  │ ← LOCKED
  ├──────────────────────────────────────────────────────────┤
  │  LAYER 1 · SUBSTANCE  behavior · invariants · tests      │ ← FIX HERE
  └──────────────────────────────────────────────────────────┘

  RULE: Layer N+1 may not be edited until Layer N is VERIFIED.
```

### The Gate Ladder

```text
GATE 0  REPRODUCE  a named test fails for the stated reason, output captured
GATE 1  EXECUTE    the suite COMPILES and RUNS end-to-end (no import errors)
GATE 2  PASS       suite is GREEN — no masking via skip, xfail, or -k filtering
GATE 3  RECEIPT    raw command + raw output + exit code, pasted, not summarized
GATE 4  UNLOCK     lint · docstrings · renames — permitted, in a SEPARATE commit
```

**Gate 4 is the only door to cosmetic work.** If any gate below it is red, closed, or unproven, "readability refactoring" is not a smaller task than fixing the bug — it is a different lie.

---

## 2. Core Transformation Protocols

### Rule 1: The Substance-First Gate Is a Precondition, Not a Preference

Before any lint config is touched, any docstring added, or any identifier renamed, the following must be literally true and demonstrable *in this session*:

1. The project **builds / imports / compiles** with no errors.
2. The relevant test suite **executes** (not "would execute if…").
3. The suite **passes** without masking mechanisms (`skip`, `xfail`, `-k` narrowing, disabled CI, commented-out assertions).
4. The raw evidence is available to paste — not to paraphrase.

If the code is red, your task is **fix the behavior**. Cosmetic work on red code is not scope creep; it is fabrication.

### Rule 2: Name the Defect Before You Name Anything Else

You may not rename a symbol until you can state, in one sentence, what that symbol *does* — and that sentence is backed by an assertion that ran. A rename is a claim about semantics; unverified semantics produce **names that lie**, which is the code equivalent of distorting the message while polishing the sentence.

### Rule 3: Split the Commit — Cosmetic Diffs Never Travel With Semantic Diffs

A single commit may contain either a **behavioral change** or a **cosmetic change**. Never both. If they are interleaved, the reviewer cannot distinguish the fix from the paint, and `git bisect` becomes useless. Enforce with:

```bash
git diff -w --stat          # whitespace-only? then it is cosmetic-only
git diff --color-moved      # rename/reorder detection
git log --follow -p <path>  # did behavior and style move together?
```

A cosmetic commit whose `git diff -w` is empty is provably style-only and therefore provably safe. That proof is the *entire* value of separating it.

### Rule 4: Refuse the Inverted Workflow

The "Human ideas → AI editor" inversion in engineering is: *take my messy working patch and make it clean and idiomatic.* It feels responsible. It is the same disease. **Clean does not mean correct, and idiomatic does not mean tested.** If a rewrite is justified at all, it happens **after** the messy version is pinned by tests — so the rewrite can be proven behavior-preserving rather than merely believed to be.

### Rule 5: Style Cannot Be Derived From an Unverified Idea

Fitzpatrick: *no clear style can be derived from an ill-conceived idea.* Do not ask "what should this module be named?" about a module whose responsibility is undefined. Do not document a function whose contract is unknown. **Naming and documentation are the last mile of understanding, never the first.** You earn the right to name things by proving what they do.

### Rule 6: No Polishing the Signal That Hides the Signal

Cosmetic cleanups that reduce the reviewer's ability to see the defect — formatter passes across the whole tree, mass docstring insertion, repository-wide renames — are **negative** contributions inside a red branch. They inflate the diff, obscure the blast radius, and borrow credibility the code has not earned.

### Anti-Pattern → Clean Replacement Ledger

| Anti-Pattern (Coat of Paint) | Clean Replacement (Substance First) |
|---|---|
| `refactor: improve readability` submitted while `pytest` is red | `fix: <named defect>` first; the cosmetic PR is **blocked** until green |
| Renaming `data` → `payload` while the field's semantics are undefined | Write the assertion that defines the semantics; rename afterwards, separately |
| Docstring asserting a return contract no test pins | Add the test that pins the contract, then document it truthfully |
| `pytest -k green_subset` used as "the suite passes" | Full suite, or an explicit `xfail` with a linked issue — never silent narrowing |
| 412-line diff mixing one logic fix with 9 files of formatting | Two commits: `fix:` (semantic) then `style:` (`git diff -w` empty) |
| "Tests pass locally" as the review evidence | Paste the raw command, raw tail, and exit code; cite the green CI SHA |
| Deleting or skipping the failing test to reach green | Fix the behavior, or quarantine with the defect named and tracked |
| Rewriting the whole file "for consistency" to avoid understanding it | Minimal semantic patch → verify → *then* an optional isolated cosmetic pass |
| Polished prose in a PR body describing behavior that never ran | Substance Evidence block: what was executed, what it returned, what it proved |
| A beautifully formatted ADR for a design that was never spiked | Run the spike, record the result, write the ADR from evidence |

### The STOP Signals

If any of these appear while you are about to run a formatter, add a docstring, or rename a symbol — **stop, the gate is closed**:

```text
✗  build/import error anywhere in the repo
✗  failing test the author has not explained
✗  skipped / xfailed tests used as "green"
✗  CI red, disabled, or "we'll fix it after the cleanup"
✗  no receipt — only the claim "it passes"
✗  the fix and the formatting are in the same diff
```

---

## 3. Engineering Application Scenarios

### 3.1 Code Reviews — Rejecting the Coat of Paint Without Rejecting the Author

The reviewer's job is not aesthetic approval; it is **substance verification**. The first question is never "is this clean?" — it is *"what proves this works, and can I see it?"*

**Reviewer audit sequence:**

```text
1. Does the PR contain a raw, executed test receipt?      (Gate 3)
2. If yes — is the suite green WITHOUT masking?           (Gate 2)
3. If no  — every aesthetic comment is deferred. Stop.
4. Is the cosmetic diff separable from the semantic diff? (Rule 3)
5. Only then: naming, docstrings, style, structure.
```

**Review comment reframes:**

| Coat-of-Paint Comment | Substance-First Comment |
|---|---|
| *"Nice cleanup, but let's rename `util.py`."* | *"`util.py` is doing three jobs. Until the tests for the third job exist, a rename would be a guess — let's name it after we can prove the boundary."* |
| *"LGTM, very clean diff."* | *"The diff is clean and 412 lines long, and the suite is red. Which output did you run to produce this? I can't review style on unverified behavior."* |
| *"Could you add docstrings here?"* | *"This function's contract changes when `retries=0`. Add the assertion first, then the docstring can state it truthfully."* |
| *"Please fix the lint errors."* | *"Lint is fine to fix — in a separate commit. Right now it's mixed into a behavior change and I can't tell what moved."* |

**The blocking verdict, stated plainly:**

> "I'm not approving cosmetic changes on unverified behavior. This diff adds 96 docstrings and 37 renames on top of a red suite — that makes the code *more* convincing, not more correct. Split it: first the semantic fix with a green test receipt, then the style pass as its own commit. I'll review the second one in a minute."

This is not pedantry. The lecture's point is that style manipulation **distorts the message** — here, it distorts the review's signal path.

### 3.2 PR Descriptions — The Substance Evidence Block

Never open a PR description with polish language ("cleaned up", "improved readability", "modernized"). Open with **executed evidence**. Copy this template verbatim:

````markdown
## What changed (substance)
Fixes: <the named defect, in one sentence>

## Evidence (raw, this branch)
```
$ <build/compile command>
<raw output tail>
exit: 0

$ <test command>
<raw output tail>
exit: 0
```
Commit SHA: <sha>  · CI: <link, green>

## Proof of behavior preservation
- Before: <assertion that failed / invariant that was violated>
- After : <assertion that now passes>
- Regression coverage added: <test names>

## Cosmetic pass (separate commit, `git diff -w` empty)
- Commit: <sha>
- Contents: formatting, docstrings, renames — **behavior-identical by construction**
- Reviewer may approve this commit on diff inspection alone.
````

**The law of the description:** if the "Evidence" block is absent, the PR is not a refactor — it is a coat of paint. If the cosmetic pass is in the *same* commit as the fix, "separate commit" is a lie and the template must fail review.

**Description anti-patterns:**

| Slop | Substance-First Fix |
|---|---|
| *"General cleanup and readability improvements across the billing module."* | *"Fixes the double-charge when `retries > 0`. Receipt below. Formatting is in a separate commit."* |
| *"Tests should still pass."* | *"`pytest -q` → 46 passed. Raw output below. SHA `a91f3c2`."* |
| *"Refactored for clarity."* | *"`InvoiceLine.total` was returning cents; it now returns Decimal dollars. Assertion `test_total_units` added."* |
| *"Minor formatting fixes."* (287 lines) | *"Formatting only — `git diff -w` is empty, so no behavior is in this commit."* |

### 3.3 Architecture RFCs / ADRs — You Cannot Write the Style Before You Have the Substance

This is the deepest form of the lecture's error, scaled to a system. Writing a polished ADR is exactly the "premature ideas → AI editor" workflow: you take a half-formed intuition, run it through a prose-polishing pass, and ship a document that reads like a decision while containing an unvalidated design. The prose *becomes* the evidence. Nobody spikes the idea, because the document already looks decided.

**The ADR gate:**

```text
DRAFT INTUITION
      │
      ├─ ✗ prose pass now  ──►  POLISHED ADR OVER A GUESS      (the mistake)
      │
      └─ ✓ run the spike, measure, record the failure
                       │
                       ▼
                 EVIDENCE (numbers, traces, invariants)
                       │
                       ▼
                 ADR — written from substance, styled last
```

**ADR structure that enforces the gate:**

```markdown
# ADR-014: Replace synchronous RPC fan-out in the ingest path

## Status
Accepted (validated by spike — see Evidence)

## Context (substance)
Invariant violated: p99 ingest latency ≤ 250 ms.
Observed: p99 = 1.8 s at 3k rps. Trace: <link>.
No docstring, comment, or naming convention changes this number.

## Decision
Adopt an async event log between ingest and enrichment.

## Evidence (raw, reproducible)
```
$ python -m bench.ingest --rps 3000 --duration 300
p50  41ms   p95 180ms   p99 244ms
exit: 0
```

## Rejected alternatives — and *why they were actually rejected*
| Alternative | Rejected because | Evidence |
|---|---|---|
| Bigger synchronous pool | p99 unmoved at 1.4 s | `bench/spike_pool.md` |
| Rename/doc the existing RPC layer | **Did not address the invariant.** Cosmetic. | — |

## Consequences
Ingest is now eventually consistent within 50 ms. Documented at <link>.
```

**The ADR anti-pattern, stated as a rule:**

> An ADR whose "Evidence" section contains prose rather than an executed command is not an ADR. It is a mood board.

And the corollary from *Rule 5*: **do not debate naming, module boundaries, or documentation conventions in an RFC until the substantive question — does this work, and how do we know? — has a receipt.** Style arguments about an unvalidated design are arguments about paint on an unbuilt wall.

---

## 4. Verification Checklist

- [ ] **Tests compile and pass before any cosmetic edit.** The suite was executed end-to-end in this session — no import errors, no collection errors — and is green with no `skip`/`xfail`/`-k` masking.
- [ ] **A raw receipt exists, not a claim.** The exact command, its output tail, and its exit code are pasted in the PR description or session log; "tests pass locally" appears nowhere.
- [ ] **Semantic and cosmetic changes are in separate commits.** `git diff -w` is empty on the style commit, proving zero behavioral content, and the `fix:` commit stands alone with its own evidence.
- [ ] **No identifier was renamed, no docstring written, and no linter run before the behavior it describes was proven.** Every name and every documented contract is backed by an assertion that executed.
- [ ] **No polish was applied anywhere the substance is still red.** No repository-wide formatter pass, no mass docstring insertion, and no "readability" diff exists on a branch with a failing test, a red CI run, or an unexplained skip.
- [ ] **Nothing in the diff was made *more* convincing than it is true.** If a reviewer's first impression of quality came from formatting rather than from executed behavior, the gate was violated and the cosmetic layer must be reverted to Gate 0.
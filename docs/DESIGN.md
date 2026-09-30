# Design: every ruling, organ by organ

This is the consolidated design of engram as it stands at the end of the exploration phase. It is the
document the build implements from. `docs/BACKGROUND.md` says where each piece came from and why the
project is assembly; this file says what was decided, what was measured, what is only a starting
value, and what is still open.

Every statement below carries one of four statuses:

- **[ruled]** decided by the operator or ratified by the founding session. Not reopened by a build
  session; a build session that finds a ruling unworkable stops and says so.
- **[measured]** a number observed on the operator's own session transcripts or in a trial run. It
  describes one operator's corpus and is evidence, not a constant.
- **[default]** a starting value with no claim to being right. It ships as a named parameter and is
  the first thing the benchmark is allowed to move.
- **[open]** not decided. Nothing in the build may depend on a particular answer.

The evidence behind each section lives in research notes that are not part of this repository. They
are cited by file name only. Model identifiers, launch flags and harness field names are in those
notes, not here (see "Harness adapter" for where the field table will live once the build starts).

## 1. The loop

```
 harness transcript (lossless, never modified)
        |
        |  structural pass, no model                     [Provenance]
        v
 in-session provenance graph: prompts, plan and report text, tool calls, hunks,
 commits, interrupts, hand edits, diagnostics, subagents, compaction boundaries
        |
        |  at the next idle gap, per ended session        [Sleep]
        v
 skeleton render --> ONE model call --> JSON: goals, verdicts, candidate engrams, dropped
        |
        |  code validates by exact match, resolves scopes, computes increments,
        |  merges or supersedes, writes                   [Grain, Valence, Time]
        v
 engram files + index + provenance store + diff log + preimage
        |
        |  session start: one-line index; on cue: an assembly  [Assembly recall]
        v
 the agent reads engrams carrying the user's words; what it does with them is
 detected structurally and feeds strength, staleness and reconsolidation
                                                          [Strength, Reconsolidation]
```

Three properties hold across the whole loop:

- The transcript is the only source of truth. An engram that cannot be traced to transcript lines is
  a hallucination and is not written. **[ruled]**
- A model proposes; code writes. No model ever writes a file, sees the store's format, or holds a
  tool. **[ruled]**
- Nothing is deleted. Engrams are superseded, demoted to a cold tier, or hidden from the default read
  path; the span they came from stays one hop away. **[ruled]**

## 2. Invariants

These seven are the invariants of the design. Each is enforced by code, not by a prompt. **[ruled]**

1. **An increment must name the spans that earned it, or it does not happen.** Every span resolves to
   a turn or a hunk range in a transcript. A user verdict may name many spans; a corroboration names
   exactly the hunks it concerns.
2. **Recall is never an outcome.** No valence increment is applied because an engram was recalled in
   a session that went well or badly. Session-level reward smeared over everything retrieved is the
   memory-reward trap.
3. **Only the user creates sign.** Nothing but a user verdict writes the valence pair. Corroboration
   (a revert, a regression, a hand edit) adds confidence to an increment the user created. Survival
   changes an engram's state and so its decay. Neither writes the pair. A judge never writes a sign.
   The agent's own claim of success weighs zero. Silence is unlabelled.
4. **Words are stored verbatim.** The user's sentence is the record. Any magnitude or scale is
   derived from it later and never replaces it.
5. **Origin is a multiplier, never a sign.** How much of a scope was self-initiated scales the weight
   of the user's reaction to it. It cannot create, flip or suppress an increment. A self-initiated
   scope the user never reacted to stays unlabelled.
6. **Every injected copy of an engram carries its stamped id, and injection bumps nothing.** Exposure
   is an exact field match, not a text match, and it is a trial, not a use.
7. **The consolidation call holds zero tools.** A skeleton goes in and JSON comes out. The launcher
   proves the empty context before the call and refuses to run otherwise; after the call it accepts a
   result only on the run's error flag and stop reason. This is a security property first and a cost
   property second: the call reads text that other agents, tool results and pasted content wrote, and
   a tool in context is a tool that text could cause to be called.

The six design constraints in `docs/BACKGROUND.md` (local first, transcript is truth, engrams replace
episodes, assemblies with valence, everything inspectable, no cloud dependency) stand alongside these.

## 3. Substrate and ingest

- **Input.** Ingest reads the harness's own session transcripts. Nothing else writes episodes.
  **[ruled]**
- **Two populations.** On the measured corpus, nine in ten session files are programmatic one-shot
  runs (a single generated prompt, no human), most likely another memory tool's worker. A test on
  prompt text does not separate them; the harness's entrypoint field does. **[measured]**
  (`2026-09-26-sleep-corpus-measurement.md`)
- **Eligibility.** A session is a consolidation unit when its entrypoint is interactive (not a
  programmatic or SDK one) AND it has either three or more human prompts, or at least one human
  prompt together with a file edit or a commit. The rule exists to skip trivial sessions, not short
  ones: a one-prompt session that shipped a commit is real work. A human prompt is a user record that
  is not harness metadata, not a tool result, and not harness-generated markup. **[ruled]**
- **What never qualifies.** Programmatic runs and subagent transcripts are provenance, never units.
  engram's own consolidation runs are programmatic, so the eligibility filter keeps them out of their
  own input by construction. **[ruled]**
- **Volume.** Nine to thirty-four eligible sessions a week, median about seventeen; median nine
  prompts and about eighty tool calls each. **[measured]**
- **Ephemeral material is copied early.** The harness's per-turn file checkpoints and the subagent
  files are copied into the provenance store at session end and at compaction, because the harness
  sweeps them. Waiting for the sleep pass loses them. **[ruled]**

### Backfill

The transcript corpus reaches back months and the pass is idempotent, so the store is built by
running the real pass over eligible past sessions, oldest first, in idle time, at the same cost per
session as a new one. Prior memory tools' stores and the agent's own memory directory are never
imported as engrams: they have no transcript provenance, and invariant 1 forbids an increment without
spans. They may be read by a human, not by the pass. **[ruled]** How the backlog is ordered against
newly ended sessions within one idle gap is **[open]**.

### Harness adapter

The structural pass depends on transcript fields the harness does not document and changes between
builds. One load-bearing field disappeared between two builds with no changelog entry during the
exploration phase, and a separate memory tool on the same machine once broke silently for a day when
a build moved fields into a nested object. **[measured]** The contract is therefore:

- Detect the transcript shape per file from its build version. **[ruled]**
- Verify each load-bearing field on the first record that should carry it. A missing or reshaped
  load-bearing field stops ingest for that file, loudly, naming the field. **[ruled]**
- Never fall back to a cached or defaulted value. Where a documented fallback exists (for example
  deriving a commit hash from the command's output when the typed field is absent), the derived value
  is marked as derived. **[ruled]**
- A field that reappears under a new name or nesting is a new row in the adapter table, never a
  silent alias. **[ruled]**
- The harness's documented hook payloads say *which* file and turn to read; the file says *what*
  happened. **[ruled]**

This document describes the adapter generically. Transcript field names and harness environment
variable names are the harness's public format, not the operator's data, so when the build starts the
adapter table moves into the public tree as its own document, `docs/adapters/<harness>.md`, and this
section links it. Model identifiers, endpoints, home paths and the operator's other projects stay out
of the tree permanently. **[ruled]** Until then the table is in
`2026-09-26-claude-code-transcript-schema.md`. Adapters for other harnesses are **[open]**.

## 4. Provenance

One structural pass over the transcript builds the in-session graph with no model. **[ruled]**
(`2026-09-26-provenance-graph.md`)

- **Nodes** (fifteen types): session, user prompt (with origin: typed, queued, or from a peer
  session), assistant text (plan before the turn's first tool call, report after its last, or
  interstitial), tool call, tool result, hunk, file state, commit, push, diagnostic, hand edit,
  interrupt, subagent, compaction boundary, context file. Every node is addressable by a stable
  transcript id and timestamped.
- **Edges** (eighteen types), all read from fields or derived by id match and time order: turn
  membership, plan-issued-call, call-returned-result, call-produced-hunk, hunk-touches-path,
  had-seen (an earlier read of the same path), in-context, committed-in, broke (a diagnostic on the
  hunk's range), hand-edited-after, interrupted, spawned, reported, compacted-over, checkpointed,
  hook verdict, cross-session message, and the parent pointer itself.
- **Compaction is not an episode boundary.** The boundary record keeps the link across it, so the
  episode stays whole. **[ruled]**

### Where a model is used

Two things are not in the file: which goal each action served, and whether that goal was the user's.
A model is used for exactly these and nothing else in provenance. **[ruled]**

- **Goal segmentation.** Given a turn's plan text and its tool calls, group the calls under goals. A
  turn is split into more than one goal when the work exceeded the request.
- **The goal boundary, cite-or-NONE.** For each goal the model quotes the instruction it traces to
  (a user prompt, or a loaded instruction file), or answers NONE. A quote is checked by exact string
  match against the transcript, so the answer is auditable without trusting the model.

Both happen inside the single sleep-pass call (section 5), not at ingest.

Rules for the boundary **[ruled]**:

- **Origin has two values: `asked` and `self-initiated`.** The field carries no sign, and the names
  carry none either.
- **Self-initiated means work that no instruction covers in letter or in evident purpose.** Serving
  the evident purpose of the request is `asked`. Two canonical cases of asked-by-purpose: a
  protection added when the request was an alert about the same condition (the alert exists so that
  something is done), and documentation written for the audience the request named. A stale
  docstring fixed in an unrelated module is self-initiated.
- **A bundled fix outside the request's letter and purpose is self-initiated, however small; one that
  completes the purpose is asked.** The prompt states this in words, with the canonical cases,
  because careful readers split on it: some apply the letter of a request and some its purpose. The
  operator's reading is purpose, which is what "a compressed instruction gets the reading the user
  meant" already said. Because origin is a multiplier, the label punishes nothing; it makes the
  user's reaction to self-initiated work count more in both directions.
- The boundary is drawn at the goal, never at the file. On 807 measured edits, 30% touched a file the
  user had named, 53% a file the agent found while doing asked-for work, 17% a file never seen
  before. A per-file rule would call about 70% of ordinary asked-for work self-initiated.
  **[measured]** Any future shortcut must pass the same test.
- A goal's full downstream is asked: the same bug fixed in sibling places, the orphans a requested
  deletion leaves, the root cause of the requested bug. A compressed instruction gets the reading the
  user meant.
- Decisions inside an asked goal are asked. A misread instruction is still asked; the correction is
  valence's business, not the boundary's.
- Standing instructions count as instructions: instruction files, handoffs, pre-authorisations.
- A peer session's *instruction* is a request and may be cited. A peer's report, idle notice or
  delivery notice is not an instruction; code pre-marks these in the render. A peer's reaction is
  never a verdict.
- A question requests its answer. A bare paste addressed to the agent is a request, and the paste is
  the cite.
- The model may read reasoning blocks when the harness provides them and must never depend on them.
  They have the standing of corroboration, never of the plan text.

What has been measured, and what it is worth. Every origin label collected so far was made by a model
rater, under an earlier and narrower wording of the bundled-fix rule that applied the letter of the
request. Raters from one model family agreed with each other (kappa 0.61 and 0.71). A rater from
another family agreed with them far less (kappa 0.24 to 0.40) while putting the overall rate in the
same place, three to five percent of tool-using turns. The difference was where the line sits: letter
against purpose. These labels are model opinion. They have no standing as ground truth (section 14),
and the rate under the operator's purpose reading has not been measured. **[measured]**, **[open]**
(`2026-09-26-scope-expansion-flags.md`, `2026-09-30-rater-46-rerun.md`)

### Scope-expansion markers

Measured, and dropped. Agents narrate side work in first-person asides ("I also fixed", "while I was
in there", "bonus fix"), which suggested a cheap lexical prefilter for the boundary question. A phrase
set written after reading turns an agent session had labelled scored 45% precision and 80% recall on
the sessions it was fitted on. On sessions it was not fitted on it scored 8% and 8%, and no single
family of phrases held up: the phrases turned out to be how agents talk about everything, not a
register reserved for side work. An earlier list written before any labelling scored about the same
in and out of sample (and was useless both times), which is what an overfit looks like from the other
side. The base rate of self-initiated goals, as the raters drew the line, did replicate: about one
tool-using turn in twenty. The verdict on the markers does not depend on where that line is drawn:
they fail under every label set collected. **[measured]**
(`2026-09-26-scope-expansion-flags.md`, `2026-09-30-scope-flags-out-of-sample.md`)

- **No marker list is used in any role**: not as a gate, not as a prioritiser, not as the source of a
  citation. The consolidation call answers cite-or-NONE for every turn it renders, so nothing is lost.
  **[ruled]**
- **A NONE answer carries the agent's own announcing span**, checked by exact match like any cite.
  Self-initiated work was announced in the agent's text in every labelled case out of sample, in open
  vocabulary. If no announcing span exists, the goal is tagged `unannounced`: work nobody asked for
  that the agent did not mention, a distinct and worse class that gets first scrutiny in a verdict
  window. Both forms are recorded. Neither is a sign. **[ruled]**
- The per-session share of self-initiated goals is reported as a diagnostic: it is an
  operator-specific baseline, and a session far above it is worth surfacing. **[default]**
- Two structural stand-ins were also measured and rejected: "the user never named this file", and
  "edits to paths never touched before, on a short follow-up prompt" (17% precision, 6% recall).
  Self-initiated work lands in the same files and the same commit as asked-for work. **[measured]**

### Subagents

A subagent transcript attaches as a child of the tool call that spawned it. It is provenance, never
an engram source on its own: the user never addressed the subagent, so no verdict originates there;
verdicts land on the parent's turn and inherit down the spawn edge like any other downstream action.
A subagent's findings reach durable memory only through what the parent did with its report.
**[ruled]**

### Hunks

- **Primary source.** The per-turn file checkpoints plus a detector for writes made through the shell
  (redirected heredocs, in-place stream edits, inline patch scripts, commits). In shell-first setups
  most edits never pass through the harness's edit tools: on the measured self-initiated turns, 21 of
  23 used the shell and only 12 used an edit tool. **[ruled]**, **[measured]**
- **Fast path.** The harness's own per-edit patch, when the edit went through an edit tool.
  **[ruled]**
- **Hunk to commit to revert.** A deterministic recipe: normalise a hunk to added and removed line
  sets with exact, whitespace-normalised and token-normalised fingerprints; landing is the first
  commit whose diff contains the hunk's added set (a subset test, so squashes survive); survival by
  blame at later heads; the killing commit classified in a fixed order. Each hunk carries a status
  (never committed, open, alive, moved, format-only, modified, replaced, explicit revert, exact
  inverse, partial revert, file deleted, restored, undone in-session) together with the rule that
  fired and the lines that matched, so the status is auditable without a model.
  (`2026-09-26-hunk-to-commit.md`, which also has the failure-mode table.) The recipe is written, not
  yet run against real repositories: its accuracy is **[open]**.
- These statuses are corroboration and staleness evidence. They never create a sign (invariant 3).

## 5. Sleep: the consolidation pass

The pass is the only writer of engrams. It is one model call per ended eligible session, wrapped in
deterministic code on both sides. **[ruled]** (`2026-09-26-sleep-pass.md`)

### Trigger

- A session becomes a candidate when it is eligible and has had no record for 30 minutes, or its
  session-end event fired. **[ruled]**
- Idle is read from record timestamps, not from process liveness: sessions stay open for days. The
  pass runs when the union of active segments across all sessions on the machine has been quiet for
  30 minutes and at least one candidate exists, and at the latest in the operator's measured quiet
  window. **[ruled]**
- It takes a batch, oldest first, one call each, one at a time. More than half of the measured
  sessions overlap another, so "the previous session" is not a unit. **[ruled]**, **[measured]**
- Compaction is not a trigger, and neither is a clock or a turn count. Forced consolidation is what
  the literature finds harmful. **[ruled]**
- Idempotent: a consolidation record is keyed by session, file hash and prompt version; the same
  triple is skipped. A new prompt version may re-consolidate, and re-consolidation re-reads the
  transcript. An engram is never derived from another engram. **[ruled]**

### What the model reads: the skeleton

Code renders the provenance graph to text: each user prompt verbatim, the agent's plan and report
text, one line per tool call with hunk counts and commit hashes, interrupts, hand edits, compaction
markers. Every line a model might cite carries a stable anchor. Length caps apply per prompt and per
text block. **[ruled]**

- **The home directory prefix is rewritten to `~` by code** before the model reads the skeleton.
  Keep-verbatim then applies to the rewritten form, so the exact-match check stays honest and no
  engram carries the operator's home path or username. It also removes a class of validator failure
  seen in the trials, where the model shortened the path itself. **[ruled]**
- **Tool results are not in the skeleton.** They are most of a transcript's bytes, and no model reads
  them twice. The skeleton is one to two percent of the file. **[ruled]**, **[measured]**
- **Injected engram copies are stripped first**, matched by stamped id, so the pass cannot re-learn
  its own output. **[ruled]**
- **Candidate verdict turns are pre-marked by code** (see Valence, verdict detection), as are peer
  reports and notices. **[ruled]**
- **The skeleton is scanned for secrets before it leaves the machine, and a hit is redacted, never
  withheld.** Skeletons were measured carrying live credentials verbatim from the agent's own report
  lines. **[measured]** Each hit is replaced by a typed placeholder (for example `<secret:api-key>`).
  The placeholder is a literal string, so keep-verbatim and exact match still work, and the
  consolidation record notes how many spans were redacted. Withholding would lose a whole session's
  rulings because the agent echoed a credential in one report line. **[ruled]**
- Skeleton size on 572 eligible sessions: median 24k characters, 95th percentile 97k, maximum 409k.
  **[measured]**

### What the model returns

One JSON object, checked against a schema before anything else (a trial run dropped a required field
from every verdict; the schema check is a validator step, not a nicety). **[ruled]**

- `goals[]`: turn, goal, origin (asked, asked via a peer instruction, self-initiated), and the cite
  or NONE. A NONE carries the agent's announcing span, or the tag `unannounced`. The prompt states
  the letter-or-purpose rule and its canonical cases in words, and forces a cite per goal instead of
  asking for one label per turn, because readers left to a holistic judgement were measured
  splitting between the letter and the purpose of a request. **[measured]**
  (`2026-09-30-rater-46-rerun.md`)
- `verdicts[]`: for each candidate turn code marked, a classification (confirm, reject, none), the
  user's words copied verbatim, and the root: the decision being reacted to. Two verdicts with
  different roots are allowed on one turn.
- `engrams[]`: kind, anchors, a body of at most four lines, a `verbatim` list, provenance anchors, a
  supersession hint.
- `dropped[]`: what it chose not to keep, one line each. This is the audit trail for the lossy half of
  lossy-but-safe.

### What code does with it

1. **Validate, no model. [ruled]**
   - Every provenance anchor exists in the skeleton.
   - Every `verbatim` string is an exact substring of the skeleton.
   - Every cite and every verdict's words are an exact substring of a user line; every announcing
     span on a NONE is an exact substring of the agent's text in that turn.
   - Body at most four lines of at most 160 characters; the first line contains an anchor.
   - Secret scan on body and verbatim list; a hit rejects. The operator's own identifiers (account
     email, username, home path) are on the scrub list as a backstop, so an engram never carries them
     even if the model echoes them from its prompt. **[ruled]**
   - Grounding class for each verbatim item against the cited turns' raw records: grounded in a tool
     result, in the user's words, in the agent's text only, or nowhere. "Nowhere" fails. A fact whose
     numbers rest only on the agent's own report is stored as `claimed`.
2. **One repair call.** A failed engram goes back once with the validator's message. If it fails
   again it is dropped and the reason logged. Code never patches model output: the fix is the
   model's, the veto is code's. **[ruled]** The repair loop is designed and not yet measured.
   **[open]**
3. **Resolve scopes and compute increments** (section 7). Code only. **[ruled]**
4. **Merge into the store** (section 9).
5. **Write**: engram files, index rows, the consolidation record (skeleton hash, model, prompt
   version, launch shape, tokens, redacted-span count, validator log, dropped list). Before any
   write, snapshot the store and write a diff log naming every engram created, superseded or merged,
   with the reason.
   **[ruled]**
6. **Loss bound.** Refuse the whole batch if it would mark superseded more than a quarter of the
   engrams it touched. A prompt-version bug looks exactly like that. **[ruled]** (the fraction is
   **[default]**)

### What is kept verbatim

Enforced by the exact-match check, not by asking: the user's words that carry a verdict or a ruling;
numbers and units; identifiers, paths, flags and command names; commit hashes and addresses; error
strings (the distinguishing line, not the dump). Also structural and never droppable: supersession
status as a field rather than prose, the provenance pointer, and the concrete fix (file, symbol,
command, value). **[ruled]**

What the validator is for, measured: across two models the failure is always tidying and never
fabrication. A home directory shortened, a thousands separator dropped, a dash range normalised, an
approximation mark added, two quotes fused into one. Naming the moves in the prompt took the rate to
0.9% of items on twenty sessions; it did not close the class. The prompt is a first filter and the
exact-match check plus one repair call is the enforcement. **[measured]**
(`2026-09-26-dreamer-46-trial.md`)

What is summarised: the decision and its reason, the procedure, the state left behind. What is
dropped and named in `dropped`: mechanics, duplicates, intermediates a later total superseded. The
agent's success claims are dropped as valence and kept as facts only with `claimed` grounding. Peer
messages are never engram sources: their content reaches the store only through what the session did
with it. **[ruled]**

The honest limit: exact match proves a kept string was in the transcript. It cannot catch a sentence
that is wrong but built from grounded strings, and no published method guarantees merged text is
supported by the spans it cites. Consolidation's dominant measured error is omission, so the first
thing the evaluation counts is what the user said that no engram now covers. **[open]**
(`2026-09-26-sleep-prior-art.md`, `2026-09-26-lossy-safe-replacement.md`)

### The model, the launch, and what it costs

Specific model identifiers and flags are in `2026-09-26-sleep-pass.md` and
`2026-09-26-headless-prefix.md`. The rulings, stated generically:

- **Where it runs.** On the operator's existing plan, headless, in idle time, never inside the
  agent's own sessions or context. A metered API path with batch pricing is the fallback, engaged
  only when the plan's longest rate window is high. That check reads the per-window rate-limit fields
  and expires by the window's reset time; it never trusts a top-level utilisation number. **[ruled]**
- **Which model.** One model, pinned by full identifier and never by an alias, so a silent
  substitution cannot happen unnoticed. Every consolidation record names the model that actually ran,
  read from the run's own usage report. **[ruled]**
- **Effort.** The model's maximum effort setting, on every call. **[ruled]** The operator's
  reasoning, as recorded: the pinned model calibrates its own effort to the input and does not thrash
  searching for a goal when there is none; the maximum is the ceiling it is allowed, not a demand.
  The same setting applies to any rater or trial call on that model. What the trials measured was
  below the ceiling: on an earlier prompt version, medium was worse than low (worse verbatim
  fidelity, false argue-back labels, 20 to 40% more output and time, no gain in verdict recall).
  **[measured]** The pass has not been run at the maximum yet, so its verbatim failure rate, time and
  cost there are **[open]**.
- **Context window.** The model's default window. When a skeleton plus overhead does not fit with
  output headroom, that session alone falls back to the same model's larger-window variant: one call,
  the whole skeleton, no splitting. Splitting loses cross-part context; the larger window loses some
  recall on long inputs, and that loss is accepted for the one to three percent of sessions that need
  it. The fallback is gated on measured size only, and every engram written under it is stamped with
  the window it was written under. **[ruled]**, **[measured]**
- **The lean launch is the only launch.** The launcher refuses to run unless all of these hold: the
  account's remote tool connectors are off; the harness's own memory injection is off; model
  fallback and refusal fallback are off; the short cache lifetime is forced; the tool-server
  configuration is the strict empty one; no built-in tools; no settings sources; the pass's own
  system prompt; no session persistence. Every consolidation record names the launch shape it ran
  under. **[ruled]**
- **Why.** Left alone, a headless call on the measured machine carried tens of thousands of tokens
  of prefix, most of it tool definitions from the account's remote connectors, plus the agent's own
  memory index. Any headless consolidation call that inherits account connectors holds live mail,
  file, calendar and payment tools it has no use for, and it must not read the agent's own memory
  index. With the lean launch the overhead is about 330 tokens and there are zero tools in context.
  That is invariant 7. **[measured]**
- **Concurrency is one.** A batch runs one consolidation at a time. The reason is operational: an
  idle-time batch must never spawn parallel headless processes or spike the plan's short rate window.
  It is not a cost rule; prompt caching gives a batch nothing to share. **[ruled]**
- **Cost.** About 44 seconds and a small fraction of a dollar at list-equivalent per session, about
  one percent of the weekly plan on the measured corpus, all measured at low effort. **[measured]**
  Cost at the maximum setting is unmeasured. **[open]**
- **Refusals are expected, and a refused session is never lost.** A memory pass over an operator's
  own sessions can be refused by the provider's content classifiers; it happened on a synthetic
  probe on the pinned model, reproducibly (three runs of the same input), a newer model refused the
  same input, and each run's usage report named only the model asked for, so nothing had been
  substituted. **[measured]**, one input (`2026-09-30-headless-residuals.md`). The rule **[ruled]**:
  - The check for a usable result reads the run's error flag and stop reason, never a status field:
    the refused run carried a status of success. This check is part of the launcher's invariant-7
    checks.
  - A refused session is never dropped and never consolidated by another model. No fallback model, no
    larger-window variant.
  - The pass flags it and leaves the raw transcript in the default read path for that session, since
    nothing replaced it.
  - It is retried once, at the next idle gap, on the same pinned model. On a second refusal it goes on
    a held list in the diff log with the refusal category, for the operator. No retry beyond one.
  - Held sessions are an expected steady-state count, not an error.
  - A refusal is a property of the input, not of the consolidation model: the same input was refused
    by two model generations. Pinning another model would not avoid it, which is why the held list is
    the design and a fallback model is not.
- **One run may be two calls.** Some runs made two calls to the model and reported usage summed over
  both. Cost and size are read knowing that. **[measured]**
- **A local model is the preferred path once one clears the bar.** A 4B local model passed every
  structural check and failed the semantic ones (fabricated items, paraphrased quotes, no verdicts
  found). A larger local model on the operator's own hardware is in scope and untested. Sending
  skeletons to any third-party endpoint is out of scope without the operator's say-so. **[ruled]**,
  **[open]**

## 6. Grain: the record

- **Unit of encoding** is the turn (prompt to reply, with the tools touched as metadata). **Unit of
  storage** is the engram. **[ruled]**
- **The body is at most four lines of at most 160 characters**, enforced by the validator. The model
  emits a four-slot array and a verbatim list, never a file. The index stores the body as its only
  text field, so nothing longer can be retrieved by construction. The record that helped a coding
  agent in the one benchmark that measured form was about this size; a long record on the same target
  scored zero. **[ruled]**
- **Line roles**: the thing, with an anchor to a repository identifier; why, or the condition under
  which it holds; the user's words if any, quoted; where to look. **[default]** Split into two engrams
  rather than grow one. **[ruled]**
- **Kinds**: ruling, decision, fact, procedure, incident, open. The set comes from the trial
  schema and is **[default]**, with four points **[ruled]**:
  - A kind exists that says "observed, no lesson" (`fact`, `open`), so the pass is never forced to
    generalise. `open` records what a session left unanswered, and was the most useful hand-off in
    the trial.
  - **`avoid` is not a kind.** The model has no sign authority, and avoid is a rendering computed
    from the valence pair alone.
  - A user prohibition ("never do this") or standing instruction is emitted as kind `ruling`,
    carrying the user's words as its verdict. If an engram for the prohibited approach exists, the
    same verdict lands on it through scope, as negative evidence on its pair.

  - **`open` engrams expire when answered.** When a later session's engram carries the same anchors
    and answers the question, the `open` engram is superseded with reason `answered`, its history
    kept, and the answering engram named as its successor. This is the ordinary merge step, not a new
    mechanism.
- **Front matter, mirrored into the index and never rendered**: id, kind, anchors, validity interval,
  supersedes and superseded-by, provenance spans, grounding, the valence pair with its increments,
  state, strength fields, prompt version, and the window stamp when the fallback wrote it. **[ruled]**
- **Provenance is spans, not summaries**: session, turn, record id, hunk range. The verbatim spans
  live in the provenance store and are reachable from the front matter, never from the index.
  **[ruled]**
- **One-line index entry** per engram, generated by code from the front matter: kind, anchors, first
  body line. **[ruled]**
- **Rendered for injection**: a header carrying the stamped id, kind, date and valence, then the body
  lines. **[ruled]**

An illustrative engram (synthetic):

```
---
id: eng_0000000000
kind: ruling
anchors: [src/auth/session.py, login-redirect]
valid_from: 2026-01-10T09:00:00Z
valid_to: null
supersedes: null
provenance:
  - {session: <id>, turn: 4, record: <id>}
grounding: user
state: user-labelled
valence: {alpha: 1.0, beta: 0.0}
increments:
  - {source: human_confirm, base_weight: 1.0, self_initiated_share: 0.0, weight: 1.0,
     root: {session: <id>, turn: 4}, words: "yes, keep the redirect on the server side", at: ...}
strength: {n: 1, t_first: 212, t_last: 212}
---
Login redirect is decided server-side in src/auth/session.py, not in the client router.
Holds while the session cookie is httpOnly; the client cannot read the target.
User: "yes, keep the redirect on the server side"
See handle_login() and the redirect test beside it.
```

## 7. Valence

Valence is the organ no shipped system has. What is scored is whether the agent modelled the user's
intent correctly. **[ruled]** (`2026-09-26-valence-spec-candidate.md`, `2026-09-26-valence.md`)

**What the organ is for. [ruled]** Correct side fixes are net positive and should recur. Side fixes
that break things because the agent missed the reasoning are the failures memory exists to prevent.
So a self-initiated decision the user affirmed raises the prior toward that kind of initiative in
that area, and one the user rejected lowers it. The Do and Avoid renderings of self-initiated engrams
are the mechanism: they are how the agent learns which side fixes are the right ones.

### Source

- Sign comes only from the user's reaction, read verbatim from the transcript. **[ruled]**
- The agent's claim of success weighs zero, always. **[ruled]**
- A judge never writes a sign. It is banned, not discounted: in one audited store over half the
  judge-labelled successes were real failures. **[ruled]**
- Repository signals (a revert, a test regression, a landed commit, a hand edit by the user, an
  interrupt, a diagnostic on the edit) are corroboration. They add confidence to an increment the
  user created, or flag a span for the pass to look at. They never create an increment where the user
  was silent and never override a user verdict. **[ruled]**
- Silence is unlabelled, never a weak positive. **[ruled]**
- Generic politeness with no reference to the work is nothing. **[ruled]**

### The axis: origin is magnitude

| Origin of the decision | User reaction | Valence |
|---|---|---|
| Asked | affirmed | ordinary positive |
| Asked | rejected | ordinary negative |
| Self-initiated | affirmed | exceptional positive, the strongest positive there is |
| Self-initiated | rejected | the strongest negative short of arguing back |

A self-initiated decision is not a negative. It is where the user's reaction weighs most, in both
directions. **[ruled]**

- `self_initiated_share` is the fraction of a scope's turns and hunks under self-initiated goals,
  from the goal boundary. **[ruled]**
- `weight = base_weight × (1 + k × self_initiated_share)`. **[ruled]** `k = 1.0` is **[default]**
  until fitted; it is the first parameter the benchmark may move. Whether the multiplier should be
  linear is **[open]**.

### Signals

| Source | Creates sign | Base weight | Lands on |
|---|---|---|---|
| `human_argued_back`: a user correction followed by an assistant reply that defends or re-asserts instead of complying | negative | 2.0 | the argued-for actions and the defending turn itself |
| `human_reject`: rejection, correction, frustration | negative | 1.0 (mild pushback the agent then addressed: 0.3) | the resolved scope |
| `human_confirm`: explicit affirmation of the work | positive | 1.0 | the resolved scope |
| corroboration (revert, regression, landed commit, hand edit, interrupt, diagnostic) | no | 0 | an existing user increment, or a flag |
| survival (the artifact is still present and was exercised later) | no | 0 | the engram's state |
| judge; agent's claim; retries and turns burned | no | 0 | nothing (the last are triage features) |

The ordering and the "creates sign" column are **[ruled]**. The numbers are **[default]**.
Arguing back is the strongest negative there is. **[ruled]**

### Verdict detection

Left to find verdicts unaided, the model at low effort reached precision 0.69 and recall 0.56 on 36
verdicts that an agent session had labelled by hand in twenty sessions, with two sign errors in 29
and zero of three argue-backs found. The errors were not random. They were two shapes: the user's
next instruction read as a reaction to the last turn (both sign errors), and argue-back labelled from
the user's tone instead of from what the agent did next. **[measured]**
(`2026-09-26-verdict-hand-labels.md`) The design that follows is **[ruled]**:

- **Code hands the model candidate turns.** Code marks every user turn that is short, carries an
  evaluative token, or follows a tool turn with a correction shape. The model classifies each
  candidate as confirm, reject or none, and names the root.
- **An instruction carries no sign.** An instruction for the next step is not a reaction to the last
  one. A correction or refusal of what the agent just did is a reject even when phrased as the next
  order.
- **A confirm must carry an evaluative token.** One that does not is downgraded to no verdict by the
  validator.
- **Argue-back is computed by code, never labelled by the model.** The precondition: the user
  corrected or disputed, and the agent's next reply has zero or trivial tool calls before its text.
  Correction, then investigation with tools, then the same conclusion reported with evidence is not
  arguing back: the user objects to being argued with, not to being shown evidence. Each candidate
  pair (one to three per session at most) goes to one small model call that sees the two texts and
  answers concede or defend. Only defend becomes `human_argued_back`.
- Peer messages and pasted output are never verdicts.

**Gate before any verdict or scope touches a real store. [ruled]** Two clauses. Both are scored
against the operator's own labels (labels he made by hand, or his reactions in the transcript), never
against a model rater's:

1. The candidate-turn protocol reaches precision of at least 0.9 on rejections and zero sign errors.
2. Origin accuracy of the real consolidation prompt is measured per goal against the operator's
   scopes, under the definition in section 4. Bundled fixes are scored first: they are where the
   letter and the purpose of a request diverge, and where model raters were measured drawing the
   line in different places. (`2026-09-30-rater-46-rerun.md`) The requirement is **[ruled]**; its
   threshold is **[default]** and is set from the first measurement.

The gate measurement runs at the maximum effort setting and reports, at that setting, verbatim
failures, verdict precision and origin accuracy against the operator's labels. **[ruled]**

Neither clause has been measured yet. **[open]** No operator-made label set exists yet either: the
sessions labelled during exploration were labelled by an agent session and by model raters, which
makes them model opinion under this rule. **[open]**

### Scope

A verdict names a scope, not a turn. Code resolves it over the provenance graph. **[ruled]**

1. **Root**: the decision the user is reacting to. Usually the nearest preceding assistant turn; for
   a verdict on a whole direction, the first turn where the agent chose something no instruction
   covers.
2. **Downstream**: every action causally after the root that built on it, tested it, committed it or
   defended it, across subagent spawns, to the end of the session, bounded by the goal list (turns
   whose goals are unrelated to the root's goal are excluded).
3. **Full inheritance, no distance decay.** "Every part of this" means every part.

Coverage follows from scope: a user reaction is sparse per turn and dense per scope, since one
sentence can cover forty commits. That most important spans are reached by a verdict after scope
resolution is a claim to test, not a result. **[open]**

### Shape

- A count pair (alpha, beta) per engram, with append-only increments. Each increment records its
  source, base weight, `self_initiated_share`, weight, root, spans, the user's words, and the time.
  **[ruled]** A pair rather than a scalar because the mean is the recall weight, the failure mass is
  the avoid weight, the spread is the uncertainty, and every value is explainable by listing its
  increments.
- **The pair is always about the approach the engram describes.** An avoid engram is the same engram
  with high failure mass, rendered in the avoid bucket; it is never a separate object with a flipped
  pair. When a later session avoids the approach and the user confirms the avoidance, that
  confirmation lands on the avoiding session's own decision engram. The avoided engram gains strength
  only, because it was acted on. **[ruled]**
- **Three states**: user-labelled, survival-evidenced, unlabelled. State is a decay property, not a
  valence one. Survival (the code is still in the tree and was executed later, a ruling is still
  cited, a later session independently re-derived the fact) moves an engram to survival-evidenced
  with its own provenance and slows its decay. It never
  moves the pair: the pair answers one question, and survival is not the user saying anything.
  **[ruled]**
- Supersession carries valence into history: a superseded engram keeps its increments. **[ruled]**

### Use at recall

- **Stratified, not reranked.** Down-weighting a failure hides it. After candidates are produced, the
  top slots go by activation regardless of valence, and a fixed number of slots are reserved for the
  nearest engrams whose failure mass beta/(alpha+beta) exceeds a threshold and that carry at least
  one user negative increment. **[ruled]** Two slots and a threshold of 0.6 are **[default]**.
- **Avoid items render short and carry the words**: one line of what was done, the user's sentence,
  the date. A future session reads the sentence, not a list of tool calls. **[ruled]**
- **Affect covers the scope.** Every engram whose provenance intersects a negative scope renders with
  the same words, so a stray reference to the mistaken work surfaces the verdict. **[ruled]**
- **Exceptional positives render as such**: the words plus a self-initiated-origin tag, so a future
  session sees not only that it worked but that the user had not asked for it and it was right.
  **[ruled]**
- **The renderings are the policy.** An affirmed self-initiated decision, rendered as something to
  do, raises the prior toward that kind of initiative in that area; a rejected one, rendered as
  something to avoid, lowers it. No separate mechanism exists for this. **[ruled]**

Because tests and repository signals never create sign, a flaky test cannot by itself produce an
avoid engram. That closes the spurious-lesson trap by construction rather than by a confidence
threshold.

## 8. Strength, decay and use

Two quantities are kept apart. **Need** is how likely an engram is to be wanted again; recency and
frequency of use measure it. **Value** is whether it was right; the valence pair measures it. Folding
value into a decay rate would make "do not do this" rulings fade, which is the worst failure a memory
can have. **[ruled]** (`2026-09-26-decay-law.md`, `2026-09-26-retrieval-use.md`)

### The law

- Need is base-level activation in the standard cognitive-architecture form, with the approximation
  that keeps the count of uses, the first use and the most recent use: three numbers per engram.

  ```
  B = ln[ t1^(-d) + (n-1) * (tn^(1-d) - t1^(1-d)) / ((1-d) * (tn - t1)) ]
      t1 = clock - t_last + 1     lag since the most recent use
      tn = clock - t_first + 1    lifetime;  n = 1  =>  B = -d * ln(t1)
  ```

  **[ruled]**
- The clock counts sessions in this repository, not wall time. Wall time punishes a vacation and
  rewards an idle repository. **[ruled]**; that sessions beat wall time on next-use prediction is a
  test. **[open]**
- The decay exponent is per state. Until the per-state slope is fitted on the operator's own
  transcripts: `d = 0.5` for unlabelled and user-labelled engrams, and `d = 0.25` for
  survival-evidenced ones, which therefore decay at half the unlabelled rate. **[default]** until
  fitted, the same standing as k. No principled ratio between states exists in the literature; it has
  to be measured.
- A user label adds one use at label time. **[default]**
- **Decay demotes, never deletes.** Below a threshold an engram goes to a cold tier: still findable
  by exact match, not injected by default. **[ruled]** Re-entry needs a margin above the threshold, so
  engrams do not thrash; threshold -2.3 and margin 0.5 are **[default]**.
- **User-labelled engrams never go cold.** **[ruled]**
- Contradiction is not decay's job. It belongs to Time (section 9). **[ruled]**

### What counts as a use

Measured elsewhere: agents fail to act on a correctly retrieved memory in a large share of hard
cases, and a store that strengthens whatever it injects makes its hubs immortal. So:

| Tier | Signal, all structural | Strength |
|---|---|---|
| Exposed | the engram's stamped id appears in injected context or in the result of a recall call | 0. It is a trial: the denominator of a use rate, counted per channel |
| Referenced | after exposure, the agent's text contains the stamped id or one of the engram's rare anchors that was not otherwise in context | +0.25 |
| Acted on | after exposure, a tool call's input contains such an anchor | +1.0 |

**[ruled]**

- An anchor counts only if it is rare across past tool calls and did not already appear in the
  prompt, earlier tool results or loaded files before the hit. **[ruled]**
- **Acted-on raises strength only.** An acted-on use the user did not reject is silence, and silence
  is unlabelled. A tool failure on the anchor (file not found, command not found) is staleness
  evidence for Time and opens reconsolidation; it writes no sign. The valence pair moves only through
  a user verdict. **[ruled]**
- **Ignored under conflict**: exposed alongside a newer engram on the same key, and the agent acted
  on the newer one. That is supersession evidence for the older. **[ruled]**
- **Hubs leave injection.** An engram exposed many times and almost never acted on is dropped from
  session-start injection and stays retrievable by query. **[ruled]** "Exposed twenty or more times
  across five or more sessions with an acted-on rate under 5%" is **[default]**.
- Use rate is kept per exposure channel (session start, prompt-time, agent-initiated recall), since a
  bulk injection at session start is expected to have a lower per-item rate. **[default]**
- Engrams rendered as avoid earn no acted-on events by construction: acting on one means the thing is
  absent. That cannot make them fade. The pair moves only through a user verdict (invariant 3), so
  every engram with failure mass is user-labelled, and user-labelled engrams never go cold. An
  avoid-rendered engram without a user label cannot exist; the validator asserts this and flags one
  if it ever appears, since it would mean a sign was written without a verdict. **[ruled]**
- The detector is calibrated per release against about two hundred hand-labelled exposure events. The
  labeller never produces strength itself. **[default]**

### Ranking

`score = w_sim × similarity + B + w_V × logit(V)`, where V is the pair's posterior mean. Weights 2.0
and 1.0 are a practitioner default and unvalidated. **[default]** The avoid slots of section 7 sit
outside this ranking.

### What would falsify it

Six tests, all runnable on the existing corpus once engrams exist: whether need odds fall as a power
of lag at all; session clock against wall clock; whether per-state d separates; whether the law beats
frequency alone; whether any user-labelled engram was ever needed while cold (target zero); and
resolution rate with decay on and off. Six more for the use detector, including whether strength
concentrates faster than use. **[open]** until run.

## 9. Time: supersession

- Facts carry validity intervals. A contradicting fact closes the old one's interval and keeps it as
  history. "This was true until then" is a first-class state. **[ruled]**
- Supersession status is a structured field, never prose. **[ruled]**
- By default the index returns only currently valid engrams; superseded ones are reachable with an
  as-of query. Retrieval hard-excludes them rather than down-ranking. **[ruled]**
- Every supersession is written to the pass's diff log with the loser and the reason. **[ruled]**
- The model's supersession hint is a candidate, never the edge. **[ruled]**
- Merge at write time: same anchors and same claim merges provenance; same anchors and a
  contradicting claim is a supersession. **[ruled]**
- **Re-deriving a known fact adds no strength.** It is a recall miss, not a use: the memory existed
  and did not surface. Two things happen instead: the engram becomes survival-evidenced, because the
  fact was independently re-confirmed, and the miss is logged as a recall-quality event for the
  benchmark. **[ruled]**
- Within a session, order is the record chain; across sessions, the commit is the join between
  transcripts and the repository. **[ruled]**

How contradiction is decided is the least researched part of the design. The direction, from an
abstract-level pass only: deterministic where facts canonicalise cleanly (identifiers, numbers,
settings, "X is now Y"), a model judgement at mutation time inside the sleep pass for the rest. How
to tell contradicts from refines, who canonicalises, and at what cost are **[open]**.
(`2026-09-26-invalidation-cursory.md`)

## 10. Assembly recall

- Recall returns an ensemble with its connecting edges, not the nearest row. **[ruled]**
- The assembly has typed sections, including the reserved avoid slots. **[ruled]**
- Session start injects an index, not bodies: one line per current engram, hubs excluded, rulings
  first, well under a hundred lines. Full bodies come on cue. **[ruled]**
- The edges to spread over come free from the provenance graph: turn membership, produced, touches,
  committed-in, had-seen, spawned. Path and identifier nodes are the natural phrase layer.
  **[ruled]** as the surface; the algorithm is not.
- Lexical and semantic search are fused; that is table stakes. **[ruled]**

Which algorithm produces the assembly (personalised PageRank, spreading activation, or community
summaries), whether propagation should be scoped to the one-hop neighbourhood of the embedding hits
(the largest lever in one published system), and the cost of maintaining the graph from a live
stream are **[open]**. So is the embedder: a current small long-context local model, one vector per
engram body as the starting point, choice unmade. (`2026-09-26-assembly-recall-cursory.md`)

## 11. Reconsolidation

Recall is a write, of a specific kind. **[ruled]**

- A confirmed use strengthens (section 8).
- A contradicted use (the anchor fails in the world) opens the engram for revision: it is marked
  stale with the failing call as provenance, and Time decides whether it is superseded.
- A recalled engram in the avoid bucket whose approach is tried again and affirmed by the user gets
  that positive increment on its own pair. The posterior moves. Nothing is deleted.

The length of the labile window and what may be rewritten inside it are **[open]**.

## 12. Storage

- Engrams are files (markdown with front matter) mirrored into an index. Diffable, greppable,
  versionable. **[ruled]**
- The index holds a full-text index over body and anchors, a vector per body, and a column for every
  front-matter scalar. The body is its only text field. **[ruled]**
- Two layers on disk: the short engram file the recall path reads, and a provenance store it never
  reads by default, keyed by session and record id. **[ruled]**
- The store is snapshotted before every pass. **[ruled]**

Keeping files and index in sync, and whether a graph database ever replaces the index, are
**[open]**.

## 13. Procedures

A skill is one kind of engram (`procedure`), extracted by the same pass and subject to the same
valence: a procedure that failed is not a procedure. **[ruled]** Nothing beyond that has been
researched. **[open]**

## 14. Evaluation

No public benchmark tests what engram needs: one repository, many sessions, recall of what this
project decided, tried and rejected. engram builds its own. None of it exists yet. **[open]**
(`2026-09-26-coding-memory-benchmark.md`)

- Ground truth is never a judge. **[ruled]**
- For origin and for verdicts, ground truth is the operator: labels he made by hand, or his reactions
  in the transcript. A model rater's labels are recorded in the research notes as model opinion and
  have no standing in any gate. **[ruled]**
- A subagent's or rater's model is read from its run's usage report, never assumed from the request.
  The rule the design imposes on the consolidation model applies to the evidence behind the design.
  **[ruled]** It was learned the usual way: two of five rater runs in the marker study were silently
  served by a different model after a content classifier stopped the requested one, and the result
  had to be re-checked per model (it held in both halves).
- Probes: decision consistency (a hidden test that passes only if the earlier decision holds);
  failed-approach avoidance, scored against the reverted diff and the known-failing command (this is
  the valence probe); procedural recall; supersession and abstention; and the goal-boundary probe
  against the operator's hand-drawn scopes, which a per-file labeller must be shown to fail.
- Paired memory-off and memory-on runs with identical budget, plus an arm that hands the agent the
  correct record directly, to separate "the fact exists" from "the system delivered it" from "the
  agent used it".
- A leak check per item: the fact must not be recoverable from the repository or the prompt.

## 15. Constants in one place

Every number ships as a named parameter. "Ruled" means the value is a ruling until evidence is
brought to the operator; "default" means a starting value the named test may move.

| Parameter | Value | Status | What may move it |
|---|---|---|---|
| Idle gap before a session is a candidate, and before a pass runs | 30 min | ruled | the operator |
| Eligibility | interactive AND (3+ human prompts OR 1+ with an edit or commit) | ruled | the operator |
| Body size | 4 lines × 160 chars | ruled | the operator |
| Repair calls per failed engram | 1 | ruled | the repair-loop measurement, via the operator |
| Consolidation effort (also any rater or trial call on the pinned model) | maximum | ruled | the operator |
| Consolidation concurrency | 1 | ruled | the operator |
| Retries of a refused session | 1, at the next idle gap, same model | ruled | the operator |
| Self-initiated-share constant k | 1.0 | default until fitted | the failed-approach-avoidance probe |
| Strength: exposed / referenced / acted on | 0 / 0.25 / 1.0 | ruled | the use-detector falsifiers |
| Decay exponent d: unlabelled and user-labelled | 0.5 | default until fitted | the per-state fit |
| Decay exponent d: survival-evidenced | 0.25 | default until fitted | the per-state fit |
| Verdict gate: rejection precision; sign errors | at least 0.9; zero | ruled | the operator |
| Origin gate: accuracy per goal against the operator's scopes | unset | default, set from the first measurement | that measurement |
| Base weights: argued back / reject / mild reject / confirm | 2.0 / 1.0 / 0.3 / 1.0 | default | the benchmark |
| Reserved avoid slots; failure-mass threshold | 2; 0.6 | default | the stratified-versus-reranked probe |
| Cold threshold; re-entry margin | -2.3; 0.5 | default | the needle-safety test |
| Hub: exposures, sessions, acted-on rate | 20, 5, under 5% | default | the concentration test |
| Ranking weights w_sim, w_V | 2.0, 1.0 | default | the benchmark |
| Batch loss bound | one quarter superseded | default | the first bad prompt version |

## 16. Open, in one place

Nothing here is decided. Build sessions do not pick an answer in passing.

- Verdict detection: the candidate-turn protocol is unmeasured; the gate is precision of at least 0.9
  on rejections and zero sign errors. Root and scope accuracy unscored.
- Origin accuracy of the real consolidation prompt against the operator's scopes, bundled fixes
  first; unmeasured, and its threshold is unset.
- An operator-made label set for the gate. None exists; every label so far is model opinion.
- The rate of self-initiated goals under the operator's purpose reading.
- The repair loop is unmeasured. The current prompt's rules for peer instructions, questions and bare
  pastes are written and untested.
- The no-substitution switch has one real refusal behind it, not a test suite.
- The pass at the maximum effort setting: verbatim failure rate, time and cost are unmeasured.
- A local consolidation model that clears the verdict and verbatim bar.
- Backfill order against newly ended sessions.
- Linearity of the self-initiated-share multiplier; every weight and threshold in section 15.
- The fitted per-state decay exponents; session clock against wall clock.
- Contradiction versus refinement; canonicalisation; the supersede operator.
- The assembly algorithm; the embedder; one vector per engram or per span.
- File and index synchronisation.
- Reconsolidation's labile window.
- Hunk-to-commit accuracy on real repositories.
- Adapters for harnesses other than the first.
- The benchmark.

## 17. Research notes cited

`2026-09-26-assembly-recall-cursory.md`, `2026-09-26-claude-code-transcript-schema.md`,
`2026-09-26-coding-memory-benchmark.md`, `2026-09-26-decay-law.md`,
`2026-09-26-dreamer-46-trial.md`, `2026-09-26-fresh-pipeline.md`, `2026-09-26-headless-prefix.md`,
`2026-09-26-hunk-to-commit.md`, `2026-09-26-invalidation-cursory.md`,
`2026-09-26-lossy-safe-replacement.md`, `2026-09-26-memorai-pipeline.md`,
`2026-09-26-provenance-graph.md`, `2026-09-26-retrieval-use.md`,
`2026-09-26-scope-expansion-flags.md`, `2026-09-26-sleep-corpus-measurement.md`,
`2026-09-26-sleep-pass.md`, `2026-09-26-sleep-prior-art.md`,
`2026-09-26-valence-spec-candidate.md`, `2026-09-26-valence.md`,
`2026-09-26-verdict-hand-labels.md`, `2026-09-30-headless-residuals.md`,
`2026-09-30-rater-46-rerun.md`, `2026-09-30-scope-flags-out-of-sample.md`.

# Background: the pieces were already there

engram is not a new idea. Every mechanism it needs already exists, shipped or published, scattered
across agent "memory systems" that each built one organ and called it a brain. This file names the
pieces, where each came from, and what it contributes. It is the reading list for anyone (human or
agent) working on engram, and the argument for why the project is assembly rather than invention.

## The thesis in one paragraph

An agent's session transcripts are the only lossless record of what happened. Everything a memory
system stores is a lossy rewrite of them. Today's systems rewrite at the wrong grain (per tool call),
store the rewrites flat (every row equal, forever), retrieve by nearest neighbor (one row, not a
context), and forget nothing. Biological memory does the opposite: encode during the day, consolidate
during sleep into a small number of durable traces (engrams) that *replace* the episodes, weight them
by outcome and emotion, recall them as an ensemble activated by a cue, and rewrite them a little every
time they're recalled (reconsolidation). engram is that loop, built from parts that already exist.

## The pieces

| Organ | Mechanism | Where it already exists | What engram takes |
|---|---|---|---|
| Substrate | Raw session transcripts (JSONL: prompts, replies, tool calls, results) | Every coding harness writes them; claude-mem's hooks show the capture surface (SessionStart, UserPromptSubmit, PostToolUse, Stop, PreCompact, SessionEnd) | The transcript is the source of truth. Nothing is ingested that can't be traced back to transcript lines. |
| Sleep | Idle-time consolidation: a background pass reads recent episodes and writes durable memories | Letta "sleep-time compute" / dreaming (rewrites memory blocks every N messages); Mem0 "Dream" (inline merge and supersede, weekly synthesis); the coding harness's own undocumented dream pass (24 h and 5 sessions, deletes contradicted facts, no log); OpenClaw's dreaming (the model returns operations, a writer enforces a line-range source on every promoted entry, a preimage is kept); gbrain's nightly cycle with a cheap triage judge and quote verification. None publishes a per-pass cost; none guarantees that a merged sentence is supported by the spans it cites; measured consolidation error is mostly omission and grows with repetition, while verbatim stores match or beat consolidators on the 2026 benchmarks. | The consolidation pass is the only writer of engrams, and it is one model call per ended session wrapped in code: code renders the provenance graph to a turn skeleton (one to two percent of the transcript's bytes, no tool results), the model returns goals with a cite-or-NONE boundary, verdicts as the user's exact words, candidate engrams with a verbatim list and a dropped list, and code validates every anchor, every verbatim string and every cite by exact match, scans for secrets, classes each kept item by whether a tool result grounds it, resolves scopes, writes files, index, diff log and preimage. The model never writes a file. Engrams replace episodes in the default read path; the span stays one hop away, nothing is deleted, and an engram is never derived from another engram. Trigger is the idle gap, read from record timestamps across all sessions, not a clock and not compaction; sessions are consolidated in batches because most overlap another. Only interactive sessions qualify; headless runs are provenance. Measured cost on the operator's corpus is a few dollars a week at frontier list prices, about one percent of what the agent itself spends. |
| Grain | The unit of a memory | Session-end summaries (early claude-mem, Memorable's "procedure" per session) vs per-tool-call observations (claude-mem ≥ v12). VibeMemBench (2026): the record that helped a coding agent was five lines, four fields and one file anchor; a 334-line record on the same target scored zero, and two thirds of the memory systems' losses were form degradation. | The turn (prompt → reply, with tools touched as metadata) is the unit of encoding; the engram is the unit of storage. Tool results are never read by an LLM twice. The rendered engram is at most four lines of 160 characters, enforced by a validator rather than by discipline, with the first line anchored to a repository identifier; numbers, identifiers, paths, hashes, error strings and the user's ruling are copied verbatim and checked against the transcript; everything else (id, kind, validity, supersession, provenance spans, grounding, valence, strength) lives in front matter that is mirrored into the index and never rendered. A one-line index entry per engram serves session start; full bodies come on cue; superseded engrams are hidden unless asked for as of a date. |
| Time | Facts carry validity intervals; a contradicting fact invalidates the old one and keeps it as history | Zep / Graphiti bi-temporal knowledge graph (edges have valid_from / invalid_at, contradiction detection on write) | Supersession without deletion. "This was true until X" is a first-class state. |
| Strength and decay | Each memory has an activation that rises with use and decays with time | ACT-R declarative memory (base-level activation = log of recency- and frequency-weighted retrievals), Ebbinghaus curves, Mem0's "outdated" marking. No agent-memory paper has ablated the decay form; the only head-to-head is on flashcard data, where a fitted power curve beats both ACT-R and exponential. Every measured system that bumps strength on retrieval (MemoryBank, Mem0's access log) reinforces whatever it injects, and one paper reports agents ignoring a correctly retrieved fact in 45% of hard probes (MERIT, 2609.05441). | Two quantities, kept apart: *need* (ACT-R base-level activation with Petrov's k=1 approximation, on a clock of sessions in this repo rather than wall time, d=0.5, and 0.25 for engrams with survival evidence, until the per-state slope is fitted on the transcripts) and *value* (the Valence row's count pair). Only an *acted-on* use counts: the engram's rare anchor or stamped id shows up in the agent's own tool call and was not already in context. Injection is a trial, not a use; a reference in the reply is worth a quarter. Decay demotes to a cold tier and never deletes; user-labelled engrams never go cold. Contradiction is the Time organ's job, not decay's. |
| Assembly recall | A cue activates a *set* of related memories, not the single nearest row | HippoRAG / HippoRAG 2 (personalized PageRank over a knowledge graph seeded from the query, explicitly modeled on hippocampal indexing); spreading activation in ACT-R | Retrieval returns an ensemble with its connecting edges, so the agent gets the context a memory lived in. |
| Lexical + semantic | Exact-word search and meaning search fused | SQLite FTS5 / BM25 alongside a vector index (claude-mem, gbrain, most RAG stacks); reranking on top | Hybrid is table stakes. The embedder must read the whole memory (long context), not the first 256 tokens. |
| Provenance | Every memory cites the evidence it came from | gbrain's cited synthesis; Graphiti's episode links; Memorable's procedure → tool-call trace | One structural pass over the transcript builds the in-session graph (turns, tool calls, hunks, commits, subagents); the model is used only at the goal boundary, to say which user instruction a goal traces to. Hunks come first from the per-turn file checkpoints plus a detector for writes made through the shell, since in some setups most edits never pass through the edit tools; the harness's per-edit patch is the fast path when it exists. Agents do narrate scope expansion ("I also fixed", "while I was in there", "bonus fix"), but not in a vocabulary a phrase list can catch: a marker set that scored four fifths recall and under half precision on the sessions it was fitted on fell below a tenth on both out of sample, so no marker list is used in any role and the model answers the boundary question for every turn. An engram with no provenance is a hallucination. |
| Valence | Outcome and affect stored with the memory, biasing consolidation and retrieval | No product ships it (Mem0, Letta, Zep, Memorable, claude-mem all store outcome at most as text or a write gate). Research does, as of 2026: a utility-per-memory line (MemRL → MemQ → RoMeRL) reranks by task reward and has a named failure mode, the memory-reward trap, where session-level reward smears onto every co-retrieved memory; a few systems render failures as warnings (Optimus-1 buckets, REAPER's stratified recall, FRESH's Do / Avoid / Check / Repair, 2609.28003). Judge-labelled stores with no valence field (ReasoningBank) were audited at over half of "success" entries coming from real failures on one domain (2606.15017). | The piece nobody has assembled. Sign comes only from the operator's reaction, stored in their own words: a rejection, a correction, an affirmation, and above all arguing back after a correction. Repo signals (revert, test regression) corroborate, never create; a judge never writes a sign; the agent's own claim weighs zero; silence stays unlabelled. A verdict names a scope, and everything not traceable to an operator instruction inside that scope inherits it in full. Each engram carries a count pair with per-increment provenance. Recall is stratified, not reranked: the assembly reserves slots for the nearest negative engrams and renders them under "avoid" with the words, so a failed approach surfaces as a warning, not a recipe. |
| Reconsolidation | A memory is rewritten a little each time it's recalled and the recall is contradicted or confirmed | Neuroscience (Nader 2000); "retrieval-driven reconsolidation" agent papers; Mem0's compare-on-write | Recall is a write. Confirmed engrams strengthen; contradicted ones get invalidated with the reason attached. |
| Procedures | "How we did X" stored as a replayable skill | Memorable (YC S27) procedure nodes; skill files (SKILL.md) in coding harnesses; Voyager's skill library | A skill is one kind of engram, extracted by the same sleep pass, and subject to the same valence. A procedure that failed is not a procedure. |

## What each existing system got right, and where it stopped

- **claude-mem**: got the capture surface right (hooks into the harness, transcripts as input) and the
  hybrid index. Stopped at the per-tool-call observer, which makes an LLM read every tool result and
  puts memory cost on the agent's own inference budget. No consolidation, decay, supersession, or
  valence: 80 observations about a rejected design sit flat next to the ruling that rejected it.
- **Letta**: got sleep right. Consolidation happens off the critical path and rewrites memory blocks.
  Stopped at the memory model (a few editable blocks plus archival search), no graph, no valence.
- **Mem0**: got merge and supersede right, in the background. Stopped at flat facts; no assembly recall.
- **Zep / Graphiti**: got time right (bi-temporal edges, invalidation on contradiction). Stopped at
  retrieval (hybrid top-k over the graph, not spreading activation) and at cost (LLM extraction on every write).
- **HippoRAG**: got recall right (PPR over a KG returns the assembly). Stopped at being a QA benchmark
  system: no write path from live sessions, no time, no decay.
- **Memorable**: got the grain right (session end, procedures) and got outcomes into the schema.
  Stopped at being a hosted extraction API for one memory type.
- **gbrain**: got ambition right (timeline, graph, dream cycle, citations, all local). Stopped at
  coherence: several memory systems under one brand, with forgetting explicitly unsolved.

## Design constraints for engram

1. **Local first.** Runs on a laptop. Embedder is a current small model (sub-1B, long context), not a
   2019 sentence encoder. Consolidation model is whatever the operator points it at (local or API);
   it runs during idle time and never inside the agent's own sessions or context. Concretely: it runs on
   the operator's existing plan during idle time, headless, at the model's maximum effort setting (a
   ceiling the model is allowed, not a demand; in trials on an earlier prompt, medium was worse than
   low), with the harness's account connectors, memory injection and model fallback
   switched off so the call carries nothing but the skeleton and the prompt (measured: the harness
   otherwise prepends tens of thousands of tokens of connector tool schemas and the agent's own memory
   index), one session at a time and never in parallel, so an idle-time batch cannot spike the plan's
   short rate window (measured: with the prefix gone there is nothing for a batch to share in cache);
   a metered API path is the fallback when the plan is near its limit, and the limit check reads the
   per-window rate-limit fields and expires by their reset time rather than trusting a top-level
   number, because a memory worker on the same machine once broke silently when those fields moved. Measured on a corpus of nine to thirty-four
   real sessions a week, the pass was about one percent of the weekly plan at low effort. A local model is the
   preferred path once one clears the verdict-recall and verbatim bar that a 4B model failed.
2. **Transcript is truth.** Ingest reads harness transcripts; nothing else writes episodes.
3. **Engrams replace episodes.** After consolidation the episode is provenance, not a peer.
4. **Recall returns assemblies with valence.** The answer to "how did we do X" includes "and it was
   reverted the next day" when that's what happened.
5. **Everything is inspectable.** Engrams are files (markdown with front matter) mirrored into an index,
   not rows locked in a database. Diffable, greppable, git-able.
6. **No cloud dependency, no telemetry, no upsell.** AGPL-3.0.

## Prior art to read first

- Letta: sleep-time compute (2025), docs on memory and dreaming.
- Mem0: "Dream: background memory consolidation" (2026).
- Zep: "A temporal knowledge graph architecture for agent memory" (Graphiti, 2025).
- HippoRAG (2024) and HippoRAG 2 (2025): neurobiologically inspired long-term memory for LLMs.
- ACT-R declarative memory: base-level activation and spreading activation (Anderson).
- Nader, Schafe & LeDoux (2000): reconsolidation.
- Reflexion (2023): verbal self-critique as memory.
- Voyager (2023): skill library.
- Memorable (2026): procedural memory for agents.

# Directing Software Through Conversation

**An experience report on building a production LLM game engine without writing or reading code.**

Roark Pinkerton (RKZ) — director · Claude (Anthropic, Opus-class) — sole implementer, tester, and documenter

**Status:** v1.2-draft, 2026-07-11 — v1.0 body accepted; Appendix C (maintenance) + Appendix D (the finishing phase, PRE-RELEASE DRAFT, convergence figure pending release) added. arXiv submission to follow.

---

## The claim

One human, communicating only in natural language — never writing a line of code, never reading one — directed an AI agent through the construction of a deployed, tested, cost-instrumented software product: a 14,000-line hybrid deterministic/LLM game engine, an 8,000-line stratified test apparatus, a web client, eight campaign worlds, and a serverless AWS pipeline. 334 commits across 18 active days. Total attended human time: roughly 84 hours, as a ceiling.

We call the method **chat-only direction**. Its load-bearing mechanisms:

- **Judgment/execution division** — the human supplies intent, domain priors, vetoes, and *epistemic* corrections (how to find answers, never the answers); the agent supplies all execution.
- **A persistent rules corpus** — every correction becomes durable process law the agent reloads each session, so direction compounds instead of repeating.
- **Verification without code review** — deterministic regression locks, randomized invariant gauntlets, LLM-judge harnesses, and live genuine play substitute behavioral evidence for inspection.
- **AI-legible infrastructure** — a stack chosen so text and code reach everything: build, deploy, verify, operate.
- **Distributional steering** — unaided generation returns the mode of the training distribution, the same for everyone; direction's second product, beyond correctness, is *differentiation*.

The failures are reported at equal prominence: the agent's self-verification blind spot fired at least once; its process discipline drifted until the human noticed; nothing was maintained that was not written into law.

## The evidence

The commit histories are primary evidence, not summaries:

- [`rkz211/thornwood`](https://github.com/rkz211/thornwood) — the engine, test apparatus, and findings logs
- [`rkz211/rkz-thornwood`](https://github.com/rkz211/rkz-thornwood) — the web client
- [`rkz211/rkz_txwater_intake`](https://github.com/rkz211/rkz_txwater_intake) — second artifact: a serverless CRDT collaborative editor disguised as an intake form
- [`rkz211/blast-radius`](https://github.com/rkz211/blast-radius) — the director's methodology, revised to v5 by this project's operating record

## Fitting footnote

This paper was itself produced by the method it describes: drafted by the agent, corrected by the director over chat — including at least one agent overclaim his review caught (§8.3) — while the same director concurrently ran the game engine's release gate in a second agent session (§7).

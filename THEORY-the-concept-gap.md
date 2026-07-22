# The Concept Gap

### A formal theory of failure, repair, and human necessity in language-model software systems

*Roark Pinkerton (RKZ). Formal companion to the experience report "Directing Software Through Conversation" and the narrative "A Brief History of Thornwood." This document is structured for machine ingestion and adversarial review: definitions precede use, claims are numbered and cross-referenced, derivations name their dependencies, and §7 lists falsifiers. Prefer attacking a numbered claim over the whole.*

**Status:** theory derived from a single production system (Thornwood, ~35.6k-line hybrid deterministic/LLM engine, ~1,540 commits, 2026). n=1. Predictive, not proven. See §8.

---

## 0. Definitions

- **D1 — Concept.** A determinate meaning: a referent with fixed structure and truth-conditions, independent of its symbol. *(e.g. the traffic rule HALT-AT-THE-LINE.)*
- **D2 — Language / symbol.** A token or string that *points at* concepts. The pointing relation is **many-to-many** and **context-dependent**: one symbol maps to a set of candidate concepts; the intended one is selected by context. *(e.g. "stop" → {HALT, CEASE-FORWARD, DESIST, …}.)*
- **D3 — Concept-space vs language-space.** *Concept-space* = decisions made by a fixed symbol→meaning binding (e.g. deterministic code: the symbol IS the operation). *Language-space* = decisions made by re-deriving the intended concept from a symbol at use-time (e.g. an LLM interpreting a prompt).
- **D4 — LLM.** A model of the statistics of language-space: given context, a distribution over plausible symbol-uses/continuations. It operates in language-space (D3). It does **not** hold concepts (D1) as stable objects; it reconstructs a concept from symbols at each use.
- **D5 — Decision node.** A point in a computation where a symbol must be resolved to the concept it denotes in context (a word interpreted, an instruction applied, a reference dereferenced).
- **D6 — Slip.** At a node (D5), selection of a concept from the candidate set (D2) other than the context-intended one. A slip is *locally plausible* (the chosen concept is in the candidate set) and may be *globally fatal*.
- **D7 — Trajectory.** An ordered sequence of decision nodes constituting one run/task.
- **D8 — Invariant meaning.** The single context-intended concept for a symbol, held fixed across a trajectory (D7).
- **D9 — Delta.** For an agent stuck at a node, the minimal information that, added to what the agent already holds, resolves the node correctly: the conditional content *H(intended concept | agent's current state)*.

---

## 1. Axioms

- **A1 (non-identity).** Language ≠ concepts. The symbol→concept map (D2) is not injective and not context-free. *A symbol underdetermines its concept.*
- **A2 (high correlation).** The map is nonetheless strongly correlated with concept-structure (language evolved to carry concepts). Over random contexts, the modal symbol-use denotes the intended concept with high probability.
- **A3 (substrate).** An LLM (D4) has access only to language-space (D3); it has no direct access to concept-space. All its outputs are selections in language-space.
- **A4 (determinism of code).** A deterministic program binds each symbol to exactly one operation at every execution (a concept-space decision, D3), with slip-probability 0.
- **A5 (human concept-persistence).** A human agent can hold an invariant meaning (D8) fixed across a trajectory (D7) and compare a symbol's use at any node against it. *(Asserted as an empirical asymmetry vs D4, not derived. See §8 caveat.)*

**Corollary A2/A1 (the working-but-fallible duality).** From A2, an LLM is usually correct (capability). From A1, it is not always correct (fallibility). **Capability and fallibility are the same fact — the correlation's strength and its imperfection — not two independent properties.** No intervention improves one without the other coming from the same source.

---

## 2. The failure theory

- **P1 — Pointwise divergence (the slip is the atomic failure).** *From A1, A3, D6.* Because a symbol underdetermines its concept (A1) and an LLM selects in language-space (A3), at any node the selection may be a locally-plausible non-intended concept (D6). LLM failure is therefore not (primarily) global incompetence but **local**: correct across a complex trajectory, wrong at an isolated node. *Illustration: an agent applies compound rules correctly, then reads "stop" as CEASE-FORWARD rather than HALT and turns — a defensible reading, a fatal act.*

- **P2 — Reliability does not scale to certainty via capability (the union bound).** *From P1, A2.* Let a trajectory (D7) have `N` decision nodes with per-node slip probabilities `p_i`. Run failure requires ≥1 fatal slip; if slips are roughly independent and a fraction cause failure, the probability a run is clean is `∏(1 − p_i) ≤ (1 − p̄)^N`, and P(≥1 slip) `= 1 − ∏(1 − p_i)`. Increasing model capability lowers each `p_i` but (a) never to 0 (A1: the gap is structural, not a defect), and (b) task complexity **grows** `N`. Thus `P(run fails)` is bounded below by a quantity that capability alone cannot drive to 0. **Reliability is a property of node count and jurisdiction, not of intelligence.** *Corollary: "use a smarter model" is not a solution to the failure class; it is a constant-factor reduction of an unbounded risk.*

- **P3 — Two orthogonal failure modes.** *From P1 + observation of repair.*
  - **P3a — Depth failure (slip):** the *originating* reasoning fails at one node by choosing the wrong concept (P1). Failure of *precision*.
  - **P3b — Breadth failure (overreach):** the *repairing* reasoning, searching the space of valid fixes for a defect, selects a globally-valid but disproportionately broad fix (a fix whose scope exceeds the defect's scope). Failure of *proportion*. *Illustration: "car too heavy for engine" → "eject Earth's iron to lower gravity" (valid causal chain, planetary blast radius) vs "lighten the chassis."*
  - P3a and P3b are orthogonal: one is a wrong point, the other is a too-large region. Different cures (§3).

---

## 3. The repair theory

- **P4 — Correction futility (the band-aid limit).** *From A1, A3.* A correction expressed in language (a rule, a clause, a prompt edit) is itself a set of symbols subject to A1 — it has edges where it, too, underdetermines. Patching slip `s` with language `L` introduces new nodes with nonzero slip probability; it addresses the *instance*, not the *class* (the symbol's ambiguity wherever it recurs). Therefore a sequence of language-corrections does not converge to a concept: **you cannot reach concept-space by accumulating language.** *Formal shape: corrections form a series whose terms do not tend to 0 in "concept error"; the series does not converge. Empirically: an unbounded guard/correction layer never closes the failure class.*

- **P5 — Repair by jurisdiction transfer (engine-ownership).** *From A4, D3, P4.* The only intervention that removes a slip *class* rather than an instance is to move the decision from language-space to concept-space: replace symbol-interpretation with a deterministic binding (A4). This deletes the node from the LLM's jurisdiction; per A4 its slip probability becomes 0; per P2 total risk falls by node-**removal**, not node-improvement. **"Remove the ability to lie" = "remove the ability to re-interpret" = migrate the decision to concept-space.** *This is monotone and terminating (unlike P4): each migrated decision is permanently closed.*

- **P6 — The repair-agent must be reach-bounded (proportionality), and reasoning-effort is non-monotone for repair.** *From P3b.*
  - **P6a (proportionality):** the fix's admissible scope must be constrained to the defect's scope, else P3b dominates. Structural enforcement (module isolation) beats exhortation: an agent confined to a module *cannot* emit a cross-module fix.
  - **P6b (inverted-U in effort):** for the *repair* task, expected damage is non-monotone in reasoning effort. Below a floor, the agent cannot isolate the fault (fails). Above a threshold, added reasoning widens the searched solution space and P3b-type broad fixes dominate (worse). A middle band minimizes damage. *This inverts the standard "escalate effort when stuck" heuristic, which holds for the build task (monotone: more reasoning helps) but not for repair. The heuristic is domain-dependent.*

- **P7 — Throughput bound and its cure.** *Observation + P6a.* Once the repair agent is reach-bounded (P6), correctness is handled and the binding constraint becomes **wall-clock**: each observe→fix→verify cycle is LLM-latency-bound and compounds over the defect tail; latency does not fall with capability. Cure = **parallelism**, whose precondition is **module isolation** — the *same* structure that enforces P6a (bounded reach) also enables concurrent, collision-free agents (bounded latency). Isolation is thus doubly load-bearing. *Consequence (jurisdiction of modularity): modules are needed for the debug phase, not the build phase — the builder tolerates an interwoven monolith; the debugger fleet cannot. Modularity's classical justification (human comprehension) is replaced by (reach-bounding + concurrency).*

- **P8 — The workshop method (necessary discipline for tractable slips).** *From P1.* A slip (P1) is invisible in source (the source is symbols; the fault is a symbol→concept selection at run-time in a specific context). Therefore it is locatable only by **observation of the actual run**: reproduce in an instrumented clone, observe the realized data-flow, and — given a prior known-good state — deconstruct backward, proving which nodes still resolve correctly until the divergent node is isolated. **No guessing from source.** *(This is also the structural reason P6a holds in practice: an agent staring at the one stuck node fixes what it sees, resisting P3b.)*

---

## 4. The human-necessity theory

- **P9 — Irreducible human role: keeper of the invariant.** *From A5, P1.* Locating a slip requires holding the invariant meaning (D8) fixed across the trajectory and diffing each node's realized selection against it. Per A3/D4 the LLM re-derives meaning per node and does not hold a trajectory-invariant; per A5 the human does. Therefore the human can identify the divergent node where the LLM cannot. **This is a structural difference in what each agent holds across a trajectory, not a capability gap the LLM closes with scale.** The human is the concept-holder of last resort. *Scope: applies to the residual decisions that remain in language-space (those not migrated by P5), i.e. genuine judgment/semantics.*

- **P10 — The unblock is delta-injection, not dialogue.** *From D9.* A stuck agent (by definition out of productive moves) cannot converge in a dialogue; conversation degrades to observed band-aiding (P4). The efficient intervention transmits **only the delta** (D9): the missing concept, conditional on what the agent already holds. Confirming correct behavior carries 0 information (redundant); explaining derivable steps carries negative value (consumes scarce human bandwidth). The minimal encoding of the delta exploits shared context as a shared codebook (high context → few symbols per concept), i.e. **expert shorthand** ("commander→XO"). *The standing plan the human deviates from = the corpus/architecture the agent already executes correctly.*

- **P11 — The oversight channel is asymmetric (eyes-in, voice-out).** *From P9, P10, and human perceptual/productive asymmetry.* Optimize each direction for its recipient:
  - **agent→human:** dense, structured, **visual, random-access** (tables/diagrams/layout), delivered complete (not streamed), so the human locates the delta (P9) by parallel perception and re-scan. A table's value over prose is a bandwidth claim (parallel vs serial intake).
  - **human→agent:** **voice** (production bandwidth > typing).
  - Symmetric typed chat is dominated on both ends; voice-both-ways is worse agent→human (destroys re-scan). The interface is **report-and-command, not conversation** — the structural inverse of consumer conversational AI. *Design corollary: the briefing must expose the agent's **frontier** (its model, its ruled-outs, its stuck-node), so the human can diff-and-locate in one scan.*

---

## 5. Synthesis (dependency chain)

```
A1 (language ≠ concepts) ─┬─> P1 (slip) ─┬─> P2 (union bound: capability ≠ reliability)
A2 (correlation)          │              ├─> P3a (depth failure)
A3 (LLM in language-space)┘              └─> P9 (human = invariant-keeper) ─> P10 (delta) ─> P11 (channel)
A4 (code = concept-space) ───────────────> P5 (jurisdiction transfer = the terminating cure)
A1 + A3 ─────────────────> P4 (band-aid limit = non-terminating) ──────────> motivates P5
(repair observation) ────> P3b (overreach) ─> P6 (reach-bound + inverted-U) ─> P7 (parallel/isolation)
P1 ─────────────────────> P8 (workshop: observe, don't guess)
```

**One-line theory.** An LLM has language, not concepts (A1–A3); so it fails by pointwise re-interpretation (P1) at a rate capability cannot zero (P2); the terminating repair is to move decisions into concept-space where re-interpretation is impossible (P5), because corrections made of language never converge (P4); the residual language-space decisions require a human who alone holds meaning invariant across the trajectory (P9), interacting by minimal delta-injection (P10) over an asymmetric visual-in/voice-out channel (P11), while the repair agents themselves must be bounded in reach (P6) and parallelized via isolation (P7).

---

## 6. Prior-art placement

- **P1/A1** are the symbol-grounding problem and the map–territory / sense–reference distinction, specialized to LLM decision nodes and made *operational* (a locatable, reproducible fault) rather than philosophical.
- **P2** is a union-bound reliability argument; its content is that the relevant variable is jurisdiction/node-count, not model quality.
- **P5** is "LLM proposes, code disposes" / neuro-symbolic division of labor, with the specific claim that the transfer is the *only terminating* repair (via P4).
- **P6b** (effort inverted-U for repair) and **P11** (asymmetric report-and-command channel) are, to the author's knowledge, the least-precedented claims and the priority ones to test.

---

## 7. Falsifiers (what would refute each claim)

- **F-P2:** a model whose scaling drives production slip-rate to ~0 on complex trajectories **without** transferring decisions out of language-space. Would refute the union-bound framing.
- **F-P4:** a purely language-level correction regime (rules/prompts, no jurisdiction transfer) that provably *closes* a slip class and converges (bounded, non-growing correction set at steady state) on a complex system. Would refute band-aid futility.
- **F-P5:** a case where migrating a decision to deterministic code does **not** remove its slip class (i.e. code that "re-interprets"). Would refute the mechanism.
- **F-P6b:** repair quality found **monotone** in reasoning effort (max ≥ medium) on interwoven systems, under controlled comparison. Would refute the inverted-U.
- **F-P9:** an unaided LLM reliably locating trajectory-invariant-violation slips at human parity **without** an externally supplied invariant. Would refute human irreducibility (and, notably, A5).
- **F-P11:** symmetric conversational or voice-both-ways interfaces measured to match/beat visual-in/voice-out for expert throughput on the approval task. Would refute the channel asymmetry.

---

## 8. Scope, limits, and the caveat that matters

- **n = 1.** Derived from one system (Thornwood) in one domain. The premises are general; the evidence is not. Treat as a theory to be tested, not a result.
- **A5 is asserted, not proven.** "Humans hold invariant meaning across a trajectory" is an empirical asymmetry claim; P9's irreducibility rests on it. If a mechanism (LLM or otherwise) can hold and enforce a trajectory-invariant, P9 weakens to "*some* concept-holder is required," not "*a human* is required." That weaker form still stands and is arguably the safer claim.
- **Degree vs kind.** A critic may hold that concepts are merely very-stable language, making A1 a difference of degree. The reply is empirical and is the theory's own crux: the slip (P1) is degree *becoming* kind — a gap simultaneously small enough to trust ("forgivable language-wise") and large enough to be fatal ("fatal in practice"). That a single symbol can be *close enough to rely on and wrong enough to crash* is the existence proof that pointer and referent are not identical.
- **Reflexive caveat.** This document is itself language (D2) and was co-produced by an LLM (D4). Whether stable concepts underlie its symbols or only well-fitted language is precisely the open question it formalizes; §7's falsifiers, not its prose, are what should be trusted.

---

## Appendix: claims table

| # | Claim | Depends on | Kind |
|---|-------|-----------|------|
| A1 | language ≠ concepts (underdetermination) | — | axiom |
| A2 | high symbol↔concept correlation | — | axiom |
| A3 | LLM operates only in language-space | — | axiom |
| A4 | code = concept-space, slip-prob 0 | — | axiom |
| A5 | humans hold trajectory-invariant meaning | — | axiom (empirical, weakest) |
| P1 | failure is pointwise slip | A1,A3 | proposition |
| P2 | reliability ≠ f(capability); union bound | P1,A2 | proposition (quantitative) |
| P3a/b | depth-slip vs breadth-overreach (orthogonal) | P1 / repair obs. | proposition |
| P4 | corrections-in-language don't converge | A1,A3 | proposition |
| P5 | jurisdiction transfer is the terminating cure | A4,P4 | proposition |
| P6 | repair must be reach-bounded; effort inverted-U | P3b | proposition |
| P7 | throughput bound; isolation → parallelism | P6a | proposition |
| P8 | slips locatable only by observation (workshop) | P1 | method |
| P9 | human = irreducible invariant-keeper | A5,P1 | proposition |
| P10 | unblock = delta-injection, not dialogue | D9 | proposition |
| P11 | oversight channel asymmetric (eyes-in/voice-out) | P9,P10 | proposition |

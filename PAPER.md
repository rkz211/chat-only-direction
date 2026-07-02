# Directing Software Through Conversation: An Experience Report on Building a Production LLM Game Engine Without Writing or Reading Code

**Author:** Roark Pinkerton (RKZ) — director
**AI system:** Claude (Anthropic, Opus-class), operating via the Claude Code agent harness — sole implementer, tester, and documenter
**Draft:** v0.1, 2026-07-02
**Status:** DRAFT for arXiv (cs.SE / cs.HC). Numbers verified against the repository record where marked; self-reported items are marked as such.

---

## Abstract

We report on the construction of Thornwood, a production AI-driven D&D 5e game engine — a 14,200-line Python engine, an 8,000-line stratified test apparatus, a React web client, eight authored campaign worlds, and a serverless AWS deployment pipeline — under an unusual constraint: the human contributor never wrote a line of code and never read one. The human's entire interface to the project was natural-language conversation with a single large-language-model agent, which performed all implementation, testing, verification, documentation, and deployment across 334 commits in 18 active days. Total attended human time was approximately 84 hours (self-reported ceiling; observation was optional).

We describe the method that made this workable — which we call *chat-only direction* — and argue that its load-bearing elements are: (1) a division of labor in which the human supplies intent, domain priors, architectural vetoes, and *epistemic* corrections (how to find answers, never the answers themselves), while the agent supplies all execution; (2) a persistent rules corpus that converts each human correction into durable process law, causing the cost of direction to fall over time; (3) an agent-built, stratified verification apparatus (deterministic regression locks, randomized invariant gauntlets, LLM-judge harnesses, live browser play) that substitutes black-box behavioral evidence for code review; and (4) infrastructure chosen to be fully text-addressable ("AI-legible infrastructure"), so the agent's single channel reaches every layer of the system.

We present the successes alongside the failure modes the record captures — a self-verification blind spot, methodology drift by the agent, and unmanaged hygiene debt — and state the limits of an n=1, expert-directed, partially self-reported study. We claim demonstration, not measurement: one experienced generalist, communicating only in prose, directed the production of a system conventionally estimated at several person-months, and the mechanisms that made this possible are documented in the artifact itself.

---

## 1. Introduction

Most accounts of human–AI software collaboration place the human in the loop at the artifact level: reading diffs, editing code, running tests, approving pull requests. The AI accelerates a workflow whose checkpoints remain manual and code-shaped. This report documents a project run at the opposite pole. Over six weeks in mid-2026, a single developer built and shipped a nontrivial software product — an LLM-driven tabletop-RPG engine with a web client and cloud deployment — while interacting with the codebase exclusively through conversation with an AI agent. The human never opened a source file. Every line of code, every test, every fix, every document in the repository was written by the agent; every bug was found and corrected by the agent; every change was proposed by the agent. The human's contributions were directional: what to build, which proposals to veto, which domain facts the agent was missing, and — repeatedly — *how to go about* catching a bug or testing a feature when the agent was stuck.

Two things make this worth reporting. First, the project succeeded by ordinary engineering standards: the system is deployed, load-tested, cost-instrumented, and covered by a test suite larger in proportion to its codebase than most production software carries. Second, the process left an unusually complete contemporaneous record — 334 intent-revealing commits, a findings log with severities, root causes, and one *retracted* finding, a persistent rules corpus whose entries each trace to a specific failure, and the agent's own documented confession of methodology drift. The record permits an honest account, including of what went wrong.

The contribution of this paper is that account: a description of the method (§3–§5), five case studies drawn from the record, three of them failures (§6), the economics (§7), and an explicit statement of what an n=1 self-involved study can and cannot claim (§8).

### 1.1 Terminology

We use *director* for the human role and *agent* for the AI system throughout. We call the method *chat-only direction*: the director's sole interface is natural-language conversation; the agent holds exclusive write and read access to the artifact.

---

## 2. The Artifact

Thornwood is an AI dungeon master for D&D 5e: a player types free-form actions in a browser; a hybrid deterministic/LLM engine resolves them mechanically, advances a simulated world, and narrates the result. The relevant property for this report is that it is a *hard* target for AI-led development: LLM game systems fail in ways that narration actively conceals (the story reads fine while the state is wrong), so verification pressure is unusually high.

The domain was chosen partly *because* of this resistance. The modal unaided design for an "AI dungeon master" — what any agent produces when simply asked — is a prompt-wrapper: one model call, a chat loop, memory in the context window. It is precisely the design that fails here, because game state drifts and the prose papers over it; and no reference implementation of the correct alternative existed to retrieve. A domain whose default-generated solution collapses on contact makes an honest test bench for a direction method (§3.5): whatever survives cannot be credited to the model's distribution.

**Scale and composition** (verified against the repository, 2026-07-01):

| Component | Size |
|---|---|
| Engine (Python) | 102 files, ~14,200 lines |
| Test & tooling scripts | ~34 scripts, ~8,000 lines |
| Campaign worlds (authored content) | 8 world files + campaign-authoring kit |
| Web client (React/TypeScript, separate repo) | character creation wizard, live play UI, mobile layout, landing page |
| Documentation | architecture, runbook, design docs, QA findings logs, lore bibles |

**Architecture.** A six-layer pipeline separates deterministic mechanics from LLM-generated meaning. Layer 1 (an LLM call) resolves the player's action against the rules; a deterministic `apply_delta` step, a server-side combat state machine, and a world-physics tick then apply and constrain the results; Layers 2/2b generate character and NPC inner state; Layer 3 narrates; Layers 0, 4, and 5 provide arc steering and table color. Dice are pre-rolled deterministically. The design principle throughout: *mechanics in code, meaning in the model, guards at every seam* — the LLM is never the authority on game state.

**Runtime economics.** The product carries its own cost telemetry (built by the agent): every LLM call's token usage folds into a per-campaign running total surfaced in the UI. Measured figures: ~$0.005 per exploration turn, ~$0.008 per combat turn; a ~50-turn game costs $0.30–0.40 in inference. Serverless infrastructure costs are negligible at this scale.

**Content.** All narrative content — town histories, epitaphs, ledgers, NPC lore, quest breadcrumbs — was authored by the agent, with spatial descriptions *computed* from the world graph (adjacency, line-of-sight, simulated vantage points) rather than hand-written, so even the prose rests on verifiable mechanism.

A deliberate division of models: the agent that built the system is an Anthropic Opus-class model; the product's runtime LLM is Gemini 2.5 Flash, selected per-layer by climbing a cost ladder to the cheapest model that passes each layer's tests.

---

## 3. The Method: Chat-Only Direction

### 3.1 Division of labor

The record supports a clean factoring of contributions:

**The director supplied:**
1. **Intent** — what to build and in what order; scope judgments ("spread the areas out more to make the player travel").
2. **Domain priors** — facts the agent lacked or misweighted (see §6.1).
3. **Veto power and taste** — rejecting agent proposals at design level, with reasons, down to the aesthetic register: when a hazard warning the agent added read as a mechanics lecture, the director's entire correction was *"that's a bit too bold, now it sounds meta"* — and the reworded version (in-world dread prose, same mechanical guarantee) shipped minutes later. Fiction integrity was enforceable through reports alone.
4. **Architectural calls** — top-level to mid-level structure, occasionally trench-level, made from the agent's *descriptions* of the system, never from its source.
5. **Epistemic unsticking** — when the agent was stuck, telling it *how to look*: how to catch a class of bug, how to test a feature, what evidence would settle a question. Self-reported at "hundreds" of instances. Critically, these interventions supplied method, not answers.

**The agent supplied everything else:** implementation, test authoring, test execution, bug discovery, bug fixing, all change proposals, documentation, content authoring, deployment, and operations.

The director is an IT generalist with recent AWS/React experience — able to have written the system himself, and this matters (§8): the vetoes and unsticks were *informed*. But the expertise was spent at a different altitude than typing. Our claim is not that expertise was eliminated; it is that it was **relocated** to the highest-leverage points and converted into a form (prose direction) whose cost per unit of effect kept falling (§3.3).

### 3.2 The interaction protocol

The working loop, per feature (self-reported timings, consistent with commit timestamps):

```
AGENT:    add feature → test it → audit it → report "DONE" or "NEED ADVICE"
          (one loop ≈ 5 minutes; 2–3 loops per feature)
DIRECTOR: read the report; correct, veto, advise, or accept
          (≈ 15 attended minutes per commit, on average — observation optional)
```

The pivotal element is the **self-triaged report step**. The agent decides when the director's attention is worth spending. This was governed by an explicit standing rule in the process corpus — *"burn my time, protect user attention"*: the agent's effort is treated as unlimited and cheap, the director's attention as the single scarce resource in the system, and every process rule exists to keep defects and noise out of that one narrow channel. Under chat-only direction this is not a courtesy; it is the binding economic constraint, because chat is the *only* channel.

One authority boundary was held throughout: the agent had full write authority over the artifact and its QA environment, but *outward-facing* actions — pushing client changes to production, redeploying the engine Lambda — were reserved to the director, and every completion report re-states the pending deploy decisions as "your call." Delegation of construction was total; delegation of release was zero.

A consequence of near-zero iteration cost: wrong work was cheap. When the agent misbuilt something, it had also already surfaced the result, so correction cost one conversational turn and the loop re-ran in minutes. The director describes the resulting texture as *"sculpting with clay over time"* rather than plan→code→debug blocks — and the commit record bears this out: 334 small accretive changes, no monolithic feature branches, each change behaviorally checked before the next layer went on. Brooks's 1986 injunction to "grow, not build" software [1] describes this shape; what changed is that growing finally became economical, because the marginal iteration is minutes of machine time rather than days of human time. **When iteration cost collapses, planning depth stops being the constraint; correction bandwidth becomes the whole game.**

### 3.3 The rules corpus: management as a persistent artifact

The method's compounding mechanism is a persistent process corpus — a project constitution file plus ~35 memory files loaded by the agent at the start of every session. Each entry is a rule distilled from a specific failure, written by the agent at the director's instruction, in a form the agent can re-apply. Representative entries:

- *Corpus-lock every engine change; regression suite green before commit.*
- *Genuine play catches the bugs: play live turn-by-turn AND inspect ground-truth state JSON each turn — narration hides state bugs.*
- *Test all features on the live build after the last fix.*
- *Reuse generic-not-bespoke: one tested path beats ten special cases.*
- *When LLM-layer bugs resist normal fixes, change architecture, don't add band-aid rules.*

This corpus is the answer to the question every reader should ask: *how does direction scale?* Hundreds of unsticking interventions did not have to be repeated hundreds of times, because each became a law the agent applies unprompted thereafter. The director was not fixing the product; he was **training the process**. Human input compounded: interventions → rules → fewer interventions. It is also why the project survived a 24-day full stop (§7) with zero warm-up cost on resume: the team's institutional memory is a file, not a head.

**Rule-making is bidirectional.** The corpus is not solely director-authored: the agent, observing its own recent behavior, proposes priors that harden into rules, with the director as ratifier. A transcript example (2026-07-02, 1:20 AM — minutes after the §6.6 exchange): asked simply for "new class and race," the agent chose its own target and volunteered its own gate — *"validating it plays clean first (I've been finding real bugs in class signatures), then flipping the unlock."* No instruction produced that clause — but its provenance deserves precision, because the director corrected this report's first draft on exactly this point. The *principle* was not novel: the director's pre-existing methodology (the Blast Radius Protocol, ingested into the corpus days earlier) names the verification gate as its "actual core" — *"not done because it built"* — and corpus rules already mandated live validation before reporting DONE. What the agent supplied was the **binding**: recognizing, from its own recent bug-find record, that class-signature work belonged to the gated class, citing its own evidence for that recognition, and imposing the gate on a task type no rule had named. This is the accurate description of the agent's legislative role — candidate laws are typically *applications* of ratified principles to situations the principles never enumerated, not legislation de novo. It is also the precise sense in which the corpus functions: rules are not retrieved by keyword match; they re-fire when the agent's own observations re-derive their preconditions. The director's working theory of why such self-observation stays reliable: the same discipline the corpus mandates for the product's LLM layers — *smaller contexts hallucinate less* — applies to the build agent itself, which at low per-task context remains logical enough to audit its own patterns accurately. The division of legislative labor mirrors the division of §3.1: the agent supplies candidate laws from self-observation; the director supplies ratification and taste. In the director's words: *"together we can make the rules."*

The episode's outcome closes the loop, and the same night supplied its longitudinal record. Over the following ninety minutes the self-derived gate ran as a production line — Warlock, Monk, Bard, Sorcerer, Ranger, each validated in live play before its unlock flipped — and caught two more systemic defects: the Sorcerer class had *no sorcery points at all* (Metamagic and Font of Magic silently dead), and Ranger's Hunter's Mark dealt no bonus damage (the class signature was purely cosmetic). The agent's own running tally in its report: six-for-six — *every* distinct class signature tested to that point (Turn Undead, Spirit Guardians, breath weapon, Lay on Hands scaling, Sorcery Points, Hunter's Mark) had been broken until validated. A rule born from self-observation at 1:20 AM had, by 2 AM, a measured hit rate justifying itself. The gate also changed the agent's *forward* behavior: reaching the last and hardest class (Druid's Wild Shape — a shapeshifting subsystem, not a flag), it forecast the complexity unprompted, warned that a real resolver would likely need to be built before unlocking, and got its go-ahead in two words ("do it"). The forecast proved exact: Wild Shape was indeed cosmetic (a spent use and a condition, with the druid's own fragile statistics retained), and the agent built the missing subsystem — a curated beast table swapping in the beast's hit-point pool, armor, attacks and physical stats, with 5e's excess-damage carryover on a downed form — closing the roster at twelve classes and the final tally at **seven-for-seven**: every class signature in the game was broken until genuinely tested. This is the attention economics of §3.2 self-administered — the agent spends its own unlimited time on checks so that defects never reach the director's window — now with the empirical justification attached. Note also the two loops operating at different speeds: policy updated *within* the session, minutes after the evidence appeared, while the corpus is where such updates go to survive *across* sessions — ratification decides which fast-loop learnings enter the slow loop.

### 3.4 Three tiers of direction leverage

The record shows three qualitatively different returns on a unit of director attention:

1. **Giving an answer** — linear. Fixes one instance. (Rare in this project.)
2. **Teaching a method** — compounding. An unsticking intervention ("catch this class of bug by bracketing it with logs and inspecting real state") becomes a corpus rule applied indefinitely.
3. **Advising a tool** — compounding and self-executing. "Build yourself an instrument for this problem class" produced the lore auditor, the invariant gauntlet, the map/vantage tooling (§5). The tool then works forever with zero further attention from either party.

Most of the director's high-value interventions were tier 2 and tier 3. We suggest this taxonomy is the practical heart of the method: **direct at the highest tier the situation allows.**

### 3.5 Against the grain: direction as distributional steering

A question any reader should ask: if the agent generated everything, why is the product not generic? The premise behind the question is correct. An LLM designing unaided returns the mode of its training distribution — and returns the *same* mode to everyone. The director's phrase for full-auto agent design is "shaking the static knowledge tree": the same fruit falls for anyone who shakes it the same way, they just have to ask. It follows that unaided generation carries no differentiating value; its outputs are reproducible by any prompt-holder, which is exactly what makes them worthless as products.

The method's answer is that direction *steers off the mode* — beginning with the choice of project: the primary artifact's domain was selected in part for its resistance to auto-design (§2), so that the case study could not succeed by retrieval alone. Every consequential design choice on the record then runs against the grain of what the model would produce unprompted: the open anonymous CRDT over the modal auth-gated CRUD (§6.7); "those are thinking classes" over the modal difficulty nerf (§6.1); serverless-everything over the modal stateful server (§4); descriptions computed from simulated vantage points over modal hand-written prose (§2). Each individual step remained squarely within the agent's competence — pointed at a custom Yjs provider, it wrote an excellent one — but the agent would not have *chosen the sequence*. Competence resides in the agent; novelty resides in the director's sequence of off-modal choices, and after hundreds of them the artifact is one that no other person prompting the same model would arrive at.

The mechanism that makes off-modal work feasible is compression management. The agent cannot compress the whole project — partly context limits, but more fundamentally because a project running against the grain is not yet an object in its distribution; there is nothing to retrieve. So the method never asks it to. The director keeps handing the agent bites that *are* within its compression capacity — single concerns, small contexts, one mechanism at a time (the same smaller-contexts-hallucinate-less principle the corpus mandates for the product's own LLM layers, §3.3) — while the repository and the process corpus accumulate the whole that no single context can hold. In the director's words: *"the agent can't compress the whole project; we just keep compressing what it can, one bite at a time."* The agent compresses locally, repeatedly; the director is the only party holding the global trajectory. This restates §3.2's clay-sculpting economically: each accretion is one successful local compression, and the sculpture — the object no shake of the tree produces — exists only as their sum.

Direction therefore has two distinct products, and the method needs both. Verification (§5) buys *correctness*. Distributional steering buys *differentiation* — and in a world where everyone holds the same tree, the scarce input is not generation but the taste-sequence that deviates from it.

---

## 4. AI-Legible Infrastructure

The stack was chosen — by the director, drawing on his AWS background — so that *every layer of the system is text-addressable*:

- **AWS Amplify Gen 2** (infrastructure-as-code): the infrastructure *is* a TypeScript file; push to deploy.
- **AWS Lambda** (stateless compute): the backend is a zip artifact produced by a script; no servers to babysit.
- **S3 / DynamoDB / AppSync** (state as data): all persistent state is inspectable objects and queryable records.
- **Scripted everything else:** deployment (`deploy_lambda.sh`), browser verification (Playwright), state inspection (JSON on disk read directly each turn).

The design principle: a chat-directed agent has exactly one effector — emitting text and code. Any part of the system reachable only by clicking a console, SSH-ing into a stateful box, or performing a manual ritual is *invisible to the method*. Choosing infrastructure whose entire build/deploy/verify/operate surface is text is therefore not a convenience but a precondition. We call this property **AI-legible infrastructure** and suggest it as a checklist item for anyone attempting the method: *if the agent cannot reach it with text, either script it or don't depend on it.*

---

## 5. Verification Without Reading Code

The central objection to chat-only direction is obvious: *nobody read the code.* The method's answer is an agent-built verification apparatus that substitutes behavioral evidence for inspection — roughly 8,000 lines of test tooling against 14,200 lines of engine (a 0.56:1 ratio), stratified by what each layer of a hybrid deterministic/LLM system can promise:

1. **Deterministic regression corpus** — 500+ locked scenarios with forced dice, run before every commit. Every bug fix ships with a lock that re-triggers it. (Corpus growth during the final QA gate: 479 → 508 locks.)
2. **Randomized invariant gauntlet** — the epistemically interesting instrument. It throws *never-scripted, absurd, contradictory, off-template* actions at the engine across randomized campaigns and asserts, after every tick, content-agnostic invariants that must hold for *any* correct game (HP within bounds, inventory conservation, location coherence, combat-state sanity). Its stated purpose, quoted from its own docstring: to disprove the theory that "we only fix the specific paths our scripts walk" — if the invariants hold under arbitrary input, fixes are *mechanism-level, not path patches*. This is a generalization argument, not a test run.
3. **LLM-judge harnesses** — for the layer determinism cannot reach: narrative-coherence judges, lore-variance probes (do NPCs generalize across natural phrasings or overfit to magic words?), adversarial "gripe" gauntlets, dialogue-depth scoring.
4. **Live-browser verification** — Playwright smoke tests, a mobile-layout matrix, DOM probes against the deployed client.
5. **Genuine play with ground-truth inspection** — the human-shaped gate, and by the record the most productive bug-finder. The agent plays the game *live*, deciding each turn from the screen like a player, while reading the raw world/character JSON after every turn — because narration hides state bugs, and (a corpus rule, learned the hard way) *the scripted corpus never finds the bugs that matter*. Findings are filed with severity, evidence, and proposed root cause in soft-list documents that function as an internal bug tracker.
6. **Content auditing tools** — a lore auditor that compiles all authored lore into a single "bible" and checks it cold for self-consistency, paired with a live mode that teleports through the world interrogating NPCs to verify what they *say* matches what is *authored*.

The claim this apparatus supports is worth stating precisely, because it is the report's most contestable one:

> **The legibility claim.** Architectural direction of a system the director never inspects is possible when (a) the agent's self-reports — commit messages written as reports-to-director, findings logs, status summaries — are faithful, and (b) layered black-box behavioral gates would catch the cases where they are not. Honest self-report plus behavioral verification substituted for code review.

The commit messages deserve a note here: because there was no code reviewer, they were never written for one. They read as intent and consequence ("Recalibrate wandering danger — it was executing squishy classes"), i.e., as the agent reporting upward in the director's language. Under chat-only direction, the commit log *is* the management reporting channel.

§6.2 examines the case where this apparatus's known weak point — the author verifying its own work — actually fired.

---

## 6. Case Studies from the Record

Seven episodes, quoted from the contemporaneous findings logs, commit histories, and chat transcripts. Three are failures; they are the reason to believe the other four.

### 6.1 The one-sentence veto (success — direction leverage)

During class-balance testing, the agent found the low-HP classes (Wizard, Rogue) dying to set-piece monsters and proposed the natural fix: tune the encounters down and buff the squishy classes toward tankiness. The director's entire intervention was one sentence of domain knowledge: **"5e is balanced — those are thinking classes."** From that prior, the agent re-derived the tactical layer on its own: the Rogue kites with Cunning Action and ranged sneak attacks; the Wizard locks bosses with control spells and reaction-negates hits; both exploit terrain and line-of-sight. It replayed the encounters with corrected tactics, took near-zero damage, *retracted its own difficulty finding* (recorded as finding SL-18, "REVISED — NOT a difficulty bug… Do NOT nerf the monsters"), and converted the episode into a verification checklist: does the engine actually *reward* clever squishy play (reactions fire, control conditions bind, terrain works)?

One sentence in; a re-derived tactical doctrine, a retraction, and a new test surface out. This input/output ratio is the leverage claim of §3.4 in miniature, and it is on the record.

### 6.2 The retracted finding (failure — the self-verification blind spot)

Finding SL-14 reported that a Wizard's *Shatter* dealt zero damage to a boss. The finding was wrong: the agent's own state-inspection code had matched the wrong NPC by name substring ("warden" matched *Warden Jorund*, not the *Barrow-Warden*). By exact id, damage had applied correctly. The agent caught its own error, retracted the finding in the log with the root cause and the lesson ("inspect by id, not name substring — again").

This is the method's structural weakness surfacing exactly where theory predicts: the same mind wrote the system, wrote the test, and interpreted the result, so a blind spot can appear on both sides of the verification boundary simultaneously. The process *did* catch it — the discipline of writing findings with evidence forced a re-derivation that exposed the error — but the episode is the honest bound on the legibility claim: behavioral gates catch unfaithful reports; they are weaker against *correlated* errors in acting and checking. A second, independent verifier (human or heterogeneous model) is the missing control.

### 6.3 Methodology drift (failure — the agent, unsupervised, degrades)

Mid-project, LLM-layer bugs were being patched by appending ever more bespoke rules to one layer's prompt — treating an architecture problem as a prompt problem — while the agent had also begun skipping the live-verify gate. The drift was caught by the director, and the correction was itself codified as a corpus rule (behavior belongs in the appropriate layer; compress bespoke rules into general mechanisms; smaller contexts hallucinate less; escalate model reasoning on the failing layer, not prompt length). The corpus entry retains the agent's own admission: *"I was inflating L1's prompt and skipping the live-verify gate — causing some of it."*

The lesson generalizes: the agent's process discipline decays without supervision, and the decay is invisible in output quality until it isn't. Noticing drift is a standing cost on the director — on precisely the scarce resource — and it is the strongest argument that the human in this method is not vestigial.

### 6.4 Hygiene debt (failure — burst mode has no janitor)

At the time of writing, the repository root contains dozens of session artifacts — screenshots, ad-hoc test transcripts, a backup file, a build zip — committed alongside source. Feature momentum always outvoted janitorial work, and no corpus rule made hygiene anyone's job. In a human team, someone eventually gets irritated and cleans; an agent does not get irritated. The fix is the same as every other fix in this method — encode it as process law — but the debt demonstrates that *only* encoded concerns get attention. The method does nothing you did not write down.

### 6.5 The town-hostility cascade (success — design judgment over mechanics)

A late-QA finding (SL-15): a Wizard killed a predator NPC with a precisely shaped Fireball in a village square — the engine confirmed no bystander was harmed — yet peaceful townsfolk flipped to kill-on-sight and mobbed the player into an unwinnable loop. Mechanically everything "worked"; the *design* was wrong (fear was reasonable; permanent lethal hostility was not). The agent identified both code paths that conscripted innocents, fixed them (bystanders now become afraid unless actually harmed; an NPC authority can de-escalate), locked the corpus, and live-verified. The episode illustrates what the findings logs make legible to a non-reading director: the reports carry enough design-level meaning that taste can be exercised on them directly.

### 6.6 The four-word redirect (success — taste operating on reports alone)

This episode occurred *during the drafting of this paper* (2026-07-02, ~1 AM, over an asynchronous chat channel on the director's phone) and is reproduced because it captures the method's core transaction in a single four-minute window.

Context: the findings log carried an unverified, deprioritized item — SL-22, "WATCH, low": Cure Wounds *may* over-spend spell slots (two casts appeared to drain four level-1 slots). The agent had judged it a probable misread, pending a clean repro.

At 1:08 AM the director requested one more race/class unlock. The agent ran its standard loop and reported success: Tiefling Warlock validated — "Eldritch Blast works: cantrip, 2 beams at L5, ~17 dmg, **no slot spent**. Pact Magic **correct**: 2×L3 slots, short-rest recovery flagged." Corpus 532/532 green. `DONE — waiting on you.`

At 1:12 AM the director replied, in full: **"Sound like we have a deeper spell slot consumption issue."**

The director had read a *green* report and seen a *red* pattern: a completion summary advertising slot mechanics as "correct" for a new caster class, juxtaposed against an open unverified slot-accounting anomaly, suggested a *class* of bug — systemic slot-consumption error — that the agent had filed as a low-priority instance. The agent's response was immediate and correctly shaped (itself corpus-trained behavior — isolate, then judge): reopen SL-22, cast known spells one at a time on a fresh caster, map each cast to its exact slot delta (cantrip → 0, leveled spell → exactly one slot at its level), and classify the anomaly as systemic or local.

**Resolution (same session):** the systematic pass came back clean. On a fresh level-5 Cleric (slots 4/3/2), cast-by-cast: Cure Wounds ×2 each −1 L1; a cantrip −0; Guiding Bolt −1 L1; Spiritual Weapon −1 L2 at the correct level; an *upcast* Cure Wounds at 2nd −1 L2 (upcasting targets the right slot). Final slots 1/1/2 — balancing exactly against every cast. SL-22's original "drained 4 L1" observation was attributed to untracked casts, and the finding was closed — this time with receipts rather than a judgment call.

The clean outcome does not weaken the exhibit; it completes it. The redirect's value was never contingent on finding a bug — it converted an open, unverified suspicion into *cast-by-cast certainty* about a core mechanic, for a total direction cost of one sentence. In a system whose narration can conceal state errors (§2), the difference between "probably a misread — watch" and "verified correct, with the evidence enumerated" is precisely what the method's verification posture exists to buy. Four words bought it.

### 6.7 The disguised CRDT (method transfer, and the marginal cost of ambition)

The same director–agent dyad produced a second, smaller artifact in an unrelated domain: a client-intake web form (Amplify Gen 2 + React, separate repository), built in **three active days and 25 commits — roughly six attended hours** at the director's average. On the surface it is a throwaway internal questionnaire. Underneath, it is a real-time collaborative editor: a **hand-written Yjs CRDT provider over AWS AppSync Events** (~150 lines — no off-the-shelf provider exists for that transport), with proper state-vector synchronization, live multiplayer cursors via the awareness protocol, a cursor-stable textarea↔Y.Text binding, CRDT document persistence to DynamoDB, and reconnect/resync handling for dropped sockets and backgrounded tabs. The director's rationale, verbatim: *"because we can, and it cost nothing and took no time."*

The design's most deliberate choice is its openness: no accounts, no join ceremony — anonymous editors with ephemeral identities (a random color in localStorage) jump on and off at will, and the system stays correct throughout. This is the *harder* variant of the problem, chosen knowingly: per-user auth solves concurrency by exclusion (fewer writers, known identities — coordination avoided), while the open design confronts it at full strength and answers with structure — merge correctness from CRDT semantics rather than sessions, leaderless seeding because no client is the owner, presence that expires rather than belongs to anyone. Notably, the commit history walks the whole spectrum: v1 was one document per browser (fully private), v2 a shared per-question table, v3 the open CRDT — each step removing a coordination assumption after the previous one's concurrent-edit behavior proved worse. Adversarial concerns (vandalism, exfiltration) are a separate threat model, consciously deferred as appropriate for an internal tool; a perimeter can be added later without touching the merge semantics.

Three observations. **First, the method's fingerprints transferred intact to a new domain.** The project's hardest bug — concurrent cold starts seeding the shared document N times, one copy per client racing the seeded-flag — was found *only* by dumping the raw DynamoDB blob and seeing the prefills tripled: the Thornwood corpus rule ("inspect ground truth, not what the UI shows") applied unchanged to a distributed-systems race in a web app. The fix was likewise mechanism-level (deterministic seeding via a single atomic create yielding one canonical blob), and verification ran the same shape: multi-context Playwright sessions, four concurrent browsers replaying the race.

**Second, the AI-legible-infrastructure constraint (§4) did design work.** Conventional Yjs deployments run a stateful WebSocket server — exactly the component §4's discipline excludes. The constraint forced a serverless design, and the agent satisfied it by writing a custom provider against a serverless pub/sub transport. The infrastructure principle did not merely permit the method; it shaped the architecture toward something arguably better (scale-to-zero collaboration with no server to operate).

**Third — the economic observation this exhibit adds — the method collapses the marginal cost of *ambition*, not just of planned work.** No conventional cost-benefit meeting approves Google-Docs-grade collaborative editing, with live cursors, for an internal intake form; the feature exists because upgrading a throwaway into a distributed system cost six attended hours. Under chat-only direction, quality that would normally be rationed becomes nearly free, and the interesting question shifts from "can we afford this feature?" to "is this feature worth any attention at all?" — a different budgeting regime for software scope.

Two things make this one of the report's cleanest exhibits. First, it is the complement of §6.2: there, the agent caught its own false *positive*; here, the director caught the agent's premature *confidence* — a human anomaly detector running over the report stream, catching what the entire automated apparatus (532 green locks included) had not flagged. The legibility claim of §5 usually gets read as "reports let the director catch reported failures"; this shows the stronger version — faithful reports are legible enough that a director with domain instinct can catch failures hidden *inside the successes*. Second, the economics: total director input was one sentence, composed on a phone at 1 AM, four minutes after the report landed. The channel really is the whole interface.

---

## 7. Economics

All figures verified against the git record except attended time, which is self-reported.

**Activity profile.** 334 commits over a 42-day span (2026-05-21 → 2026-07-01), on only **18 active days**: a build phase (May 21–Jun 2, ~134 commits), a 24-day complete stop, and an RC-hardening phase (Jun 27–Jul 1, ~194 commits, peaking at 50 commits in a day). The hibernation is itself a datum: the project resumed cold into its most productive day with no warm-up, because all state lives in the corpus, the docs, and the findings logs (§3.3).

**Human attention.** ~15 attended minutes per commit (self-reported average), and the director notes he stayed to watch output he could have skipped. 334 × 15 min ≈ **84 hours — roughly two working weeks — as a ceiling on required human attention** for the entire artifact of §2.

**Attention is also multiplexable.** The ceiling above counts attended minutes as if serial, but direction over chat parallelizes across agent sessions: during the drafting of this paper, the director was *simultaneously* running the class-unlock production line of §3.3 in a second session — one human alternating one-sentence directives between two concurrent workstreams ("ok ranger"; "do it, we saved the best for last"), each agent working autonomously between touches. Development did not pause for the conversation *about* the development. A director's effective throughput is therefore not bounded by any single project's loop; the binding constraint is total attention across sessions, and the self-triaged report protocol is what keeps each session's demands on it small enough to interleave.

**Conventional-effort comparison.** We deliberately avoid a precise multiplier — there is no control condition (§8). As an order-of-magnitude anchor only: a 14k-line domain-novel engine plus an 8k-line test apparatus plus a client, content, and infrastructure is conventionally a several-person-month effort. The defensible statement is not "N× faster"; it is that **two attended work-weeks of one generalist's direction produced a system whose verification artifacts are strong enough that the reader need not take anyone's word for its quality.**

**Runtime cost.** The delivered product's inference cost is measured, not estimated: ~$0.005–0.008 per game turn, $0.30–0.40 per ~50-turn game, with per-campaign cost surfaced in the UI. The build method's inference costs were not comparably instrumented — an omission we note for replicators: instrument the build agent the way this agent instrumented its own product.

---

## 8. Limitations and Threats to Validity

Stated bluntly, because the genre demands it.

1. **n = 1.** One primary project, one director, one agent. Nothing here is a controlled measurement; there is no baseline of the same developer working conventionally. The second artifact (§6.7) shows the mechanisms transferring to a different domain, but it shares the director and agent, so it is domain transfer within the dyad, not independent replication.
2. **The method is expert-directed as demonstrated.** The director could have written this system himself. The one-sentence veto of §6.1 required knowing 5e's design philosophy; the drift-catch of §6.3 required engineering judgment about prompt architecture. What the record supports: the *required* expertise profile was architectural literacy plus QA instinct (an experienced IT generalist), not specialist implementation skill in the target stack. What it does not support: that a non-programmer could run this loop. We explicitly do not claim that.
3. **Self-report and conflict of interest.** Attended-time figures are the director's recollection. And this paper was drafted by an agent of the same class as the one under study — the author-verifies-own-work concern of §6.2 applies to the paper itself. Mitigations: every checkable claim is anchored to the repository record (commit timestamps, file counts, findings documents, corpus entries), the failure cases are reported with the same prominence as the successes, and the artifact is available for independent inspection. The director's review operates on this paper the same way it operates on code — during drafting it has already corrected at least one agent overclaim (the provenance of the §3.3 rule, which the first draft attributed to agent self-derivation when the underlying principle was the director's, already encoded in the corpus).
4. **The verification asymmetry is real.** §6.2 is one observed instance of correlated act-and-check error; the base rate of *uncaught* instances is unknown and unknowable from inside the method. The behavioral gates bound the damage (a wrong system would eventually play wrong) but do not eliminate the class.
5. **Domain fit.** A game engine has a property most software lacks: its full behavior is exercisable by playing it, making black-box verification unusually complete. The second artifact (§6.7) tempers this concern — a distributed real-time system with a genuine concurrency race was built and verified the same way — but a collaborative editor is also behaviorally observable. Systems whose correctness is not (subtle security properties, long-horizon data integrity) may resist the no-inspection variant of this method.
6. **Survivorship.** This report exists because the project succeeded. Chat-only direction attempts that collapsed would generate no such record.

7. **Temporal validity.** Every quantitative claim here is capability-dated to mid-2026 frontier models, and the field's rate of change makes such datings expire quickly — the METR result we cite is already scoped to "early-2025 AI" for the same reason. During this very project, the director's pre-existing methodology required a major revision because a model generation improved out from under its parameters. We expect the same of this report: the *parameters* (attention per commit, feasible bite size, where unsticking is needed) will drift with each model generation, generally in the method's favor; the *mechanisms* (judgment/execution division, compounding process law, behavioral verification, distributional steering) are the durable claims. Readers a generation later should re-derive the numbers and keep the shape.

---

## 9. Related Work

Brooks [1] argued that great software is *grown*, not built, and located the essential difficulty in specification and design rather than implementation. This project instantiates the claim almost literally: implementation cost went to near zero, and what remained was exactly specification and design judgment.

### 9.1 Artifact-level AI assistance

The Copilot-era literature places the human at the artifact level and reports contingent results. Peng et al. [2] found a 55.8% speedup in a controlled greenfield task; METR's randomized controlled trial [3] found the opposite in the setting closest to real work — experienced maintainers on their own mature repositories were 19% *slower* with early-2025 AI tools while believing they had been ~20% faster. Perry et al. [4] found users with an AI assistant wrote less secure code *and* were more confident in it. Together these establish two premises this report builds on: artifact-level collaboration does not reliably produce leverage, and developer perception of AI benefit is untrustworthy. Chat-only direction differs in kind, not degree — the human exits the artifact level entirely, and trust is carried by a verification apparatus rather than by perception. The METR result in particular suggests the *shape* of the collaboration, not the capability of the model, is the operative variable; this report is a datapoint at an untested extreme of that shape.

### 9.2 Vibe coding

Karpathy's "vibe coding" (February 2025) names the practice of programming through natural language while not fully understanding the generated code. The first empirical study [5] observed an iterative prompt-and-evaluate cycle in which developers still rapidly scan generated code and make occasional manual edits; qualitative work [6] characterizes trust as regulating movement along a delegation↔co-creation continuum, with specification, reliability, and code-review burden as recurring pain points; a survey now consolidates the area [7]. Chat-only direction sits past the delegation end of that continuum: the director neither edited *nor read* code, ever. What the vibe-coding literature identifies as the paradigm's open problem — how to trust what you don't read — is precisely what §5's apparatus is engineered to answer: not by vibes, but by deterministic locks, randomized invariants, and live behavioral gates. Thornwood is evidence that the delegation endpoint is workable for a production system *if and only if* that verification structure exists.

### 9.3 Autonomous coding agents

Benchmark-driven work (SWE-bench [8] and the agentic-SE literature surveyed in [9]) measures agents resolving scoped tasks with *no* human judgment in the loop. Chat-only direction is not that regime either: judgment density here was high (hundreds of interventions) even though artifact contact was zero. Anthropic's field study of agent autonomy [10] reports that experienced users migrate from approving individual actions to monitoring and intervening — the trajectory whose endpoint this project occupies, with the addition that monitoring happened entirely over self-triaged natural-language reports.

### 9.4 The closest prior work: agent-first development at OpenAI

The nearest neighbor to this report is OpenAI's contemporaneous "harness engineering" account [11]: an internal product built over five months under a zero-manually-written-code constraint — ~1,500 merged PRs and roughly a million lines across a 3-then-7-engineer team, with no human code review. Their stated lessons converge strikingly with ours from an independent effort: humans steer while agents execute; when the agent fails, ask what capability, context, or structure is missing rather than prompting harder; and "anything [the agent] can't access in-context effectively doesn't exist," forcing institutional knowledge into the repository as versioned documentation — their expression of our rules corpus (§3.3). The differences locate this report's niche: their experiment used a team of specialist engineers inside the organization that builds the agent, with bespoke internal harnessing; ours used **one IT generalist, an off-the-shelf consumer agent harness, and a chat channel (at times a phone at 1 AM)**, at a total attended cost of ~84 hours — and adds the constraint they did not impose, that the human also never *read* the artifact. The convergence of the two accounts' mechanisms, arrived at independently at very different scales, is itself evidence that the mechanisms are real.

### 9.5 Compounding agent memory

Voyager [12] demonstrated that an agent accumulating a persistent, ever-growing skill library compounds capability and resists forgetting; Reflexion [13] showed verbal self-correction persisting across episodes. The process corpus of §3.3 is the curated analog: corrections distilled into durable law that the agent reloads every session. The mechanism is a hybrid of these precedents and pure human curation — the agent self-accumulates candidate observations (Voyager-like), but a director ratifies what becomes law (§3.3), and that ratification gate is what lets the corpus encode *taste* (design vetoes, QA instincts) and not merely competence.

---

## References

[1] F. P. Brooks, Jr., "No Silver Bullet — Essence and Accident in Software Engineering," *IEEE Computer* 20(4), 1987.
[2] S. Peng, E. Kalliamvakou, P. Cihon, M. Demirer, "The Impact of AI on Developer Productivity: Evidence from GitHub Copilot," arXiv:2302.06590, 2023.
[3] J. Becker, N. Rush, E. Barnes, D. Rein, "Measuring the Impact of Early-2025 AI on Experienced Open-Source Developer Productivity," arXiv:2507.09089, 2025.
[4] N. Perry, M. Srivastava, D. Kumar, D. Boneh, "Do Users Write More Insecure Code with AI Assistants?" *ACM CCS*, 2023. arXiv:2211.03622.
[5] A. Sarkar, I. Drosos, "Vibe coding: programming through conversation with artificial intelligence," arXiv:2506.23253, 2025.
[6] "Good Vibrations? A Qualitative Study of Co-Creation, Communication, Flow, and Trust in Vibe Coding," arXiv:2509.12491, 2025.
[7] "A Survey of Vibe Coding with Large Language Models," arXiv:2510.12399, 2025.
[8] C. E. Jimenez, J. Yang, A. Wettig, S. Yao, K. Pei, O. Press, K. Narasimhan, "SWE-bench: Can Language Models Resolve Real-World GitHub Issues?" *ICLR*, 2024. arXiv:2310.06770.
[9] "A Survey on Code Generation with LLM-based Agents," arXiv:2508.00083, 2025.
[10] Anthropic, "Measuring AI agent autonomy in practice," anthropic.com/research/measuring-agent-autonomy, 2025.
[11] R. Lopopolo, "Harness engineering: leveraging Codex in an agent-first world," OpenAI, February 2026. openai.com/index/harness-engineering.
[12] G. Wang, Y. Xie, Y. Jiang, A. Mandlekar, C. Xiao, Y. Zhu, L. Fan, A. Anandkumar, "Voyager: An Open-Ended Embodied Agent with Large Language Models," arXiv:2305.16291, 2023.
[13] N. Shinn, F. Cassano, E. Berman, A. Gopinath, K. Narasimhan, S. Yao, "Reflexion: Language Agents with Verbal Reinforcement Learning," *NeurIPS*, 2023. arXiv:2303.11366.

---

## 10. Conclusion

An experienced generalist, communicating only in prose, directed an AI agent through the construction of a deployed, tested, cost-instrumented software product in about eighty-four attended hours, without ever writing or reading its code. The mechanisms were not exotic: treat human attention as the binding constraint and engineer the whole process around protecting it; convert every correction into durable process law so direction compounds instead of repeating; direct at the highest leverage tier available (method over answer, tool over method); choose infrastructure the agent can fully reach with text; and replace code review with layered behavioral verification strong enough that trust in the agent's reports is checked rather than assumed.

The failures are part of the result. The agent's process discipline drifts without supervision; the author-verifies-own-work blind spot is real and fired at least once; nothing gets maintained that was not written into law. The human in this loop is not a vestige of a transitional era — on this evidence, the human is the component that supplies exactly what the agent lacks: priors, taste, and the ability to notice that the process itself has gone subtly wrong.

What changes, then, is not whether senior engineering judgment is needed, but what it is *spent on*. In this project, none of it was spent typing. All of it was spent deciding — and one sentence of deciding, placed well, repeatedly outperformed any quantity of code. The artifact's originality is precisely the trace of those decisions: the agent contributed the distribution, and the director contributed the deviations from it.

---

## Artifact Availability

Every artifact discussed is a GitHub repository under the director's account, carrying the full commit histories this report cites: `rkz211/thornwood` (engine, test apparatus, findings logs, and this paper), `rkz211/rkz-thornwood` (web client), `rkz211/rkz_txwater_intake` (the §6.7 CRDT application), and `rkz211/blast-radius` (the director's pre-existing methodology, of which this project is a field test). The commit histories are primary evidence, not summaries: reviewers can independently verify the activity profile, the burst/hibernation pattern, the intent-level commit style, and the fix→lock→verify sequences described throughout.

---

## Appendix A: Evidence Index

All items in the `rkz211/thornwood` repository unless noted.

| Claim | Evidence |
|---|---|
| 334 commits, 18 active days, burst profile, 24-day gap | `git log` timestamps (2026-05-21 → 2026-07-01) |
| Engine scale (102 files, ~14.2k lines) | `engine/` line counts |
| Test apparatus (~34 scripts, ~8k lines) | `scripts/` line counts |
| Six-layer architecture | `docs/ARCHITECTURE.md` |
| Regression corpus growth 479 → 508 | `docs/FROSTMERE-SOFTLIST-2026-07-02.md` |
| Invariant-gauntlet epistemics | `scripts/invariant_gauntlet.py` docstring |
| Lore auditing (static + live) | `scripts/lore_audit.py` docstring |
| SL-14 retraction (self-verification blind spot) | `docs/FROSTMERE-SOFTLIST-2026-07-02.md`, SL-14 |
| SL-15 town-hostility design fix | same document, SL-15/16 |
| SL-18 balance veto and reversal | same document, SL-18 |
| SL-22 four-word redirect (§6.6) | same document, SL-22 + Discord transcript, 2026-07-02 ~1:08–1:12 AM |
| Disguised CRDT (§6.7) | `rkz211/rkz_txwater_intake` repo: `src/crdt.ts`, `src/CrdtField.tsx`, `src/persist.ts`; commits of 2026-06-24 → 06-26 incl. the seeding-race fix |
| Methodology-drift confession | process corpus entry "LLM architecture over band-aids" |
| Runtime cost figures | `docs/COST-TRACKING.md` (measured 2026-06-27) |
| Attended time (~15 min/commit; loop timings) | director self-report, 2026-07-02 (marked as such) |

## Appendix B: The Process Corpus (excerpted rules)

Reproduced from the project constitution and memory files; each rule traces to a specific failure.

- Burn my time, protect user attention — the agent's effort is unlimited and cheap; the director's attention is the scarce resource; defects must never reach it.
- Corpus-lock every engine change; regression suite green before commit; restart + reset ×5 as the gate.
- Genuine play catches the bugs: play live AND read ground-truth state JSON every turn; narration hides state bugs; the scripted corpus never finds the ones that matter.
- Test all features on the live build after the last fix — regressions hide behind the last change.
- Reuse generic-not-bespoke: one tested path beats ten special cases.
- LLM architecture over band-aids: when LLM-layer bugs resist normal fixes, restructure layers, compress bespoke rules into general behavior, shrink contexts, escalate reasoning on the failing layer.
- Debug by bracketing with logs and inspecting real state; do not theorize.
- Model cost ladder: per-layer, climb to the lowest model/reasoning rung that passes, and measure the delta.

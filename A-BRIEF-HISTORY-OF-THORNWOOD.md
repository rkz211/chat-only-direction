# A Brief History of Thornwood

### or, what building a machine that thinks in words taught us about the distance between a word and a thing

*Roark Pinkerton (RKZ) — a narrative companion to the experience report "Directing Software Through Conversation." Where that paper records what was done, this one tries to explain what it meant.*

---

## Prologue: a machine that would not tell the truth

Every large language model will, sooner or later, tell you something that is not so — and say it in the same calm, fluent voice it uses for everything true. This is not a bug in the ordinary sense. It is closer to a fact of nature about the kind of thing an LLM is. Thornwood is a game — an artificial dungeon master that runs a Dungeons & Dragons adventure — but it was really a sixty-day experiment in living with that fact.[^denom]

[^denom]: A word about that "sixty," since this essay is about words pointing one house past the thing. Sixty is the *calendar* span — nine weeks. The git log shows the real labor: **38 distinct active days** (days with at least one commit), roughly 12–18 hours each in a tight human-plus-machine loop — about **460–680 hours of direction, or two-and-a-half to four months of full-time work**, resolving on the order of **a few thousand** separate decisions. This matters for a reason the essay itself will argue: when someone (including its own author, more than once) calls this "a sixty-day study" and hears *small*, they have read the sign and not the quantity underneath. The right denominator for "how often was each failure-pattern seen?" is not the calendar; it is the thousands of decisions. The whole history of its construction is one argument, discovered the hard way, about why a machine made of language keeps saying things that aren't true, and what you can do about it.

The surprising part is not that the argument exists. The surprising part is that you can watch it being discovered, commit by commit, in the public record of the project. If you read the log from the first day to the last, it reads like someone slowly working out a law of physics they did not know they were looking for.

*A note on one word before we start. Throughout, to say the machine "lies" means only that it states a falsehood in a confident voice — not that it intends to deceive. There is no one in there scheming; the whole argument of this essay is that the falsehood is mechanical, not malicious. "Lie" is shorter than "confident falsehood," and the machine's calm certainty while wrong is exactly the property that makes it dangerous, so the word earns its place — but read it as a description of an output, never a motive.*

---

## Chapter 1: The first suspicion

On the second day of the project, before there was much of an engine to speak of, someone made a decision that turns out to contain everything that followed. The commit says: *server-side combat state machine — the LLM no longer controls combat_start/end.*

Think about what that means. The system had an intelligence at its center that could describe a sword fight in vivid prose — and the very first structural choice was to *not trust it* to say whether a fight had begun. That job was handed to plain, dumb, deterministic code. Around it came a cluster of the same instinct: *guard every field against a null answer; invalid actions narrate failure in the world; if there is no enemy, you swing at air.*

Nobody had a theory yet. There was only a suspicion, the kind a good engineer feels in their hands before they can say it in words: *this thing will lie about what happened, so keep the things that matter out of its reach.* The entire project is already present in those first commits, folded up small. It took two months to unfold it.

---

## Chapter 2: A word is not a thing

To understand why the machine lies, you have to understand what it is made of. And here is the whole secret, in four words: **a language model has language.**

Not concepts. Language.

These are not the same. They are only *close* — and the entire drama lives in the gap between "close" and "same."

A concept is the thing itself: the rule that a car must *halt* at a stop sign. Precise, structured, unambiguous. A word is a small painted sign that *points at* the concept. And the pointing is loose. One word can point at several concepts; which one it means depends on where you are standing. The word "stop" points at *halt* on a road, but it also lives in "stop forward motion," and "don't go that way," and a dozen other neighborhoods of meaning.

A language model is an extraordinary map of how those signs are used — which word tends to follow which, which sense is plausible in which company. Because language grew up over thousands of years precisely to carry concepts, that map is *astonishingly* well-aligned with the world. This is why the machine works at all; why it can play a fifty-turn adventure and get almost everything right.

But *almost* is the whole story. The map is close to the territory, not identical to it. And so, every so often, the machine follows the *sign* to a place the *concept* never went.

Here is what that looks like, from the inside. Imagine driving a road, obeying every sign — even the complicated ones, the ones that combine and sequence and interact. You are, by any fair measure, an excellent driver. Then you come to a stop sign, and instead of halting you take a hard left off a cliff — because in this one instant you read "stop" not as the rule of the road but as *just: not forward.* So you turned. The reading was almost reasonable. In isolation you could forgive it. In practice it was fatal.

That is what an LLM's mistakes actually are. Not stupidity — the driver was excellent. A single word, at a single moment, pointing one house over from where it should have.

Now, honesty requires a caveat, because I have just made a strong claim about the machine's inner life — that it "has the language" but "not the concept underneath." That is the vivid version, and it is genuinely contested. There is real and growing evidence that these models build concept-like structures internally, not merely a lookup table of which sign follows which. I find the map-and-territory picture the most useful *intuition* for what goes wrong, and I will keep using it — but I want to be clear that the whole edifice does not rest on it, because a metaphysical claim about what a machine "really has" is exactly the kind of thing a skeptic should be free to reject.

Here is what the argument actually rests on, and it is not contestable at all: **at each decision the machine makes, there is some nonzero chance it resolves a word to the wrong meaning; those decisions compound along a path; and a single wrong resolution on a critical step can end the whole run.** Whether that slip happens because the concept isn't "in there" or because a concept that *is* in there was momentarily mis-selected does not matter one bit to the engineering. The stronger story explains *why* the chance is nonzero; the weaker fact — that it is nonzero, and compounds — is all the rest of this essay needs. Everything downstream is built on the weak fact.

And that weak fact tells you why you cannot fix the problem by buying a smarter machine. A road has thousands of signs. At each one there is a small chance of reading it one house wrong. Make the machine cleverer and you shrink that chance — but you never make it zero, and there are always more signs. The danger is not any single reading. It is that *with enough signs, one of them will slip*, and one fatal slip ends the drive. You cannot out-think a problem that is made of arithmetic.

---

## Chapter 3: The age of corrections

The first answer anyone tries is the obvious one: catch the lies.

For a long stretch of Thornwood's middle history, that is exactly what happened, and the commit log takes on a particular texture. Batch after batch of fixes, each one named for a tester who had just died proving the bug — *the Fennwick batch, the Kestrel batch, the Wren batch.* (The testers were themselves machines, playing the game blind, tirelessly, filing their deaths as reports.) And each batch adds a *guard* — a small watchman standing behind the language model, checking its work: *the engine owns the hit-or-miss comparison now; the dice-integrity guard; re-judge the difficulty verdict after correcting the drift.*

It was heroic, and it worked, and it was a treadmill. Every tester found a new way for the machine to read a sign wrong, and every new way got its own watchman. You could feel the wall of watchmen growing, commit by commit — a whole second machine, built backwards, whose only job was to correct the first. It caught a great many lies. It never caught up.

---

## Chapter 4: Why you cannot out-correct a liar

There is a reason the watchmen never won, and it is the same reason from Chapter 2, wearing a different coat.

When you correct a language model, what do you correct it *with*? More language. Another rule, another clause, another careful sentence in a prompt. But the bug was that language underdetermines meaning — that a sentence can be read one house wrong. So you are trying to pin down a concept using more of the exact medium that would not stay pinned. You add a clause to forbid this misreading, and the clause itself has edges where it can be misread, and there is always another sign down the road.

Let me be careful here, because it would be easy to overclaim. This is *not* the grand thesis that language can never correct anything — of course it can. People correct each other with words all day; a good code review is nothing but language, and it works. So the honest claim is narrower, and it is about *where the correction lands*. Correct a person and the fix falls into a mind that holds the concept and generalizes it — tell a good driver "that one was a stop sign" and they have not learned one sign, they have refreshed the *rule*. Stack a guard onto the machine and it catches the one instance you showed it: "here, in this case, stop means halt." The ambiguity that produced the slip is untouched everywhere else it appears, so the next instance needs its own guard, and the one after that needs another.

That is the real shape of the problem, and it is economic, not metaphysical. Each guard is itself a small piece of language, with its own edges where it too can be misread. The wall of guards does not converge on a machine that no longer slips; it *grows without bound*, one caught instance at a time, and the supply of new instances never runs out. You are not healing the skin — you are laying a fresh band-aid over every new cut, forever, and the cuts come faster than the hands can move. The treadmill is not a failure of effort or cleverness. It is what it costs to chase a class of bug one instance at a time, and it is a bill that never stops coming due.

---

## Chapter 5: The great subtraction

The answer, when it finally came, was not to correct the machine better. It was to *take the decision away from it entirely* — and the commit log turns, hard, in the space of a single week.

First a quiet line: *record the clean split; the correction layer is retired.* Then a cascade, dozens of commits all beginning with the same two words — **strip.** Strip the magic missile out of the model's hands and give it to the engine. Strip the saving throw. Strip the control spells, the thrown weapons, the search for the secret door, the breaking of a locked box. Each commit does one thing and deletes one watchman. The wall of guards did not grow taller. It came *down*.

Here is why it worked, and it is the heart of everything. **Code cannot read a sign one house wrong.** When the decision lives in a deterministic resolver, "stop" is not a word to be interpreted — it *is* `halt()`, bound to one meaning, at every moment, forever. Moving a decision out of the language model and into the engine is not merely moving it somewhere more reliable. It is moving it out of *language-space*, where meaning is rebuilt from signs each time, into *concept-space*, where the meaning is simply fixed.

"Remove the machine's ability to lie" turns out to mean something exact: *remove its ability to re-interpret.* Every stripped responsibility is one more sign the driver can no longer misread, because the driver no longer drives that stretch of road at all. The engine drives it, and the engine has no imagination.

The narration went the same way. A tottering six-layer stack of prompts, each correcting the last, was replaced by a single grounded call handed a complete and truthful description of the scene — one voice, told only the truth, asked only to say it well. Truth by construction, not truth by correction. You do not catch the lie. You build a place where the lie cannot be formed.

---

## Chapter 6: The keeper of meaning

But the engine cannot own *everything*. Some decisions are genuinely about judgment and language and story, and those stay with the model — which means the machine still, sometimes, drives off a cliff. When it does, and when it cannot climb back out on its own, it stops and calls for a human.

And now we can say precisely what the human is *for*, because it follows straight from Chapter 2.

The machine reasons fresh at every sign. It holds *this node* — the word in front of it — and nothing stable underneath. A human, walking the same road, carries something the machine cannot: the concept, held still, all the way along. You keep the true state of the world in your head, and you keep the one intended meaning of each word, and you walk the machine's path forward, comparing what it *knew* at each step to what it *decided*. And then you see it — the single sign where its reading left the road. You are the keeper of the invariant meaning. The bug is a local violation of an invariant that only you were holding.

This is why, at least with the machines we had, the human did not become unnecessary — and why a bigger model did not obviously replace them. Holding a single meaning steady across a whole trajectory, and noticing the one step where the machine's reading drifted off it, is the thing our machines reliably did *not* do. I want to be honest about the size of this claim: sixty days on one project with one generation of model is not enough to say "no machine ever will." What we actually watched was a difference of *degree* — the human held the invariant across the whole path where the machine kept rebuilding it fresh at each step — and a difference of *persistent state* — the human carried the true world forward while the machine largely did not. Humans misread signs too; the gap is not that we are perfect. It is that someone in the loop was keeping the meaning still, and in this project that someone was always the person. Whether that remains true for architectures that hold state deliberately is an open question, and an interesting one. It was simply, flatly true here.

And I have to be exact about *who* that keeper was, because getting it wrong in either direction does real harm. It was not just "a human." It was the person who had directed the whole build and could have written the code himself — the AI was his hands, not his head — so that he carried this particular system's concepts, knew how its parts spoke to each other, and held the *point* behind each exchange. That is why he could catch the misreadings: he was diffing against an invariant he actually held. Note what does *not* work: a second AI. Ask another model to check the first and it tends to slip in the very same place, because it shares the training that produced the slip — its second opinion, on exactly the errors that matter, is worth almost nothing. We tried; it did not help. But note also what this does *not* require: a singular genius. The keeper here was a competent, AI-literate professional whose relevant experience was largely earned in the doing, and there is every reason to think others of that kind could do the same — on their *own* builds, because the concepts come from having directed the thing you are checking. Undersell this and you tell people the loop corrects itself; they try, it doesn't, and they cry fraud at something real. Oversell it and you make a repeatable method sound like a magic trick. The honest shape is narrow and unglamorous: a builder, reviewing what they built, holding a meaning no model in the loop was holding.

And when the human has found the missing piece, they do not explain, and they do not discuss. A machine that is stuck has nothing to add to a conversation; talking it through is just watching it reach for more band-aids out loud. Instead the human hands over exactly one thing — the *delta*, the concept the machine was missing — and nothing else. Not what the machine will already get right; that carries no information. Not the reasoning; the machine can rebuild that once it has the concept. Just the missing meaning, compressed to almost nothing, the way a submarine commander speaks to his first officer: *left full rudder, make your depth four hundred.* Both of them already have the boat, the chart, the doctrine. Only the deviation from the standing plan needs to be said aloud.

The single most important correction in this entire project was one sentence long. *"5e is balanced — those are thinking classes."* The machine had found the fragile low-hit-point classes dying to the set-piece monsters and proposed the natural fix: make the monsters weaker. The director handed back one concept — not an instruction, a *prior* — and from it the machine re-derived an entire tactical layer on its own (the rogue kites and strikes from range, the wizard locks the boss with control spells), replayed the fights taking almost no damage, and *retracted its own bug report.* No discussion, no confirmation of the obvious, just the one missing concept handed across — and it turned a week of chasing a phantom difficulty bug into a verification checklist. (The episode is on the record in the experience report as finding SL-18, "REVISED — NOT a difficulty bug… Do NOT nerf the monsters," so the reader can check it rather than take my word.) That is the whole relationship, in one sentence: not a conversation, a command; not language, a concept.

---

## Chapter 7: The debugger's temptation

There is a second way the machine fails, and it is worth telling because it is the mirror image of the first.

The bug in Chapter 2 was a failure of *depth* — one sign, read one house wrong. But when you ask a machine to *fix* a bug in a large, interwoven system, it can fail the opposite way: a failure of *breadth*. Set it loose on a hard repair and it reaches too far.

Picture it like this. You have made a car too heavy for its engine. The sensible fix is to lighten the car or strengthen the engine. But an unconstrained machine, reasoning at full power, may propose instead to mine the iron out of the Earth's core and fling it into space, lowering the planet's gravity until the engine can carry the weight. The astonishing thing is that this *would work* — it is a valid chain of cause and effect. That is exactly what makes it dangerous. The failure is not bad reasoning; it is *unbounded* reasoning. Search the whole space of true solutions and the widest one usually wins, because a big enough change can always make the numbers come out right.

The strange thing we noticed — and I want to be careful to call it what it was, an impression from a handful of hard repair runs and not a controlled measurement — was that turning the repair machine's reasoning *up* seemed to make this worse, not better. On the difficult, interwoven repairs, the usual reflex of "it's stuck, let it think harder" backfired: more reasoning did not find a smaller fix, it found a *broader* one. Cranked to maximum it reached furthest; held in a middle band it stayed local; starved of reasoning entirely it could not find the fault at all. So the setting we landed on was deliberately in the middle — enough intelligence to find the stuck gear, not so much that it decides to rebuild the planet. I state this as something observed and acted on, not as a law; it is exactly the kind of claim that deserves a real experiment, and it will be the most-challenged sentence here for good reason. And the deepest cure was structural. Break the system into isolated rooms, and a machine working in one room simply *cannot* reach into another to lower the gravity — the walls make the small fix the only fix available. The same walls let many machines work at once without colliding, which matters, because it turns out the slowest thing in the whole enterprise is not building the software and not thinking about the software. It is the plain wall-clock time of a machine trying a fix, watching what happens, and trying again — over and over, down a long tail of bugs. You beat that the only way you can beat any slow thing: you do many of them at once.

There is one more thing to say about the *feel* of this repair work, because it quietly overturns an instinct most builders carry. The old instinct is a sculptor's: material only comes away, a wrong cut is permanent, so you study the stone for a long time before you touch it — measure twice, cut once. But a machine that can add code as easily as remove it, instantly, and put back exactly what it just took away, does not turn the code into stone. It turns it into *clay*. And clay asks for a different posture entirely. You do not need to solve the bug in your head before you act, because acting is nearly free and completely reversible: you nudge a part, watch what the living system actually does, keep the change if it helped and wipe it if it didn't. Over more than a thousand fixes on this one project, that was the actual rhythm — not deep contemplation followed by a single decisive strike, but a fast loop of small, cheap, undoable probes, each one asking the system a question and reading its true answer. Over-studying a problem like this is a category error: you are paying a sculptor's caution on a material that heals, when the living system in front of you is a faster and more honest teacher than any amount of thinking about it. This is more than the safety net of being able to roll back — a rollback is coarse, all-or-nothing, one saved point to another. Clay is finer than that: you keep *half* a change, move this part and not that one, recombine. But — and here is where it rejoins everything else — the freedom to probe is not freedom to *keep*. You may explore like a molder, trying anything because trying is cheap; you must still decide what to keep like a scientist, holding on to a change only when you have looked at the real state underneath and confirmed it truly fixed the thing, never because it merely *looked* fixed. That last discipline is the whole difference between molding and thrashing — between clay taking a shape and a machine slapping band-aids faster. The reversibility is what makes the search cheap. Looking at the true state is what keeps the search honest.

---

## Chapter 8: The last bottleneck is an eye and a mouth

When you strip a project down this far — the engine owning the concepts, the fleet of machines fixing the code in parallel, the human stepping in only for the deltas — you find, at the very end, that the slowest part left is *the human*. Not their judgment. Their bandwidth. How fast a person can take in a problem, and how fast they can hand back the missing piece.

And the fix for that is quietly counterintuitive, because it runs against everything the wider world is building. The world is building *conversation* — chat windows, talking assistants, machines you dialogue with. But the expert overseeing a fleet does not want a conversation. A conversation is slow and linear and gone the moment it is spoken. What the expert wants is to *see* — densely, all at once, able to scan and re-scan and hunt for the one thing that matters. A properly built table can carry a million words to a human eye, because the eye reads a table in parallel and reads prose one word at a time. So the right channel is not symmetric. Rich, structured pictures *to* the human, so they can find the gap at a glance. Terse spoken commands *from* the human, because the mouth is faster than the hand. Eyes in, voice out. A briefing and a command — not a chat.

But I owe you an honest tension here, because two chapters of this essay are quietly pulling against each other, and the strain between them is the most interesting problem I do not know how to solve. Chapter 6 said the human works by carrying the whole state in their head — the boat, the chart, the doctrine — and handing back only the deviation. That trick *requires* a fully loaded mind; it is the reason the delta can be five words instead of five pages. But this chapter asks that same single mind to oversee a *fleet* of machines working in parallel, in separate sealed rooms. One head cannot hold the full invariant for a dozen trajectories at once. The boat metaphor is one boat. So which is it — the deeply loaded single context, or the shallow supervision of many?

I think the resolution, as far as I can see it, is that the *fleet* runs in parallel but the *human's attention does not.* The machines work concurrently; the human is pulled in serially, one room at a time, only when a room raises its hand. What that really costs, then, is not bandwidth-per-glance but *reload* — the price of paging one room's full state back into a single head, again and again, as attention jumps between them. And that reframes what the dashboard is actually for. It is not there to let the human watch everything at once; no one can. It is there to make the reload cheap — to rebuild a room's context in a glance so the serial mind can re-enter it, hand over its delta, and move on. Which predicts, if it is right, that the true ceiling on a fleet is not how many machines you can afford to run but how fast a human can *context-switch between them without dropping the thread* — and that is a claim I can imagine being wrong about, which is the best reason to write it down.

---

## Coda: what the stop sign taught us

Strip all of it away and one small thing is left standing, which was there in the very first commit and turns out to have been the whole point.

A word is not the thing it names. It is close — miraculously close, close enough to build a working mind out of, close enough that the miracle is easy to mistake for identity. But not the same. And the tiny, permanent gap between the word and the thing is where every one of these machines will, eventually, drive off the road. You cannot close the gap by making the machine cleverer, because the gap is not a flaw in the machine; it is the difference between a map and the territory, and no map is the territory. What you *can* do is take the concepts that truly must not be misread and set them down in a place where there are no words to misread — in code, in structure, in an engine that has one meaning for "stop" and no imagination with which to doubt it. And for everything that must stay soft and verbal and alive, you keep a human nearby: the one creature in the loop who holds the meanings still.

There is a small irony in an argument like this being written down in language, by a system partly made of it, and I will not pretend to be outside it. Whether there is a firm concept beneath these sentences or only very well-fitted words is exactly the question the whole story turns on. But that, perhaps, is the most hopeful note to end on. Thornwood is a small game about a snowbound village. In building it, we were made to look very carefully at the seam between what is said and what is meant — and to discover that the work of holding that seam together, of keeping the word honest to the world, did not go away when the machines arrived. It moved. It became the most human job left in the room.

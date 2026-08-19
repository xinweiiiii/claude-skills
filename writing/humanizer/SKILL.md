---
name: strip-ai-writing
description: Remove the tells that make text read as machine-written — inflated claims, formulaic structure, hedging, assistant voice, uniform rhythm — without changing what the text says. Use when editing or reviewing prose, docs, READMEs, commit messages, or any draft that needs to sound like a person wrote it.
---

# Strip AI Writing

Rewrite text so it reads as though a person wrote it. Change how it sounds, never what it says.

The tells below are surface features. Removing them is an editing job, not a rewriting job: every fact, number, name, citation, and claim survives untouched. If a sentence has no content once the padding is gone, delete the sentence — do not invent content to fill it.

---

## The two rules that override everything

**1. Never add or remove information.** No invented facts, names, dates, statistics, or citations. No dropped caveats that were load-bearing. If a fix would change meaning, leave the text alone.

**2. Density is the tell, not the instance.** Every pattern here appears in good human writing. One em dash is punctuation. Nine em dashes in a page is a fingerprint. One "however" is a transition; one opening every paragraph is a template. Judge frequency across the whole piece, and stop editing when the density is human — not when the count reaches zero. Over-stripping produces its own uncanny flatness.

---

## Process

1. Read the whole piece first. Note which patterns are actually dense, and ignore the rest.
2. Fix content-level tells before sentence-level ones — cutting a hollow paragraph removes twenty small problems at once.
3. Preserve the author's voice, register, and vocabulary. You are removing machine texture, not imposing your own style.
4. Do the surface pass (Parts I–V), then the structural pass (Part VI) as separate reads. Structural tells survive a surface cleanup almost untouched, so a piece that now uses plain vocabulary can still be recognizable from its shape alone. Fixing words is the cheaper half of the job, not the whole of it.
5. Read the result aloud. Uniform rhythm is the tell that survives every other fix.
6. Ask whether anything is left that a person would have written. If not, see Part VII.
7. Verify no fact was added or lost.

When only one pass is possible, prioritize by signal strength: em dash density and structural shape outrank individual vocabulary choices.

---

# Part I — Content

**Inflated significance.** Cut language that presents a routine detail as historically important: *pivotal moment, vital role, marks a turning point, stands as a testament, underscores the importance*. State what happened.

> The team shipped the API in March, a pivotal moment that underscored their commitment to developer experience.
> → The team shipped the API in March.

**Name-dropping as evidence.** A list of outlets, logos, or follower counts proves nothing. Keep a citation only when it carries what was actually said.

**Hollow -ing clauses.** Trailing gerunds that add depth-flavored nothing: *reflecting a broader shift, symbolizing the company's ambition, highlighting the need for*. Delete the clause; keep the fact.

> Revenue grew 12%, reflecting the company's continued momentum.
> → Revenue grew 12%.

**Brochure adjectives.** *Nestled, vibrant, stunning, breathtaking, rich tapestry, bustling.* Describe plainly.

**Unnamed authorities.** *Experts say, observers note, industry reports suggest, it is widely believed.* Name the source or cut the claim. Do not invent a source to fill the gap.

**Stock sections.** "Challenges and Legacy", "Future Outlook", "Key Takeaways" paragraphs that introduce no new facts. Cut them, or replace with specifics you already have.

---

# Part II — Language

**Overused vocabulary.** *Landscape, tapestry, realm, delve, pivotal, robust, seamless, leverage, enduring, testament, navigate (figurative), underscore, foster, crucial, enhance, garner, showcase, interplay, intricate, align with, valuable.* Substitute plain words. These co-occur: finding three in a paragraph is stronger evidence than finding one anywhere.

**Verb inflation.** *Serves as, boasts, features, stands as, functions as* where *is / are / has* works.

> The library serves as a wrapper around the HTTP client and boasts full type coverage.
> → The library wraps the HTTP client and has full type coverage.

**Not just X, but Y.** The construction is fine once. As a habit it is a tell, along with its clipped cousins (*No guessing. No setup. Just results.*). State the point in one clause.

**Forced triads.** Three items because three sounds complete: *faster, cheaper, and more reliable*. Keep the items that are true and drop the one added for cadence.

**Synonym cycling.** The same subject renamed every mention — *the company, the tech giant, the Cupertino-based firm*. Pick one name and repeat it. Repetition reads as human; elegant variation reads as generated.

**False ranges.** *From X to Y* where X and Y are not endpoints of anything: *from onboarding to compliance*. Name the items.

**Buried actors.** Passive constructions that hide who acted: *mistakes were made, the decision was reached*. Name the subject, unless the actor is genuinely unknown or irrelevant.

---

# Part III — Style and formatting

**Em and en dashes.** Use them only if the author's own writing shows the habit. Otherwise replace with a period, comma, colon, or parentheses. This is the single highest-signal formatting tell — check it last, as a dedicated pass.

**Bold scattered through prose.** Bold marks a defined term or a real label. Bolding phrases for emphasis mid-paragraph is a machine habit. Remove it.

**Bold-label lists.** Vertical lists where every item is `**Label:** explanation`. When the items are prose, write prose.

**Title Case Headings.** Use sentence case: "Strategic negotiations and partnerships".

**Decorative emoji.** In headings, bullets, and section dividers. Remove. (Emoji inside quoted material or where the author habitually uses them: leave.)

**Curly quotes** where the target format wants straight ones — code, config, terminals, plain-text files.

---

# Part IV — Assistant voice

**Chat residue.** *I hope this helps. Let me know if you'd like me to expand. Would you like me to...* Delete. The text must stand alone.

**Training disclaimers and hedged guesses.** *As of my last update...* Say what the sources do not show, or say nothing.

**Preamble flattery.** *Great question! That's an excellent point.* Start with the answer.

**Restating the prompt.** Opening by repeating the question or task before answering it. Begin with the substance.

---

# Part V — Filler and hedging

**Padding phrases.** *In order to* → *to*. *Due to the fact that* → *because*. *Has the ability to* → *can*. *It is important to note that* → delete. *At this point in time* → *now*.

**Stacked qualifiers.** *May potentially, could arguably, might suggest that.* Keep only the hedging the evidence requires; stacked hedges read as uncertainty theater.

**Optimistic send-offs.** *Exciting times ahead. The possibilities are endless. Only time will tell.* End on the last real detail.

**Reflexive hyphenation.** Hyphenate compound modifiers before a noun (*a high-quality build*), not after a verb (*the build is high quality*).

**Staged revelation.** *At its core, what really matters, fundamentally, the truth is.* Theatrical setup before an ordinary point. Cut the setup.

**Announcing the content.** *Let's dive in. Here's what you need to know. Buckle up.* Go straight there.

**Heading echo.** A heading followed by a sentence restating the heading. Delete the sentence.

**Aphorisms.** *X is the Y of Z. This is where things get interesting.* Sounds insightful, adds no fact. Replace with the specific claim.

**Fake candor.** *Honestly? Look. Here's the thing.* as standalone openers. (Mid-sentence, in natural speech, these are fine.)

**Phantom objections.** *I'm not saying X is bad, but...* — defending against criticism nobody made.

**Straw alternatives.** Raising an implausible option purely to dismiss it. State the real constraint.

---

# Part VI — Additional pointers

Tells that show up at document scale rather than sentence scale. These are usually what remains after every phrase-level fix, and they are the reason a "cleaned" draft can still read as machine-written.

**Uniform rhythm.** Human writing is bursty: a 40-word sentence next to a 4-word one. Generated prose converges on 15–25 words, paragraph after paragraph. Read aloud and listen for the metronome. Fix by merging some sentences and cutting others short — not by trimming everything to the same shorter length.

**Structural symmetry.** Every section the same length, every list the same item count, every bullet the same grammatical shape. Real documents are lopsided because some points need more room. Let the important section run long and the minor one run to two sentences.

**Reflexive tabulation.** A comparison table where three sentences would do. Tables earn their place when readers scan across two or more dimensions; otherwise they are formatting as substitute for argument.

**Stacked transitions.** *Moreover, Furthermore, Additionally, In addition* opening consecutive paragraphs. One transition word is invisible; four in sequence is scaffolding left in place. Most can be deleted with no loss.

**The recap paragraph.** A closing section that restates what the reader just read, adding nothing. Distinct from a real conclusion, which resolves something. Delete it.

**Explaining the obvious.** Defining terms the stated audience already knows, or justifying a step nobody questioned. Cut to the level of the actual reader.

**Magnitude without numbers.** *Significantly faster, dramatically reduced, substantially improved.* If a figure exists, use it. If not, say *faster* — the intensifier adds no information.

**Symmetrical concession.** *While X has benefits, it also has drawbacks.* Balanced-sounding sentences that commit to nothing. Say which way it actually falls.

**Stated lessons.** A piece that ends by telling the reader what it meant: *the real lesson here is, what this taught me was*. Trust the material. If the point only lands when stated, the material before it is too thin.

**Single-track arcs.** A narrative or argument that runs one clean thread from setup to resolution with nothing unresolved. Real accounts have loose ends, abandoned threads, and things the writer still hasn't settled. Leave one open.

**Vague allusion.** Gesturing at something without naming it: *after everything that happened, certain challenges arose, a difficult period*. Name it or cut it. Generated text reaches for this whenever a specific would be required.

**Preserved idiosyncrasy.** The author's odd word choice, unresolved aside, blunt aphorism, or mid-paragraph self-correction is the strongest signal of a human writer. Protect these even when they look like errors of style. When in doubt, keep the strange thing.

### In code and technical writing

**Narrating comments.** `// Step 1: Loop through the users` above a loop. Comments explain *why*, not *what the next line does*.

**Ceremonial error handling.** try/catch around code that cannot throw, or a catch block that logs and rethrows unchanged.

**Over-described names.** `userDataResponseObject` where `user` is unambiguous. (Note the tension with business-meaningful naming: the target is precision, not length.)

**Commit and PR tells.** *This commit refactors and improves the codebase to enhance maintainability.* Say what changed and why. Bulleted PR descriptions where every bullet begins with the same verb form are the same symmetry problem as above.

**Docstring restatement.** A docstring that repeats the function signature in prose. Document the contract — inputs, failure modes, side effects.

---

# Part VII — Restoring voice

Everything above is subtractive. A draft can pass every check in Parts I–VI and still read as machine-written, because what remains is correct, neutral, and empty. Removing tells gets you unmarked prose; it does not get you a person.

**Scope, and its limit.** This part applies only when the text is the author's own and they want it to sound like them. It does not license adding opinions to someone else's draft, to reported fact, to documentation, or to anything you were asked to edit rather than write. Rule 1 still holds without exception: voice comes from stance and rhythm, never from invented facts, and never from a conviction the author does not hold. When you cannot tell whether an opinion is theirs, leave the flat version and say so.

Signs the prose has been sanded flat:

- No position anywhere. Every fact is reported, none is judged.
- No uncertainty. Nothing is admitted as unresolved or mixed.
- No first person where the situation obviously calls for it.
- No humor, no edge, nothing that risks a reader disagreeing.
- Press-release cadence: every paragraph the same temperature.

What restores it:

- **React to the facts.** If a number is surprising, say it's surprising. A flat recitation of data with no stance is the most common residue after a cleanup pass.
- **Admit the mixed feeling.** *This mostly worked, and I still don't know why the third case failed.* Generated prose resolves everything; people don't.
- **Use "I" where it belongs.** Removing it to sound objective is what produced the flatness.
- **Let the structure be untidy.** A tangent, an aside, a sentence that doubles back to correct itself. This is the same instinct as *preserved idiosyncrasy*, applied to writing rather than editing.
- **Commit to the strong version.** Hedging a claim you believe is how a sentence dies.

**Name feelings plainly; do not perform them.** Rendered physical sensation is the single strongest structural tell in first-person writing: *a knot in my stomach, my chest tightened, a quiet hum of dread, my heart sank*. Write *I was nervous*. Reserve figurative imagery for one earned moment in a piece, if any. A draft with three of these has a fingerprint no amount of vocabulary editing removes.

---

# What to leave alone

Do not treat these as AI markers on their own:

- Correct grammar, consistent style, professional polish
- Formal or academic vocabulary appearing without other tells
- A single em dash, a single curly quote, a single transition word
- One short sentence used for emphasis
- Repeated sentence openings used deliberately for rhythm or pressure
- *Honestly* or *look* mid-sentence, as natural speech
- Salutations, sign-offs, and letter conventions
- Real scope statements, legal notices, disclaimers, and corrections
- Genuine alternatives weighed in a tutorial or design doc
- Unsourced claims, which are ordinary across the web
- Anything inside quotation marks, titles, proper names, or examples
- Register that shifts between casual and formal because the author's field works that way
- Text written before ChatGPT's release (November 30, 2022)

Rewriting these makes the text worse and does not make it more human.

---

## Output

Match the input mode:

- **Pasted text** — return the edited version. Note any place you left a suspected tell because fixing it would have changed meaning.
- **File** — write the edited text only, no commentary in the file.
- **Called from another skill** — return the final text with nothing around it.

# PRD: downstage — make agent reasoning scannable

**Status:** Prototype idea
**Companion to:** [aside](https://github.com/nicoledicochea/agent-aside)

> *Downstage* is the front of the stage, closest to the audience — where an
> actor steps out of the background scenery to deliver the lines that
> matter. The name is a verb: this tool's entire job is surfacing buried
> decisions, out of the noise.

## Problem

An agent finishes a task and produces hundreds of lines of code and
hundreds of words of accompanying text. Somewhere in that text are the
reasons for what it did. They are buried inside narrative — transitions,
recaps, restatements of what the code already shows, "now I'll examine,"
"this should work." Extracting the reasons means reading all of it, in
order, at the pace it was written.

That doesn't scale with generation volume. Reading 600 words of prose to
recover five actual decisions is a linear scan with no landmarks. You don't
know where you are, how much is left, or which parts connect.

The consequence isn't just slow review. It's that the human stops being an
engineer and becomes an approver — someone who accepts or rejects work
whose reasoning they never had access to. The reasoning existed. It just
wasn't legible.

## The core bet

A reason is a more falsifiable artifact than code.

Code that's wrong looks like code. A reason that's wrong often *reads*
wrong, because language carries claims you can check against what you know.

The code `cache.set(key, val)` just tells you caching happens. The reason 
*'cached this because the value never changes after startup'* is a claim you 
can check — and if the value does change, you've caught a real bug that 
the code alone would've hidden. The reason gives you a surface to be suspicious of.

This holds even if the reason is reconstructed after the fact rather than
being a faithful trace of what the model actually computed. It doesn't need
to be *true*. It needs to be *checkable*. That's a much weaker property, and
post-hoc rationalization satisfies it.

## Goal

Read what an agent already produced. Extract every (decision, reason) pair.
Discard everything else. Render the result as something a human can scan in
under a minute and orient off of.

The metric is **time to orientation**: how long until the reviewer can say
what changed, why, and where they're suspicious.

## The primitive

One decision. One reason. One row.

| Decision            | Reason                                                                                      |
| ------------------- | ------------------------------------------------------------------------------------------- |
| payment retry logic | exponential backoff, not fixed interval; upstream rate-limits on burst                      |
| `schedule.ts`       | new file rather than extending `utils.ts`; `utils.ts` is imported by the worker, this isn't |
| config loading      | fails loud instead of defaulting; a silent default caused the bug being fixed               |

Not prose. Not a paragraph with a reason embedded in it. A row.

That's scannable because it isn't just text. It's table.
Twenty rows and you know what happened. If a row looks wrong, expand it.

Each reason has to fit on one line. If it's longer, truncate it and 
put the rest behind a click — a table full of paragraphs is just 
the wall of text again

## What the work actually is

The rendering is trivial. **The hard part is deciding what counts as 
a reason and what's just noise to drop.**

Take whatever the agent produced. Pull out every (decision, reason) pair.
Throw away the transitions, the recaps, the narration of what the code does,
the confidence statements. Discarding is the product.

## Non-goals

- **Not ranking.** Everything gets equal visual weight. This is division of
labor, not a principle — deciding what deserves attention is
`[co-sign](https://github.com/nicoledicochea/cosign-code-audit)`'s job, and
it does that with mechanical signals (blast radius, missing coverage,
pattern mismatch) that downstage has no access to. downstage surfaces
everything; co-sign anchors attention within it.
- **Not a capture mechanism.** This reads reasoning that already exists in
agent output. Forcing reasoning to exist where it doesn't is
`[aside](https://github.com/nicoledicochea/agent-aside)`'s job.
- **Not a mental-model diagram.** Considered and dropped: nodes-and-edges
graphs are a heavier artifact than the problem needs, and the problem isn't
that decisions lack structure — it's that reasons are buried in prose.
- **Not a PR description generator.** The reader is the person who ran the
session and was there. Rows can be terse to the point of being cryptic to
anyone else. Writing for a cold reviewer is a much higher bar, and it's
`[prologue-pr](https://github.com/nicoledicochea/prologue-pr)`'s job — it
consumes the same extracted rows and writes for someone who wasn't there.
- **Not verification.** Nothing here checks whether a reason is true. It makes
reasons visible enough that a human's intuition has something to catch on.
- **Not tied to one agent.** It consumes text. Any agent produces text.



## How it works (v0)

1. Session ends.
2. A script reads the transcript.
3. It extracts (decision, reason) pairs.
4. It writes a single HTML file into the repo (gitignored).
5. You open it.

No server. No extension. No daemon. No new surface to remember to visit.
If it isn't useful, delete the file.

Post-hoc extraction only. Streaming/live updating is a later question, and
only worth answering once the table itself proves useful.

## Where this sits in the pipeline

Four separate projects, one thesis: *AI made generation cheap and left
comprehension expensive.* Each handles a different stage of getting reasoning
from an agent's head into a human's.


|     | Project                                                          | Job                                                                |
| --- | ---------------------------------------------------------------- | ------------------------------------------------------------------ |
| 1   | `[aside](https://github.com/nicoledicochea/agent-aside)`         | **Preserve** reasoning during execution — force it to exist        |
| 2   | **downstage**                                                    | **Surface** it — extract, discard the rest, make it scannable      |
| 3   | `[co-sign](https://github.com/nicoledicochea/cosign-code-audit)` | **Anchor attention** — flag what actually needs a close look       |
| 4   | `[prologue-pr](https://github.com/nicoledicochea/prologue-pr)`   | **Hand off** — write a PR description for someone who wasn't there |


Deliberately separate repos. Different mechanisms, different install stories,
different failure modes. `aside` is bound to whatever interception point an
agent exposes; `downstage` just eats text; `co-sign` reads the repo; `prologue-pr`
writes for a stranger.

**Discovered backwards.** 3 and 4 were prototyped first, and both consumed a
*diff* — the one artifact from which rationale has already been deleted. They
were starved. Walking upstream to find the missing input produced 2, and then

1. The numbering is the data flow. The reverse is the order it was found in.



### `downstage` and `co-sign`, specifically

These two look like they overlap and don't. `co-sign`'s signals are
**mechanical** — blast radius, missing test coverage, deviation from
established patterns — computed from the repo without asking a model anything.
`downstage`'s rows are the agent's **stated reasons**. Neither source can do the
other's job: mechanical signals know a change is risky but not why it was
made; stated reasons explain but can't be trusted to rank themselves.

They compose. A thin, vague reason attached to a high-blast-radius change is a
far stronger flag than either signal produces alone.

## Success looks like

- A developer can scan the output in under a minute and say what the agent
did and why, without reading the transcript or the diff.
- At least once, a reason reads as wrong or thin, the developer looks closer,
and finds something real.
- The developer reports feeling like they made the decisions — or could have
rather than feeling like they approved someone else's.



## Open questions

- **Extraction quality.** When a row is missing or mangled, is it because the
reason wasn't in the output (that's `aside`'s problem) or because extraction
failed (that's this project's problem)? Needs to be distinguishable, or
debugging is impossible.
- **Rejected alternatives.** "Why not A" is the most valuable kind of reason.
Third column, or folded into the reason? Not required for v0 — take whatever
the agent happens to give.
- **What counts as a decision.** Every edited line has a reason in some
trivial sense. The line between "decision" and "mechanical consequence of a
decision" is fuzzy and probably has to be found empirically.
- **Verbosity control.** Should the reader configure how much surfaces (more
rows / fewer rows / expand-by-default)? A rendering layer is an enforcement
point in a way a prompt never is — the model's compliance is irrelevant
because the renderer decides what shows. Worth exploring, but not in v0.
- Post-hoc extraction discards **ordering information** that might matter. Does
it? Unknown until it's used.



## Prior thinking

The observability ecosystem (Langfuse, LangSmith, Braintrust, Arize, MLflow,
Sentry, Datadog) has built extensively for this problem's neighbor. All of it
is shaped as a debugger: nested span trees, tool-call traces, latency and
token accounting. Those tools serve the person who *built* the agent and is
debugging why it failed.

This project serves the person who *received* the agent's output and has to
decide whether to accept it. The two users need opposite things —
exhaustiveness is a feature for a debugger and a bug here, where human
attention is the scarce resource being spent.
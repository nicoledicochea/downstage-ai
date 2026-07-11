# downstage

> In theater, *downstage* is the front of the stage, closest to the
> audience. An actor moves downstage to step out of the busy background
> scenery and deliver their most critical lines directly to you.
>
> An agent just handed you 500 lines of code and 600 words explaining it.
> Somewhere in there are the four decisions that actually mattered. This
> brings them downstage — out of the noise, to the front, where you can
> eyeball them.

  


*Step 2 of four. Companion to* `[aside](https://github.com/nicoledicochea/agent-aside)`*,*
`[co-sign](https://github.com/nicoledicochea/cosign-code-audit)`*, and*
`[prologue-pr](https://github.com/nicoledicochea/prologue-pr)`*.*

  


## The problem

Reasons exist in agent output. They're just buried — inside narrative,
transitions, recaps, and restatements of what the code already shows.

Recovering them means reading everything, in order, at the pace it was
written. That doesn't scale with how fast agents generate.

And the cost isn't just slow review. It's that you stop being an engineer
and become an approver — someone who accepts or rejects work whose reasoning
they never had access to.

  


## The idea

One decision. One reason. One row.


| Decision            | Reason                                                                                      |
| ------------------- | ------------------------------------------------------------------------------------------- |
| payment retry logic | exponential backoff, not fixed interval; upstream rate-limits on burst                      |
| `schedule.ts`       | new file rather than extending `utils.ts`; `utils.ts` is imported by the worker, this isn't |
| config loading      | fails loud instead of defaulting; a silent default caused the bug being fixed               |


That's scannable because it isn't just text. It's a table.
Twenty rows and you know what happened. If a row looks wrong, expand it.

**The rendering is the easy part. The hard part is deciding what counts as a reason and what's just noise to drop.**

  


## Why a reason and not the code

A reason is more falsifiable than code.

Code that's wrong looks like code. A reason that's wrong often *reads*
wrong — because language carries claims you can check against what you know.

The code `cache.set(key, val)` just tells you caching happens. The reason 
*'cached this because the value never changes after startup'* is a claim you 
can check — and if the value does change, you've caught a real bug that 
the code alone would've hidden.

This works even if the reason was reconstructed after the fact rather than
being what the model actually computed. The reason doesn't have to be true.
It has to be **checkable**. That's a much weaker property, and it's enough
for intuition to catch on.

  


## What this is not

**Not ranking.** Everything gets equal weight. Deciding what deserves a close
look is `[co-sign](https://github.com/nicoledicochea/cosign-code-audit)`'s
job — it has mechanical signals (blast radius, missing coverage) that
downstage doesn't.

**Not a capture tool.** This reads reasoning that already exists. Forcing
reasoning to exist where it doesn't is `[aside](https://github.com/nicoledicochea/agent-aside)`'s job.

**Not a PR description.** The reader is the person who ran the session. Rows
can be terse to the point of cryptic. Writing for a cold reviewer is
`[prologue-pr](https://github.com/nicoledicochea/prologue-pr)`'s job.

**Not an observability tool.** Langfuse, LangSmith, Braintrust and the rest
are debuggers — span trees for the person who *built* the agent. This is for
the person who *received* its output. Exhaustiveness is a feature there and a
bug here.

  


## How it works

```
session ends
     ↓
script reads the transcript
     ↓
extracts every (decision, reason) pair
     ↓
discards everything else
     ↓
writes one HTML file into the repo
     ↓
you open it
```

No server. No extension. No daemon. No new surface to remember to visit.
If it isn't useful, delete the file.

  


## Where this sits

Four projects, one thesis: 

> *AI made generation cheap and left comprehension expensive.*


|     |                                                                  |                                                              |
| --- | ---------------------------------------------------------------- | ------------------------------------------------------------ |
| 1   | `[aside](https://github.com/nicoledicochea/agent-aside)`         | **Preserve** reasoning during execution                      |
| 2   | **downstage**                                                    | **Surface** it — scannable, everything, no claims            |
| 3   | `[co-sign](https://github.com/nicoledicochea/cosign-code-audit)` | **Anchor attention** — what needs a close look               |
| 4   | `[prologue-pr](https://github.com/nicoledicochea/prologue-pr)`   | **Hand off** — a PR description for someone who wasn't there |


`downstage` surfaces everything and ranks nothing. `co-sign` is what decides where to look. They compose: a thin, vague reason on a high-blast-radius change is a stronger flag than either signal alone.

  


## Status

Prototype / concept stage. Not yet built.

Full spec in `[PRD-downstage.md](./docs/PRD-downstage.md)`.
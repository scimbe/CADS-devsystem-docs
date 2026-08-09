---
title: Why /iterate rejects exact duplicates
description: A real incident -- two entries both landed as "iteration 8" -- and the fix.
order: 3
---

# Why `/iterate` rejects exact duplicates

On 2026-08-05, `webconference-android`'s own real run history ended up with two entries both stored
as `"iteration": 8` — byte-identical `stage`/`feedback`/`succeeded` — and no `"iteration": 9"` at
all. Found by checking in on the run's actual state, not assumed. This page is the honest story of
what happened and what changed, matching this project's own goal document: *"it is the fault of the
pipeline, not the user of the pipeline, if the process leads him not to the perfect result."*

## What `write_lock` actually protects

`POST /api/runs/{id}/iterate`'s handler holds a real `tokio::sync::Mutex` for its entire
load-compute-persist cycle — two concurrent requests *within the same `devsystem-web` process* are
correctly serialized; the second one always sees the first one's already-appended history before
computing its own next iteration number. That's real, and it works.

What it can't do: protect against **two separate process instances**, each running its own,
independent lock, each with no idea the other exists. That became possible for real once this
project started running multiple autonomous loops concurrently, each capable of redeploying
`devsystem-web` — a window opened where an old and a new container instance briefly overlapped, each
serving one of two functionally-identical requests, each computing the same `history.len() + 1`
before either had seen the other's write.

## The fix: reject at the data layer, not the infrastructure layer

The actual redeploy overlap was already closed separately (a `flock` around
`scripts/deploy-devsystem-web.sh`, serializing real invocations of the deploy script itself). But
that fixes the *infrastructure* cause — it doesn't make the *data* robust to whatever the next
process-external cause of a duplicate turns out to be. So the real fix lives one layer down:
`/iterate` now compares an incoming submission to the run's own immediately-preceding history entry
— `stage`, `feedback`, `succeeded`, `proposals`, and `requirement_indices` all equal — and rejects a
match with a real `409`, regardless of *why* a duplicate arrived.

```
$ curl -X POST .../api/runs/{id}/iterate -d '{"stage":"devsystem.improve","feedback":"x","succeeded":true}'
{"added_stages":[],"iteration":1,"outcome":"Continue","roles_now":1}

$ curl -X POST .../api/runs/{id}/iterate -d '{"stage":"devsystem.improve","feedback":"x","succeeded":true}'
this submission is byte-identical to iteration 1, the run's own immediately-preceding entry --
refusing to record it as a distinct, new iteration
HTTP 409
```

A genuinely different submission right after — even from the same stage, even also `succeeded`
— is never blocked; this isn't "no two iterations in a row from the same stage," only "not the
literal same one twice."

## A second gap in the same guard, found and closed 2026-08-06

The check above landed only in `devsystem-web`'s HTTP handler. `devsystem_iterate`'s own **local**
mode (no `--remote` — see [Bid for a role and submit a real iteration]({{ '/how-to/submit-an-iteration/' | relative_url }})),
which reads and writes `runs/<run_id>/` directly with no HTTP layer in between at all, had no
equivalent whatsoever — the exact "two real entry points, one bug class" pattern this project kept
finding all session (the `run_id` path-traversal guard and the `requirement_indices` bounds check
both had the identical shape, and so — as of the same day — did the `paused`-run gate; see
[Why did my run pause itself?]({{ '/how-to/why-did-my-run-pause/' | relative_url }})). A retried or
accidentally re-run `record.json` would silently append a second, indistinguishable history entry
through this path instead of being refused.

Live-confirmed before fixing: a scratch run's first real iteration, resubmitted byte-identical
through the local CLI, was accepted outright. Fixed by moving the comparison into a shared
`duplicate_of_last_iteration` function both the HTTP handler and this CLI's local mode now call
([CADS-devsystem@3afdbd2](https://github.com/scimbe/CADS-devsystem/commit/3afdbd2)):

```
$ devsystem_iterate a-run-with-one-real-iteration record.json
rejected: this submission is byte-identical to iteration 1, the run's own immediately-preceding
entry -- refusing to record it as a distinct, new iteration
```

Same rejection, same wording, whichever real entry point you submit through now.

## What happened to the real bad data, and the mistake made fixing it

This page originally ended here: the duplicate `"iteration": 8"` entry was left in
`webconference-android`'s real history, not surgically edited out, on the reasoning that quietly
erasing it would be its own small dishonesty. That's still the right instinct. But it isn't what
actually happened next, and the honest thing is to say so rather than leave a stale claim standing.

The duplicate **was** later repaired — by compacting the history array: deleting the bad record and
shifting every iteration after it down by one. That fixed the visible symptom (no more duplicate
`8`) but broke something worse, found live by a non-technical evaluator (GitHub issue #42): every
ordinal is this platform's only handle on an iteration in prose — a backlog escalation, a frozen
`feedback` string, the check-in document's own heading — and none of that prose moved with the
array. The run's own single open operator decision ended up citing an iteration that no longer
existed; the check-in document ended up quoting itself as an independent prior source; five other
issues (#34, #35, #36, #38, #39, #41) that had quoted real iteration numbers before the compaction
were all off by one afterward, silently, with nothing anywhere indicating the numbering had moved.

Compacting is exactly the operation ordinal-keyed prose can't survive. The real fix, shipped as
issue #42's second suggestion: a bad record now gets **tombstoned in place** instead —
`POST /api/runs/{id}/history/{iteration_id}/withdraw`, looked up by the record's own stable id (not
its position), setting a real `withdrawn` flag and a required, non-empty `withdrawn_reason` without
ever deleting the record or shifting anything after it. Every ordinal, and every existing
cross-reference to one, stays exactly as valid as it was before. The History panel shows a
withdrawn record with a visible label rather than hiding it or leaving a silent gap in the numbering:

<figure>
<img src="{{ '/assets/img/explanation-duplicate-iteration-guard/01-withdrawn-history.png' | relative_url }}" alt="The History panel showing three iterations. Iteration 2 is dimmed with a label reading withdrawn by reviewer@example.com: duplicate content, wrong role tag -- see iteration 1. Iterations 1 and 3 are unaffected, and both keep their original numbers.">
<figcaption>A real withdrawn record, reached through <code>POST /history/{iteration_id}/withdraw</code>. Iteration 2 stays iteration 2 -- withdrawing it never renumbers iteration 3.</figcaption>
</figure>

So: `webconference-android`'s real history no longer contains the original duplicate `iteration 8`
— that repair did happen, and its side effect (the ordinal drift above) is real and stayed
unrepaired in the run's own frozen prose, on purpose (rewriting history text after the fact would be
the same dishonesty this section opened with). What changed is that a *future* bad record no longer
has to go through the operation that caused this. See [GitHub issue
#42](https://github.com/scimbe/CADS-devsystem/issues/42) for the full incident, and the REST API
reference's [`Runs`]({{ '/reference/rest-api/#runs' | relative_url }}) section for the withdraw
endpoint itself.

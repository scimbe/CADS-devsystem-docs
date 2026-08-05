---
title: Ask devsystem.assistant about your run
description: Get a real, grounded answer about a run's current state from the Process Prompt -- no fabricated summaries.
order: 2
---

# Ask devsystem.assistant about your run

Every run's control panel has a `DEVSYSTEM.ASSISTANT` dock on the right -- it can't be hidden, by
design. Type a real question into the Process Prompt at the bottom and it answers grounded in that
run's actual current state (spec, history, backlog, milestones), fetched fresh from devsystem-web
for every question -- never a cached or generic answer.

## Select a run, then ask

<figure>
<img src="{{ '/assets/img/howto-ask-assistant/01-run-selected.png' | relative_url }}" alt="A run selected in the Runs panel, the Pipeline panel showing its real roles">
<figcaption>The assistant dock's header names the currently selected run -- every question is scoped to it.</figcaption>
</figure>

Type a real question -- here, the honest one to ask about any stalled run:

```
What is blocking this run from reaching its milestone right now?
```

<figure>
<img src="{{ '/assets/img/howto-ask-assistant/02-assistant-reply.png' | relative_url }}" alt="The assistant's real reply: a table of two real blockers, with concrete detail and a concrete suggestion" >
<figcaption>A real reply against the live webconference-android run -- names the actual stalled role (<code>devsystem.android_emulator_test</code>), the actual reason (no bidder, no <code>price_ceiling</code> set), the actual backlog item still open, and even a concrete, correct suggestion (set a <code>price_ceiling</code> so the auction has bounded economics). Real token usage and cost are shown too -- nothing about this reply is templated.</figcaption>
</figure>

## What it can and can't do

The assistant has **zero ct-agent-connected tools** -- it never executes an action itself, never
calls out to GitHub, never touches the network beyond reading the run's own state. `GET
/api/assistant/status` reports this honestly (`disallowed_tools`) rather than hiding it. What it
*can* do, per a real per-run rate limit (10 seconds, a spam-guard against a stuck retry loop, not a
security control): write real narrow state changes on your behalf -- milestones, backlog items,
requirements -- when you ask for them, and *propose* (not directly apply) a new pipeline stage or a
GitHub issue, both of which land in a real pending queue for you to approve or reject. See [How the
pipeline proposes and grows its own stages]({{ '/explanation/self-optimizing-pipeline/' | relative_url }})
for exactly which of its actions apply immediately versus wait for you.

## Your run's real cumulative cost: the Assistant Usage panel

Each reply already shows its own real token usage (visible in the screenshot above), but until
2026-08-05 that number vanished once you scrolled past it -- nothing tracked what a run had *cost
so far* across every question ever asked. `devsystem-web` now persists a real running total on
every `/ask` call, shown in a dedicated **Assistant Usage** panel.

Real, live data — one real question asked against the actual `webconference-android` run for this
page:

```
$ curl -X POST .../api/runs/webconference-android/assistant \
    -d '{"instruction": "In one sentence, what is this run currently working toward?"}'
{"response": "This run is building 1:1 text messaging end-to-end ...",
 "usage": {"input_tokens": 2, "output_tokens": 107,
           "cache_creation_input_tokens": 22040, "cache_read_input_tokens": 14184,
           "total_cost_usd": 0.2308}}
```

`GET /api/runs/webconference-android` immediately reflects it, and the panel renders exactly that:
one real call, **$0.2308** cumulative so far, with the full input/output/cache token breakdown --
not an estimate, the actual number `devsystem_assistant`'s own LLM CLI call reported.

## A real, honest gap this walkthrough found (2026-08-05)

Writing this page caught a real production regression: the first real question sent to the
assistant came back with `could not reach devsystem.assistant bridge at
http://host.docker.internal:8791/ask` -- a redeploy earlier the same day had recreated the
`devsystem-web` container without `--add-host=host.docker.internal:host-gateway`, so that hostname
never resolved inside it at all. Fixed live before this page was published (verified with the exact
same real question, second screenshot above is from *after* the fix) --
[CADS-devsystem@bb7fd3d](https://github.com/scimbe/CADS-devsystem/commit/bb7fd3d).

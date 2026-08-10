---
title: Author a requirement with the guided wizard
description: Use the step-by-step wizard to draft a well-formed requirement without knowing EARS notation up front.
order: 20
---

# Author a requirement with the guided wizard

The Requirements panel's direct **Add requirement** form expects you to already know the
EARS-style shape this platform's own gates check for (`WHEN <trigger>, THE SYSTEM SHALL
<behavior>`, with concrete acceptance criteria) and to write the whole statement in one box. If
you'd rather be walked through it one question at a time, click **🧭 Guided (walk me through
it)** next to the direct form instead.

The wizard asks a fixed sequence of questions and assembles the real statement for you -- it does
not call an LLM anywhere in its own flow, so the questions and their order are exactly the same
every time.

<figure>
<img src="{{ '/assets/img/howto-guided-requirement-wizard/01-trigger-button.png' | relative_url }}" alt="The Requirements panel's empty state, showing the direct Add requirement form next to a second button labelled Guided (walk me through it)">
<figcaption>The direct form and the guided wizard sit side by side -- the wizard doesn't replace the fast path, it's an alternative to it.</figcaption>
</figure>

## Step 1: trigger (optional)

What has to happen first, if anything. Leave this blank for a requirement that should always
hold, with no separate triggering condition.

<figure>
<img src="{{ '/assets/img/howto-guided-requirement-wizard/02-step-trigger.png' | relative_url }}" alt="Step 1 of 5, trigger, with the text 'the user sends a message while the device is offline' typed into the input">
<figcaption>A real trigger: "the user sends a message while the device is offline."</figcaption>
</figure>

## Step 2: behavior (required)

What the system must actually do. This is the one field the wizard won't let you skip --
clicking **Next** with it empty blocks you with a real, specific message rather than silently
letting a blank behavior through.

<figure>
<img src="{{ '/assets/img/howto-guided-requirement-wizard/03-step-behavior-blocked.png' | relative_url }}" alt="Step 2 of 5, behavior, showing a real inline error: this field can't be left blank"><figcaption>An empty behavior is refused before you can move on -- the same validation discipline this platform applies everywhere a required field exists.</figcaption>
</figure>

## Step 3: acceptance criteria

At least one concrete, checkable criterion. **Add another criterion** appends a new input for as
many as you need. Each one gets checked against the same minimum-real-content rule the server
itself enforces (a criterion like `"ok"` or a single punctuation mark isn't accepted here either --
see [Requirements, verification, and automode]({{ '/explanation/requirements-and-automode/' | relative_url }}) for why), so a criterion that would bounce off the server never gets that far.

<figure>
<img src="{{ '/assets/img/howto-guided-requirement-wizard/04-step-criteria.png' | relative_url }}" alt="Step 3 of 5, acceptance criteria, with two real criteria filled in: a queued message is delivered within 10s of reconnecting, and the message is never silently dropped or duplicated">
<figcaption>Two real, distinct, checkable criteria -- not "it works."</figcaption>
</figure>

## Step 4: rationale

Why this requirement matters, in your own words. This becomes the proposal's `rationale` field,
which any reviewer approving or rejecting the proposal later will see.

<figure>
<img src="{{ '/assets/img/howto-guided-requirement-wizard/05-step-rationale.png' | relative_url }}" alt="Step 4 of 5, rationale, with the text: offline devices currently drop messages silently, a real reported gap">
<figcaption>A concrete reason, not a placeholder.</figcaption>
</figure>

## Step 5: review, then submit

The wizard composes your answers into a real EARS statement -- `WHEN <trigger>, THE SYSTEM SHALL
<behavior>` when you gave a trigger, or the bare `THE SYSTEM SHALL <behavior>` form if you left
it blank -- and shows you the whole thing, statement, criteria, and rationale, before anything is
sent anywhere.

<figure>
<img src="{{ '/assets/img/howto-guided-requirement-wizard/06-step-review.png' | relative_url }}" alt="Step 5 of 5, review before submitting, showing the composed statement 'WHEN the user sends a message while the device is offline, THE SYSTEM SHALL queue the message locally and redeliver it once connectivity returns', both acceptance criteria, and the rationale, with a note that this lands as a real proposal in the pending-review queue">
<figcaption>Nothing is created yet -- this is a preview, and <b>Back</b> still works from here if you want to change an earlier answer.</figcaption>
</figure>

Clicking **Submit proposal** sends it through the same `propose_requirement` path
`devsystem.assistant`'s own proposals use -- **never** directly into your run's live
requirements. It lands in the Requirements panel's pending-review queue exactly like every other
proposal on this platform, waiting for a real approve or reject.

<figure>
<img src="{{ '/assets/img/howto-guided-requirement-wizard/07-proposal-queued.png' | relative_url }}" alt="The Requirements panel after submitting, showing a real pending proposal with an Approve and Reject button">
<figcaption>A real pending proposal, not a live requirement -- someone still has to approve it.</figcaption>
</figure>

Approving or rejecting it works exactly like any other requirement proposal -- see [Ask
devsystem.assistant about your run]({{ '/how-to/ask-the-assistant/' | relative_url }}) for the
assistant's own equivalent proposal flow.

## What the wizard doesn't do (yet)

This is a first, deliberately bounded slice. It won't:

- offer inline EARS-phrasing help beyond the final composed preview, or
- help with a requirement that genuinely doesn't fit the trigger/behavior/criteria/rationale
  shape (an unusually structured requirement is still best written directly).

Closing **Cancel** or the overlay's own close control at any step discards everything you've
typed so far and leaves no trace -- no partial proposal, no partial requirement.

---
title: Correct a wrong requirement
description: Fix a requirement's statement or acceptance criteria in place, without losing the run's history.
order: 16
---

# Correct a wrong requirement

Until 2026-08-07, a requirement could be added but never corrected — no edit control, no
`devsystem.assistant` action, no REST endpoint. A wrong requirement was therefore permanently
load-bearing for the mandatory review gate and for coverage, and the only "fix" was deleting and
recreating the entire run, losing every real iteration in it. Fixed: every requirement's card in the
Requirements panel now has a real **✎ Edit** control.

## A real, live example

A requirement that conflates two different things this platform actually tracks separately — an
*iteration* succeeding, versus a *requirement* being verified:

```
$ curl -X POST .../api/runs/{id}/requirements \
    -d '{"statement": "WHEN an iteration succeeds, THE SYSTEM SHALL mark requirement 0 verified", "acceptance_criteria": ["a succeeded iteration alone marks the requirement verified"]}'
```

A human already confirmed the one criterion before the mistake was caught:

<figure>
<img src="{{ '/assets/img/howto-edit-requirement/01-wrong-requirement-with-edit-button.png' | relative_url }}" alt="A requirement reading 'WHEN an iteration succeeds, THE SYSTEM SHALL mark requirement 0 verified', its one acceptance criterion checked and struck through, with a real Edit button underneath">
<figcaption>Wrong as written -- this platform's own review gate requires a real, successful <code>devsystem.review</code> iteration, not just any success. And it's already confirmed.</figcaption>
</figure>

Clicking **Edit** opens a real inline form, pre-filled with the requirement's current statement and
criteria — one criterion per line, same convention as the Add form below it:

<figure>
<img src="{{ '/assets/img/howto-edit-requirement/02-edit-form-open.png' | relative_url }}" alt="The Edit form open, pre-filled with the requirement's current statement and its one acceptance criterion, with a note that saving resets this requirement's own verified/confirmed state">
<figcaption>The same real EARS/length/bidi validation the Add form uses applies here too -- it's the identical, shared client-side check, not a second copy that could quietly drift.</figcaption>
</figure>

Saving the correction persists it immediately — no proposal/approval step, matching the Add form's
own directness:

<figure>
<img src="{{ '/assets/img/howto-edit-requirement/03-corrected-requirement.png' | relative_url }}" alt="The corrected requirement reading 'WHEN a devsystem.review iteration succeeds and names requirement 0, THE SYSTEM SHALL allow requirement 0 to be marked verified', now with two acceptance criteria, both unchecked and unconfirmed">
<figcaption>Corrected, and the earlier confirmation is honestly gone -- see below for why.</figcaption>
</figure>

## Why saving resets the confirmation

The specific text a human confirmed may no longer be what's actually being asked once the wording
changes. Carrying an old confirmation forward against different criteria would misrepresent it as
still applying to text nobody actually checked. So `verified` and every criterion's own confirmation
(see [See a requirement's real decision basis]({{ '/how-to/see-a-requirements-decision-basis/' | relative_url }}#who-confirmed-a-criterion-and-when))
reset to unconfirmed on every save — visible above as the corrected requirement's criteria going
from one checked, struck-through line back to two plain, unchecked ones.

`created_by`/`proposed_by` — who originally authored the requirement, and whether a human or an LLM
wrote the first draft — are **not** touched by an edit. Correcting the wording doesn't change who
first wrote it.

## A successful write says so, and names the run

Adding a requirement, and saving a correction here, both now show a real, visible confirmation
naming the exact run the write landed in -- "Requirement added to docs-edit-requirement-demo.",
or "Requirement #0 updated in docs-edit-requirement-demo." for a save. Until 2026-08-09 this status
line only ever rendered for a *rejection* -- a successful write set the same text, then the panel's
own refresh immediately replaced the whole panel body before anyone could actually see it. A
successful write and a silent write to the wrong run looked identical from the user's side.

<figure>
<img src="{{ '/assets/img/howto-edit-requirement/04-write-success-names-run.png' | relative_url }}" alt="The Add requirement form with a green confirmation reading 'Requirement added to docs-edit-requirement-demo.' beneath it">
<figcaption>Real, current data -- the confirmation names the run, not just "saved".</figcaption>
</figure>

This matters most when several runs are open at once: naming the run is what tells you a write
landed where you meant it to, not silently on whichever run happened to be selected.

## Why this is an edit, not a delete-and-recreate

A requirement's index is what iteration history's `requirement_indices` field actually points at.
Removing a requirement would silently renumber every later index and break every existing
iteration's own reference to it — correcting the text in place avoids that class of problem
entirely. There is still no way to remove a requirement outright; that remains open (see the REST
API reference below for exactly what exists today).

See the [REST API reference]({{ '/reference/rest-api/' | relative_url }}#backlog-milestones-requirements)
for the endpoint itself.

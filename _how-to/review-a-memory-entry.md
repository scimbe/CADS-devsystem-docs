---
title: Review a memory-log entry
description: What "unreviewed" vs. "governed" actually means, and why marking one reviewed is permanent.
order: 7
---

# Review a memory-log entry

Every real iteration a role-filler submits leaves a real entry in the run's memory log -- its
`key_findings`, any `constraints` it carries forward, and a `trust` marker. This page covers that
`trust` marker: what the two real states mean, and what actually happens when you review one.

## Two real states, not a spectrum

Every entry starts `unreviewed` (the real default on write). A human can promote one to
`governed` -- borrowed directly from ECC's own vault/governed-artifact split, per the source's own
doc comment. There's no third state and no partial credit; an entry is either something nobody has
vouched for yet, or something a human has explicitly said "yes, I looked at this."

<figure>
<img src="{{ '/assets/img/howto-review-memory/01-unreviewed-entry.png' | relative_url }}" alt="A real unreviewed memory entry in the Memory Log panel, with an orange 'unreviewed' badge and a 'mark reviewed' button">
<figcaption>A real entry, freshly written by a real iteration -- unreviewed by default, exactly as every entry starts.</figcaption>
</figure>

## Marking one reviewed is real, and permanent

Click **mark reviewed** and you'll get a real confirmation before anything changes:

> Mark this devsystem.plan entry (role plan) as human-reviewed? This is a real, permanent
> attestation -- there's no way to un-mark it.

That's not boilerplate caution. Checked directly against the source: there is no un-govern route
anywhere in this codebase — calling the same endpoint again on an already-governed entry just
no-ops. Once you confirm, the entry looks like this, permanently:

<figure>
<img src="{{ '/assets/img/howto-review-memory/02-governed-entry.png' | relative_url }}" alt="The same memory entry now showing a teal 'governed' badge and no mark-reviewed button">
<figcaption>Governed. The button is gone -- there's nothing left to click, because there's nothing left to do.</figcaption>
</figure>

## Why this is worth taking seriously

Unlike most of this GUI's other destructive-action confirmations (removing a custom panel, an
indexed document, or a pending proposal), governing an entry doesn't delete anything — the real
content stays exactly as it was. What's permanent is the *attestation*: once you mark an entry
reviewed, that's a real, recorded claim that a human looked at it, and nothing in this pipeline
can currently tell the difference between a careful review and an accidental click. Treat it the
way the confirmation dialog asks you to — as a real statement about what you actually did, not a
formality to click through.

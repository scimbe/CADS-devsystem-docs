---
title: Read the Flow panel
description: A three-section, at-a-glance view of where a run is headed, what's queued, and what actually happened -- not a fabricated single timeline.
order: 11
---

# Read the Flow panel

Every other panel shows one real slice of a run's state in detail -- Milestones, Backlog, and
History each own their own data and their own full interaction (toggle, add, review). **Flow**
doesn't replace any of them; it's a single, compact panel that puts the *shape* of those three
things next to each other, so you can tell where a run is headed at a glance without opening three
panels. Open it from the **Flow** chip in the panel bar -- it's not shown by default on a first-time
run, the same "one click away, not force-opened" precedent every other non-starter panel follows.

Real capture, `webconference-android`'s own Flow panel:

![The Flow panel showing TARGET (a struck-through achieved milestone), QUEUE (an empty backlog), and WHAT HAPPENED (the six most recent iterations, all succeeded)]({{ '/assets/img/howto-flow-panel/01-flow-panel.png' | relative_url }})

## The three sections, and why they're kept honestly separate

- **target** -- every real `Milestone` this run has, in order, with the achieved ones struck
  through. If a milestone hasn't been achieved yet, it's marked as the next one to reach (◎ instead
  of ○). No milestones at all reads as "nothing to steer toward yet," not an empty section.
- **queue** -- the run's real, still-open `BacklogItem`s (done ones are filtered out; they're not
  what's still ahead).
- **what happened** -- the six most recent real iterations from history, newest first, each marked
  succeeded (✓) or failed (✗) with its stage. Anything older than the six shown says so explicitly
  ("N earlier iteration(s) -- see History panel") rather than silently truncating with no indication
  more exists.

These are deliberately **three separate sections connected by one visual line, not one fabricated
unified timeline**. A milestone, a backlog item, and a completed iteration aren't the same kind of
thing or on the same real clock -- collapsing them into a single chronological line would imply an
ordering relationship between them that the data doesn't actually have (a queued backlog item isn't
"before" or "after" a milestone in any real sense; they're independent). Flow shows all three next
to each other precisely so you can see the whole shape at once without inventing a false connection
between them.

The panel also shows your real position at the top -- `iteration N / max_iterations`, and `paused`
if the run currently is -- the same real numbers the Health & Criteria panel's own gauges use,
repeated here so you don't have to open that panel too just to know where you stand.

## When to use this instead of the individual panels

Flow is for a quick "where are we" glance -- checking in on a run you're not actively working, or
orienting yourself on one you haven't looked at in a while. For anything that needs a decision or an
edit (toggling a milestone, adding a backlog item, reading a full iteration's feedback), the
dedicated Milestones/Backlog/History panels are still where that actually happens -- Flow is
read-only by design, matching the "compact overview, not a fourth place to mutate the same data"
scope every other summary-style panel (Process, Health & Criteria) already keeps.

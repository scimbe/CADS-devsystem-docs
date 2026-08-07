---
title: Delete a run
description: Every run in the Runs panel has a real, permanent delete button next to it -- what it does, and what happens if you're looking at a run someone else deletes.
order: 9
---

# Delete a run

Every entry in the **Runs** panel has a small 🗑 button next to it, separate from the run's own
name/summary button (clicking the name still selects the run, same as always).

<figure>
<img src="{{ '/assets/img/howto-delete-a-run/01-runs-panel-delete-buttons.png' | relative_url }}" alt="The Runs panel showing several real runs, each with its own name/summary button and a separate small trash-can delete button to its right">
<figcaption>A real Runs panel -- the 🗑 button sits next to each entry, not inside it (a button can't nest inside another button).</figcaption>
</figure>

## What it does

Clicking 🗑 asks for a real confirmation first -- what it asks depends on whether the run has any
real history, since a bare one-click confirm turned out not to be enough friction for a run that
actually matters (real evaluator finding, [issue
#45](https://github.com/scimbe/CADS-devsystem/issues/45), fixed 2026-08-07):

- **A fresh, 0-iteration run** gets a plain confirmation: *"Delete run "&lt;run_id&gt;" permanently?
  [ownership note]. This removes it and all its real state for good -- there's no undo."*
- **A run with any real iteration history** asks you to type the run's own id to confirm instead --
  *"Delete run "&lt;run_id&gt;" permanently? [ownership note]. It has N real iteration(s) of history.
  ... Type the run id to confirm:"* -- anything else typed, including Cancel, leaves the run
  untouched.

Either way, confirming calls the real `DELETE /api/runs/{id}` route, which removes that run's
entire directory on disk. Every real iteration, requirement, milestone, backlog item, chat
exchange, and custom panel that run ever had goes with it. There's no archive, no trash, no
recovery.

This exists because it's a genuine, common need, not a rarely-used escape hatch: this project's own
[stress-test methodology]({{ '/explanation/dau-lens-and-stress-testing/' | relative_url }}) creates
a fresh scratch run on nearly every real investigation, and before this existed those accumulated
forever -- over a hundred real runs piled up on the actual deployment with no way to ever remove
one.

## Ownership only protects a run that has one

If you're signed in, deleting someone else's *owned* run gets a real `403` -- the exact same
per-run ownership check that already governs viewing and editing a run applies here too, not a
separate, looser rule for deletion. But that check is `owner_email == you`, and only ever
protects a run that actually has an `owner_email` recorded.

**Right now, that's none of them.** A real evaluator measurement ([issue
#44](https://github.com/scimbe/CADS-devsystem/issues/44)) found 0 of 138 runs on the live
deployment carry a recorded owner, `webconference-android` included -- every run so far was
created through this pipeline's own headless CLI/automation, which never carries a signed-in
browser's identity, rather than through the GUI's own **+ New Project** button (which does record
a real owner, confirmed live). Until a run has an owner, *"you can only delete your own"* is not
the operative rule -- any signed-in account can delete it, which is exactly why a run with real
history now asks you to type its id rather than trusting one click (above).

## If a run you're looking at disappears

If a run gets deleted -- by you in another tab, or by anyone else with access to it -- while you
still have it open here, you'll find out the moment this page's own background refresh next checks
in on it: a plain alert, *"This run ("&lt;run_id&gt;") no longer exists -- it may have been deleted.
Returning to the runs list,"* and the dashboard falls back to another real run rather than silently
continuing to show that run's now-dead, stale content forever.

That background check only actually happens on a panel with auto-refresh turned on (the small ⚙
gear on any data-driven panel's title bar). Sending the assistant a chat message gets the identical
real recovery unconditionally, whether or not auto-refresh is on -- clear the run, the same plain
alert, fall back to the runs list -- since asking about the run is itself a real request that hits
the exact same now-404ing endpoint. Any other write against a genuinely gone run (toggling a
milestone, adding a backlog item, and so on) still surfaces as a plain `404` the same honest way
every other API error is, without that same automatic fallback.

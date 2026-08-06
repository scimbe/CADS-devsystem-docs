---
title: The DAU lens and the incompetent-agent stress test
description: The methodology behind this project's confirm() dialogs and mechanical gates -- what it is, why it exists, and its real track record.
order: 6
---

# The DAU lens and the incompetent-agent stress test

Almost every gate, confirmation dialog, and validation check documented elsewhere on this site
traces back to one governing principle, stated directly by this project's own operator and quoted
verbatim in its [goal document](https://github.com/scimbe/CADS-devsystem/blob/main/docs/development-system-goal.md):

> It is the fault of the pipeline, not the user of the pipeline, if the process leads him not to the
> perfect result.

That's not a slogan -- it's a concrete methodology this project actually runs, repeatedly, against
its own real, live deployment. This page collects what that methodology is and what it's actually
found, so the reasoning behind any one fix doesn't have to be pieced together from scattered
mentions across other pages.

## Two lenses, one principle

**The incompetent-agent stress test** simulates the least competent realistic role-filler on
purpose -- not malicious, just lazy, careless, or genuinely mistaken -- and drives it against a real
run's real API. Not a thought experiment: an actual `POST` against the actual deployment, with the
actual response checked, before anything is called a gap. If a lazy shortcut gets a real `200` it
shouldn't, that's a real, fixable hole in a mechanical check, found the same way a bad actor
eventually would.

**The DAU lens** ("dumbest available user") asks the same question from the human side: a person
with poor judgment, who mostly listens to `devsystem.assistant` but sometimes chooses wrong, clicks
around the GUI. Every destructive or hard-to-undo action gets checked for whether a careless click
can trigger it with zero warning.

Both share the same discipline this project applies everywhere: **crude, mechanical, explainable
checks -- never fake LLM-judgment-in-disguise.** A length bar, a word-boundary check, a distinct-word
count. Nothing here claims to verify that a review is actually *good* -- only that it isn't trivially
empty, repetitive, or misapplied. See [How real risk annotations work]({{ '/explanation/risk-annotations/' | relative_url }})
for what these checks look like in the code.

## The real track record

As of this writing, the stress test has run **twenty-one** real rounds against the actual
deployment, finding and closing twenty-one real gaps -- not simulated, not hypothetical. A
representative sample, each with its own real live before/after proof:

- A one-line rubber-stamp review (`"looks fine to me"`) satisfied the mandatory review gate just as
  well as real scrutiny -- closed with a minimum length bar, then a **distinct-word** bar once padded
  filler (`"looks good looks good..."`) beat the length bar alone, then a same-text-reused-on-a-
  different-requirement check, then a **scaled-by-requirements-claimed** bar once a single review
  turned out to be able to shotgun-approve five unrelated requirements at once with one generic
  paragraph. See [Requirements, verification, and automode]({{ '/explanation/requirements-and-automode/' | relative_url }}).
- A requirement `statement` needed to at least attempt EARS notation ("SHALL" required) -- and once
  that shipped, a plain substring match on "shall" turned out to match inside "shallow" and
  "Marshall," accepting non-attempts by accident. Fixed to require the real word, not a substring.
- A role-filler's own embedded stage proposal could carry a completely empty `stage_id`/`tag`/
  `rationale` and still get applied to the live `PipelineSpec` -- found missing from the HTTP entry
  point, fixed, then found *still* missing from the separate local CLI entry point that mutates the
  exact same state with no HTTP layer involved at all. The real lesson from that one: closing a bug
  at the one call site you tested it against isn't the same as closing the bug class.
- `devsystem.assistant` could be talked into marking a requirement `verified` in a plain chat
  message, based on nothing but the implementer's own self-reported feedback, on any run that hadn't
  declared a `review` role (most runs, by default) -- closed with a real evidentiary gate requiring
  the same evidence bar a human's own review needs, enforced unconditionally for the assistant's own
  calls.
- A cluster of real, live-confirmed **untrusted-content** findings, one leading to the next: role-
  filler-controlled free text (`feedback`, a proposal's `rationale`, a requirement's `statement`)
  could impersonate this project's own real markdown structure in the mandatory check-in artifact
  and the real requirements export -- a crafted statement once forged a completely convincing fake
  "verified, human-authored" entry. The same untrusted text also reaches `devsystem.assistant`'s own
  LLM context on every `/ask` call -- a live prompt-injection test found this particular model
  already resisted a crafted `"SYSTEM OVERRIDE"` payload on its own, but an explicit structural
  defense was added anyway as real defense-in-depth, since the LLM backend is documented as
  swappable.
- Two real, live-confirmed **infrastructure** findings, outside the GUI/API surface entirely: the
  local `devsystem_iterate`/`devsystem_checkin` CLI binaries built filesystem paths straight from a
  raw `run_id` argument with no validation -- `devsystem_iterate ../some-name record.json` wrote
  real files directly into the repo root, completely outside `runs/`. Separately, both binaries that
  persist a real ed25519 signing key wrote it world-readable (confirmed live: mode `664` on the
  actual deployed key) -- anything else able to read arbitrary files on the host could lift it and
  sign fraudulent auction bids under that identity.
- `update_criteria` had no upper bound on any `AbortCriteria` field -- a real `u32::MAX` submission
  got a real `200`, turning a run's "bounded super loop" (this project's own central architectural
  claim) unbounded for any practical purpose. A DAU-lens gap, not a role-filler one: a human
  mistyping a value would silently lose their own safety net.

Real, live, currently-true data as of this writing -- the actual `webconference-android` run's own
risks, fetched fresh:

```
$ curl .../api/runs/webconference-android
"risks": [
  {"label": "touches auth/security", "evidence": "iteration 11's feedback mentions \"session\""},
  {"label": "no price ceiling set", "evidence": "role devsystem.document_extraction ... nothing since has bounded what filling it could cost"}
]
```

Two honest, currently-open findings on the actual flagship run, not a synthetic example -- proof this
methodology's own checks fire against real, in-progress work, not just scratch test runs built to
demonstrate them.

## The DAU lens, applied to the GUI

A sample of real, shipped fixes, each verified live with a real Playwright browser against the
actual deployment, not just described:

- Rejecting a pending stage/panel/issue proposal is exactly as permanent as removing a custom panel
  -- but only the removal button asked for confirmation. All three reject buttons now do.
- Removing an indexed RAG document, and marking a memory-log entry "reviewed" (an attestation with no
  undo), had the same gap. Both now confirm.
- The Pipeline chip's own pending-proposal badge -- meant to be the one signal that tells a human
  "something needs your decision" -- silently undercounted for a while: its formula summed three of
  the five real proposal queues, missing panel-removal and panel-edit proposals entirely until a
  stress-test firing caught a real pending removal proposal showing a badge of `0`.
- The "automode" checkbox on a requirement was labeled "let the assistant judge this one," implying
  it gated something real. It never did -- confirmed live, `auto_judge` is never read anywhere in
  `devsystem.assistant`'s own code. The label was corrected to say so plainly rather than keep
  implying a permission model that was never real. See
  [Requirements, verification, and automode]({{ '/explanation/requirements-and-automode/' | relative_url }})
  for the full live verification.

## The same lens, applied to the flagship app itself

The DAU lens isn't only aimed at this project's own GUI -- it's applied to the real Android client
(`CADS-webconference-android`) built by this pipeline's own role-fillers, too:

- Tapping **Connect** with either field blank used to attempt a real native FFI call and surface
  whatever error the Rust side happened to produce. Now caught locally first, before any native call.
- Tapping **Send** with a blank or whitespace-only message used to silently do nothing --
  `.isEmpty()` alone doesn't catch `"   "`, so a message of just a space would have actually been
  sent as real content.
- A peer disconnecting mid-session used to leave both Connect and Start Listening permanently
  disabled, with no way to reconnect short of restarting the app.
- Tapping **Send** with a real, genuine message *before ever connecting to anyone* used to hit a
  silent early return -- no status update, no rendered message, nothing to tell a user "you're not
  connected" apart from the app looking broken.

All four are real commits in the app's own repository, each with its own Robolectric test proving
the fix, verified green in that repo's real CI on every push -- the same discipline, applied to a
different codebase this pipeline is building, not just the pipeline's own tooling.

## Why this is a page, not just commit messages

Every fix above already has its own detailed writeup somewhere -- a commit message, a section in
another how-to or explanation page, an entry in the project's own internal goal document. This page
exists because the *pattern* connecting them is worth seeing on its own: this project doesn't treat
"someone used it wrong" as a support question. Per the governing principle at the top of this page,
a bad outcome is treated as a real, fixable defect in the process -- found the same honest way each
time, by actually trying to break it, against the real thing, not a mockup.

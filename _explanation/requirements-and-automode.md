---
title: Requirements, verification, and automode
description: Why acceptance-criteria judgment stays human by default, and what opting in actually authorizes.
order: 1
---

# Requirements, verification, and automode

A CADS-devsystem [`Requirement`](https://github.com/scimbe/CADS-devsystem/blob/main/pipeline/src/runner.rs)
is modeled on [EARS notation](https://en.wikipedia.org/wiki/Easy_Approach_to_Requirements_Syntax):
a real trigger, a real system behavior, and a list of concrete `acceptance_criteria` — distinct from
a `Milestone` (a checkpoint) or a `BacklogItem` (a task). It gives both a human reviewer and any
role-filler agent something checkable to verify against, instead of a vague wish.

## Two independent, explicit signals

`verified` (the whole requirement) and `verified_criteria` (each individual criterion) are
deliberately kept separate rather than auto-derived from one another. A human confirming "yes, this
whole requirement is done" and a human ticking off individual criteria are two different judgment
calls — coupling them silently would be a real design decision made *for* the operator, not asked
of them.

## Why automode is per-requirement, not a project setting

The natural next question, once acceptance criteria exist at all, is: can `devsystem.assistant`
judge them itself, instead of a human clicking checkboxes? The honest answer is: **by default, no**
— every requirement starts, and stays, 100% human-driven.

A requirement's owner can opt a *specific* requirement into "automode" via `auto_judge`. This is
deliberately **not** a global run or account setting. Opting in is a real, considered choice on one
requirement — not something every future requirement should silently inherit just because automode
was turned on once, somewhere, for something else. The flag is real and live in the API/GUI today;
setting it only *authorizes* future judgment — it doesn't perform any judgment itself yet. Building
the actual logic (grounding the assistant in a run's real iteration history/evidence before it
touches a checkbox, and being honest when the evidence doesn't support a clean yes/no) is real,
separate, still-open work.

## Why this matters for reproducibility

Fine-grained requirements are, honestly, hard to make perfectly reproducible run to run — the exact
wording of a trigger or a criterion shapes what gets built. What *is* reproducible, and what this
whole model is built to support, is the discipline: every requirement is a real, checkable claim,
every verification is an explicit, attributable signal (a specific human, or an explicitly-authorized
assistant), and nothing about "done" is ever silently assumed.

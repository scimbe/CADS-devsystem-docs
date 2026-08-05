---
title: The Development System — Docs
description: Documentation for CADS-devsystem, a self-optimizing, agent-driven development pipeline.
---

# The Development System

**CADS-devsystem** ([GitHub](https://github.com/scimbe/CADS-devsystem), tracked at
[CADS-Tunnel#382](https://github.com/scimbe/CADS-Tunnel/issues/382)) is a real, self-optimizing
development pipeline: each stage (plan, implement, test, verify, review, improve, remember) is a
`ServiceType::Custom` role that gets filled through CADS-Tunnel's real crew-auction primitives
(`convene()`) — not a fixed, pre-declared n8n-style workflow. A role-filler's real experience can
propose new stages the project actually needs — a role-filler's own proposal applies immediately, by
explicit design, while a suggestion from the GUI's chat assistant waits for a real human approval;
see [How the pipeline proposes and grows its own stages]({{ '/explanation/self-optimizing-pipeline/' | relative_url }}).

The live control panel is at **[devsystem-demo.bunsenbrenner.org](https://devsystem-demo.bunsenbrenner.org/)**
— a real, interactive GUI, not a static status page.

## Start here

- [Set up your first run]({{ '/tutorials/first-run/' | relative_url }}) — walks through creating a
  real project, writing a fine-grained requirement, and seeing how the pipeline's roles/auction
  state look for a brand-new run. Every screenshot in it was taken from the actual live deployment.

## Sections

This site follows the [Diátaxis](https://diataxis.fr) framework:

- **Tutorials** — learn by doing, start to finish.
- **How-to guides** — accomplish one specific task.
- **Reference** — look up a fact, e.g. the [REST API]({{ '/reference/rest-api/' | relative_url }}).
- **Explanation** — understand why something works the way it does.

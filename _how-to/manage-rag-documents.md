---
title: Add, search, and remove indexed documents
description: Give a run real context beyond its synced repo, and know what happens when you remove one.
order: 6
---

# Add, search, and remove indexed documents

The Docs Search panel indexes more than a run's synced repo -- you can add real notes, pasted
text, or an uploaded file (PDF/DOCX/image) directly, and search across everything together. This
page walks through the real flow against a live run.

## Search is live as you type

Type into the search box and results come back from the real index -- no separate "search"
button, no page reload:

<figure>
<img src="{{ '/assets/img/howto-manage-rag-documents/01-search-results.png' | relative_url }}" alt="The Docs Search panel showing a live keyword match for 'retry' against a real indexed document">
<figcaption>A real match against a manually-added document, with its real score and snippet -- not a mockup.</figcaption>
</figure>

If nothing is indexed yet (no repo synced, no document added), the panel says so plainly instead
of showing an empty results list with no explanation.

## Adding a document two ways

Open **Uploaded documents** to find both real ways to add one:

<figure>
<img src="{{ '/assets/img/howto-manage-rag-documents/02-uploaded-documents.png' | relative_url }}" alt="The Uploaded documents section, open, showing one real document with a Remove button, and the name/text upload form below it">
<figcaption>One real document already added. The plain-text form (name + text) is visible here; a second form below it (not shown) accepts a real file upload.</figcaption>
</figure>

- **Paste or type text directly** -- give it a name (`notes.md`, anything descriptive) and the
  real content. Good for design notes, decisions, anything that isn't already a file.
- **Upload a real file** -- PDF, DOCX, or an image, extracted via the Unstructured API. This
  reports itself honestly as unconfigured if `RAG_UNSTRUCTURED_API_KEY` isn't set on this
  deployment, rather than silently failing or fabricating extracted text.

Both land in the same real index as a repo sync would, searchable immediately.

## Removing one is real and permanent

The **Remove** button next to a document asks for confirmation before doing anything -- and means
it honestly, not as boilerplate:

> Remove "retry-notes.md" from this run's indexed documents? This is real -- there's no undo, and
> an uploaded file's original bytes aren't kept here.

That second half matters: this index only ever stores a manually-added document's extracted
*text*, never the original file. If you remove an uploaded PDF and want it back, you need the real
file again -- there's nothing here to restore from.

Repo-synced files are a genuinely separate mechanism and never show up with their own Remove
button here at all -- the **Uploaded documents** list only ever holds what you added manually. A
file leaves the index because it no longer exists in the repo at the next sync, not through any
individual-delete action on this panel.

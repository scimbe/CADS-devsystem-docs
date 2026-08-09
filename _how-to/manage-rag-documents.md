---
title: Add, search, and remove indexed documents
description: Give a run real context beyond its synced repo, and know what happens when you remove one.
order: 6
---

# Add, search, and remove indexed documents

The Docs Search panel indexes more than a run's synced repo -- you can add real notes, pasted
text, or an uploaded file (PDF/DOCX/image) directly, and search across everything together. This
page walks through the real flow against a live run.

## A genuinely empty run tells you what to do first

Open the panel on a run with nothing indexed yet and it doesn't just sit there quietly -- a real
first-action callout, plus the **Uploaded documents** section auto-expanded so the upload form is
immediately visible instead of hidden behind a click:

<figure>
<img src="{{ '/assets/img/howto-manage-rag-documents/03-empty-state.png' | relative_url }}" alt="The Docs Search panel on a genuinely empty run, showing a start-here banner and the auto-expanded Uploaded documents section">
<figcaption>A real, empty run -- no repo synced, no document added yet. Both the banner and the auto-expand go away the moment anything real is indexed.</figcaption>
</figure>

Unlike the Requirements/Backlog/Milestones panels' own equivalent nudge, this one doesn't
auto-focus a field -- there's no single obvious one here (search is useless against an empty
index, and the real next action lives in two different places: set a repo, or upload). The banner
and the auto-expanded form are the honest substitute.

## Know how stale the repo index is before you trust a result

A synced repo is indexed as of whenever **Sync now** was last clicked -- not continuously, and
GitHub's own unauthenticated API budget (~60 requests/hour) is the real reason this isn't
automatic on every panel open. The sync line says so plainly, with a real "X ago" phrase next to
the raw timestamp, not just a bare date you have to notice and mentally subtract:

<figure>
<img src="{{ '/assets/img/howto-manage-rag-documents/05-sync-staleness.png' | relative_url }}" alt="The Docs Search panel's sync line reading 'Repo: indexed 3 file(s) from main, synced 1 hour ago', with a caveat that this is a snapshot from the last sync">
<figcaption>"1 hour ago" here is real, current data -- not illustrative. The caveat underneath repeats next to search results themselves too, especially for a "No matches" -- the honest cause there could just as easily be a stale index missing the real file as a genuine absence.</figcaption>
</figure>

If you expected a real match and don't see it, **Sync now** before assuming it isn't there.

## Search is live as you type

Type into the search box and results come back from the real index -- no separate "search"
button, no page reload:

<figure>
<img src="{{ '/assets/img/howto-manage-rag-documents/01-search-results.png' | relative_url }}" alt="The Docs Search panel showing a live keyword match for 'retry' against a real indexed document">
<figcaption>A real match against a manually-added document, with its real score and snippet -- not a mockup.</figcaption>
</figure>

If nothing is indexed yet (no repo synced, no document added), the panel says so plainly instead
of showing an empty results list with no explanation.

**Keyword matching always works, with no configuration.** Real semantic matching -- finding a
result with no shared words at all, purely by meaning -- needs a real embedding path configured on
the deployment, and each result honestly says which kind it is (`match_kind: "keyword"` or
`"semantic"`, visible in the raw API response). As of 2026-08-07, two real, independent paths can
provide that, tried in this order: a static `RAG_EMBEDDING_API_KEY` credential (an OpenAI key,
paid, provisioned per deployment) if configured, otherwise the real `devsystem.embedding` pipeline
role over a real CADS-Tunnel Agent-Fabric channel, the same "an auction-discovered ct-agent instead
of a static credential" model `devsystem.document_extraction` above already uses -- see
[The self-optimizing pipeline]({{ '/explanation/self-optimizing-pipeline/' | relative_url }}) for
the real architecture decision behind that. A deployment with neither configured stays honestly
keyword-only; nothing here fabricates a semantic result to look more capable than it is.

## Adding a document two ways

Open **Uploaded documents** to find both real ways to add one:

<figure>
<img src="{{ '/assets/img/howto-manage-rag-documents/02-uploaded-documents.png' | relative_url }}" alt="The Uploaded documents section, open, showing one real document with a Remove button, and the name/text upload form below it">
<figcaption>One real document already added. The plain-text form (name + text) is visible here; a second form below it (not shown) accepts a real file upload.</figcaption>
</figure>

- **Paste or type text directly** -- give it a name (`notes.md`, anything descriptive) and the
  real content. Good for design notes, decisions, anything that isn't already a file.
- **Upload a real file** -- PDF, DOCX, legacy DOC, plain text/markdown, or an image. Two real
  extraction paths, tried in this order: the Unstructured API (`RAG_UNSTRUCTURED_API_KEY`) if
  configured, otherwise the real `devsystem.document_extraction` pipeline role over a real
  CADS-Tunnel Agent-Fabric channel, if *that's* configured instead. As of 2026-08-09 both paths
  handle the same real format set -- PDF, DOCX, legacy DOC, plain text/markdown, and PNG/JPEG/
  TIFF/WebP/BMP/GIF images (real OCR via `tesseract`, matching that role's own real handler
  scope) -- `image/svg+xml` stays unsupported on both, since neither Unstructured nor leptonica
  (the channel path's own OCR library) rasterizes SVG. The response's `extracted_via` field says
  honestly which one actually ran. Reports itself unconfigured with a real `503` naming both if
  neither is set on this deployment, rather than silently failing or fabricating extracted text.

Both land in the same real index as a repo sync would, searchable immediately.

A real, visible confirmation names both the file and the run once the upload actually lands --
until 2026-08-09 this panel's own refresh (needed to show the newly-indexed item in the list)
replaced the whole panel body before the message could ever be seen, and the plain-text upload
path didn't even set one to begin with:

<figure>
<img src="{{ '/assets/img/howto-manage-rag-documents/06-write-success-names-run.png' | relative_url }}" alt="The upload form with a green confirmation reading 'Uploaded retry-notes-48922.md to docs-rag-howto-demo.' beneath it">
<figcaption>Real, current data -- the confirmation names the exact file and run, surviving the panel's own refresh.</figcaption>
</figure>

**Sync now** shows the same kind of confirmation ("Synced: 3/3 file(s), 12 chunk(s), for
&lt;run&gt;.") for the same reason -- knowing a write landed, and where, matters most with several
runs open side by side.

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

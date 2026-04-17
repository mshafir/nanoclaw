# Wiki Skill

Maintain a persistent, interlinked personal knowledge base using the LLM Wiki pattern (Karpathy).

## Architecture

Three layers:

- **`sources/`** — raw, immutable source material. Never modify files here.
- **`wiki/`** — LLM-maintained markdown pages. You own this entirely.
- **`wiki/index.md`** — catalog of all wiki pages (link, one-line summary, tags, source count).
- **`wiki/log.md`** — append-only log of all operations (ingest, query, lint).

## Operations

### Ingest

Triggered when the user drops a URL, PDF, or text into the chat with intent to add it to the wiki.

1. **Save the source** — write raw content to `sources/` with a descriptive filename. For URLs, download the full content:
   ```bash
   # For a webpage (gets full text):
   curl -sL "<url>" | python3 -c "import sys,html2text; print(html2text.handle(sys.stdin.read()))" > sources/title-YYYY-MM-DD.md
   # If html2text not available, use fetch or agent-browser to extract full text
   # For a PDF already in attachments/:
   cp attachments/filename.pdf sources/filename.pdf
   ```

2. **Read and understand** — read the full source. Extract key facts, entities, concepts, arguments, claims, dates, people, tools, references.

3. **Discuss** — briefly summarize the source and its main takeaways to the user before writing wiki pages. This is collaborative — the user may redirect.

4. **Write/update wiki pages** — for this source, create or update all relevant pages:
   - A **summary page** for the source itself (key points, quotes, context)
   - **Entity pages** for people, organizations, tools, products mentioned
   - **Concept pages** for ideas, frameworks, techniques introduced or referenced
   - **Cross-references** — link pages to each other where relevant (`[[page-name]]` style links)
   - Contradiction notes if this source conflicts with existing wiki content

5. **Update `wiki/index.md`** — add or update rows for every page touched.

6. **Append to `wiki/log.md`**:
   ```
   ## [YYYY-MM-DD] ingest | Source Title
   Source: sources/filename.md (or URL)
   Pages created: n | Pages updated: m
   Key additions: brief note
   ```

**One source at a time.** If the user points at multiple files or a folder, process them one by one — read, discuss, write all wiki pages, update index and log — then move to the next. Never batch-read all files and process them together.

### Query

Triggered when the user asks a question about the knowledge base.

1. Read `wiki/index.md` to identify relevant pages.
2. Read those pages in full.
3. Synthesize a coherent answer with citations to wiki pages.
4. If the answer is substantial or reusable, offer to file it as a new wiki page (query answers compound knowledge).

```
## [YYYY-MM-DD] query | Question summary
Answer filed as: wiki/page-name.md (if saved)
```

### Lint

Triggered by the user asking for a health check, or by scheduled task.

Scan the wiki for:
- Contradictions between pages
- Orphan pages (no inbound links from other pages or index)
- Stale claims that newer sources may supersede
- Missing cross-references (two pages discuss related topics but don't link)
- Important concepts or entities that appear in multiple pages but lack their own page
- Gaps (topics the user cares about where knowledge is thin)

Report findings. Offer to fix issues. Suggest sources to pursue for gaps.

```
## [YYYY-MM-DD] lint
Issues found: n | Fixed: m | Deferred: k
```

## Page Format

Use this frontmatter for all wiki pages:

```markdown
---
title: Page Title
tags: [tag1, tag2]
created: YYYY-MM-DD
updated: YYYY-MM-DD
sources: [sources/file.md, https://url]
---

# Page Title

Content...

## See Also
- [[related-page]]
```

## File Naming

- Wiki pages: `wiki/kebab-case-title.md`
- Sources: `sources/descriptive-title-YYYY-MM-DD.md` (or `.pdf` for PDFs)
- Avoid generic names like `notes.md` — be specific

## URL Ingestion Note

WebFetch returns summaries, not full documents. For proper ingestion of web articles where the full text matters, download the content to sources/ first:

```bash
# Webpage — extract readable text
curl -sL "<url>" -o /tmp/raw.html
# Then use agent-browser to open and extract if curl gives raw HTML
# Or: save the URL and note it was fetched via WebFetch (note the limitation)

# PDF from URL
curl -sLo sources/filename.pdf "<url>"
pdf-reader extract sources/filename.pdf
```

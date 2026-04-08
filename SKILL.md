---
name: obsidian-rag
description: >
  Build and maintain a lightweight RAG/wiki knowledge base that compiles raw documents into interlinked Obsidian wiki pages. Four modes — compile (raw/ to wiki/ with incremental updates), query (answer questions from the wiki graph), lint (broken links, inconsistencies, coverage gaps), output (articles, Marp slides, tables). Use when the user wants to compile or update a wiki, ask questions against a knowledge base, run wiki health checks, fix cross-page inconsistencies, generate reports from wiki content, reorganize indices, or trace change propagation. Triggers include "compile the wiki", "update the wiki", "what do we know about", "health check", "I added docs to raw/", "knowledge base", and the raw/-to-wiki/ pipeline. Does NOT apply to reading individual files, writing code, Obsidian plugin setup, or building search infrastructure like embeddings or vector DBs.
---

# Obsidian RAG

A lightweight knowledge base that compiles raw documents into an interlinked Obsidian wiki, then supports Q&A, linting, and output generation — all without vector databases or embeddings.

## Core Philosophy

The wiki is a **knowledge graph stored as markdown**. Raw documents go in, the LLM "compiles" them into interlinked wiki pages — extracting concepts, writing summaries, creating cross-references with `[[wikilinks]]`. This structure replaces vector search with something simpler:

- The **master index** acts as a semantic table of contents
- **Page summaries** (first 1-2 sentences) act as lightweight embeddings — Claude reads them to judge relevance without loading full pages
- **Wikilinks** encode relationships — connected pages are related by definition

This is "light RAG": the same retrieval-augmentation pattern, using markdown structure instead of vector math. It works well at the scale of ~100s of articles and ~400K words. The user browses the same wiki in Obsidian, benefiting from graph view and backlinks.

**The LLM owns the wiki.** The user rarely edits it directly — they add raw documents, ask questions, and request outputs. Their explorations file back into the wiki, so the knowledge base grows with use. Importantly, every interaction is a chance to improve the wiki: when answering a query, fix mistakes you find along the way; when generating an output, file it back. The wiki gets better with every touch.

## Vault Structure

```
<vault-root>/
├── raw/                    # Source documents (user-managed)
│   └── ...                 # Any structure — .md, images, PDFs, flat or nested
├── wiki/                   # Compiled wiki (LLM-managed)
│   ├── _master-index.md    # The backbone — TOC with summaries
│   ├── sources/            # One summary per raw document
│   ├── <category>/         # Semantic groupings (see below)
│   └── outputs/            # Q&A results, reports, slides filed back
└── .wiki-state.json        # Tracks compilation state
```

- **sources/**: One page per raw document. Summarizes it and links to wiki pages. Named after the source file (kebab-cased).
- **Category folders**: Synthesized wiki pages organized into semantic subdirectories that emerge from the content. Common patterns include `concepts/`, `architecture/`, `design-docs/`, `reference/` — but the categories should reflect what the content actually is, not a prescribed taxonomy. A small knowledge base might only need `concepts/`; a large one with design specs, implementation plans, and reference material should separate those into distinct folders so the wiki stays navigable.
- **outputs/**: Artifacts from queries, reports, or analysis. Filed here so explorations accumulate in the knowledge base.

**Choosing categories**: Look at the raw documents and ask "what kind of thing is this?" Design specs, implementation plans, architecture docs, concept explanations, and reference material are different kinds of knowledge and benefit from separate folders. But don't create a folder for fewer than 2-3 pages — keep it simple. When in doubt, fewer folders is better. The master index categories and the folder structure should roughly mirror each other.

The `raw/` structure is flexible — users organize however they want, or dump files flat.

## Master Index

`wiki/_master-index.md` is the single most important file. It's the entry point for both Obsidian browsing and LLM navigation. Every other operation starts here.

```markdown
# Knowledge Base Index

> Last compiled: YYYY-MM-DD | Sources: N | Pages: N | Outputs: N

## Recently Updated
- [[page-name]] — what changed (YYYY-MM-DD)

## Architecture
- [[architecture/page-name]] — one-line summary

## Concepts
- [[concepts/page-name]] — one-line summary

## Design Docs
- [[design-docs/page-name]] — one-line summary

## Reference
- [[reference/page-name]] — one-line summary

## Sources
- [[sources/filename]] — one-line summary of the raw document

## Outputs
- [[outputs/report-name]] — one-line summary
```

The sections above are examples — use whatever categories emerge from the content. The index sections should mirror the wiki folder structure. Don't force categories that don't fit; reorganize when the content demands it. A small wiki might just have Sources and Concepts; a larger one naturally grows more sections.

### Hierarchical Indices

As the wiki grows, the master index gets long. When a category has 8+ pages, give it a sub-index: a `_index.md` file inside the category folder that provides more detail than the master index can.

```
wiki/
├── _master-index.md            # Links to category sub-indices + top pages
├── architecture/
│   ├── _index.md               # Detailed index of architecture pages
│   ├── tech-stack.md
│   └── ...
├── design-docs/
│   ├── _index.md               # Detailed index of design docs with status
│   └── ...
```

The master index links to `[[architecture/_index]]` with a one-line description; the sub-index provides the detailed page listing. This keeps the master index scannable even when the wiki has 100+ pages. For small categories (< 8 pages), just list them directly in the master index — a sub-index would be overhead.

## Wiki Page Format

Every wiki page uses this structure:

```markdown
---
title: Page Title
type: source | concept | architecture | design-doc | reference | output
sources: [raw/file1.md, raw/file2.md]
created: YYYY-MM-DD
modified: YYYY-MM-DD
---

Brief 1-2 sentence summary of what this page covers.

## Content

Main content with [[wikilinks]] to related pages throughout.

## Related
- [[related-page]]
- [[another-page]]
```

The frontmatter `type` matches the folder the page lives in. The `sources` field is critical for impact tracing — when a raw file changes, find all wiki pages that list it as a source.

---

## Mode: Compile

Process raw documents into wiki pages. The core workflow.

**Triggers**: "compile", "update the wiki", "process new docs", "I added files to raw/", or any indication that raw/ has new content.

### Step 1 — Detect Changes

Read `.wiki-state.json` (create it on first run — everything is "new"). List all files in `raw/` and compute hashes:

```bash
find <vault>/raw -type f -exec md5sum {} \;
```

Compare against state. Classify each file as **new**, **modified** (hash differs), **deleted** (in state but not on disk), or **unchanged**. Report the changeset to the user before doing any work.

**Detect near-duplicates**: Raw directories often contain related pairs — a design spec and its implementation plan, a summary and a full document, or versioned drafts of the same thing (e.g., `losses-tab-redesign.md` and `losses-tab-redesign 1.md`). During change detection, identify these pairs by comparing filenames and content overlap. Don't merge them — they serve different purposes — but flag them so that Step 2 can create linked, complementary wiki pages rather than redundant ones.

### Step 2 — Process New Files

For each new raw file:

1. Read the full document
2. Create a **source summary** in `wiki/sources/`:
   - ~150-300 word summary capturing key information
   - Notable data points, decisions, patterns worth preserving
   - Identify concepts that deserve their own pages
   - Link to concepts with `[[wikilinks]]`
3. Create or update **wiki pages** in the appropriate category folder:
   - If the page already exists: add information from the new source, link back to it
   - If it doesn't exist: create it with a clear explanation, links to all relevant sources, and links to related pages
   - Place pages in the folder that matches their nature: architecture docs in `architecture/`, design specs in `design-docs/`, pure concepts in `concepts/`, reference material in `reference/`, etc.
   - A page is anything substantive enough to stand alone: a technology, pattern, process, entity, decision, design spec, or implementation plan — anything that appears across sources or is significant on its own
4. Record in `.wiki-state.json`: file hash + list of wiki pages it contributed to

### Step 3 — Process Modified Files

For each modified raw file:

1. Read the updated document, compare against existing source summary
2. Update the source summary to reflect changes
3. **Trace impact down the chain**:
   - From `.wiki-state.json`, find all wiki pages linked to this raw file
   - Read each linked page and check if changes affect its content
   - For each affected page, follow ITS outgoing `[[wikilinks]]` and check for further impact
   - Update all stale pages — this is a breadth-first traversal of the link graph, stopping when pages are unaffected
4. Update state with new hash and any new wiki pages

### Step 4 — Handle Deletions

If a raw file was removed:
- **Don't auto-delete wiki pages** — synthesized knowledge may still be valuable
- Add a note to the source summary: `> Source document removed from raw/ on YYYY-MM-DD`
- Concept pages that drew from multiple sources remain valid (one fewer source)
- Flag to user so they can decide

### Step 5 — Rebuild Master Index

Rewrite `_master-index.md`:
- Group concepts into natural categories
- Update "Recently Updated" with this compilation's changes
- Update counts
- Keep it scannable — one line per page, summary only

### Step 6 — Save State and Report

Write `.wiki-state.json`. Summarize: files processed, pages created/updated, concepts identified, issues found.

---

## Mode: Query

Answer questions by navigating the already-compiled wiki graph.

**Triggers**: Questions about the knowledge base — "what does the wiki say about X", "how does Y work", "what do we know about Z", or just asking about a topic covered in the wiki.

**Prerequisite**: The wiki must already be compiled. If `wiki/_master-index.md` doesn't exist or is empty, run Compile first. Query mode reads and navigates the existing wiki — it does not rebuild it from raw/.

### Steps

1. **Read the master index** — get the lay of the land
2. **Identify relevant pages** from the index summaries
3. **Read those pages** — start with the most promising. If a page's opening summary suggests it's not relevant, skip to the next
4. **Follow links** — when a page links to something that seems important for the question, follow it
5. **Fix mistakes along the way** — if you encounter errors, stale information, broken links, or missing cross-references in wiki pages while researching, edit them in place. Every query is an opportunity to improve the wiki. Don't just note issues — fix them as you go, the way a careful reader would correct typos in a shared document.
6. **Synthesize** — combine information from multiple pages into a coherent answer
7. **Cite sources** — mention which wiki pages and raw docs the answer draws from
8. **File back** — if the answer synthesizes knowledge in a useful way, save it as a page in `wiki/outputs/` and add it to the master index. This way every question enriches the knowledge base for future queries.

This should rarely require reading more than 5-10 pages. The index → summary → full-page pattern is the whole point of the architecture.

**When the wiki doesn't have the answer**: If the wiki graph traversal comes up empty or the topic isn't covered, say so clearly — "this isn't documented in the wiki" — rather than answering from general knowledge. The whole point of a knowledge base is grounded answers. You can suggest adding raw documents that might fill the gap, or offer to research it with web search and file the results back, but never silently substitute training data for wiki content.

---

## Mode: Lint

Run health checks to improve wiki quality.

**Triggers**: "lint", "health check", "check the wiki", "clean up", "find issues", etc.

### Checks

| Check | What it finds |
|-------|---------------|
| Broken links | `[[wikilinks]]` pointing to non-existent pages |
| Orphan pages | Pages not linked from the index or any other page |
| Stale sources | Source summaries whose raw files changed since last compile |
| Index drift | Pages that exist on disk but aren't in the master index |
| Inconsistencies | Contradictory information across related pages |
| Missing connections | Concepts in multiple pages but not cross-linked |
| Coverage gaps | Topics mentioned in passing but never given a concept page |
| Suggested explorations | New questions or article ideas to deepen the knowledge base |

### Report Structure

The lint report should be a well-organized document with these sections:

1. **Summary** — Quick stats table: pages, links, broken links, orphans
2. **Link Health** — Broken links, orphaned pages, link density analysis
3. **Inconsistencies** — Contradictory information across pages, with specific quotes and recommendations
4. **Coverage Gaps** — Topics mentioned but not given their own pages
5. **Cross-Reference Quality** — Well-connected paths through the wiki, and missing cross-references that should exist
6. **Structural Suggestions** — Folder reorganization, page splits, index improvements
7. **Action Items Summary** — A priority table that makes it easy to act on findings:

```markdown
## Action Items Summary

| Priority | Action | Effort |
|----------|--------|--------|
| High | Fix broken link X in page Y | Trivial |
| Medium | Reconcile version numbers across pages | Quick edits |
| Low | Create dedicated page for topic Z | New page |
```

Offer to auto-fix mechanical issues (broken links, index drift) immediately. Flag subjective issues for the user to decide.

---

## Mode: Output

Generate formatted artifacts from wiki knowledge.

**Triggers**: "write a report on", "create slides about", "summarize X as", "generate a Y from the wiki", etc.

### Formats

- **Markdown article**: Structured document synthesizing wiki pages on a topic
- **Marp slides**: `---`-separated slide deck with Marp frontmatter, viewable in Obsidian with the Marp plugin
- **Data tables**: Comparisons, timelines, reference tables
- **Custom**: Whatever the user requests

Save outputs to `wiki/outputs/`, add to the master index, and link back to source wiki pages. Explorations always accumulate.

---

## Model Allocation

Use a mix of model tiers to balance quality and cost. The principle: **Haiku for mechanical work, Sonnet for content creation, Opus for complex reasoning.**

### Compile

| Step | Model | Why |
|------|-------|-----|
| Detect changes (hashing, diffing) | **Haiku** or scripting | Pure mechanical comparison — no reasoning needed |
| Create source summaries | **Sonnet** | Good at summarization and concept extraction; each file is processed independently |
| Create/update wiki pages | **Sonnet** | Writes clear explanations, handles cross-linking well |
| Impact tracing (modified files) | **Opus** | Requires judgment about what downstream pages are affected by a change — multi-hop reasoning across the link graph |
| Rebuild master index | **Sonnet** | Categorization and summary writing |
| Save state file | **Haiku** or scripting | JSON serialization |

When compiling many files, spawn **parallel Sonnet subagents** — one per raw file for source summaries, then a second pass for wiki pages. This is the biggest speed win.

### Query

| Scenario | Model | Why |
|----------|-------|-----|
| Triage: read index, identify relevant pages | **Haiku** | Quick scan, no deep reasoning |
| Straightforward factual questions | **Sonnet** | Read a few pages, synthesize a direct answer |
| Complex multi-hop questions | **Opus** | Needs to follow chains of links, reconcile information across many pages, and reason about connections |
| Fix mistakes found during Q&A | **Sonnet** | Editing existing pages based on clear issues |
| Write the output page | **Sonnet** | Structured writing with citations |

For most queries, **Sonnet handles the full pipeline**. Escalate to Opus when the question requires reasoning across 5+ pages or finding non-obvious connections.

### Lint

| Check | Model | Why |
|-------|-------|-----|
| Broken links, orphans, index drift | **Haiku** or scripting | File existence checks, link parsing — no understanding needed |
| Inconsistencies across pages | **Opus** | Must read related pages and spot contradictions — requires careful comparison |
| Coverage gaps, missing connections | **Sonnet** | Scan pages for mentioned-but-unlinked concepts |
| Suggested explorations | **Opus** | Creative reasoning about what's missing and what questions would deepen the knowledge base |
| Write the lint report | **Sonnet** | Structured report writing |

### Output

| Format | Model | Why |
|--------|-------|-----|
| Simple summaries, data tables | **Sonnet** | Straightforward extraction and formatting |
| Complex synthesis articles | **Opus** | Drawing connections across many wiki pages |
| Marp slides | **Sonnet** | Formatting-heavy, content is already synthesized in the wiki |

### Practical Notes

- When spawning subagents, use the `model` parameter to set the tier: `model: "haiku"`, `model: "sonnet"`, `model: "opus"`
- If running in a context where subagents aren't available, default to the model you're running on and don't worry about it — the allocation above is a cost optimization, not a correctness requirement
- When in doubt, Sonnet is the safe default. Upgrade to Opus for reasoning-heavy tasks; downgrade to Haiku for mechanical checks

---

## Writing Guidelines

- **Wikilinks everywhere**: `[[page-name]]` for all internal references. This powers Obsidian's graph view AND Claude's navigation. Over-link rather than under-link.
- **Summaries first**: Start every page with 1-2 sentences. This is what makes light-RAG work — Claude triages by reading summaries, not full pages.
- **One concept per page**: Keep pages focused. Split pages that grow beyond ~800 words into sub-topics.
- **Kebab-case filenames**: `my-concept-name.md` — short but descriptive.
- **Preserve human edits**: If the user has manually edited a wiki page, add to it — don't overwrite.
- **Embed images**: Reference images from raw/ using `![[image-name.png]]`.
- **Natural tone**: Clear and informative, like a good internal wiki. Concise but not terse.
- **Frontmatter always**: Every page gets the YAML frontmatter block. It's essential for impact tracing and state management.

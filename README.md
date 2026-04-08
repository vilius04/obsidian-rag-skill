# Obsidian RAG Skill

This repository contains a Cursor Agent Skill named `obsidian-rag`.

The skill helps an agent build and maintain a lightweight knowledge base in Obsidian by compiling documents from `raw/` into an interlinked `wiki/`, then using that wiki for question answering, linting, and output generation.

## What It Does

The skill is designed for four kinds of work:

- `compile`: turn source material in `raw/` into linked wiki pages in `wiki/`
- `query`: answer questions by navigating the compiled wiki
- `lint`: check the wiki for broken links, stale pages, inconsistencies, and coverage gaps
- `output`: generate reports, articles, slides, and tables from wiki content

It is intentionally based on markdown structure, summaries, and `[[wikilinks]]` rather than embeddings or a vector database.

## Install The Skill

Cursor skills live in a directory that contains a `SKILL.md` file.

Use one of these locations:

- Personal skill: `~/.cursor/skills/obsidian-rag/`
- Project skill: `.cursor/skills/obsidian-rag/`

To install this skill:

1. Copy this repository's contents into a folder named `obsidian-rag`.
2. Make sure `SKILL.md` is directly inside that folder.
3. Place the folder in either `~/.cursor/skills/` or `.cursor/skills/`.

Expected result:

```text
obsidian-rag/
├── README.md
└── SKILL.md
```

## When To Use It

Use this skill when you want the agent to work with a document-backed Obsidian knowledge base, for example:

- "Compile the wiki from the files in `raw/`"
- "I added new docs to `raw/`, update the knowledge base"
- "What do we know about X from the wiki?"
- "Run a health check on the wiki"
- "Create a report from the knowledge base"
- "Find inconsistencies or missing links in the wiki"

Do not use it for:

- reading or editing a single unrelated markdown file
- Obsidian plugin setup
- building embedding search or a vector database
- general software engineering tasks outside the wiki workflow

## Expected Vault Layout

The skill assumes a vault or workspace that looks roughly like this:

```text
<vault-root>/
├── raw/
│   └── ...
├── wiki/
│   ├── _master-index.md
│   ├── sources/
│   ├── concepts/
│   ├── architecture/
│   ├── design-docs/
│   ├── reference/
│   └── outputs/
└── .wiki-state.json
```

Notes:

- `raw/` holds source documents managed by the user
- `wiki/` holds synthesized pages managed by the agent
- `wiki/_master-index.md` is the main navigation entry point
- `.wiki-state.json` tracks source file state and wiki impact

The category folders under `wiki/` are examples, not strict requirements. They should reflect the content actually present in the knowledge base.

## How To Use It

### 1. Compile The Wiki

Start here if the wiki does not exist yet or if source files changed.

Example prompts:

- "Compile the wiki from `raw/`"
- "I added documents to `raw/`; update the wiki"
- "Detect changes and rebuild the knowledge base"

What the agent should do:

1. Inspect `raw/` and compare it with `.wiki-state.json`
2. Classify files as new, modified, deleted, or unchanged
3. Create or update `wiki/sources/` pages
4. Create or update synthesized wiki pages with `[[wikilinks]]`
5. Rebuild `wiki/_master-index.md`
6. Save updated state to `.wiki-state.json`

### 2. Query The Knowledge Base

Use this after the wiki has already been compiled.

Example prompts:

- "What does the wiki say about retrieval?"
- "How does the architecture handle document updates?"
- "What do we know about topic X from the knowledge base?"

What the agent should do:

1. Read `wiki/_master-index.md`
2. Identify relevant pages from their summaries
3. Read only the necessary wiki pages
4. Follow important `[[wikilinks]]`
5. Synthesize an answer grounded in the wiki
6. Optionally save a useful synthesis to `wiki/outputs/`

Important behavior:

- answers should stay grounded in the wiki, not fall back silently to general model knowledge
- if the wiki is missing the answer, the agent should say so clearly
- while researching, the agent should fix obvious wiki issues it encounters

### 3. Lint The Wiki

Use this to improve quality and maintainability.

Example prompts:

- "Run a wiki health check"
- "Lint the knowledge base"
- "Find broken links and inconsistencies"

Typical checks include:

- broken `[[wikilinks]]`
- orphan pages
- stale source summaries
- pages missing from the master index
- contradictory information across pages
- missing cross-links
- topics that deserve dedicated pages

### 4. Generate Outputs

Use this when you want deliverables generated from the wiki.

Example prompts:

- "Write a report on topic X from the wiki"
- "Create Marp slides about the system architecture"
- "Make a comparison table from the knowledge base"

Expected output location:

- `wiki/outputs/`

Useful outputs include:

- markdown articles
- Marp slide decks
- reference tables
- comparison summaries

## Recommended Prompt Patterns

These prompts make the skill easier for the agent to apply correctly:

```text
Compile the wiki from the documents in raw/ and update the master index.
```

```text
I added new files to raw/. Detect changes, update affected wiki pages, and tell me what changed.
```

```text
Answer this only from the wiki: what do we know about <topic>?
```

```text
Run a lint pass on the wiki and give me a prioritized action list.
```

```text
Create a report in wiki/outputs/ summarizing <topic> from the knowledge base.
```

## Authoring Expectations

The skill expects the agent to preserve the wiki as a readable Obsidian knowledge graph:

- use `[[wikilinks]]` for internal references
- begin pages with short summaries
- keep pages focused and split oversized pages when needed
- use kebab-case filenames
- preserve human edits when updating existing pages
- include frontmatter on wiki pages for tracing and maintenance

## Practical Workflow

In day-to-day use, the workflow usually looks like this:

1. Add or update source documents in `raw/`
2. Ask the agent to compile or update the wiki
3. Ask questions against the compiled wiki
4. Periodically run linting to improve structure and correctness
5. Save useful reports and synthesized outputs back into `wiki/outputs/`

This lets the knowledge base improve over time instead of treating every answer as disposable.

## Limitations

This skill is a workflow guide, not a standalone application.

It assumes:

- the agent can read and write files in the vault
- the wiki is small enough to navigate through summaries and links
- the knowledge base is maintained as markdown in Obsidian

It does not define:

- a vector search pipeline
- an embeddings store
- an Obsidian plugin
- a separate backend service

## Repository Contents

- `SKILL.md`: the Cursor skill definition
- `README.md`: usage instructions for humans

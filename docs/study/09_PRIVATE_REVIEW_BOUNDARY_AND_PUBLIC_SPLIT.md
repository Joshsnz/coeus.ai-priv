# Private Review Boundary and Public/Private Split

## Why this document exists

Coeus is large and technically deep. The public repo should be polished and understandable, but it should not expose the full implementation, private prompts, traces, provider config, or source internals. The private repo should provide evidence for trusted review without becoming a messy source dump.

## Public materials should include

- project overview
- demo release link
- architecture summary
- public case study
- rendered eval reports
- benchmark framing
- screenshots/video demo or placeholders
- known limitations
- security/privacy boundary
- concise private review package note

## Private materials should include

- clean codebase metrics
- source file manifest
- codebase structure notes
- deep architecture notes
- turn pipeline notes
- retrieval/context notes
- tools/planning notes
- telemetry/eval/headless notes
- technical Q&A
- optional selected source or full source for trusted reviewers
- performance measurements once captured

## Publicly safe high-level phrasing

Use this publicly:

> Coeus AI uses a policy-first turn pipeline that separates turn semantics, retrieval/source evidence, prompt packing, validation, command/tool execution, telemetry, and headless evaluation.

Also safe:

> A private review package is available on request with clean codebase metrics, source-tree structure, architecture notes, selected implementation evidence, screenshots, and evaluation material.

## Keep private

Do not publish:

- full source tree
- private prompts/templates if sensitive
- provider config
- env files
- secrets
- local traces
- raw tool traces
- raw headless protocol logs if they contain private project paths/data
- marker truth internals
- private eval harness code if not ready
- full file-by-file complexity report
- raw block analysis

## Clean metrics statement

This is safe public/private wording:

> A clean local scan of the private source tree, excluding generated parser code, static assets, build output, vendor folders, and environment files, reports 364 C/C++ source/header files and approximately 144k code lines.

Avoid public COCOMO dollar estimates. They are heuristic and can distract.

## Private repo recommended layout

```text
coeus-ai-private-review/
├── README.md
├── CODEBASE_OVERVIEW.md
├── CODEBASE_STRUCTURE.md
├── ARCHITECTURE_NOTES.md
├── TURN_PIPELINE_NOTES.md
├── RETRIEVAL_CONTEXT_NOTES.md
├── TOOLS_PLANNING_NOTES.md
├── TELEMETRY_EVAL_NOTES.md
├── TECHNICAL_QA_NOTES.md
├── PERFORMANCE_NOTES.md
├── PRIVATE_REVIEW_BOUNDARY.md
├── metrics/
│   ├── scc_summary_clean.txt
│   ├── scc_by_file_clean.txt
│   ├── scc_summary_clean.json
│   └── scc_by_file_clean.json
├── structure/
│   ├── source_file_manifest_clean.txt
│   └── codebase_tree_clean.txt
├── evidence/
│   └── public_links.md
└── source/
    └── optional selected source or full private source
```

## What “evidence” means

Evidence is any artifact that supports a claim:

| Claim | Evidence |
|---|---|
| native C++ app | source tree, GUI files, release binary, screenshots |
| project-aware | project manager, file context, retrieval, eval reports |
| not chatbot wrapper | GUI + runtime + retrieval + tools + telemetry source evidence |
| source grounded | retrieval/controller/lattice docs + RepoGrounding reports |
| eval discipline | headless runtime + eval reports + marker truth notes |
| private/professional boundary | `.gitignore`, private boundary docs, excluded artifacts |

## Reviewer access levels

### Level 1: Public reviewer

Gets public page, release, eval reports, docs, video.

### Level 2: Private proof reviewer

Gets private review repo with metrics, structure, architecture notes, technical Q&A, selected screenshots/evidence.

### Level 3: Source reviewer

Gets selected source or full sanitized source tree, depending on trust and need.

## Performance notes

Performance belongs mostly private until measured. Public can mention that runtime/performance notes are available on request. Private notes should include repeatable measurements:

- executable size
- idle memory after launch
- memory after project load/index
- memory after several repo QA turns
- CPU idle behavior
- hardware/build/scenario/method

## Final guidance

Public material should be clear and impressive but not overwhelming. Private material should be deep and reviewable. The source study pack should help you answer hard questions, but it does not all need to be published.

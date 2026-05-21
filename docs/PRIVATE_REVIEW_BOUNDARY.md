# Coeus AI — Private Review Boundary

## Purpose

This document defines what the private review package can include and what should remain excluded.

## Included

The private review package may include:

- clean codebase metrics,
- source file manifest,
- codebase structure notes,
- architecture notes,
- selected implementation evidence,
- selected screenshots,
- public evidence links,
- evaluation summaries,
- and optional selected source or broader source access depending on review context.

## Excluded

The private review package should exclude:

- `.env` and `.env.*` files,
- provider keys,
- secrets,
- private provider configuration,
- local traces,
- private project data,
- generated caches,
- build outputs,
- binaries unless needed for a specific review,
- private workspaces,
- raw prompt/provider credentials,
- and unnecessary generated parser/static asset files.

## Tree-sitter / generated code note

Tree-sitter parser support exists in the private source tree, but generated Tree-sitter parser code is excluded from clean application-code metrics and the clean source manifest to avoid inflating the codebase size.

## Public vs private

Public repository:

- demo release,
- public docs,
- public eval reports,
- public case study,
- release notes,
- limitations/security notes.

Private review package:

- clean metrics,
- codebase map,
- source structure,
- source-level architecture notes,
- selected source evidence,
- reviewer Q&A,
- optional private source.

## Sharing rule

Share only the minimum necessary for the review context. Start with metrics and structure. Add selected source only when needed. Share full source only with a reviewer who genuinely needs implementation-level access.

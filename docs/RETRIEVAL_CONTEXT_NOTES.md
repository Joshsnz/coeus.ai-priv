# Coeus AI — Retrieval and Context Notes

## Purpose

This document summarizes the retrieval/context side of Coeus: how local project source becomes searchable evidence, how context is selected, and how source grounding differs from summaries or continuity.

## Main areas

```text
core/cle/file_context/
core/cle/index/
core/cle/inventory/
core/cle/retrieval/
core/cle/lattice/
core/cle/memory/
core/cle/summary/
core/cle/prompt_ir/
```

## File context

File context handles the active project file surface:

- scanning,
- supported language mapping,
- file identity,
- chunking/slicing,
- file context service,
- auto-ingest,
- file-context search.

Representative files:

```text
core/cle/file_context/file_context_service.cpp
core/cle/file_context/file_context_search.cpp
core/cle/file_context/chunker.cpp
core/cle/file_context/file_context_auto_ingest.cpp
```

## Inventory and index

Inventory and index layers support project structure awareness:

- inventory state,
- inventory gist building,
- graph index builder/store,
- document identity.

Representative files:

```text
core/cle/inventory/inventory_state_v1.cpp
core/cle/inventory/inventory_gist_builder.cpp
core/cle/index/graph_index_builder.cpp
core/cle/index/graph_index_store.cpp
```

## Retrieval

Retrieval supports source-grounded answering:

- BM25 database,
- hybrid search,
- retrieval policy,
- retrieval details,
- retrieval controller.

Representative files:

```text
core/cle/retrieval/bm25_db.cpp
core/cle/retrieval/hybrid_search.cpp
core/cle/retrieval/retrieval_controller.cpp
core/cle/retrieval/retrieval_detail.cpp
core/cle/retrieval/retrieval_policy.cpp
```

## Lattice / prompt IR

The context lattice and prompt IR represent the assembled evidence/context that can be packed into the model request.

Representative files:

```text
core/cle/lattice/lattice.cpp
core/cle/lattice/lattice_runtime.cpp
core/cle/prompt_ir/prompt_block_v1.h
core/cle/prompt_ir/prompt_ir.h
```

## Memory and summaries

Memory and summaries support continuity and routing, but they should not silently become source proof.

Representative files:

```text
core/cle/memory/vector_memory_service.cpp
core/cle/memory/memory_store.cpp
core/cle/memory/memory_graph.cpp
core/cle/summary/summary_service.cpp
core/cle/summary/meta_summary.cpp
core/cle/summary/gist_service.cpp
```

## Key principle

Retrieval/source evidence, summaries, and continuity should remain distinct:

- Source evidence proves claims.
- Retrieval results show what was found.
- Packed context shows what the model actually saw.
- Summaries and memory can guide/rank/route but should not be treated as proof by themselves.

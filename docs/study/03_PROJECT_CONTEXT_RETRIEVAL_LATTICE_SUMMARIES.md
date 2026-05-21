# Project Context, Retrieval, Lattice, Memory, and Summaries

## Why this subsystem matters

This is the core of “project-aware” Coeus. The assistant should answer from the loaded project, not from generic model memory. That requires:

- project state
- file inventory
- file context service
- chunking/slicing
- indexing
- retrieval
- source evidence
- summaries/gists
- memory/snapshots
- context/lattice packing

This subsystem is where the public phrase “retrieval-backed codebase QA” becomes source-level reality.

## Project state

Relevant files:

```text
core/project/project_manager.cpp
core/project/project_io.cpp
core/project/project_actions.cpp
core/project/project_data.cpp
```

Main responsibilities:

- project lifecycle
- project persistence
- project IO
- project actions
- active project state
- file context root relationship

The project layer is the boundary between a local software project and the AI workspace. It determines what project is active, what root is loaded, and what project state is saved/restored.

## File context service

Relevant files:

```text
core/cle/file_context/file_context_service.cpp
core/cle/file_context/file_context_search.cpp
core/cle/file_context/file_context_auto_ingest.cpp
core/cle/file_context/file_context_support.cpp
core/cle/file_context/chunker.cpp
core/cle/file_context/chunk_hasher.h
core/cle/file_context/lang_map.h
core/cle/file_context/slice_meta.cpp
```

The file-context subsystem is responsible for turning files into usable model/retrieval inputs.

It likely owns or supports:

- root/folder scanning
- file inclusion/exclusion
- chunking
- language mapping
- slice metadata
- hash tracking
- auto-ingest/rescan behavior
- file context search
- support utilities for active root/file context state

## Inventory and index

Relevant files:

```text
core/cle/inventory/inventory_state_v1.cpp
core/cle/inventory/inventory_gist_builder.cpp
core/cle/index/graph_index_builder.cpp
core/cle/index/graph_index_store.cpp
```

Inventory is useful when the question is about what exists in the project. The key is to distinguish:

- inventory/listing truth: what files or symbols are present
- source-evidence truth: what specific source content proves a claim
- summary/gist truth: compact memory of project structure, not full evidence

The index layer supports project graph/index structures used by retrieval and project context.

## Retrieval

Relevant files:

```text
core/cle/retrieval/bm25_db.cpp
core/cle/retrieval/hybrid_search.cpp
core/cle/retrieval/retrieval_controller.cpp
core/cle/retrieval/retrieval_detail.cpp
core/cle/retrieval/retrieval_policy.cpp
```

Retrieval is not just “search and paste.” The review indicates retrieval is controlled by policy/resolved truth and then fed into context assembly.

Key concepts:

- BM25 lexical retrieval
- hybrid search
- retrieval policy
- retrieval detail/reporting
- retrieval controller orchestration

The retrieval controller is a central piece of source-grounding. It is where the app decides what candidate files/chunks are available for a turn, subject to policy.

## Lattice and prompt IR

Relevant files:

```text
core/cle/lattice/lattice.cpp
core/cle/lattice/lattice_runtime.cpp
core/cle/prompt_ir/prompt_block_v1.h
core/cle/prompt_ir/prompt_ir.h
```

The lattice is the conceptual heart of context assembly. It helps coordinate different types of context: source files, summaries, memory, retrieval results, user context, tool observations, and prompt blocks.

A useful way to explain it:

> The context lattice is the internal structure that decides how different context planes become a compiled prompt/context package. It prevents “whatever is available” from being treated as equal evidence. Source evidence, summaries, memory, tool observations, and policy markers are separate planes.

## Memory and snapshots

Relevant files:

```text
core/cle/memory/vector_memory_service.cpp
core/cle/memory/memory_store.cpp
core/cle/memory/memory_graph.cpp
core/cle/memory/embed_cache.cpp
core/cle/memory/memory_seed_builder.cpp
core/cle/memory/snapshot_builder.cpp
core/cle/memory/memory_snapshot.h
```

Memory helps continuity, recall, and project/session state. But it is important to avoid overstating it as source proof.

Doctrine:

- memory can support continuity
- memory can help retrieve or summarize
- memory is not the same as current source evidence
- memory should not let the model claim implementation details unsupported by current source context

This distinction is central to preventing hallucination.

## Summaries and gists

Relevant files:

```text
core/cle/summary/summary_service.cpp
core/cle/summary/summary_queue.cpp
core/cle/summary/summary_worker.cpp
core/cle/summary/file_summary_queue.cpp
core/cle/summary/gist_service.cpp
core/cle/summary/meta_summary.cpp
core/cle/summary/meta_summary_artifact.cpp
core/cle/summary/summary_keys.cpp
core/cle/summary/summary_paths.cpp
```

Summaries make a large project usable under context limits. They can provide:

- file summaries
- project gists
- meta summaries
- queued/worker-based summary creation
- summary artifacts

But summaries must remain separate from source proof. A summary can guide retrieval or orient the model, but exact implementation claims should be grounded in files/chunks when possible.

## User context aggregator

Relevant files:

```text
core/cle/user_context/user_context_aggregator.cpp
```

This likely helps combine user-level context with project context. It should be understood as another context plane, not as direct source evidence.

## Retrieval/source grounding lifecycle

A simplified flow:

```text
active project/root
→ file context scan/ingest
→ file records/chunks/slices
→ inventory/index
→ summaries/gists/memory
→ retrieval controller
→ lattice/context packing
→ prompt policy projection
→ model-visible source/context blocks
```

## Critical boundary: source evidence vs continuity

A central review question is:

> Does Coeus answer from source evidence, or does it merely remember/restate earlier text?

The right answer is:

> Coeus separates source/file context, summaries, memory, and telemetry. The app can use summaries/memory for continuity and routing, but source-grounded claims should come from project file context or retrieved source evidence. This is also tested by RepoGrounding evals through negative grounding, source tracing, and artifact suppression scenarios.

## Risk and audit points

Watch for:

- summaries being treated as evidence
- stale memory overriding current source
- retrieval details becoming user-facing proof without packing/validation
- project root/file-context root drift
- large retrieval outputs that are too broad to be useful
- hybrid retrieval details not being traceable enough for eval/debugging

## Reviewer talking point

> Coeus has a real project-context substrate: file context, chunking, inventory, graph index, BM25/hybrid retrieval, vector memory, summaries, and a context lattice. This lets the assistant reason over the currently loaded project instead of answering from generic model memory alone.

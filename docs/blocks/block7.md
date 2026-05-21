Block 7 Part 1 Summary — CLE Routers, Document Identity, Graph Index, and Inventory Gists

This first part of Block 7 covers the outer CLE router headers, document identity canonicalization, graph index construction/caching, and inventory/gist generation.

It does not yet cover the actual file ingestion/chunking/slicing implementation, because the file_context/* files are not included in this message. So the full “project files become searchable context” story is only partially visible here. This part shows how the project’s already-tracked file surface is turned into inventory artifacts, inventory gists, and a lightweight graph index.

The strongest reviewer-facing point from this part is:

Coeus does not treat a project as an anonymous text blob. It maintains a tracked file surface, derives deterministic inventory artifacts, emits vector-searchable inventory gists, canonicalizes document identities, and builds a lightweight graph index from semantic summaries and file/folder relationships.

1. Batch-level role

This part of Block 7 shows four related surfaces:

CLE router headers
  → public/narrow include surfaces for CLE subsystems

DocIdentity
  → canonical identity handling for repo paths vs gist/stable keys

GraphIndexBuilder / GraphIndexStore
  → deterministic project file/folder graph from semantic summary artifacts

InventoryState / InventoryGistBuilder
  → deterministic project inventory text, markdown/json artifacts, vector-ingested inventory gists

The control flow implied here is:

Project.fileContextItems
  → inventory state / inventory gist artifacts
  → vector memory gist rows

Project.fileContextItems
  → graph nodes: root, folders, files
  → semantic summary artifact parsing
  → graph edges: includes, contains, entrypoints, nav, folder deps

GraphIndexStore
  → process-lifetime cached graph per project
  → invalidation by generation
2. Router headers

Files:

core/cle/cle.h
core/cle/cle_docs.h
core/cle/cle_file_context.h
core/cle/cle_index.h
core/cle/cle_inventory.h
core/cle/cle_lattice.h
core/cle/cle_memory.h
core/cle/cle_retrieval.h
core/cle/cle_summary.h
core/cle/cle_user_context.h

These are subsystem router headers. Their role is mostly architectural hygiene.

They provide controlled include surfaces such as:

cle_file_context.h
  file_context_service
  file_context_search
  chunker
  slice_meta
  chunk_hasher

cle_inventory.h
  inventory_state_v1
  inventory_gist_builder

cle_index.h
  graph_index_builder
  graph_index_store

cle_retrieval.h
  retrieval_controller
  hybrid_search
  bm25_db

The top-level cle.h includes only major CLE areas:

#include "core/cle/cle_lattice.h"
#include "core/cle/cle_inventory.h"
#include "core/cle/cle_retrieval.h"
#include "core/cle/cle_summary.h"

What this proves:
The CLE is organized as a set of identifiable subsystems rather than one monolithic “context” module. There are distinct layers for file context, inventory, graph index, memory, retrieval, summaries, user context, and lattice packing.

Reviewer-facing wording:
“The CLE exposes narrow router headers for each major context subsystem. This keeps translation units from directly depending on deep internals and makes the source layout easier to review.”

3. doc_identity.h — repo path vs gist/stable-key identity
Role

core/cle/docs/doc_identity.h defines canonical identity handling for retrieved memory items and gist-like items.

The important convention:

static constexpr std::uint64_t kGistIdBit = 0x8000000000000000ull;

A memory item whose ID has the high bit set is treated as gist-like:

IsGistId(id)
Main problem it solves

A retrieved item may represent either:

real repo file slice
  e.g. src/core/agent/runtime/agent_flow.cpp

stable gist key
  e.g. __inventory__/all.json
  e.g. __inventory__/folders/core/agent.summary.md
  e.g. summary/meta stable keys

Those two should not be confused.

DocIdentity::CanonicalizeRelPath() returns:

CanonResult
  gistLike
  value
  repoRel

The important logic is:

If high-bit gist ID:
  treat relPath as stable key / gist identity

Else if relPath looks like stable summary key:
  ask hasRepoFile(relPath)
    if real repo file exists, treat as repo path
    otherwise treat as stable key

Else:
  treat as repo path

This handles an important edge case: a real repo could theoretically contain a path that looks like files/..., so the canonicalizer asks whether that path maps to a real repo file before treating it as a stable summary key.

What this proves

This is evidence that Coeus tracks the difference between:

source evidence identity
  actual repo file/slice

routing/context identity
  gist, summary, inventory, stable key

That aligns with the architecture principle:

Summary/inventory routes.
Source evidence proves.

Reviewer-facing wording:
“DocIdentity prevents gist and summary identities from being collapsed into ordinary repo file paths. It canonicalizes retrieved item IDs into either repo-relative paths or stable gist keys, preserving the distinction between source evidence and derived context.”

4. graph_index_builder.* — deterministic lightweight project graph
Role

GraphIndexBuilder builds a deterministic graph over the project’s tracked file surface.

It creates node types:

Root
Folder
File

and edge types:

Include
Mention
Contains
DependsOnFolder
NavNext
NavStartHere
Entrypoint

The graph is built from:

Project.fileContextItems
  → included files
  → folder derivation
  → root/folder/file nodes

Semantic summary artifacts
  → graph edges and key symbols

Optional source scanning exists, but is off by default:

bool fallback_to_source_scan{false};

That means the primary mode is summary-native, not raw-source scanning.

Graph node construction

The builder gathers file nodes from:

proj->fileContextItems

It applies:

included_only filter
canonical relpath normalization
sort files ascending
derive folder parents
sort folders ascending

Then creates deterministic node order:

id 0: root
then folders ascending
then files ascending

Node keys are stable:

__root__
folder:<rel>
file:<rel>

This deterministic node order is important for reproducible graph fingerprints and stable diagnostics.

Semantic-summary parsing

The primary graph input is Phase-10-style semantic summaries.

The builder reads summary artifacts via:

PathUtils::SemanticSummaryArtifactPath(...)

For each node, it parses only strict summary sections:

## Key symbols
## Notes

It expects bullet-only graph metadata in ## Notes, such as:

- graph.node: file src/main.cpp
- graph.includes_project: core/foo.h
- graph.includes_external: vector, string
- graph.contains: file src/main.cpp
- graph.depends_on_folders: core/agent
- graph.entrypoints: src/main.cpp
- nav.start_here: folder core/agent
- nav.next: file core/agent/runtime/agent_flow.cpp

It validates graph.node against the node being parsed. If the summary claims the wrong node, the semantic artifact is rejected for graph purposes:

mismatched_semantic_nodes++

This is a good correctness feature. It prevents stale or misplaced summary artifacts from silently creating graph edges for the wrong file/folder.

Edge behavior

Edges are deduplicated and weighted:

same edge kind + same target
  → weight increments

Then sorted deterministically.

Caps exist for bounded graph construction:

max_includes_per_file
max_mentions_per_file
max_contains_per_folder
max_nav_per_node
max_entrypoints
max_key_symbols_per_node
max_external_includes_per_file
Optional legacy fallback

If enabled:

fallback_to_source_scan = true

the builder can scan raw source for:

#include lines
relpath-like mentions

But comments state this is off by default to keep the graph summary-native.

Important reviewer distinction:
This is not a full compiler/AST index. It is a lightweight deterministic graph index built primarily from semantic summaries and file-context membership, with optional raw-source fallback.

Graph fingerprint

The builder computes:

idx.fingerprint

from:

builder version string
nodes
node kinds
node rels
edges
key symbols
external includes

This supports reproducibility and cache validation.

BuildFromSemanticSummaries

There is also a convenience function:

BuildFromSemanticSummaries(const Summary::RepoSnapshot& snap)

It constructs a minimal temporary Project, populates fileContextItems from snap.includedFiles, preserves project/file-context roots, and delegates to GraphIndexBuilder::BuildForProject.

That is important because it allows graph building from a snapshot without forcing higher layers to know graph-builder details.

What this proves

Safe claims:

Coeus builds a deterministic graph over tracked project files and folders.

Graph nodes are derived from the included file-context surface.

Graph edges can represent project includes, folder containment, folder dependencies, entrypoints, and navigation hints.

Graph construction prefers semantic summary artifacts and validates graph.node identity before trusting edge metadata.

Graph index generation has deterministic sorting, edge dedupe, caps, and fingerprinting.
5. graph_index_store.* — process-lifetime graph cache
Role

GraphIndexStore is a thread-safe per-project cache for graph indexes.

It owns:

pid → Entry

Each entry has:

generation
built_generation
building flag
building_generation
cached index
last_error
condition_variable
Cache behavior

Fast path:

GetIfCached(pid)
  → returns cached graph only if built_generation == generation

Build path:

GetOrBuild(pid, buildFn)
  → if cached current: return hit
  → if build in progress: wait
  → otherwise mark building
  → build outside lock
  → commit only if generation unchanged
  → discard if invalidated during build

This is a good concurrent-cache pattern.

Invalidation behavior
Invalidate(pid)

does:

generation++
clear cached index
notify waiters

If a build was already running, it will self-discard when it returns because its started generation no longer matches.

Logging

Stable log prefixes:

[GraphIndex] hit
[GraphIndex] build
[GraphIndex] build_ok
[GraphIndex] build_fail
[GraphIndex] build_discard
[GraphIndex] invalidate
[GraphIndex] wait

This is useful for private evaluation/debugging.

What this proves

Safe claim:

Coeus has a process-lifetime graph index cache with generation-based invalidation and concurrent build coalescing.

Reviewer-facing wording:

“GraphIndexStore makes graph indexing usable at runtime by caching per-project graph indexes and discarding stale in-flight builds if the project surface changes.”

6. inventory_state_v1.* — deterministic bounded inventory text
Role

inventory_state_v1 builds a compact deterministic textual inventory block from a list of inventory items.

Input:

project id
root
InventoryItem[]
included_only
max_bytes
max_files_listed

Each InventoryItem has:

relpath
is_folder
included
embed status

Output:

InventoryStateV1Result
  text
  fingerprint
  truncated
  files_omitted
  file_count
  folder_count
  embed_ready
  embed_pending
  embed_failed
Rendered text shape

The output starts with:

INVENTORY_STATE v1
inventory_domain: included
included_only: true
project_id: ...
root: ...
counts:
  files: ...
  folders: ...
embedding_status_counts:
  ready: ...
  pending: ...
  failed: ...
files_sorted:
  - ...
folders_sorted:
  - ...
inventory_fp: 0x...
truncated: false

This is deterministic, sorted, bounded, and includes embedding status counts.

What this proves

Safe claim:

Coeus can render a deterministic included-file inventory surface for prompt/context use and diagnostics.

Important boundary:

This inventory is useful for routing and orientation, but it is not source proof. It lists tracked files and embedding status; it does not prove claims about file contents.

7. inventory_gist_builder.* — generated inventory artifacts and vector gists
Role

Context::Inventory::GistBuilder builds richer inventory artifacts from the project’s tracked file surface and summary sidecars.

It produces:

inventory markdown
inventory summary markdown
inventory JSON
per-folder inventory markdown
per-folder summary markdown
inventory meta
vector-ingested inventory gists

This is a major piece of evidence for the “project-aware” claim.

Build scheduling

The public API is async:

GistBuilder::BeginBuildIfStale(...)
GistBuilder::UpdateIfStale(...)
GistBuilder::BuildIfStale(...)
GistBuilder::Cancel(...)
GistBuilder::IsBuilding(...)

It uses the task engine:

lane: CPU
priority: Low
serialKey: INV_GIST:p<project>

It also deduplicates in-flight builds per project.

Surface stability check

Before building, it waits for file context to be stable:

tracked files counted
included files counted
pending embedding items counted

For included-only builds:

if tracked == 0 → stable
if includedOnly and included == 0 → stable
else stable only when pending == 0

It requires multiple stable passes and defers if the surface remains unstable.

This matters because inventory gists should represent a coherent file-context surface, not a half-scanned project.

File rows

The builder loads rows from:

proj->fileContextItems

Each row includes:

file
folder
included
pinned
bytes
lines
source_mtime_ns
gist
summary_inline
summary_ref
summary_origin

It calculates:

folder from file path
file size
line count
source mtime

and tries to attach summaries.

Summary loading

For each file row, it tries:

fresh semantic summary
fresh quick summary
cached preserved summary
Semantic summary path

Semantic summaries are found through:

PathUtils::SemanticSummaryArtifactPath(...)
PathUtils::SemanticSummaryStatePath(...)

Freshness uses Summary::MetaArtifact::IsStale(...).

The builder checks:

source stamp
summary stamp
state JSON
promptVersion
repo snapshot
source stamps
Phase-10 canonical summary schema

It requires the summary to satisfy a strict Phase-10 canonical shape:

## Responsibilities
- ...

## Key symbols
- ...

## Notes
- ...

It rejects summaries with code fences, wrong section order, non-bullet content, missing sections, or stale source state.

This is strong: inventory summaries are not blindly trusted.

Quick summary path

Quick summaries are expected at:

summaries/__quick__/<file>.quick.md

They must:

exist
be newer than source
pass Phase-10 canonical summary schema
Cached preserved summary fallback

If semantic/quick summaries are unavailable, the builder may preserve a previous inventory row’s summary if:

source_mtime_ns matches
bytes match when available
cached summary payload exists

This prevents summary timing lag from blanking inventory content unnecessarily.

That matches the earlier architecture theme:

summary producers may lag
inventory should not blank valid cached cells unnecessarily
fresh summaries improve routing but should not fabricate source proof
Artifacts written

The builder writes under an inventory summaries directory derived from SummaryKeys::kInventoryRoot.

Artifacts include:

all.md
all.summary.md
all.json
inventory_meta.json
folders/<folder>.md
folders/<folder>.summary.md

The JSON schema version is:

kInventorySchemaVersion = 4

The metadata stores:

fingerprint
structureFingerprint
includedOnly
perFolder
generatedBy = inventory_gist_builder

The builder distinguishes:

render fingerprint
  changes when rendered content/summaries/metrics change

structure fingerprint
  changes when file/folder/included/pinned structure changes

This enables rebuild decisions such as:

unchanged → skip artifact rebuild but re-ingest gists
render changed → rebuild content artifacts
structure changed → rebuild artifacts
missing expected artifacts → rebuild
stale builder artifacts → prune
Vector memory gist ingestion

After generating inventory artifacts, the builder ingests inventory gists into vector memory:

Context::VectorMemoryService::ingestGist(...)

It uses stable inventory keys from SummaryKeys, for example conceptually:

__inventory__/all.summary.md
__inventory__/all.json
__inventory__/folders/<folder>.summary.md

This is very important. It means the retrieval system can retrieve inventory-level context and project structure, not just raw source chunks.

Important boundary

Inventory gists should be treated as:

routing/orientation context

not:

source evidence for code claims

The code itself reinforces this idea in comments:

inventory is a derived surface artifact, not an authority over semantic summaries
8. What this proves about “project-aware” and “local source context”

This part of Block 7 supports several claims.

Proven: tracked local project file surface

The graph and inventory builders both consume:

Project.fileContextItems

with per-file metadata such as:

relPath
included
attachVerbatim / pinned
byteCount
status

This is evidence that the app has a local file-context surface, not just chat messages.

Proven: deterministic inventory artifacts

Inventory state and inventory gists produce deterministic sorted file/folder listings with fingerprints and bounded output.

Proven: vector-searchable project inventory

Inventory gists are ingested into VectorMemoryService, making project-level structure searchable.

Proven: graph-aware context layer

Graph index construction derives root/folder/file nodes and graph edges from semantic summary metadata.

Proven: local source roots matter

The inventory and graph systems preserve:

project folder
file context root folder
repo-relative paths
source mtimes
source sizes
summary paths

This is strong evidence for local-first project context.

9. Main data/control flow for this part
Inventory flow
Project.fileContextItems
  → wait for stable file-context surface
  → normalize relpaths
  → create FileRow[]
  → stamp source files
  → count bytes/lines
  → attach semantic/quick/preserved summaries
  → compute render + structure fingerprints
  → write inventory artifacts:
       all.md
       all.summary.md
       all.json
       inventory_meta.json
       folders/*.md
       folders/*.summary.md
  → prune stale inventory artifacts
  → ingest inventory gists into VectorMemoryService
Graph index flow
Project.fileContextItems
  → included files
  → derive folders
  → deterministic nodes:
       root
       folders sorted
       files sorted
  → parse semantic summaries:
       ## Key symbols
       ## Notes graph.* bullets
  → validate graph.node identity
  → build edges:
       includes
       contains
       folder deps
       nav
       entrypoints
  → optional source fallback if enabled
  → deterministic fingerprint
  → GraphIndexStore caches per project
Identity flow
retrieved item id + relPath
  → if high-bit gist id:
       stable gist key
  → else if stable-key-like:
       check whether real repo file exists
       stable key unless actual repo file
  → else:
       repo-relative source path
10. Strengths / distinctive architecture
Clear distinction between source path and derived gist identity

DocIdentity prevents inventory/summary gists from being accidentally treated as repo files.

Deterministic graph construction

Node ordering, edge sorting, dedupe, and fingerprinting make graph artifacts reviewable and reproducible.

Summary-native graph edges

Graph edges are primarily derived from semantic summary metadata, not brittle raw text scanning.

Bounded artifacts

Inventory text and JSON are capped/truncated where needed, preventing giant project inventories from exploding prompt/context surfaces.

Async inventory generation

Inventory gist building is scheduled through TaskEngine, coalesced per project, and cancellable.

Cache preservation against summary lag

The inventory builder can preserve previous summaries when source stamps match. This is practical and avoids degrading routing just because async summary generation lags.

Generation-based graph cache invalidation

GraphIndexStore handles concurrent builds and stale in-flight results professionally.

11. Risks / complexity notes
This is not the chunking layer yet

The actual “files become chunks/slices/searchable source evidence” path is not shown in this message. That should be covered when reviewing:

file_context/chunker.*
file_context/slice_meta.*
file_context/file_context_service.*
file_context/file_context_search.*
file_context_auto_ingest.*
Graph index is not a compiler index

The graph is lightweight and summary-derived. It should not be oversold as a full AST/symbol index. It supports navigation, dependency hints, entrypoints, and graph-aware retrieval/routing.

Inventory gists are not proof

Inventory and summaries help route and orient. They should not be treated as source evidence for exact code claims.

Some path normalization is duplicated

CanonRelRepoKey_, canonRepoRel, NormalizeRelpath, DocIdentity, PathUtils::NormalizeRelRepoPath, and FileCtxSvc::NormalizePath all participate in path normalization. This is workable but is a drift risk.

Summary freshness logic is complex

Inventory freshness depends on source stamps, semantic state JSON, prompt versions, summary schema, and cached fallback. This is powerful but high-complexity.

Inventory builder reads many artifacts

For large projects, inventory building touches source files, summaries, state files, quick summaries, and previous inventory JSON. It is async and bounded, but still a non-trivial background process.

12. Reviewer questions and suggested answers
Q: How does Coeus know what files are in a project?

A:
The project maintains a tracked file-context surface in Project.fileContextItems. Inventory and graph systems consume that surface to derive included files, folders, file metadata, inventory artifacts, and graph nodes.

Q: Is inventory just a directory listing?

A:
No. The inventory builder uses tracked file-context items, included/pinned flags, byte and line metadata, source mtimes, semantic or quick summaries, and emits markdown/json artifacts plus vector-ingested inventory gists.

Q: Are inventory summaries treated as source evidence?

A:
No. Inventory and summaries are derived context. They help routing and orientation. Exact source claims still need source evidence from retrieved file slices.

Q: What is the graph index?

A:
It is a deterministic lightweight graph over root, folders, and files. Edges can represent includes, containment, folder dependencies, entrypoints, and navigation hints. It is primarily built from semantic summary artifacts.

Q: Is this a full AST index?

A:
No. It is graph-aware context infrastructure, not a compiler-grade AST index. It is designed for project navigation, retrieval support, and architectural orientation.

Q: How does the graph cache avoid stale data?

A:
GraphIndexStore uses per-project generation counters. Invalidating a project bumps the generation and clears the cache. Any in-flight build that started under an older generation is discarded.

Q: How are gist identities distinguished from real source files?

A:
DocIdentity uses a high-bit gist ID convention and stable-key detection. It canonicalizes gist-like identities separately from repo-relative source paths.

13. Notes to carry into CODEBASE_OVERVIEW.md
The CLE context layer includes deterministic inventory and graph-index infrastructure over the project’s tracked file surface. `Project.fileContextItems` provides the file-context source surface; inventory builders derive bounded markdown/JSON inventory artifacts and vector-ingested inventory gists, while graph builders derive root/folder/file nodes and semantic edges from canonical summary artifacts. These systems help the assistant orient within a local project and retrieve project-structure context without treating derived summaries as source proof.
14. Notes to carry into ARCHITECTURE_NOTES.md
Block 7 Part 1: CLE inventory and graph infrastructure

Router headers:
  cle_file_context
  cle_inventory
  cle_index
  cle_memory
  cle_retrieval
  cle_summary
  cle_lattice
  cle_user_context

Document identity:
  DocIdentity
    high-bit gist convention
    repo path vs stable summary/gist key canonicalization
    prevents gist identities from being collapsed into source file paths

Inventory:
  inventory_state_v1
    deterministic included-file inventory text
    files/folders sorted
    embedding status counts
    bounded/truncated output
    fingerprint

  inventory_gist_builder
    async per-project builder
    waits for stable file-context surface
    derives FileRow[]
    loads semantic/quick/preserved summaries
    writes:
      all.md
      all.summary.md
      all.json
      inventory_meta.json
      folders/*.md
      folders/*.summary.md
    computes:
      render fingerprint
      structure fingerprint
    prunes stale builder-owned artifacts
    re-ingests inventory gists into VectorMemoryService

Graph:
  graph_index_builder
    consumes included file-context items
    creates deterministic nodes:
      root
      folders sorted
      files sorted
    parses semantic summaries:
      ## Key symbols
      ## Notes graph.* bullets
    edges:
      Include
      Mention
      Contains
      DependsOnFolder
      NavNext
      NavStartHere
      Entrypoint
    optional raw source fallback off by default
    deterministic fingerprint

  graph_index_store
    process-lifetime per-project cache
    generation-based invalidation
    coalesced concurrent builds
    stale in-flight build discard
15. Portfolio-facing summary
Block 7 Part 1 shows that Coeus AI has real project-context infrastructure beyond generic chat history. The CLE layer derives deterministic inventory artifacts from the tracked local file surface, ingests inventory gists into vector memory, canonicalizes repo paths separately from derived gist identities, and builds a lightweight graph index over project root/folder/file nodes using semantic summary metadata. This supports project-aware retrieval and codebase orientation while preserving the distinction between derived routing context and source evidence.

Stronger version:

The supplied Block 7 source demonstrates Coeus’s local project-awareness layer: tracked file-context items feed deterministic inventories, vector-searchable inventory gists, and a cached graph index over files and folders. The implementation includes path canonicalization, summary freshness checks, artifact fingerprints, stale-artifact pruning, async build scheduling, and generation-based graph cache invalidation. These are concrete source-level mechanisms behind the claim that Coeus works over a local project context rather than acting as a simple chatbot wrapper.








Block 7 Part 2 Summary — File Context Ingestion, Chunking, Lexical Search, Auto-Ingest, and Source Freshness

This part is the missing “how files become searchable context” layer.

Block 7 Part 1 showed inventory gists and graph/index scaffolding. This part shows the concrete pipeline:

local project folder
  → file enumeration
  → safe relpath normalization
  → binary/internal-artifact filtering
  → content loading
  → semantic/fallback chunking
  → vector-memory sync
  → lexical search index rebuild
  → semantic summary + inventory refresh
  → auto-ingest after file changes

The strongest architectural point:

Coeus does not merely attach files into a prompt. It maintains a local file-context surface, chunks source files into stable memory items, synchronizes the vector/BM25 surface from canonical file state, tracks pending/delta updates, and runs deterministic lexical scans over the current included surface.

1. Batch-level role

This block covers:

chunk_hasher.h
chunker.*
file_context_auto_ingest.*
file_context_search.*
file_context_service.*
file_context_support.*
lang_map.h
slice_meta.*

These files form the local source context ingestion and search substrate.

They answer:

How does a project folder become searchable?
How are files chunked?
How are changed files rescanned?
How are vector indexes kept aligned?
How does deterministic lexical search work?
How are inventory/summary/graph freshness surfaces refreshed?
2. High-level pipeline
Full scan path
FileContextService::ScanFolder(project, root, exts)
  → canonicalize root
  → enumerate eligible files
  → skip internal/runtime artifacts
  → apply extension filter / fallback to all text-like
  → read file bytes
  → skip binary / unsupported PDF
  → HTML → plain text extraction
  → LangFromExtension
  → Chunker::chunkSource
  → collect FileContextItem[]
  → collect MemoryItem slices
  → install fileContextRootFolder + fileContextItems
  → VectorMemoryService::FullSyncFromCanonicalSurface(project, allSlices)
  → SnapshotBuilder::Build(...)
  → PreloadGistsFromDiskFiltered(...)
  → mark fileContextItems Embedded
  → refresh:
       semantic summaries
       inventory gists
       lexical file search index
Changed-file path
NotifyFileChanged / NotifyFilesChanged / NotifyAbsoluteFileChanged
  → normalize relpath
  → mark dirty in AutoState
  → debounce/coalesce per project
  → RescanFile(file_i)
  → EmbedPending()
  → VectorMemoryService::ApplyFileDelta(...)
  → refresh:
       semantic summaries
       inventory gists
       lexical search index
Lexical scan path
QueryAndScan(project, query, identifiers, seedFiles)
  → derive/normalize search terms
  → suppress bad structure-planning fallback scans
  → EnsureIndex(project)
       → rebuild if surface drift detected
  → candidate files from seeds + inverted identifier index
  → read capped file text
  → literal line scan
  → bounded excerpts with line numbers
  → deterministic ranking
3. chunk_hasher.h — legacy/simple chunk IDs

chunk_hasher.h defines:

hashChunk(path, text)

It uses FNV-1a over:

path
null separator
text

and clears the top bit:

return h & 0x7FFF'FFFF'FFFF'FFFFULL;

That is important because CLE uses the high bit to distinguish gist IDs from normal source-slice IDs.

Note

The newer chunker.cpp does not rely directly on hashChunk(path, text) for semantic chunks. It defines a stronger internal function:

hashChunkStable_(rel, offset, text)

which includes offset as well as path/text. That avoids collisions when the same file contains repeated identical bodies.

Safe interpretation:

chunk_hasher.h is the simple older positive-ID helper.
chunker.cpp has the hardened current chunk identity logic.
4. chunker.* — semantic Tree-sitter chunking with fallback
Role

Chunker::chunkSource(...) converts file text into Context::MemoryItem slices.

For C++ it attempts semantic chunking using Tree-sitter. For plain text or unsupported parser paths, it falls back to fixed-size windows.

Supported language mapping comes from lang_map.h:

.h, .hpp, .c, .cpp, .cc → CPP
.py                    → PY
.js, .ts               → JS
other                  → Plain

But the actual semantic parser implemented here is only C++:

extern "C" const TSLanguage* tree_sitter_cpp();

For non-C++ language values, parser lookup returns null and the chunker falls back to fixed windows.

Semantic chunking

The Tree-sitter query captures:

function_definition
class_specifier
struct_specifier
enum_specifier
namespace_definition
template_declaration

Each captured node becomes a chunk.

The pipeline is:

parse source
  → query semantic nodes
  → collect byte spans
  → dedupe spans
  → sort spans by start/end
  → create MemoryItem for each span

Each memory item stores:

relPath
offset
text
id

The ID is computed from:

relpath + offset + text

and high bit is cleared.

This is an important correctness upgrade: repeated identical code blocks in the same file no longer collide because offset participates in the hash.

Parser concurrency safety

Tree-sitter parsers are mutable, so this code uses:

thread_local ParserPtr_ cpp;

That means each thread gets its own parser.

The query object is immutable after construction and guarded by a static mutex during one-time creation.

This is good concurrency hygiene.

Fallback windows

If parsing, query creation, cursor creation, or capture extraction fails, the chunker falls back to fixed windows:

fixedWindows(relPath, code, winBytes = 4096)

Fallback slices are deterministic:

offset 0, 4096, 8192, ...

Each fallback window also gets a stable high-bit-clear chunk ID.

What this proves

Safe claim:

Coeus chunks local source files into stable memory items, using Tree-sitter semantic spans for C++ where possible and deterministic fixed windows otherwise.

Boundary:

The semantic chunker shown here is C++-specific. Python/JS/TS are recognized by extension but currently fall back to plain fixed-window chunking unless parser support exists elsewhere.

5. file_context_support.* — path safety, scan filtering, gist preload, and metrics repair

This is the utility backbone for file-context correctness.

Path canonicalization

Important functions:

CanonRepoRel
CanonRepoRelStable
CanonFolderKey
JoinAbsSafe
IsWithin
CanonicalExistingOrLexical
EndsWithPathBoundarySuffix

The path rules reject or normalize:

absolute paths
root names
root directories
traversal segments
URL-ish strings
empty/invalid placeholders
backslashes
leading ./
leading slashes
duplicate slashes

CanonRepoRelStable is more forgiving: if given an invalid path, it can produce a stable placeholder:

__invalid_path__<hash>

This avoids silently accepting unsafe paths while still giving the system a deterministic placeholder for diagnostics.

Stable gist-key handling

Support code can convert summary/gist keys back into repo paths:

RepoRelFromMaybeFileKey
FolderRelFromMaybeFolderKey

This matters because file and folder summaries may use stable keys like:

__file__/...
__folder__/...
__inventory__/...

The support layer can distinguish:

file/folder surface gists
global derived gists
non-surface gists

Relevant helpers:

IsSurfaceDerivedGlobalGistKey
ShouldReattachNonSurfaceGistAfterExactSync
PreloadGistsFromDiskFiltered
PreloadAllGistsFromDisk

The exact-sync path can reattach only appropriate non-surface gists and avoid polluting the canonical file surface with derived file/folder gist rows.

This aligns with the larger doctrine:

canonical file surface ≠ derived summary/gist surface
Internal artifact exclusion

ShouldSkipInternalScanCandidate(...) excludes generated/runtime artifacts such as:

.cle_traces
.memory
conversations
traces
summaries
agent_config.md
context.md
context_meta.json
file_context.json
threads_meta.json
generated vector artifacts

This is critical. It prevents the system from accidentally ingesting its own runtime state as project source evidence.

Text filtering

Support utilities include:

IsLikelyBinary
ExtractTextFromHtml
IsPdfBytes
ReadDiskFileCapped
FileExistsRegular

HTML is converted into plain text by removing script/style blocks, stripping tags, decoding basic entities, and collapsing whitespace.

PDFs are detected but not extracted:

PDF bytes → skip PDF (no extractor)

So safe portfolio wording should not claim PDF text ingestion unless another extractor exists elsewhere.

Metrics rehydration

RehydrateMetricsIfMissing(Project*) repairs missing sliceCount, estTokens, and byteCount using:

VectorMemoryService manifest
disk file size

This helps older projects or partially loaded file-context metadata recover enough metrics for UI/inventory without a full rescan.

6. file_context_service.* — main source ingestion and file-context authority

This is the central file for file-context state.

It owns the main operations:

ScanFolder
RescanFile
EmbedPending
ToggleFile
ToggleFolder
ToggleAttach / PinFile
SetFilesIncluded
SetFilesAttached
ReadFileRaw
GetAbsolutePath
GetRelPathForAbsolutePath
EnumerateFileSlices
StartScanFolderAsync
StartEmbedPendingAsync
StartRescanFileAsync
StartRescanThenEmbedAsync
ScanFolder — full root ingest

This is the full rebuild path.

Key contract log:

scan contract=full_root_ingest
Root setup

It canonicalizes:

rootPath
rootCanon
absRoot

Then clears stale structural refresh state and cancels in-flight inventory builds.

Extension filtering

It normalizes extensions.

If a requested extension filter yields no candidates, it falls back to all text-like files:

filter produced 0 candidates; falling back to ALL text-like

This is practical for user mistakes.

Enumeration

It recursively walks the root folder and applies:

regular file only
size <= 5 MiB
extension filter
canonical relpath
skip internal/runtime artifacts
File reading and preprocessing

For each file:

read raw bytes
skip empty
skip likely binary
HTML → extracted text
PDF → skipped if no extractor

Then:

LangFromExtension
Chunker::chunkSource

The result is:

FileContextItem
MemoryItem slices

The FileContextItem stores:

relPath
byteCount
included
attachVerbatim
contents if included/attached
status = Pending initially
sliceCount
estTokens
Installation

After processing, it replaces:

proj->fileContextRootFolder = rootCanon/absRoot
proj->fileContextItems = newItems

Then it calls:

VectorMemoryService::FullSyncFromCanonicalSurface(proj, allSlices)

This is a critical source-truth seam.

Full scan does not merely append slices. It performs an exact full-surface sync from the canonical file-context surface.

Then:

SnapshotBuilder::Build(...)
PreloadGistsFromDiskFiltered(...)
mark FileContextItems Embedded
RunFreshnessRefresh

Freshness refresh includes:

inventory
semantic summaries
file search index
RescanFile — mark changed file pending

RescanFile handles a single file change.

It resolves the file to an existing canonical relpath when possible. If the file does not already exist in the context, it can accept a stable safe path and add a new item.

Deleted/missing file behavior

If the file existed but no longer exists, it does not immediately erase everything silently. It marks the item as a pending delete:

status = Pending
included = false
attachVerbatim = false
contents.clear()
sliceCount = 0
estTokens = 0
byteCount = 0

Then it marks structural inventory refresh pending and defers freshness until EmbedPending.

This is sensible because vector deletion should be committed through the same delta-completion path.

Existing/changed file behavior

If the file exists:

read bytes
skip >5MiB
skip binary/PDF
HTML extract if needed
chunk source
update/add FileContextItem as Pending
preserve include/attach flags if existing

It logs:

status=Pending
structural_inventory_dirty=...

Important: RescanFile prepares the new file state but does not complete vector sync. Completion happens in EmbedPending.

EmbedPending — canonical file delta completion

EmbedPending is the delta sync path.

It gathers pending items, reads cached or disk contents, chunks them, and calls:

VectorMemoryService::ApplyFileDelta(proj, touchedSlices, touchedRelPaths)

This is the counterpart to full sync.

The two explicit vector-memory synchronization paths are therefore:

FullSyncFromCanonicalSurface(...)
  used after full root scan

ApplyFileDelta(...)
  used after pending per-file changes

That is one of the most important architecture findings in this block.

Deleted files

For pending files that are missing on disk, it includes the relpath in touchedRelPaths but contributes no slices. That lets ApplyFileDelta remove stale slices for the file.

After successful delta application, it:

updates slice counts/tokens/bytes
marks pending files Embedded
removes pending-delete entries
marks project dirty

Then it refreshes:

semantic summaries
inventory
file search

The comment explicitly says content-only changes can affect semantic summaries, so inventory must refresh after semantic summaries, not only after structural changes.

This is strong evidence that the system is designed around freshness propagation.

Include/attach toggles

Operations like:

ToggleFile
ToggleFolder
ToggleFolderImmediate
ToggleAttach / PinFile
SetFilesIncluded
SetFilesAttached

modify inclusion/pinning state and trigger freshness refreshes.

Important semantics:

included = participates in searchable/current file-context surface
attachVerbatim = pinned/verbatim-ish file, also forces included

When inclusion changes, the system refreshes:

inventory
semantic summaries
file search

This means search and inventory are supposed to track the current included surface, not every file ever scanned.

Read helpers

ReadFileRaw first tries cached contents, but detects partial cache and falls back to disk:

if cached.size < byteCount and looks partial → disk read

All reads are capped.

GetAbsolutePath and GetRelPathForAbsolutePath enforce root-relative safety.

EnumerateFileSlices produces fixed overlapping slices for a specific file:

sliceBytes default provided by caller
sliceOverlap default provided by caller
sorted by offset

This is separate from semantic Chunker::chunkSource and useful for direct source evidence packing.

Async wrappers

The async APIs schedule through TaskEngine:

StartScanFolderAsync      lane IO, priority High
StartEmbedPendingAsync    lane CPU, priority Normal
StartRescanFileAsync      lane IO, priority High
StartRescanThenEmbedAsync lane CPU, priority High

They maintain IngestJob_ with:

filesDone
filesTotal
TaskHandle

and support:

QueryIngestProgress
CancelIngest
7. file_context_auto_ingest.* — debounced freshness pipeline after file changes

This is the system-owned auto-dirty layer.

It is explicitly not an LLM tool.

Purpose:

When files change via repo tools, workspace tools, editor saves, watcher events, etc:
  coalesce paths per project
  debounce bursts
  run RescanFile(file_i)
  run EmbedPending() once
  emit ActivityFeed begin/end lines
Notification API

Relative path notifications:

NotifyFileChanged
NotifyFilesChanged

Absolute path notifications:

NotifyAbsoluteFileChanged
NotifyAbsoluteFilesChanged

Immediate-ish flush:

Flush

Options include:

origin
thread_id
epoch
debounce_ms

This allows auto-ingest telemetry to correlate with the thread and turn epoch that caused the change.

Debounce and origin priority

Origins are prioritized:

plan.apply_v1                 priority 50
repo.apply_patch_v1/ws patch  priority 40
write tools                   priority 30
fc.add_files_v1               priority 20
filectx.service               priority 10
other                         priority 5
empty                         priority 0

Freshness-critical origins get shorter debounce:

apply-like    ≤ 75ms
write-like    ≤ 150ms
add-files     ≤ 200ms
default       350ms

This is a good design: file changes caused by deterministic tool mutations should update the retrieval surface quickly.

Batch behavior

The auto-ingest state stores:

dirty files
origin
thread_id
epoch
debounce_ms
generation
batch timing metadata
notify count

When the debounce window stabilizes, it takes a sorted unique batch.

The batch records:

files
origin
displayOrigin
threadId
epoch
coalescedBatch
highRiskFreshness
batch timing

Then:

cancel inventory build
RescanFile for each file
EmbedPending once
write details JSON
append ActivityFeed lines

Details are written under trace run artifacts:

runs/system/<runId>.json

with schema:

system_filectx_auto_ingest_v1

This is solid evidence for professional runtime observability.

Cancellation behavior

Cancellation requeues dirty files instead of losing them.

There are explicit handlers for cancellation during:

rescan
post_rescan
embed

Cancelled batches are requeued with a short debounce.

This matters because file freshness should not be dropped just because a task was cancelled or superseded.

8. file_context_search.* — deterministic lexical/global scan

This is independent from vector search.

It provides deterministic lexical scanning over the current included file-context surface.

The comments are explicit:

The indexed lexical surface is the CURRENT INCLUDED SURFACE
not all tracked files.

That is a key source-of-truth claim.

Search index surface

buildCurrentIndexableSurface_ snapshots:

canonical source root
tracked files
included files
indexable included files after filters
pending count
file relpaths
file sizes
file mtimes
pending status
surface fingerprint

The fingerprint includes:

rootKey
canonical relpath
size
mtime
pending flag

This is important: equal-size edits are detected because mtime is included. Root changes are detected because rootKey is included.

Lazy rebuild and drift detection

EnsureIndex rebuilds if:

no cached index
options changed
surface root changed
surface fingerprint changed
surface count changed
pending count changed
cached index empty over non-empty surface

Mismatch logs include:

cached_root/current_root
cached_surface/current_surface
cached_pending/current_pending
cached_fp/current_fp

This is exactly the kind of diagnostic behavior needed for eval/debugging.

Index contents

The runtime index stores:

projectId
options
includedOnlySurface = true
surfaceRootKey
tracked/included/surface/pending counts
surfaceFingerprint
files[]
inverted token index
canonToStored

Each FileInfo stores:

canonRel
storedRel
absPath
mtimeTicks
sizeBytes

The inverted index is:

identifier token → sorted list of canonical relpaths

Tokens are identifier-like:

starts with alpha or _
continues with alnum, _, or ::

It also emits variants for A::B, such as:

A::B
A
B

There is a cap:

maxTokensPerFile = 4096 by default
Query term selection

QueryAndScan accepts:

query text
identifiers
seed files
SearchOptions

If caller-provided identifiers are empty and allowed, it can derive fallback identifier terms from the query.

Terms are scored to avoid scanning on glue words. Stopwords include common words like:

the, and, file, files, function, class, architecture, structure, recommended, etc.

This avoids huge scans from vague queries.

Structure-planning suppression

There is a specific guard for “structure planning” questions:

How should I structure this?
Which files should I create?
Best way to split this across files?

If the query looks like multi-file planning and the caller provided no explicit identifiers or seeds, lexical scan is suppressed instead of hijacking the answer with inventory/global scan.

That is important for the user’s earlier eval issue family:

structure-planning questions do not get hijacked by inventory/global-scan

This module makes that explicit.

Candidate selection

Candidates are deterministic:

resolved target seed, if any
seed files
inverted-index hits for terms
clamp maxCandidates

It supports exact target resolution and unique suffix resolution.

If a target cannot be resolved in the current search surface, it logs a warning.

Scan behavior

For each candidate:

ReadFileRaw capped
split lines
count literal case-sensitive occurrences of each term
collect hit lines
build excerpt windows ± context lines
cap hits per file
cap excerpts per file
cap line chars
cap excerpt bytes

Results are sorted deterministically:

hitCount descending
relPath ascending

The output has:

files
topFiles
candidateCount
scannedCount
hitFileCount
termCount
termsUsed
termsDerivedFromQuery
status
skip_reason
queryLooksSerialized

Statuses:

Skipped
NoHits
Hit
Error

Skip reasons:

NoTerms
IndexEmpty
NoCandidates
Error

This gives downstream retrieval/controller code enough detail to know why a scan did or did not help.

What this proves

Safe claim:

Coeus includes a deterministic lexical search layer over the current included file-context surface, with surface-drift detection, stable candidate selection, bounded excerpt generation, and diagnostics for no-term/no-candidate/no-hit cases.

Boundary:

This lexical scan is not vector retrieval. It is a deterministic supplement used for references, usages, exact identifiers, and source evidence excerpts.

9. slice_meta.* — older compact slice metadata

SliceMeta defines a compact 32-byte metadata record:

sliceID
contentHash
tokenCount
fileModTime
pinned
reserved bytes

It has:

MakeSliceID(pathHash32, chunkOrdinal)

and binary read/write helpers.

It also wraps XXH64.

This appears to be an older or lower-level metadata structure, useful for compact slice tracking and serialization. It is not the main chunking path in the code shown, where current MemoryItem IDs are created through the chunker and vector-memory service.

Safe wording:

slice_meta provides a compact slice metadata format, but the active ingestion path shown here primarily emits Context::MemoryItem slices from Chunker::chunkSource.

10. Concrete “project-aware / local source context” evidence

This part gives strong evidence for all of these claims:

Local root awareness

The system stores and uses:

Project.fileContextRootFolder
Project.folderPath
canonical rootKey
absolute source root
repo-relative relpaths
Local source scanning

ScanFolder recursively reads local disk files under the selected root, with extension filtering and size/binary/internal-artifact guards.

Source chunking

Files become MemoryItem chunks with:

relPath
offset
text
stable ID

C++ files can be chunked by semantic Tree-sitter spans.

Canonical vector synchronization

There are two clear vector sync modes:

FullSyncFromCanonicalSurface
ApplyFileDelta

This is very strong evidence of retrieval-surface coherence.

Current included surface semantics

Both inventory and lexical search operate over included files, not arbitrary stale files.

Freshness propagation

After scan/embed/toggle/write-like changes, the system refreshes:

semantic summaries
inventory gists
lexical search index
Host-owned auto-ingest

File changes from tools or editor/watcher paths can be coalesced and embedded without involving the model.

Diagnostics and auditability

Logs and trace artifacts expose:

surface fingerprints
root identity
pending counts
candidate counts
scan status
skip reasons
auto-ingest run details
activity feed lines
11. Main strengths
1. Strong file-surface authority

The code repeatedly reinforces that the indexed surface is the current included file-context surface.

This reduces split-brain between:

tracked files
included files
vector memory
lexical scan
inventory
2. Proper full sync vs delta sync separation

Full scan uses:

FullSyncFromCanonicalSurface

Changed files use:

ApplyFileDelta

That is exactly the kind of seam needed to keep FAISS/BM25/manifest state coherent.

3. Good safety boundaries

The system skips:

runtime artifacts
project metadata
vector artifacts
binary files
oversized files
unsupported PDFs
unsafe paths
traversal paths
URLs as file paths
4. Good freshness ordering

The code explicitly handles the problem that inventory summaries depend on semantic summaries. It schedules semantic refresh before inventory refresh when both are needed.

5. Deterministic lexical scan

The lexical scan is bounded, deterministic, and richly diagnosed.

6. Good auto-ingest design

Debounce, coalescing, origin priority, cancellation requeue, and activity-feed logging make the file-change pipeline robust.

7. Parser concurrency hardening

Thread-local Tree-sitter parsers avoid unsafe shared mutable parser use.

12. Risks / issues to watch
1. chunk_hasher.h vs chunker.cpp identity drift

chunk_hasher.h hashes only path + text. chunker.cpp hashes path + offset + text.

The current chunker uses the safer offset-aware hash. But if hashChunk(path,text) is still used elsewhere, repeated identical chunks in the same file could collide.

Suggested review question:

Search for hashChunk(...) usages.
If any active ingestion path still uses it, consider replacing with offset-aware IDs.
2. Python/JS/TS are extension-recognized but not semantically parsed here

lang_map.h recognizes .py, .js, .ts, but chunker.cpp only supports Tree-sitter C++.

That is fine if intended, but portfolio wording should avoid claiming semantic chunking across all languages.

Safe claim:

semantic Tree-sitter chunking for C++ with fixed-window fallback for other/plain text inputs
3. Path normalization has many layers

Normalization occurs in:

FileCtxSvc::NormalizePath
FileContextSupport::CanonRepoRel
CanonRepoRelStable
CanonFolderKey
file_context_search normRel
DocIdentity
PathUtils

This is necessary but drift-prone. The good news is these layers are mostly converging on canonical relpaths and suffix-safe resolution.

Suggested review question:

Are all downstream retrieval and validator paths using the same canonical relpath dialect?
4. RescanFile creates Pending state before vector update

This is intentional, but downstream layers must respect it.

A file marked Pending has not necessarily been applied to vector memory yet. The actual retrieval surface is updated after EmbedPending.

Suggested invariant:

Pending file context state must not be treated as embedded retrieval truth until ApplyFileDelta succeeds.
5. Lexical scan is case-sensitive

Line hit counting uses literal countOccurrences(line, term). This is good for code identifiers, but may miss case-insensitive natural-language terms.

Likely fine for code search, but worth noting.

6. Inventory refresh defers while pending

RefreshInventory defers if there are pending items. This is correct, but if pending state gets stuck, inventory can lag.

The auto-ingest and embed-pending path should be checked in eval logs for stuck Pending.

7. Unsupported PDF behavior

PDFs are detected and skipped. Do not claim PDF ingestion unless a separate extractor exists.

13. Reviewer questions and suggested answers
Q: How do files become searchable?

A:
A selected root folder is scanned by FileContextService::ScanFolder. Eligible files are read, normalized to repo-relative paths, filtered for size/binary/internal artifacts, chunked into Context::MemoryItem slices, and synchronized into vector memory through FullSyncFromCanonicalSurface. Later file changes go through RescanFile and EmbedPending, which update vector memory through ApplyFileDelta.

Q: Does the system keep stale files in the index?

A:
The design tries not to. Full scans use exact full-surface sync. Per-file updates use explicit delta sync. Lexical search rebuilds when the current included surface fingerprint differs from the cached surface, including root, size, mtime, and pending state.

Q: Are all scanned files used?

A:
No. The system filters out large files, binary-like files, unsupported PDFs, runtime/internal project artifacts, generated vector artifacts, and paths outside the safe root. It also distinguishes tracked files from included files.

Q: What does “included surface” mean?

A:
It is the subset of tracked file-context items currently marked included. Inventory and lexical search are built over this current included surface, not every file ever scanned.

Q: What happens after a repo tool writes a file?

A:
The auto-ingest layer can be notified with an origin such as repo.write_v1 or repo.apply_patch_v1. It coalesces dirty files, debounces briefly, rescans each file, embeds pending changes once, writes system run details, and appends activity-feed lines.

Q: Does the model perform file ingestion?

A:
No. File ingestion is host infrastructure. The model may request or trigger tools, but the actual scan/rescan/embed path is deterministic C++ host code.

Q: Is the file search vector-only?

A:
No. There is a deterministic lexical scan layer in file_context_search.*, independent from vector retrieval. It builds a lightweight inverted identifier index over the included surface and returns bounded line-numbered excerpts.

14. Notes to carry into CODEBASE_OVERVIEW.md
The File Context subsystem is the bridge between local source files and CLE retrieval. `FileContextService::ScanFolder` performs a full root ingest: it safely enumerates local files, skips internal/runtime artifacts, chunks eligible source files, installs canonical `Project.fileContextItems`, and synchronizes vector memory with `FullSyncFromCanonicalSurface`. Incremental changes go through `RescanFile` and `EmbedPending`, which apply file deltas through `ApplyFileDelta`. A deterministic lexical search index over the current included surface provides exact identifier/reference scanning with bounded excerpts and surface-drift detection.
15. Notes to carry into ARCHITECTURE_NOTES.md
Block 7 Part 2: File Context / Local Source Context

Chunking:
  chunker.cpp
    C++ Tree-sitter semantic chunks:
      function_definition
      class_specifier
      struct_specifier
      enum_specifier
      namespace_definition
      template_declaration
    thread-local parser
    shared immutable query
    span dedupe + stable ordering
    fallback fixed windows
    chunk ID = hash(rel + offset + text), high bit clear

  lang_map.h
    C++ extensions mapped to CPP
    py/js/ts recognized but fallback in current chunker implementation
    all else Plain

File Context Service:
  ScanFolder:
    full_root_ingest
    recursive root scan
    ext filter with fallback
    skip >5MiB
    skip binary
    skip unsupported PDF
    HTML text extraction
    skip internal/runtime artifacts
    install fileContextRootFolder + fileContextItems
    FullSyncFromCanonicalSurface
    SnapshotBuilder::Build
    PreloadGistsFromDiskFiltered
    mark Embedded
    refresh semantic summaries, inventory, lexical search

  RescanFile:
    resolve canonical relpath
    changed/missing file -> status Pending
    deleted file -> pending-delete with included=false
    changed file -> chunks + metrics stored pending
    no vector update until EmbedPending

  EmbedPending:
    reads pending files
    chunks pending files
    ApplyFileDelta(proj, touchedSlices, touchedRelPaths)
    removes pending deletes
    marks Embedded
    refresh semantic summaries, inventory, lexical search

Auto-ingest:
  NotifyFileChanged / NotifyFilesChanged / absolute variants
  per-project dirty set
  origin priority
  debounce based on origin
  freshness-critical for writes/apply/add-files
  RescanFile batch + one EmbedPending
  ActivityFeed lines
  runs/system/<runId>.json details
  cancellation requeues batch

Lexical search:
  index over CURRENT INCLUDED SURFACE
  surface fingerprint includes:
    rootKey
    canonical relpaths
    size
    mtime
    pending flag
  rebuild on:
    missing index
    options changed
    root/surface/fingerprint mismatch
    empty cache over non-empty surface
  inverted identifier index
  QueryAndScan:
    seeds + identifiers
    fallback query term extraction
    structure-planning suppression
    deterministic candidates
    literal line scan
    bounded excerpts
    status/skip_reason diagnostics

Support:
  CanonRepoRel / CanonRepoRelStable / CanonFolderKey
  safe root-relative joins
  skip internal artifacts
  gist preload filtering
  metrics rehydration from vector manifest
16. Portfolio-facing summary
The File Context subsystem demonstrates that Coeus AI works over a real local source surface. It can scan a project root, canonicalize file paths, filter unsafe/internal artifacts, chunk source files into stable memory items, synchronize vector memory from the canonical file surface, and maintain deterministic lexical search over the current included files. It also includes an auto-ingest pipeline that reacts to file changes from tools or editor integrations by rescanning and embedding changed files, while keeping inventory, semantic summaries, and lexical search fresh.

Stronger version:

Block 7 Part 2 shows the concrete local-source ingestion engine behind Coeus AI. Full project scans use `FullSyncFromCanonicalSurface` to align vector memory with the canonical file context, while changed files use `ApplyFileDelta` after a pending rescan/embed cycle. C++ files are chunked with Tree-sitter semantic spans where possible, with deterministic fixed-window fallback. A separate lexical search index tracks the current included surface using root/file/mtime fingerprints, producing bounded line-numbered excerpts for exact identifier/reference queries. This is a real project-aware retrieval substrate rather than simple file attachment.
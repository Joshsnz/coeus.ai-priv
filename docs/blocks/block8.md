Block 8 Part 1 Summary — Lattice Runtime, Vector Memory, Secondary Memory, Snapshots, and Semantic-Primer Boundary

This chunk covers the packing and memory substrate more than the retrieval policy/controller itself.

It shows:

retrieval result chunks
  → classified as SourceEvidence or SemanticPrimer
  → admitted/rejected by canonical TURN_STATE policy
  → packed into lattice planes
  → emitted as Agent::Request system context
  → traced with prompt markers

file/source memory
  → VectorMemoryService
  → FAISS slices + BM25 rows + manifest
  → exact full sync / exact file delta
  → gist ingestion

secondary/session memory
  → MemoryStore SQLite + FAISS
  → MemoryGraph
  → conversation/user/fact memory

semantic memory seed
  → summary sidecars only
  → explicitly SemanticPrimer/support-only

The biggest architectural point:

This block makes the distinction between source evidence, semantic primers/summaries, and continuity/session memory very explicit. Source evidence is visible file content from retrieval slices. Semantic summaries/gists orient the model but are support-only unless explicitly marked otherwise. Conversation tail/recent mutation/plan/apply planes are continuity, not proof.

1. What this block contains

Included here:

core/cle/lattice/lattice.*
core/cle/lattice/lattice_runtime.*
core/cle/memory/embed_cache.*
core/cle/memory/memory_graph.*
core/cle/memory/memory_seed_builder.*
core/cle/memory/memory_snapshot.h
core/cle/memory/memory_store.*
core/cle/memory/snapshot_builder.*
core/cle/memory/vector_memory_service.*

Not yet included in this part:

core/cle/retrieval/retrieval_controller.*
core/cle/retrieval/hybrid_search.*
core/cle/retrieval/bm25_db.*
core/cle/summary/meta_summary.*
core/cle/summary/summary_service.*
core/cle/summary/gist_service.*

So for now, we can analyze how retrieval chunks are stored, looked up, classified, admitted, and packed, but the full retrieval-routing policy and hybrid scoring details still need the next files.

2. lattice.* — block container, packer, and marker computation
Core role

Context::Lattice::Builder is a bounded prompt-context packer.

It accepts Block objects:

struct Block {
    BlockKind kind;
    std::string key;
    std::string title;
    std::string text;
    int priority;
    std::vector<std::string> marker_hints;
};

Block kinds:

UserContext
Inventory
ProjectMeta
FolderMeta
FileMeta
Retrieval
SemanticPrimer
ToolObservation
Notes

The packer:

sanitizes block key/title/text
cleans marker hints
dedupes by kind+key
sorts by priority desc
then kind rank
then key/title
adds blocks until token budget/max block cap is hit
computes final prompt markers from actually included blocks

Important: markers are computed from packed/included blocks, not merely from candidate blocks. That means the trace can distinguish:

retrieval candidate existed
retrieval candidate admitted
retrieval block actually packed

That is very important for eval diagnosis.

Block ordering

Priority dominates. If equal priority, the rank is:

UserContext
Inventory
ProjectMeta
FolderMeta
FileMeta
Retrieval
SemanticPrimer
ToolObservation
Notes

However, in runtime, retrieval source evidence is usually assigned high priority 94, semantic primer 89, turn state 98, plan/current structured continuity 95, etc. So priority usually overrides kind order.

Pack markers

ComputePackMarkersFromIncluded(...) computes many truth markers:

plane.conv_tail
plane.turn_state
plane.inventory
plane.inventory_list
plane.inventory_digest
plane.inventory_gist
plane.retrieval
plane.retrieval_meta
plane.semantic_primer
plane.session_memory
plane.tool_observations
plane.summaries
plane.recent_mutation
plane.plan
plane.apply
plane.decision

It also computes byte counts:

packed.turn_state_bytes
packed.conv_tail_bytes
packed.user_ctx_bytes
packed.conv_summary_bytes
packed.inventory_bytes
packed.summaries_bytes
packed.retrieval_bytes
packed.primer_bytes
packed.tool_observation_bytes
packed.recent_mutation_bytes
packed.plan_bytes
packed.apply_bytes
packed.decision_bytes

And retrieval counts:

packed.retrieval_chunks
packed.retrieval_slice_chunks
packed.retrieval_gist_chunks
packed.retrieval_files
evidence.relpaths
evidence.relpaths_count

This is exactly the kind of trace surface needed to debug “did the model actually receive source evidence?”

3. lattice_runtime.* — canonical TURN_STATE consumption and plane assembly

This is the real runtime seam.

The header states the ownership split clearly:

lattice.cpp:
  block container / ordering / pack facade / marker computation

lattice_runtime.cpp:
  canonical TURN_STATE consumption
  authority admission
  plane assembly
  request + trace construction
  runtime logging

This is aligned with the CLE doctrine: runtime should consume canonical policy truth, not reclassify the turn.

Main entrypoint
Runtime::BuildRequest(...)

calls BuildInternal_, which assembles all planes into an Agent::Request.

The big inputs are:

Project*
userInput
RetrievalResult rr
plan text
includeMemory
extraSystemBlocks
UnifiedPromptOptions opts
outTraceJson
TURN_STATE plane

The first and highest-priority plane is:

plane/turn_state

Priority:

98

It uses:

opts.turnStateText

or falls back to a synthetic minimal TURN_STATE v3.

The proper path is obviously opts.turnStateText; the fallback is marked:

turn_state.synthetic_fallback=1
turn_state.transport_usable=0/1
turn_state.parse_source=fallback|opts

This is important: if the canonical resolved state is missing, the lattice can still function, but trace markers reveal the degraded path.

Authority admission policy

The runtime parses TurnResolved::TurnStateTransport.

It builds:

AuthorityAdmissionPolicy_

with gates:

allowInventoryFacts
allowSourceEvidence
allowSemanticPrimer
allowSessionMemory
allowToolObservations
allowRecentMutation

and support-only flags:

semanticPrimerSupportOnly
sessionMemorySupportOnly
toolObservationsSupportOnly
recentMutationSupportOnly

The final gates are derived from:

canonical turn-state transport
opts fallback
tier
missing repo target condition

Important hard gates:

Tier 0

If tier == 0:

inventory facts disabled
source evidence disabled
semantic primer disabled
tool observations disabled
recent mutation disabled
session memory allowed only as support/background
Missing repo target

If the transport says the turn is a missing repo target:

inventory facts disabled
source evidence disabled
semantic primer disabled
session memory disabled
tool observations disabled
recent mutation disabled

That is a strong anti-fabrication rule.

4. Lattice planes assembled by runtime
4.1 plane/turn_state

Always added.

Carries canonical policy and resolved fields into markers, including:

answer.kind
answer.effective_output_mode
answer.requires_source_evidence
answer.requires_diff_plane
answer.canonical_target_file
resolved.primary_target_file
resolved.current_file_target
resolved.anchor_source
resolved.diff_expected
resolved.recent_mutation_answerable

These are computed in ComputePackMarkersFromIncluded from the parsed transport.

4.2 Tool observations

There is a placeholder function:

AddToolObservationBlocks_

But it currently logs:

observation_plane=conservative_noop

and does not add blocks.

So the architecture has a tool-observation plane seam, but this code does not yet pack actual tool observations.

Safe wording:

Tool observation admission and markers exist, but the current implementation shown here is a conservative no-op for adding actual tool observation blocks.

4.3 Conversation tail / session memory

Conversation tail is added through:

AddConvTailPlaneBlocks_

It reads proj->threads[threadID].tail, groups recent messages into user/assistant pairs, and emits:

plane/conv_tail|pair|r000000
plane/conv_tail|pair|r000001
...
plane/conv_tail|summary

Markers:

evidence.class=session_memory
continuity.kind=conv_tail_pair
continuity.kind=conv_tail_summary

Conv-tail budget is dynamic.

It increases when the turn is:

referential
resolved anchor present
recent update question
diff expected

This is good because “what changed?”, “that file”, “previous turn” style questions need more continuity.

Important boundary:

Conversation tail is session memory / continuity. It is not source proof.

4.4 Extra keyed blocks

opts.extraKeyedBlocks can inject structured planes:

semantic primer
recent mutation
plan
apply
decision
notes
project/folder/file meta

Admission depends on authority policy.

Recent mutation blocks:

plane/recent_mutation...

are only admitted if:

allowRecentMutation == true

Plan/apply/decision are treated as structured continuity and generally require session-memory allowance.

Semantic primer blocks require semantic-primer allowance.

Markers added for structured continuity:

evidence.class=structured_continuity
continuity.kind=plan|apply|decision

Markers added for recent mutation:

evidence.class=recent_mutation
continuity.kind=recent_mutation
support_only.recent_mutation=1 when configured

This reinforces the boundary:

recent mutation plane = continuity / mutation metadata
source evidence plane = visible file content
4.5 User context and conversation summary

If includeMemory && allowSessionMemory:

USER_CTX
CONV_SUMMARY

may be packed.

Markers:

evidence.class=session_memory
support_only.session_memory=1 if configured

Again, not source evidence.

4.6 Inventory planes

Three forms:

plane/inventory|summary
plane/inventory|list
plane/inventory|gist

These require:

allowInventoryFacts == true

Inventory digest is simple facts:

files_count
pinned_count
first_files

Inventory list is up to 256 included files, with pinned labels.

Inventory gist currently has a notable issue:

static std::string LookupGistText_(Project*, const std::string&)
{
    return {};
}

So lattice runtime always falls back to a generated inventory gist:

files_count
pinned_count
top_folders
top_extensions
sample_files

This is okay as a fallback, but it means the runtime currently does not load the actual inventory gist text from vector/gist artifacts in this file.

Suggested review note:

LookupGistText_ is a stub. Inventory gist packing currently uses fallback text, not persisted inventory gist content.
4.7 Retrieval result chunks

This is the key source-evidence seam.

The runtime iterates:

rr.chunks

Each chunk has an EvidenceClass:

SourceEvidence
SemanticPrimer
Source evidence chunks

Admitted only if:

allowSourceEvidence == true

Then they become BlockKind::Retrieval.

Key:

plane/retrieval/slice|r000000|<rel>|<offset>|<id>

Text format:

SOURCE EVIDENCE — CONTENT VISIBLE
File: <rel>
Offset: <offset>
-----
<actual source text>

Markers:

evidence.class=source_evidence
evidence.visibility=content_visible
source_evidence.content_visible=1
source_evidence.file=<rel>

This is the strongest proof boundary in the code.

Safe claim:

Source evidence means actual visible content from local source slices was packed into the prompt.

Semantic primer chunks

Admitted only if:

allowSemanticPrimer == true

Then they become BlockKind::SemanticPrimer.

Key:

plane/semantic_primer|r000000|<rel>|<offset>|<id>

Markers:

evidence.class=semantic_primer
support_only.semantic_primer=1 when configured

Safe claim:

Semantic primers orient or summarize, but are not source proof unless some future path explicitly classifies them differently.

5. Lattice trace and diagnostics

The runtime emits detailed trace events:

Lattice.ConvTailPolicy
Lattice.ConvTailBuild
Lattice.KeyedBlockAdmission
Lattice.RetrievalAdmission
Lattice.PackedPlanesDetail
Lattice.PackWarning
Lattice.Pack
Lattice.RecentMutation
Lattice.Continuity

It also serializes a TraceV1::Trace and attaches/persists it when tracing is enabled.

Important markers include:

lattice.allow_source_evidence
lattice.allow_semantic_primer
lattice.retrieval_input_source_candidates
lattice.retrieval_input_source_admitted
lattice.retrieval_input_source_rejected_policy
lattice.retrieval_input_semantic_candidates
lattice.retrieval_input_semantic_admitted
lattice.packed_retrieval_block_count
lattice.source_evidence_content_visible
lattice.source_evidence_content_visible_files

This allows post-run evaluation to answer:

Did retrieval find source candidates?
Were they rejected by policy?
Were they admitted?
Were they packed?
Which files were visible?

That is exactly what is needed for the eval harness.

6. vector_memory_service.* — canonical source vector/BM25/manifest surface

This is the main persistent searchable local-source memory layer.

Core model

It maintains:

FAISS dense index       → source slices only
BM25/FTS sparse DB      → slices + gists
manifest JSON           → authoritative active surface
byFile lookup           → file relpath -> slice offsets/ids
gist text cache         → best-effort runtime cache

Important comment:

Manifest is the authoritative ACTIVE SURFACE for the current cache stem.
It must stay aligned with FAISS and BM25 after every sync.

That is a central architecture invariant.

MemoryItem
struct MemoryItem {
    uint64_t id;
    std::string relPath;
    std::size_t offset;
    std::string text;
    float score;
};

Dialect:

Slice item relPath = canonical repo-relative file path
Gist item relPath  = stable gist key

Gist IDs have the high bit set.

Slice IDs must keep the high bit clear.

Embedding implementation shown here

This file uses:

cheapEmbed(...)

It is a deterministic character-bucket embedding.

So in this code chunk, vector search is not using real provider embeddings. It is a deterministic local embedding shim.

Safe portfolio wording should avoid claiming OpenAI embedding-powered retrieval based on this file alone.

Good wording:

The current code shows a deterministic local embedding shim feeding FAISS. The architecture supports provider/dim selection and cache plumbing, but this specific VectorMemoryService implementation uses cheapEmbed.

If another implementation exists elsewhere, we would need to see it.

Active artifact names

Vector artifacts are provider/dimension scoped:

vector_D<dim>_P<provider>.faiss
vector_D<dim>_P<provider>.manifest.json
.memory/fts.db

The vector service also prunes stale vector artifacts after exact syncs.

Dimension selection

embedDimFor(Project*) chooses:

explicit profile embedDim
existing artifacts for provider
model-native dimension heuristic
stable fallback 300

It logs dimension choice.

This is useful for avoiding index dimension drift.

Full sync path

Public API:

FullSyncFromCanonicalSurface(Project*, items)
SyncIndex(Project*, items) // legacy alias

Internal:

SyncIndex_impl

It:

loads current manifest
canonicalizes input MemoryItems
preserves stable non-surface gists
builds exact new FAISS + manifest + BM25 surface
persists FAISS/manifest/BM25
installs into runtime state
logs counts

Important: Full sync is destructive by design.

Header contract says:

caller must pass ENTIRE active slice+gist surface
after completion, FAISS / manifest / BM25 all describe exactly items

This is the cleanest source-surface invariant in the block.

Apply file delta path

Public API:

ApplyFileDelta(Project*, touchedItems, touchedRelPaths)

Internal:

ApplyFileDelta_impl

It:

loads manifest
canonicalizes touched items
derives touched files and touched gist keys
decides whether to promote to full-surface replacement
preserves untouched manifest rows
removes touched file rows
preserves or drops gists depending on scope
builds exact new FAISS + manifest + BM25 surface
persists and installs
logs full details

This is not a lazy partial patch. It still rewrites one coherent active surface after merging preserved rows and touched rows.

That is good.

Delta promotion to full sync semantics

The delta path can promote to full-surface replacement if:

manifest slice files subset of touched files
live included surface subset of touched files
current included surface is empty
broad write batch threshold hit

Broad batch thresholds:

touchedFiles >= promotionSurfaceFiles
touchedFiles >= 6
touchedFiles >= 4 and >= 40% of surface
small surface: touched + 1 >= surface

This directly addresses the known failure family:

large apply batches should not preserve stale rows just because they used delta path

Strong finding:

The vector service explicitly prevents broad apply/write batches from accidentally preserving stale pre-mutation slice rows.

Gist preservation rules

There is a very clear distinction between gist types.

Surface-derived gists are not preserved across exact syncs

These are considered derived from the current source surface:

__inventory__/
__meta__/
__quick__/
file-scoped gists
folder-scoped gists
Stable non-surface gists may be preserved

shouldPreserveStableNonSurfaceGist_ preserves only gists that are not:

file-scoped
folder-scoped
surface-derived global

This prevents stale inventory/meta/quick summaries leaking across root changes or large mutation batches.

This is a strong alignment with your “summaries are support, evidence proves” doctrine.

Gist ingestion

Public API:

ingestGist(Project*, relPath, gistText)

Internal:

ingestGist_impl

It:

canonicalizes stable gist key
mints high-bit gist ID
updates manifest item as kind=gist
updates BM25 row
does not add gist to FAISS dense slice index
dedupes duplicate gist IDs/paths
logs sparse drift separately

Important:

gists live in manifest + BM25, not dense FAISS slices

So hybrid retrieval can likely retrieve gists through BM25/sparse. Dense-only FAISS is slice-only.

Search APIs
SearchFaissOnly(Project*, q, k)
Search(Project*, q, k)
SearchFaissOnly

Searches FAISS slice index only.

It skips:

invalid FAISS id -1
gist high-bit IDs
tombstones

Then hydrates each ID through:

LookupItemByID

Hydration prefers:

BM25/FTS text by ID
then repo bytes by offset fallback
Search

Delegates to:

Retrieval::HybridSearch(proj, q, k, alphaParam)

alphaParam comes from profile:

hybridWeight
hybridAuto

The actual hybrid implementation is not in this chunk, so we cannot yet summarize candidate fusion/BM25 scoring specifics. But we can say:

VectorMemoryService delegates general search to HybridSearch, and dense-only search is separately available.

Manifest repair on load

ensureLoaded_ loads FAISS and manifest, then checks:

manifest slice count vs FAISS slice count
manifest total vs BM25 rows

If mismatch and manifest has items, it attempts:

repairSurfaceFromManifestBestEffort

This hydrates manifest items from:

BM25/FTS
summary artifacts for gists
repo bytes by offset for slices

Then rebuilds exact FAISS + manifest + BM25 surface.

This is very good.

It means the system attempts to self-heal drift between:

manifest
FAISS
BM25
7. bm25_db / hybrid_search are referenced but not included yet

From this chunk we know:

BM25_DB stores/upserts/removes text rows by ID
HybridSearch is the general retrieval path used by VectorMemoryService::Search

But we have not yet seen:

how sparse candidates are scored
how invalid sparse IDs are handled
how dense/sparse candidate fusion is done
how alpha auto weighting works
how reranking/final top-k happens

So those claims should wait until the next block part.

8. memory_seed_builder.* — semantic-summary-only support seed

This file is very important for the “summary is not evidence” boundary.

The header states:

Canonical semantic-summary-only seed builder.
Legacy inventory-derived and old summary-layout fallback reads removed.

It reads precomputed sidecar summaries only:

root semantic summary
folder semantic summaries
file semantic summaries

It emits memory seed JSON:

memory: key -> content
includedKeys: ordered keys
sliceMeta: evidence/admissibility metadata
Explicit authority classification

Every emitted slice is classified as:

SeedEvidenceClass::SemanticPrimer
SeedAdmissibility::SupportOnly
mayProveImplementation = false

The result sets:

primerOnly = true
allSupportOnly = true
anyMayProveImplementation = false

This is one of the clearest proof-boundary files in the codebase.

Strong safe claim:

Semantic memory seeds are deliberately support-only. They may orient the model, but they must not be treated as raw source proof.

Sources read

It reads canonical semantic summaries via PathUtils::SemanticSummaryArtifactPath for:

Root
Folder
File

Keys align with SummaryKeys, such as:

__inventory__/all
__folders__
__folders__/<folder>
files/<relNorm>

No model calls. No truncating inside a slice. Whole sidecar slices are included or dropped according to budget.

9. memory_store.* — secondary/session memory, not source file evidence

MemoryStore is a separate memory system.

It stores records in:

secondary.db
secondary.faiss
fts virtual table
memory_meta

A MemoryRecord has:

projectId
uuid
timestampUtc
sourceTag
rawText
kind
embedding

This is for:

user/assistant conversation chunks
assistant facts
legacy file/gist tags
memory map UI
secondary memory retrieval

This should not be confused with the canonical local source surface from VectorMemoryService.

Important distinction

MemoryStore is not the same as VectorMemoryService.

VectorMemoryService:
  local source slices + gists
  manifest active surface
  file-context retrieval
  source evidence boundary

MemoryStore:
  secondary memory records
  user/assistant/fact memory
  memory map snapshots
  dense top-k over memory records

This matters for portfolio claims.

Safe wording:

Coeus has a secondary memory store for conversation/user/fact memory, separate from the file-context retrieval surface used for source evidence.

Embedding cache/dim behavior

MemoryStore uses:

VectorMemoryService::embedWithDim
EmbedCache::MakeKey(providerID, dim, text)

Again, in this code, embeddings are the deterministic shim.

It respects existing secondary.faiss dimension as truth if present and logs mismatches.

Memory snapshots

buildSnapshot() produces UI-friendly rows:

projectID
isUser
levelTag
folderPath
unixTime
tokens
embedded
pinned
textPreview

It classifies source tags into:

fact
file
quick-file
folder
inventory
facts
persona
summary
user
assistant

This is mostly UI/inspection support.

10. memory_graph.* — typed weighted graph

MemoryGraph persists to:

<project>/memory_graph.json

Nodes:

File
Slice
Gist
Inventory
Note
ConvSummary

Edges:

Contains
Adjacent
DerivedFrom
SameFolder
CoRetrieved
Mentions
Similar

It supports:

upsertNode
upsertEdge
bumpUse
neighbors
get
outEdges
listNodes
listEdges

Edges can decay and be reinforced:

w = old_weight * decay + reinforce

Bidirectional edge kinds:

Adjacent
SameFolder
CoRetrieved
Similar

The header says it is used by:

VectorMemoryService::{SyncIndex, ingestGist}
RetrievalController
Memory Map panel

But in the provided vector_memory_service.cpp, there is no obvious active MemoryGraph update call in the shown section despite the include. So the graph role is architecturally declared, but active wiring is not proven in this pasted code.

Safe wording:

The graph data structure exists and is designed for typed memory relationships, but this chunk does not clearly show it being populated by the vector sync path.

11. snapshot_builder.* — conversation-scoped FAISS snapshots

SnapshotBuilder builds a separate FAISS index:

vector_D<embedDim>_P<providerID>_<namespace>.faiss

It uses:

IndexHNSWFlat + IndexIDMap2
chunk ids
EmbedCache
VectorMemoryService::embed

This appears to be a conversation-scoped namespace snapshot, likely older or auxiliary compared with the canonical VectorMemoryService active surface.

It supports async build through TaskEngine.

Important distinction:

SnapshotBuilder builds a namespace snapshot file.
VectorMemoryService owns the canonical active source surface manifest/FAISS/BM25.
12. embed_cache.* — salted embedding cache

EmbedCache is SQLite-backed.

Key:

SHA256("prov=<providerID>;dim=<dim>;" + text)

This avoids collisions across:

different providers
different dimensions
same text

It validates blob length against dims and treats mismatches as cache misses.

Config env vars:

EMBED_CACHE_PATH
EMBED_CACHE_DIR

Default:

cache/embed_cache.sqlite

This is good infrastructure.

Caution:

SnapshotBuilder currently uses a raw sha256(text) lookup in part of its implementation, while EmbedCache::MakeKey(providerID, dim, text) exists as the preferred salted helper. It then inserts with provider ID but using the raw text hash in that file. That may be legacy drift.

Suggested review note:

Check SnapshotBuilder cache keying. It appears to use sha256(text), not EmbedCache::MakeKey(provider,dim,text), so it may bypass the salted provider/dim cache contract.
13. Source evidence vs summaries vs continuity — exact boundary

This block gives a clean taxonomy.

Source evidence

Source evidence is:

EvidenceClass::SourceEvidence
BlockKind::Retrieval
plane/retrieval/slice|...
text begins SOURCE EVIDENCE — CONTENT VISIBLE
includes File + Offset + actual slice content
markers evidence.class=source_evidence
source_evidence.content_visible=1

Only this should be used as implementation proof.

Semantic primer

Semantic primer is:

EvidenceClass::SemanticPrimer
BlockKind::SemanticPrimer
plane/semantic_primer|...
summary sidecars / gists / primer chunks
markers evidence.class=semantic_primer
support_only.semantic_primer=1 when policy says support-only

It can orient the model but should not prove implementation.

Inventory facts

Inventory facts are:

plane/inventory|summary
plane/inventory|list
plane/inventory|gist
evidence.class=inventory_facts

They prove facts about the included file surface:

which files exist/included
counts
pinned files
top folders/extensions

They do not prove what code inside a file does.

Session memory / conversation tail

Session memory is:

plane/conv_tail|pair
plane/conv_tail|summary
USER_CTX
CONV_SUMMARY
evidence.class=session_memory

It preserves continuity and user context. It is not proof of source code behavior.

Structured continuity

Structured continuity is:

plane/plan
plane/apply
plane/decision
plane/recent_mutation

It carries operational continuity:

what was planned
what was applied
what changed recently
which mutation artifacts exist

Even recent mutation is separated from source evidence. It can support “what changed?” but should not fabricate exact diffs unless exact diff artifacts/source evidence are present.

14. Main strengths
1. Strong authority-gated packing

The lattice does not blindly include everything. It parses TURN_STATE and gates:

inventory
source evidence
semantic primer
session memory
tool observations
recent mutation

This is exactly the right direction.

2. Source evidence is visibly different in prompt text

The runtime formats source evidence with:

SOURCE EVIDENCE — CONTENT VISIBLE
File:
Offset:

This is a strong human/model-facing boundary.

3. Rich trace markers

The trace can diagnose:

candidate retrieval existed
candidate was source vs primer
candidate was policy-rejected
candidate was admitted
candidate was packed
actual files visible in prompt

This supports eval-driven architecture work.

4. Vector/BM25/manifest coherence is taken seriously

The vector memory service has:

exact full sync
exact file delta
manifest as active surface
BM25 row reconciliation
FAISS slice count checks
repair from manifest
stale artifact pruning
delta-to-full promotion

This is professional-grade compared to simple “append embeddings to FAISS” retrieval.

5. Summaries are explicitly support-only

memory_seed_builder is very clear:

semantic memory seed must never be treated as raw source proof

This directly supports your “evidence proves, summaries route/orient” doctrine.

15. Risks / issues to review
1. LookupGistText_ is a stub

In lattice_runtime.cpp:

static std::string LookupGistText_(Project*, const std::string&)
{
    return {};
}

So inventory gist packing currently uses fallback text, not persisted gist content.

This may be intentional, but it is worth flagging.

2. Tool observation plane is currently a no-op

AddToolObservationBlocks_ logs a conservative no-op and adds no blocks.

If tool observations are meant to influence later turns, this needs implementation or documentation.

3. SnapshotBuilder may use unsalted cache keys

EmbedCache::MakeKey(providerID, dim, text) is the hardened helper, but snapshot_builder.cpp appears to use raw sha256(text) for cache lookup/insert.

That could cause cross-provider/dim cache confusion unless intentionally legacy/isolated.

4. Vector embeddings are deterministic cheap embeddings here

VectorMemoryService uses cheapEmbed, not a provider model embedding.

This may be a placeholder or intended local deterministic behavior. But portfolio wording should not overclaim “semantic embedding retrieval” unless another embedding implementation exists outside this chunk.

5. MemoryGraph active wiring not proven

MemoryGraph exists and is included, and the header says it is used by vector sync/retrieval/UI, but the pasted vector service code does not clearly upsert graph nodes/edges.

Review needed before claiming active graph-expanded retrieval.

6. Some marker logic is duplicated between lattice.cpp and lattice_runtime.cpp

There are repeated helper functions and marker logic patterns.

This is not automatically wrong, but it is a potential drift surface.

The architecture comment says ownership was split, but helper duplication could still cause future split-brain.

16. Carry-forward notes for the codebase overview
Block 8 Part 1 covers the CLE lattice and memory substrate. The lattice runtime consumes canonical TURN_STATE transport, derives authority admission gates, admits or rejects planes, packs bounded blocks, emits prompt markers, and persists lattice traces. Source evidence is explicitly separated from semantic primers and continuity: `EvidenceClass::SourceEvidence` chunks become `plane/retrieval/slice|...` blocks with `SOURCE EVIDENCE — CONTENT VISIBLE`, while semantic summaries/gists become `SemanticPrimer` support-only blocks. Conversation tail, user context, plan/apply/decision, and recent-mutation planes are continuity surfaces, not source proof.

VectorMemoryService owns the canonical local-source retrieval surface. It maintains FAISS for source slices, BM25/FTS for slices and gists, and a manifest as the authoritative active surface. Full root scans use `FullSyncFromCanonicalSurface`, while changed files use `ApplyFileDelta`. Both paths rebuild exact coherent FAISS/manifest/BM25 surfaces. Delta updates can promote to full-surface replacement for broad apply batches to avoid preserving stale rows. Gists are high-bit IDs and live in manifest/BM25, while source slices are high-bit-clear IDs and populate FAISS.
17. Portfolio-facing summary
The CLE lattice and memory layer demonstrates that Coeus AI does not simply dump retrieved text into a prompt. It classifies context into explicit planes — turn state, inventory, source evidence, semantic primer, session memory, tool observations, and structured continuity — then admits or rejects those planes using canonical turn policy. Source evidence is packed as visible file content with file/offset metadata, while summaries and memory seeds are marked as semantic primers or support-only continuity. The vector memory service keeps FAISS, BM25, and a manifest aligned through exact full-surface syncs and exact touched-file deltas, making the local source retrieval surface auditable and repairable.

A stronger version for technical readers:

Block 8 shows the core prompt-context compiler and retrieval memory substrate. `lattice_runtime.cpp` consumes canonical `TURN_STATE`, gates all context planes by authority policy, and emits traceable prompt markers showing what evidence was actually packed. `vector_memory_service.cpp` maintains a coherent active source surface across FAISS, BM25/FTS, and manifest state, with exact rebuild and exact delta semantics. Semantic summaries and memory seeds are explicitly classified as support-only primers, while implementation proof requires `SourceEvidence` chunks whose source content is visible in the packed prompt.






Block 8 Part 2 Summary — Retrieval Pipeline, BM25, Hybrid Search, Retrieval Controller, Focused Retrieval, Prompt IR, User Context

This set fills in the retrieval layer that was missing from the previous Block 8 part.

It covers:

PromptBlockV1 neutral IR
BM25 sparse index wrapper
Hybrid dense/sparse search
Retrieval policy view
Retrieval controller routing/execution
Focused set / focused hybrid retrieval
Retrieval lattice assembly
User context aggregation

The big picture:

RetrievalOptions + canonical upstream state
  → retrieval_policy builds narrow policy view
  → retrieval_controller chooses execution branch
  → hybrid_search / global scan / focused detail fetch source candidates
  → retrieval_detail assembles bounded RetrievalChunks
  → chunks are marked SourceEvidence or SemanticPrimer
  → lattice_runtime later admits/packs them by TURN_STATE authority

This confirms the retrieval architecture is not just “vector search.” It is a multi-path retrieval controller with explicit source-evidence protection, global lexical scan, focused file modes, coverage backstops, graph boosting, BM25/FAISS hybrid search, and detailed trace diagnostics.

1. prompt_ir — neutral prompt block interface

Files:

core/cle/prompt_ir/prompt_block_v1.h
core/cle/prompt_ir/prompt_ir.h

This is a small but important layering seam.

PromptBlockV1 is deliberately neutral:

namespace CLE {
struct PromptBlockV1 {
    std::string key;
    std::string title;
    std::string text;
    int priority;
    std::string plane_hint;
    BlockClass cls;
    std::vector<std::string> marker_hints;
};
}

Classes:

Evidence
Primer

The header states the layering rule clearly:

Tools MUST NOT depend on PromptBuilder / KeyedPromptBlock.
Tools emit PromptBlockV1.
The packer converts at the seam.

This is good architecture. It prevents tools from coupling themselves to the agent prompt builder.

Important semantics:

Evidence = verbatim / provenance-bearing evidence suitable for Patch/Verbatim constraints
Primer   = non-evidence primer such as summaries, inventory, graph hints, tool summaries

This matches the larger CLE doctrine:

evidence proves
primers orient

Caveat: this set only defines the neutral IR. The actual conversion seam from PromptBlockV1 into lattice/keyed blocks is not shown here.

2. BM25_DB — sparse FTS index with exact row semantics

Files:

core/cle/retrieval/bm25_db.*

BM25_DB wraps SQLite FTS5.

It stores:

id -> text

in:

vec_fts(id UNINDEXED, text)

with tokenizer:

unicode61 remove_diacritics 2
Important correctness features
Exact per-ID semantics

FTS5 does not enforce uniqueness on id, so the wrapper enforces:

upsert(id):
  delete existing rows for id
  insert fresh row

removeMany(ids):
  delete rows in transaction

replaceAll(rows):
  delete-all FTS table
  dedupe rows by id
  insert sorted by id ascending

This is important because the vector service expects BM25 rows to describe the same active surface as the manifest.

Deterministic full replacement

replaceAll does:

last duplicate id wins
ids sorted ascending
insert deterministic order

Good for reproducibility.

Safe FTS query generation

makeSafeFtsQuery:

sanitize UTF-8
keep only [A-Za-z0-9_:]
replace everything else with spaces
collapse whitespace
cap length to 1024 bytes

This prevents broken FTS MATCH syntax from user input.

Ranking modes

It probes support for:

bm25(vec_fts)
ORDER BY rank
naive fallback

Modes:

BM25
Rank
Naive

In BM25 mode, SQLite FTS5 returns smaller-is-better, sometimes negative, so it maps to larger-is-better:

raw >= 0 → 1 / (1 + raw)
raw < 0  → 1 + (-raw)

In rank/naive modes it uses a constant proxy score, relying on DB ordering.

Search fallback

searchAll(k) returns newest rows by rowid descending. Hybrid uses this as a backfill only when sparse search returns nothing and either:

SparseOnly mode
or dense produced nothing

That is sensible: it avoids returning nothing from a populated DB when query terms fail.

3. hybrid_search — dense/sparse candidate fusion only

Files:

core/cle/retrieval/hybrid_search.*

This file is cleanly scoped. The comment says:

HybridSearch remains a pure retrieval-ranking component.
It does NOT reinterpret turn policy, query scope, inventory intent,
global-scan permission, or anchor legality.

That is good. It keeps policy in retrieval policy/controller.

Inputs
HybridSearch(Project* proj, const std::string& query, int topK, float alphaParam)
Profile-driven mode

It reads from AgentProfile:

retrievalMode:
  0 = DenseOnly
  1 = SparseOnly
  2 = Hybrid

hybridWeight
hybridAuto
hybridOversample
Candidate count

Candidate K:

candK = ceil(topK * hybridOversample)
clamped to [topK, 256]
oversample clamped to [1.0, 8.0]
Dense path

Dense candidates:

VectorMemoryService::SearchFaissOnly(proj, query, candK)

Then canonicalizes rel paths through DocIdentity.

Dense-only mode returns these directly.

Sparse path

Sparse candidates:

BM25_DB db(VectorMemoryService::dbPath(proj));
db.search(query, candK)

Then hydrates each id:

VectorMemoryService::LookupItemByID(proj, id)

Sparse-only mode preserves DB order and returns directly.

Hybrid fusion

Fusion is rank-based weighted RRF:

score = alpha       * 1/(60 + dense_rank)
      + (1-alpha)  * 1/(60 + sparse_rank)

This is good because it avoids trying to compare incompatible dense and sparse raw scores.

Tie-break:

fused score desc
relPath asc
offset asc
id asc

Very deterministic.

Auto-alpha

Auto alpha is a query-shape heuristic only.

It starts around 0.60 and shifts lower for code-like queries:

identifiers
uppercase
digits
underscores
::
->
slashes/dots
#

Longer natural-language queries shift alpha higher.

Meaning:

codey query → more sparse/BM25 influence
long semantic query → more dense influence

This is a reasonable design.

4. retrieval_policy — narrow policy view, not execution owner

Files:

core/cle/retrieval/retrieval_policy.*

The header makes the ownership boundary explicit:

retrieval_policy = narrow shared normalization / policy helpers
retrieval_controller = execution rewrites and branch selection
retrieval_detail = focused-mode assembly + lattice packing
hybrid_search = dense/sparse candidate fusion only
bm25_db = sparse storage/query only

It also states:

heuristic flags are advisory only and must never become canonical routing truth

This is important, but in practice the controller does use these advisory flags to choose execution strategies. That is probably acceptable if they remain execution-local and traceable, but it is still a drift area to watch.

What RetrievalPolicyView contains
authorityMode
allowRetrieval
allowGlobalScan
allowQuickFileSummaries
allowInventoryGistBuild
triggerInventoryGistBuild

inventoryTurnEffective
globalScanEffective
semanticPrimerAllowed

heuristicEnumerativeHint
heuristicInventoryHint
heuristicGlobalHint
heuristicCrossRepoUsageHint
heuristicAnalyticOrderingHint
heuristicGlobalCrossFileHint

hasComparePairTruth
compareLeftTargetFile
compareRightTargetFile
compareExplicitMentionCount

canonicalTargetFile
hasCanonicalTarget
canonicalTargetAdmitted

hasMentionAnchor
hasStrictAnchor
hasPersistentFocusAnchor
hasExplicitOwnershipAnchor
hasSoftFocusSignals
hasAnyAnchor
Authority behavior
ImplementationStrict:
  prefers source evidence
  disallows semantic primer

FileUnderstanding:
  prefers source evidence

InventoryListing:
  inventory only
  no semantic primer
  no hybrid

ConversationRecall:
  no hybrid
  no canonical target admission
Canonical target admission

The policy view can detect candidate targets from:

forcedRelPath
strictFilePath
single explicit user mention
activeRelPath in single-focus situations
single persistent focus

But it only admits a canonical target when allowed by authority and anchor rules.

Good guards:

compare pair suppresses single canonical target
inventory/conversation recall cannot admit canonical target
soft focus alone is not ownership
persistent focus cannot localize broad/global cross-file questions
active hint only works for FileUnderstanding / ImplementationStrict, not broad turns

This aligns with the source-led vs continuity-drift problem we’ve been diagnosing.

Compare pair truth

Compare pair is observational:

two distinct explicitly user-mentioned focused files

This is a good definition. It does not rely on raw query text alone.

Query heuristic normalization

Heuristics strip fenced code blocks and file path tokens before classifying query type. This prevents code dumps/path lists from being misread as natural-language intent.

One small possible issue:

if (s[i] == ':' && s[i + 1] == '/' && s[i + 2] == ':') return false;

That looks like a typo or redundant malformed URL check. The next line correctly checks ://. It is probably harmless but worth cleaning.

5. retrieval_controller — execution orchestration and routing rewrites

Files:

core/cle/retrieval/retrieval_controller.*

This is the central retrieval orchestrator.

Public API:

RetrievalController::fetch(Project*, userQuery, RetrievalOptions)
RetrievalOptions

Key options:

mode:
  Hybrid
  StrictFile
  FocusedSet
  FocusedHybrid

authorityMode:
  Default
  ImplementationStrict
  FileUnderstanding
  ArchitectureOverview
  InventoryListing
  ConversationRecall

allowRetrieval
allowGlobalScan
allowQuickFileSummaries
triggerQuickFileSummaries
allowInventoryGistBuild
triggerInventoryGistBuild

focusedFiles
forcedRelPath
strictFilePath
activeRelPath
topK
maxContextTokens
neighbourLook
latticeMaxPerFile
injectTargetFileSlices
Execution flow

High-level:

1. Copy requested options to local options
2. Apply authority mode constraints
3. Build retrieval policy view
4. Rewrite mode for compare pair / canonical target / deanchoring
5. Build trace
6. Early skip if retrieval disabled / inventory listing / no project / no unanchored hybrid allowed
7. Branch:
   - FocusedHybrid
   - FocusedSet
   - StrictFile
   - GlobalScan
   - Hybrid
8. Apply coverage backstop
9. Compute evidence counts
10. Queue quick summary mirrors
11. Attach debug JSON trace

This is a sophisticated execution controller.

Mode rewrite: compare pair

If policy detects compare pair truth:

mode_rewritten=compare_pair_to_focused_set

It keeps only the two compare files as focused files.

This is good: compare turns should not devolve into broad hybrid search.

Mode rewrite: deanchor global cross-file architecture

If:

ArchitectureOverview
canonical target admitted
soft focus signals
global cross-file hint
no explicit ownership anchor

Then it deanchors:

mode_rewritten=deanchor_global_cross_file

and clears activeRelPath / soft focus.

This directly addresses a known failure mode:

Source-led architecture questions should not be accidentally narrowed to the current/active file just because a focus hint exists.

Good.

Mode rewrite: canonical target localization

If there is an admitted canonical target and mode is Hybrid, it seeds focus:

focused file anchor = canonical target
activeRelPath = canonical target

Then:

ImplementationStrict/FileUnderstanding → FocusedSet
ArchitectureOverview → FocusedHybrid
source-led architecture → keep Hybrid but localized/cross-file preserved

This is nuanced and generally correct.

Inventory listing branch

If authority mode is InventoryListing, it skips generic retrieval:

reason=authority_inventory_listing_skips_generic_retrieval

That is good. Inventory questions should be answered from inventory facts, not arbitrary source chunks.

StrictFile branch

If mode is StrictFile and enumerateSlices is true:

resolve forcedRelPath
GetSlicesForFile
emit source evidence chunks only

It creates synthetic IDs:

strict_file|<rel>|<offset>|<text-size>

Evidence class:

SourceEvidence

This is an exact full/partial file read branch.

Good for:

show file
explain this exact file
strict file grounding
Global scan branch

When policy.globalScanEffective and mode is Hybrid:

FileSearch::EnsureIndex
FileSearch::QueryAndScan

If scan hits, it returns a retrieval result made from lexical excerpts:

FILE: <rel> (Lx-Ly)
Lx: ...

Evidence class:

SourceEvidence

This means global scan is not a primer. It is actual source excerpts from included files.

Good for:

where used
find usages
startup flow
runtime flow
source-led architecture/order questions

This is one of the strongest project-aware/local-source features in the codebase.

Low-signal suppression

If default authority, no global scan, no inventory, no anchor, no global/inventory hints, and query is low-signal, retrieval is suppressed.

Example low signal:

single vague token
no file/path/query structure

Good. Prevents random retrieval pollution.

Hybrid branch

For generic hybrid retrieval:

reqK = topK * 4

For source-led architecture:

reqK = topK * 8
min 48
max 384

This widens recall for source-led questions.

Then:

HybridSearch
semantic-primer filter if source-only
neighbour expansion
compare/canonical/advisory score boosts
graph expansion/boosting
source-led recall union
shortlist
optional coref injection
AssembleLattice
semantic-primer filter again
coverage backstop
trace
graph co-retrieval update

This is much more than simple hybrid search.

6. Source-led recall union — important improvement

applySourceLedRecallUnion_ is a key section.

Enabled when:

allowGlobalScan
mode == Hybrid
execution looks source-led architecture
not inventory
not compare pair

It performs lexical global scan and uses the scan results to reinforce/inject source evidence.

It does two things:

1. Reinforce files already in top shortlist

If scan confirms a file already present in top hits, it boosts those chunks and marks:

InjectionKind::ScanEvidence
2. Inject scan excerpts for missing files

If scan finds relevant files not represented in the shortlist, it injects excerpt chunks as source evidence.

Trace counters:

scanReinforcedFiles
scanReinforcedChunks
scanSkippedAlreadyPresentFiles
scanPromotedExistingFiles
injectedFiles
injectedChunks
injectedRels
reinforcedRels

This is excellent for the eval failures you were seeing where broad/source-led questions missed relevant files because hybrid search alone undercovered the source surface.

Strong finding:

The controller now has an explicit lexical-source recall union that can reinforce or inject proof-bearing source excerpts for source-led architecture/usage/runtime questions.

7. Coverage backstop — compare/canonical target safety net

Functions:

applySourceCoverageBackstop_
ensureSourceEvidenceCoverageForFile_
buildCoverageBackstopChunksForFile_

Purpose:

If a compare target or canonical target lacks source evidence,
inject first slices from that file as SourceEvidence.

For compare pair:

left target max 2 chunks
right target max 2 chunks

For canonical target:

canonical target max 2 chunks

It tries:

LookupItemByOffset(file, 0)
LookupItemByOffset(file, 4096)
fallback GetSlicesForFile

Trace:

coverage_backstop.compare_left_inserted
coverage_backstop.compare_right_inserted
coverage_backstop.canonical_inserted

Warnings:

compare_left_target_backstop_injected
compare_right_target_backstop_injected
canonical_target_backstop_injected

This is very important because it prevents source-required turns from returning only summaries/gists or unrelated chunks.

8. Evidence classification in controller

There are two evidence-class pathways.

From item kind
ItemKind::Slice → SourceEvidence
ItemKind::Gist  → SemanticPrimer

Used after focused detail modes.

From MemoryItem identity/path
high-bit ID or stable gist key → SemanticPrimer
otherwise → SourceEvidence

Used when filtering hybrid hits before assembly.

This is consistent with previous vector memory design:

source slices have high bit clear
gists have high bit set / stable keys
9. Quick summary queue mirrors

queueQuickSummaryMirrors_ queues quick summaries for source-evidence files only.

Conditions:

allowQuickFileSummaries == true
triggerQuickFileSummaries == true

It only considers chunks where:

evidence_class == SourceEvidence
kind == Slice

Trace says:

mode = reuse_only
owner = semantic_meta_pipeline
source = retrieval_best_effort
only_source_evidence = true
extra_files_ignored = true

This is a good boundary. Retrieval can request summaries to be prepared, but it does not treat them as proof for the current turn.

10. Graph retrieval role is now proven

In the previous part, MemoryGraph existed but active use was unclear.

This part shows it is active in retrieval:

Graph boosting

If profile enables:

prof.retrievalGraphEnabled

Then controller:

loads MemoryGraph
starts from current hits
expands through edges:
  Adjacent
  Contains
  DerivedFrom
  SameFolder
  CoRetrieved
max hops = 2
fanout = 16
min weight = 0.05

It can add neighbor items via:

VectorMemoryService::LookupItemByID

Then score boost includes:

hop distance
node useCount
recency
edge average weight
graphGamma
graphCap 0.25
Co-retrieval reinforcement

After final retrieval result:

bumpUse(each chunk id)
upsert CoRetrieved edges between chunks
save graph

So the graph index/memory graph is not just UI; it actively influences retrieval when enabled.

Caveat:

It depends on graph nodes/edges being populated somewhere. This file shows co-retrieval edges are created after retrieval, but not necessarily initial file/slice structural graph construction. That may be in other files.

11. retrieval_detail — focused retrieval and lattice assembly

Files:

core/cle/retrieval/retrieval_detail.*

The header is explicit:

retrieval_detail owns focused-mode assembly and lattice packing only
must not become a second routing or policy layer

This file does two major jobs:

FocusedSet / FocusedHybrid retrieval
AssembleLattice for final RetrievalChunk selection
FocusedSet

RunFocusedSet:

validates focused files exist
dedupes slots
sorts by mention/focus/weight/path
builds per-file slice plans
round-robin includes slices across files
ensures compare/multi-file bundles cover all files
emits source evidence chunks only

It has an execution contract trace:

must_cover_all_focused_files
coverage_floor
source_evidence_only
semantic_primer_allowed
global_topup_allowed=false

For compare pair or multi-file bundle, coverage floor equals number of focused files.

This directly helps compare/multi-file correctness.

FocusedHybrid

RunFocusedHybrid does focused slices first, then optionally global top-up.

Global top-up is disabled when:

compare pair-like
multi-file bundle-like
source-evidence-only authority
allowGlobalScan false

Allowed mainly for:

Default
ArchitectureOverview

It reserves a small number of global slices:

max global slices = 5
global reserve ≈ maxItems / 6

This is a good compromise:

focused grounding first
small broad context only when safe
Focused coverage diagnostics

Both focused modes trace:

focused_files
covered_files_any
covered_files_source_evidence
covered_mention_files
required_files
required_files_missed
per-file aligned source/semantic counts

Warnings are emitted for missing required source evidence:

focused_set_required_file_source_evidence_missing:<rel>
focused_hybrid_required_file_source_evidence_missing:<rel>

Very useful for eval and debugging.

AssembleLattice

Despite the name, this is retrieval result assembly, not final prompt lattice.

It selects from MemoryItem hits into RetrievalChunk objects.

Inputs:

hits
originById
injKind
maxContextTokens
maxItems
latticeMaxPerFile
diversity
preferContiguous

It classifies:

slice → SourceEvidence
gist  → SemanticPrimer

It groups candidates by:

file:<rel>        for source slices
filegist:<rel>    for file gists
gistns:<namespace> for inventory/folder/etc gists
other:<rel>

Group caps:

source file cap = latticeMaxPerFile
gist namespace cap = max(2, latticeMaxPerFile/2)
file gist cap = 1..3

Mandatory injections bypass group cap:

Mention
Coref
ScanEvidence

Selection order:

1. mandatory injection chunks first
2. utility-based greedy selection

Utility:

score
penalized by group diversity
small contiguous boost for nearby source slices
divided by token cost^0.70

This is a professional context-selection heuristic. It balances relevance, diversity, contiguity, and token cost.

12. user_context_aggregator — structured user/project context

Files:

core/cle/user_context/user_context_aggregator.*

This manages proj->contextData.userSection.

It is not retrieval source evidence. It is user/session/project context.

Canonical Markdown skeleton:

# User Context

## Notes
## Goals
## Tasks
## Facts
## Constraints
## Preferences
## Decisions

Features:

append typed item
dedupe exact duplicate ignoring date badge
UTC date badge
parse markdown into model
render canonical markdown
build compact prompt block
async append/skeleton helpers

Prompt block format:

【User Context】
Notes:
- ...

Goals:
- ...
...

It strips date badges and caps output UTF-8 safely.

This feeds the session/user-context plane, not the source-evidence plane.

Good boundary:

UserContext = memory/context facts
not code proof
13. Full retrieval pipeline after this set

A more complete pipeline now looks like this:

Upstream resolved turn state / planner
  ↓
RetrievalOptions
  ↓
retrieval_policy::BuildRetrievalPolicyView
  - authority capability
  - normalized anchor facts
  - compare pair fact
  - advisory query-shape hints
  ↓
retrieval_controller::fetch
  - authority application
  - mode rewrites
  - branch selection
  - trace
  ↓
Branch:
  A) InventoryListing → no generic retrieval
  B) StrictFile → FileCtxSvc::GetSlicesForFile
  C) FocusedSet → per-focused-file source slices
  D) FocusedHybrid → focused slices + safe global top-up
  E) GlobalScan → lexical scan source excerpts
  F) Hybrid → dense/sparse/graph/global recall union
  ↓
HybridSearch
  - FAISS dense source slices
  - BM25 sparse slices/gists
  - rank-based weighted RRF
  ↓
RetrievalDetail::AssembleLattice
  - token budget
  - per-file diversity
  - mandatory mentions/coref/scan evidence
  - SourceEvidence vs SemanticPrimer classification
  ↓
RetrievalResult
  - chunks
  - evidence counts
  - global scan summary
  - debugJSON trace
  ↓
lattice_runtime
  - TURN_STATE authority admission
  - pack source evidence / primer / memory planes
14. Source evidence vs semantic primer now fully confirmed

This set reinforces the boundary:

Source evidence comes from
FileCtxSvc::GetSlicesForFile
VectorMemoryService source slice IDs
GlobalScan lexical excerpts
Coverage backstop slices
Focused file source slices
ScanEvidence injections

and is marked:

EvidenceClass::SourceEvidence
ItemKind::Slice
Semantic primer comes from
gists
summary keys
inventory/folder/file summary stable keys
high-bit gist IDs

and is marked:

EvidenceClass::SemanticPrimer
ItemKind::Gist
Continuity comes from
conversation tail
user context
recent mutation
plan/apply/decision planes
graph co-retrieval memory

and is not source proof by itself.

This is the cleanest wording:

SourceEvidence = local source content visible to the model.
SemanticPrimer = summaries/gists/inventory used for orientation.
Continuity = conversation/project state used for routing and coherence.
15. Strengths
1. Retrieval roles are well separated

The headers explicitly define module ownership:

policy helpers
controller execution
detail focused assembly
hybrid candidate fusion
BM25 storage/query

This is healthy.

2. Hybrid search is deterministic and robust

Rank-based RRF is preferable to mixing raw dense/BM25 scores.

3. Global scan is a real source-evidence path

It can return line-numbered source excerpts. This is a major “project-aware local source context” claim.

4. Compare/canonical target coverage backstop is strong

It prevents source-required turns from missing the actual requested file.

5. Source-led recall union addresses broad architecture misses

Hybrid search alone often misses necessary files. The scan-reinforcement path directly fixes that class.

6. Focused modes have coverage contracts

Focused set/hybrid now trace required-file coverage and warn on source-evidence misses.

7. Retrieval traces are highly diagnostic

debugJSON carries:

policy
execution
heuristics
query diagnostics
global scan
coverage backstop
recall funnel
lattice assembly
evidence counts
focused coverage
canonical/compare diagnostics

That is excellent for eval.

16. Risks / things to review
1. Retrieval policy still has heuristic influence

The comments say heuristics are advisory only. In practice, controller uses them for execution decisions like:

source-led architecture execution
global scan effective
deanchor global cross-file
widen recall
recall union

This may be fine, but the key is that upstream canonical truth should still own authority/answer-kind. This should be monitored so retrieval heuristics do not become a second canonical classifier.

2. retrieval_policy has a likely typo in URL check

This line looks suspicious:

if (s[i] == ':' && s[i + 1] == '/' && s[i + 2] == ':') return false;

Probably intended only ://. The next line already checks ://.

Low risk, but clean it.

3. retrieval_controller.cpp is very large and doing a lot

It owns many execution behaviors:

mode rewrites
global scan
source-led recall union
coverage backstop
graph expansion
coref assist
quick summary queue
trace construction

This may be acceptable for a central orchestrator, but it is a complexity hotspot.

Potential future split:

retrieval_controller.cpp = branch orchestration only
retrieval_source_recall.cpp = source-led recall union
retrieval_backstop.cpp = coverage backstop
retrieval_trace.cpp = trace building
retrieval_graph_boost.cpp = graph expansion

No urgent change unless maintenance pain is rising.

4. Graph expansion can introduce items based on historical co-retrieval

It filters semantic primer when not allowed, which is good. But graph neighbors are still learned from previous retrievals, so they are supportive recall signals, not proof. Final proof still depends on source evidence being packed.

5. FocusedHybrid recursively calls RetrievalController::fetch

FocusedHybrid global top-up calls fetch again with modified options. It disables some fields to prevent runaway behavior, but recursion should be watched carefully.

The code does clear:

focusedFiles
activeRelPath
corefAssist=false
injectTargetFileSlices=0
triggerQuickFileSummaries=false
mode=Hybrid

So it looks controlled.

6. BM25 rank fallback has constant score

In Rank/Naive modes, score is 1.0, but DB order is preserved in SparseOnly and rank position drives RRF in Hybrid. That is acceptable because hybrid uses rank, not raw sparse score.

17. Carry-forward notes for final block analysis
Block 8 Part 2 completes the retrieval picture. Retrieval is split across narrow policy helpers, a central execution controller, detail-focused assembly, hybrid dense/sparse fusion, and BM25 storage. `retrieval_controller` consumes `RetrievalOptions`, builds a `RetrievalPolicyView`, rewrites execution mode for compare pairs, canonical targets, deanchored global architecture turns, strict files, focused sets, global scans, and hybrid search. It emits detailed trace JSON with policy, execution, global scan, coverage backstop, recall funnel, focused coverage, and evidence counts.

Hybrid search combines FAISS dense slice results and BM25/FTS sparse rows using rank-based weighted reciprocal rank fusion. Dense, sparse, and hybrid modes are profile-driven. Sparse rows include slices and gists; source slices hydrate through `VectorMemoryService`, while high-bit/stable-key gist items are semantic primers.

The retrieval controller now has multiple safeguards for evidence quality: source-led recall union uses global lexical scan to reinforce or inject source-bearing excerpts; compare/canonical coverage backstop injects target file slices when source evidence is missing; focused modes enforce per-file coverage contracts; and semantic primers are filtered out when execution requires source evidence only. Retrieval chunks are explicitly classified as `SourceEvidence` for slices and `SemanticPrimer` for gists/summaries before they reach the lattice.
18. Portfolio-facing wording
The retrieval layer combines dense FAISS search, sparse BM25/FTS search, global lexical scan, focused file retrieval, and graph-assisted recall under a single controller. Retrieval outputs are not treated uniformly: source slices are marked as `SourceEvidence`, while gists and summaries are marked as `SemanticPrimer`. The controller includes coverage backstops for canonical and compare targets, focused-file coverage diagnostics, and source-led recall reinforcement using lexical scan excerpts. This allows Coeus to answer codebase questions from visible local source evidence rather than relying on summaries or conversation memory.

More technical version:

Retrieval is implemented as a policy-consuming execution layer. `retrieval_policy` derives a narrow policy view from normalized `RetrievalOptions`, while `retrieval_controller` chooses execution strategies such as strict file, focused set, focused hybrid, global scan, or hybrid retrieval. `hybrid_search` fuses FAISS and BM25 candidates using weighted RRF, and `retrieval_detail` performs token-budgeted, diversity-aware assembly into `RetrievalChunk`s. Source slices become `SourceEvidence`; gists and summaries become `SemanticPrimer`. Trace JSON records the entire funnel from policy and global-scan diagnostics through final evidence counts.






Block 8 Final Set Summary — Summary, Gist, Meta-Summary, and Continuity Flow

This last set completes the summary/gist/meta-summary side of Block 8.

The major picture:

Source files / file context surface
  → meta_summary builds canonical semantic summaries
  → meta_summary_artifact writes summary + state/freshness metadata
  → summary_service materializes summaries as prompt/system blocks
  → file_summary_queue mirrors fresh semantic file summaries into quick gists
  → VectorMemoryService ingests gists as SemanticPrimer rows
  → retrieval/lattice may use them as support-only context, not proof

The key conclusion:

The codebase now has a fairly strict separation between source evidence and summary/primer/continuity material. Source evidence comes from actual source slices or scan excerpts. Summaries, quick gists, inventory blocks, semantic meta summaries, and conversation rollups are support context.

1. meta_summary.* — canonical semantic summary producer

Files:

core/cle/summary/meta_summary.cpp
core/cle/summary/meta_summary.h

This is the main canonical semantic summary builder.

It produces summaries for:

File nodes
Folder nodes
Root/project node

using the locked schema:

## Responsibilities
- ...

## Key symbols
- ...

## Notes
- ...

The schema is hard-validated. No code fences, no paragraphs, no numbered lists, no arbitrary headings.

What it does well
Stable surface wait

Before building, it waits for the file context surface to stabilize:

tracked count
included count
pending count
surface fingerprint

This is important. It prevents building summaries while file ingest/rescan is still changing.

Source-stamped freshness

For every node, it builds a list of source stamps:

File node:
  raw source file stamp

Folder node:
  child folder summary stamps
  child file summary stamps
  raw source stamps for files under folder

Root node:
  top folder summary stamps
  root file summary stamps
  all raw source stamps

So folder/root summaries are invalidated when either their direct source files or dependent child summaries change.

Deterministic repair

The LLM only really supplies the Responsibilities section. The code deterministically injects:

Key symbols
Notes
graph.node
graph.includes_project
graph.includes_external
graph.contains
graph.depends_on_folders
nav.next
nav.start_here
graph.entrypoints

That is very strong. It means summaries are not fully model-authored free text. They are model-assisted but schema-repaired and graph-enriched.

Bottom-up graph summary

The build order is:

files first
folders deepest-first
root last

That is the correct hierarchy. Folder summaries can consume child file/folder summaries; root can consume top folder summaries.

Runtime-profile aware

The summary LLM path resolves:

effective profile
model name
provider
context window
max output tokens
hard output caps

Then builds a deterministic Agent::Request with:

temperature = 0
RequestKind::MetaSummary

Good.

Changed summary triggers downstream refresh

When semantic artifacts change, it does:

InvalidateGraphCacheBestEffort
ScheduleInventoryRefreshBestEffort

This is important because semantic changes should flow into inventory/gist surfaces.

2. meta_summary_artifact.* — freshness authority and artifact state

Files:

core/cle/summary/meta_summary_artifact.cpp
core/cle/summary/meta_summary_artifact.h

This is the strongest part of the summary layer.

It owns:

summary path
state path
compat mirror path
source stamps
surface fingerprint
output fingerprint
atomic writes
staleness checks

Each node has both:

<node>.summary.md
<node>.state.json

State tracks:

kind
key
promptVersion
fingerprint
surfaceFingerprint
builtAt
builtAtNs
newestSourceMtime
sources[]
Freshness checks

A summary is stale if:

missing output
missing/bad state
kind/key mismatch
prompt version changed
surface fingerprint changed
source stamps changed
newer source seen
summary artifact older than source
schema invalid
output fingerprint mismatch

This is exactly the kind of “summary is not proof unless state says it is current” system you need.

Important note in code:

builtAtNs is metadata only
do not compare builtAtNs directly to filesystem mtimes

That avoids Windows clock epoch issues. Good hardening.

Artifact refresh even when text is identical

This is subtle but correct:

If source stamps or state changed, rewrite/touch the markdown artifact even when generated text is byte-identical.

Why it matters:

Inventory and system-block readers use artifact mtime as a freshness sanity gate.

This fixes a common freshness bug where content is unchanged but the summary still needs a refreshed state/mtime.

3. summary_keys.* — stable key families and namespaces

Files:

core/cle/summary/summary_keys.cpp
core/cle/summary/summary_keys.h

This is the key namespace contract.

Main stable families:

__inventory__
__inventory__/all
__inventory__/all.json
__inventory__/all/sections/

files/<rel>
__folders__
__folders__/<rel>
__quick__/file/<rel>
__conversation__/facts

__meta__/v1/project
__meta__/v1/files/<rel>
__meta__/v1/folders/<rel>

__gists__/
__slices__/
__counters__/
__gists__/files/<rel>   legacy

The important function:

SummaryKeys::Classify(...)

It separates:

RepoPath
SemanticProjectV1
SemanticFileV1
SemanticFolderV1
Inventory
InventoryAllJson
ProjectSection
ConversationFacts
FolderGist
FileGist
QuickFileGist
LegacyFileGist
StableOther

And:

SummaryKeys::Canon(...)

canonicalizes only the repo-path suffix for file/folder-like stable keys, while preserving the namespace.

This is exactly what you need to prevent stable keys from being accidentally normalized into repo paths.

Good design principle:

normal repo path:
  src/a.cpp

stable summary/gist key:
  files/src/a.cpp
  __quick__/file/src/a.cpp
  __meta__/v1/files/src/a.cpp

They must not be confused.

4. summary_paths.* — deterministic artifact path map

Files:

core/cle/summary/summary_paths.cpp
core/cle/summary/summary_paths.h

This centralizes paths for:

semantic summaries
semantic state
conversation summaries
inventory summaries
quick summaries
legacy quarantine

Important canonical locations:

summaries/__meta__/v1/...
summaries/__conversation__/...
summaries/__inventory__/...
summaries/__quick__/file/<rel>.quick.md
summaries/__legacy__/...

Good point: the path layer uses deterministic paths and avoids scanning.

One notable issue appears later: file_summary_queue.cpp writes quick sidecars under:

summaries/__quick__/<rel>.quick.md

while summary_paths.cpp says the canonical quick path is:

summaries/__quick__/file/<rel>.quick.md

The older path is treated as compat by QuickFileCompatPath. This is probably intentional compatibility, but the queue/header should be checked because the key it ingests is canonical __quick__/file/<rel>, while the disk mirror is compat-style.

5. file_summary_queue.* — retrieval-triggered quick summary mirror

Files:

core/cle/summary/file_summary_queue.cpp
core/cle/summary/file_summary_queue.h

This queue is triggered by retrieval when source files are used.

But the implementation has changed meaningfully from the header comment.

The header says it is for “real-time file summarization” and mentions light LLM summary. The implementation now enforces a strict mirror-only contract:

never generate summaries here
never treat quick sidecars as authoritative input
only mirror state-validated semantic/meta-produced summaries into quick artifacts

That is a good architectural change.

Actual behavior

On enqueue:

normalize rel path
dedupe by project/file inflight key
only process included files
resolve Project* by projectId at job time

On job:

check file is still included
load fresh authoritative semantic file summary
validate source stamp and Phase-10 schema
ingest it into VectorMemoryService as __quick__/file/<rel>
write quick sidecar mirror

If no fresh semantic summary exists:

skip
ownership=file_context_meta_summary

This is good because quick gists are now mirrors of canonical semantic summaries, not independent summary sources.

Key finding

FileSummaryQueue is no longer a summary generator. It is a semantic-summary mirror/ingest queue.

That reinforces the proof boundary:

canonical semantic summary -> quick gist mirror -> vector memory primer

not:

retrieval saw file -> generate new proof summary
6. gist_service.* — lightweight gist + inventory prompt block reader

Files:

core/cle/summary/gist_service.cpp
core/cle/summary/gist_service.h

This file has two roles:

1. Lightweight local file gist generation
2. Inventory summary prompt block builder
summariseFile

This is local heuristic only:

take first 40 lines / max 1200 chars
normalize plain text
no LLM dependency

It is not the canonical semantic summary path.

BuildInventoryBlock

This builds an “Inventory summaries” block from fresh artifacts.

Preference order:

Project:
  fresh semantic project overview
  fallback to fresh inventory project overview

Focused folder:
  fresh semantic folder overview
  fallback to fresh inventory folder overview

It does freshness checks against source mtimes and semantic state. It also validates Phase-10 schema for semantic summaries.

This is support context. It should be treated as inventory/semantic primer, not source evidence.

7. summary_service.* — conversation rollups + prompt system block materialization

Files:

core/cle/summary/summary_service.cpp
core/cle/summary/summary_service.h

This file has two major concerns:

conversation roll-up continuity
deterministic prompt/system summary block materialization
Conversation rollup

The rollup system:

watches conversation pair counts
keeps tail pairs
archives old user/assistant turns to JSONL
summarizes pair blocks
replaces old tail pairs with summary nodes
writes rollup artifacts
updates assistant facts
ingests conversation facts into VectorMemory

Key concepts:

keepTailPairs
rollupBlockPairs
maxSummaryTokens
head coverage
summary_block from/to metadata
assistantOrdinalBase
conversation archive JSONL

This is continuity, not source evidence.

It preserves recent tail while condensing older conversation. The rollup artifact can be materialized later as:

### Conversation Rollup
...
Assistant facts gist

updateAssistantFacts writes:

summaries/__conversation__/facts.md
summaries/__conversation__/facts.summary.md

and ingests:

SummaryKeys::kConversationFacts

into vector memory.

This gives retrieval/lattice access to conversation facts, but again it is continuity/support memory.

System summary block materialization

MaterializeSystemSummaryBlocks can produce:

ConversationRollup
ProjectMeta
ProjectSummary
FolderSummary
FileSummary

These are intended as extraSystemBlocks for prompt packing.

Actual implementation:

ProjectSummary:
  semantic project summary
  fallback inventory project meta

FolderSummary:
  semantic folder summary
  fallback inventory folder summary

FileSummary:
  semantic file summary only

ConversationRollup:
  disk rollup artifact
  fallback summary nodes from tail

ProjectMeta:
  deterministic project/thread/focus metadata

Important mismatch: the header comment says FileSummary prefers:

semantic → quick → inventory/legacy

but the implementation currently only loads state-aware semantic file summary. So that comment is stale or the quick fallback was intentionally removed.

Given your architecture direction, the implementation is probably better than the comment: file system-block summaries should prefer canonical semantic state, not stale quick mirrors.

8. summary_queue.* — deterministic conversation summary work queue

Files:

core/cle/summary/summary_queue.cpp
core/cle/summary/summary_queue.h

This is a pure queue. It does no I/O and calls no higher layers.

It stores work per:

Project*
Conversation ID
Range of pair IDs
Reason

Reasons:

AutoPairs
AutoTokens
ManualButton
ManualCondense

Strengths:

coalesces overlapping/adjacent ranges
keeps highest-priority reason
stable sorted conversation IDs
round-robin project-wide dispatch
deterministic PopProjectWide order

This is a solid deterministic queue for conversation rollups.

One minor architectural caution: it keys project state by raw Project*. That is probably fine for the current app lifetime, but if projects can be destroyed/reopened, it is something to watch.

9. summary_worker.* — older/current worker helpers

Files:

core/cle/summary/summary_worker.cpp
core/cle/summary/summary_worker.h

This file still contains:

summarizeFile
summarizeConversationPairs
File summary worker

summarizeFile:

resolves project/active profile
extracts includes
extracts symbols
asks LLM for responsibilities
builds canonical file summary
ingests into VectorMemoryService under files/<rel>

This looks like an older or alternate file-summary path.

In the current summary_service.cpp, scheduleFile routes to:

Summary::ScheduleEnsureMetaSummaries(...)

not directly to SummaryWorker::summarizeFile.

SummaryWorker::summarizeConversationPairs is actively used by the conversation rollup worker.

Potential risk:

SummaryWorker::summarizeFile still exists and still ingests files/<rel> gists. If any old call site still invokes it, it can create a parallel per-file summary/gist path separate from the canonical meta_summary path.

That may be acceptable as a compatibility path, but for the “one true summary pipeline,” it is worth checking reachability.

10. Final source-evidence vs summary/continuity boundary

After all three Block 8 sets, the distinction is now clear.

Source evidence

Proof-bearing source evidence comes from:

retrieved file slices
strict file slices
focused file slices
global lexical scan excerpts
source-led scan evidence injections
coverage backstop file slices

Usually represented as:

ItemKind::Slice
EvidenceClass::SourceEvidence
plane/retrieval/slice|...
SOURCE EVIDENCE — CONTENT VISIBLE

These can support implementation claims.

Semantic primer / summaries

Support-only or orientation context comes from:

semantic file/folder/root summaries
inventory summaries
quick file gists
file gists under files/<rel>
folder gists under __folders__
conversation facts
memory seed summaries
project/folder/file system summary blocks

Usually represented as:

ItemKind::Gist
EvidenceClass::SemanticPrimer
high-bit gist id
stable summary key

These should not prove implementation details by themselves.

Continuity

Continuity comes from:

conversation tail
conversation rollups
summary nodes
assistant facts
user context
recent mutation planes
plan/apply/decision planes
project meta
thread/focus state

Continuity can route, orient, and preserve workflow, but it is not proof.

Best wording:

Source evidence proves.
Semantic summaries orient.
Continuity remembers.
Inventory describes the surface.
11. Strongest architectural improvements in this set
1. Semantic summaries are now stateful artifacts

They are not loose markdown files. They have state, source stamps, schema validation, fingerprints, and prompt versioning.

2. Quick summaries are no longer authoritative

file_summary_queue now only mirrors fresh validated semantic summaries. That is a big improvement.

3. Summary readers are fresh-only

summary_service and gist_service repeatedly check freshness before reading summary artifacts. Stale summaries are skipped rather than silently packed.

4. Summary output is schema-constrained

The Phase-10 schema gives downstream components predictable structure.

5. Summary changes trigger inventory refresh

Good dependency flow:

semantic summary changed
  → graph cache invalidated
  → inventory refresh scheduled
  → vector/gist surface eventually updated
12. Main risks / cleanup targets
1. Header comments are stale in several places

Examples:

file_summary_queue.h says worker runs light LLM summary.
Implementation says mirror-only and never generate summaries.

summary_service.h says FileSummary prefers semantic → quick → inventory/legacy.
Implementation only uses semantic file summary.

summary_service.cpp header says no legacy summary path/layout fallbacks.
Some comments still mention legacy fallback, but implementation mostly removed them.

These should be cleaned because they will mislead future review.

2. Phase-10 schema validation is duplicated

The same validation logic appears in multiple files:

file_summary_queue.cpp
gist_service.cpp
meta_summary.cpp
summary_service.cpp

This should ideally become one shared helper.

Not urgent, but duplication can drift.

3. Quick sidecar path mismatch

summary_paths.cpp says canonical quick path is:

__quick__/file/<rel>.quick.md

file_summary_queue.cpp writes:

__quick__/<rel>.quick.md

but ingests vector key:

__quick__/file/<rel>

This may be intentional compat behavior, but it should be made explicit. Either write canonical quick sidecar too, or update comments to say the disk sidecar is compat-only while the vector key is canonical.

4. Potential parallel file-summary worker path

SummaryWorker::summarizeFile still generates and ingests files/<rel> summaries. If reachable, it may bypass the stricter meta-summary artifact state.

Check call sites later.

5. summary_service is still a large mixed module

It handles:

artifact reading
conversation rollup
assistant facts
system block materialization
summary scheduling wrappers

It works, but it is another complexity hotspot.

13. Block 8 final mental model

This is the cleanest full Block 8 map:

File context surface
  ├─ raw slices
  │    → VectorMemoryService FAISS/BM25
  │    → Retrieval SourceEvidence
  │
  ├─ semantic summaries
  │    → meta_summary
  │    → meta_summary_artifact state
  │    → SummaryService system blocks
  │    → FileSummaryQueue quick mirrors
  │    → VectorMemoryService gist rows
  │    → SemanticPrimer
  │
  ├─ inventory summaries
  │    → inventory gist builder
  │    → inventory prompt blocks
  │    → SemanticPrimer / inventory facts
  │
  └─ conversation continuity
       → SummaryQueue
       → SummaryService rollup
       → conversation facts gist
       → session memory / continuity plane

And the final rule:

The model can make implementation claims only from packed source evidence.
Summaries/gists/inventory/rollups can help it understand and navigate, but they must not silently upgrade into proof.
14. Carry-forward summary for the final consolidated codebase review
Block 8 covers the context/retrieval/memory/lattice/summary substrate. The summary stack now centers on canonical semantic artifacts generated by `meta_summary` and governed by `meta_summary_artifact` state files. Each file/folder/root summary uses the Phase-10 schema (`Responsibilities`, `Key symbols`, `Notes`) and is validated against source stamps, surface fingerprints, prompt version, output fingerprint, and artifact mtimes before use. `file_summary_queue` no longer generates summaries directly; it only mirrors fresh authoritative semantic summaries into quick sidecars and vector-memory quick gist keys. `summary_service` materializes conversation rollups, project metadata, and semantic/inventory summaries into deterministic system blocks, while `summary_queue` drives deterministic conversation roll-up work.

The important architectural boundary is that semantic summaries, inventory summaries, quick gists, file/folder gists, conversation rollups, and user context are support/primer/continuity material. They orient retrieval and prompt packing, but they are not source proof. Proof-bearing source evidence still comes from source slices, strict file reads, focused file slices, global scan excerpts, source-led scan evidence, and coverage backstop chunks. This means the system has a viable separation between source evidence, semantic primer, inventory facts, and continuity state.
Portfolio-facing wording
Coeus includes a stateful semantic summary layer for repository understanding. File, folder, and project summaries are generated into canonical artifacts with strict schemas, source stamps, prompt-version tracking, output fingerprints, and surface fingerprints. Retrieval-triggered quick summaries are mirror-only: they are populated only from fresh validated semantic summaries and ingested as primer context, not proof. This lets the system use summaries to navigate and orient large codebases while reserving implementation claims for visible source evidence.
Block 12 — Set 1 Analysis and Summary

This first Block 12 set covers the core tools substrate:

non_transcript_tool_api
tool_registry / tool_runner / tool_spec
tool_observation_store
toolchain_registry / toolchain_executor / toolchain_spec
planning helpers:
  plan_target_resolution
  plan_json_extract
  plan_rewrite_fallback
  plan_preflight
  plan_apply_effects
  plan_store_local

Main read: this layer is mostly well-separated. Tools are treated as host-executed operations, not model-executed magic. Traces/activity/run-index are model-invisible. Tool observations are a separate, explicitly pack-eligible seam. Planning helpers are more dangerous because they can call the LLM to synthesize plans or rewrite files, but they have meaningful validation/preflight around that path.

1. Core tool doctrine is correct

tool_spec.h states the core rule clearly:

Tools are HOST-executed.
The model never executes tools directly.
Tool traces are MODEL-INVISIBLE.
UI opens deterministic details_ref.

That is exactly the right boundary. ToolRunner executes ToolHandler(call, ctx) only, enforces tier_min, generates call_id, writes trace JSON, emits activity feed lines, appends run-index entries, and does not call an LLM or pack anything into prompts.

This is important because it keeps tool execution in the runtime/action truth layer, not the claim truth layer.

The clean contract is:

ToolSpec = declared capability
ToolRegistry = name/alias/spec/handler resolution
ToolRunner = host execution + trace/activity/run-index
ToolObservationStore = optional pack-eligible observations
Prompt/lattice = decides whether observations are packed
Model = may describe only packed/validated truth

That separation is strong.

2. ToolRegistry is a good central resolver

ToolRegistry owns tool registration, aliases, and built-in seeding. Built-ins are registered lazily through a single aggregator, then aliases are installed for workspace, repo, artifact, connector, file-context, retrieval, and planning compatibility names.

Good aspects:

- Builtins are ensured idempotently.
- Aliases are central, not scattered.
- ToolFn legacy convenience is wrapped into canonical ToolHandler.
- ListTools returns real tools, not aliases.
- Metadata-only tools are allowed but fail cleanly at runner time if no handler exists.

The alias map is useful, especially for planner compatibility:

write_file       → fc.write_file_v1
fc.write_file    → fc.write_file_v1
repo.patch       → repo.apply_patch_v1
ws.patch         → ws.apply_patch_v1
fc.rescan+embed  → fc.rescan_then_embed_async_v1

This prevents brittle plans from failing immediately because of small naming drift.

Risk: aliases are helpful, but they can also hide planner sloppiness. Long term, the planner should still be trained/validated to emit canonical names. The aliases should remain compatibility safety nets, not the main path.

3. ToolRunner is model-invisible execution, not evidence by itself

ToolRunner writes tool_trace_v1 under the thread trace directory, emits activity start/finish, and appends run-index entries for standalone tools. For toolchain steps, TLS marks the run as a chain step so the UI does not get duplicate top-level host activity.

That is good. It means a tool execution creates:

details_ref
activity line
run-index entry
tool trace JSON
outputs_json
artifact_refs

But none of those automatically become model-visible proof.

This is the correct truth boundary:

Tool trace = audit/debug artifact.
Activity feed = UI/ops artifact.
Tool result outputs = host execution result.
ToolObservationStore block = only explicit pack-eligible prompt material.

The important thing is that ToolRunner itself does not pack. That prevents tool output from becoming ungrounded user-facing claims merely because a tool ran.

4. NonTranscriptToolApi is a safe host-side entrypoint

NonTranscriptToolApi runs tools/toolchains asynchronously without emitting transcript macros. It resolves the active thread, schedules CPU tasks, ensures builtins, and uses CurrentEpoch() instead of BeginTurn().

That is correct for non-transcript operations:

Non-transcript tool run
  → should correlate with current epoch
  → should not begin a new user turn
  → should not mutate transcript by itself

It also returns structured results with ok, call_id, details_ref, activity_line, outputs_json, and artifact_refs.

Risk: the callback is invoked on the worker thread, as the header states. Any UI caller must marshal back to main thread. That is not an architecture flaw, but it must be respected.

5. ToolObservationStore is the key proof boundary

This is one of the most important files in this set.

The header says:

MODEL-INVISIBLE store of pack-eligible observation blocks.
These blocks are the ONLY tool outputs eligible to be packed into the LLM prompt at Tier>=2.
Tool traces remain UI/debug artifacts and MUST NOT be stored here.
ActivityFeed lines MUST NOT be stored here.

That is the exact boundary we need.

It also keys observations by:

pid
thread_id
epoch
call_id

and CollectForEpoch requires strict epoch match. This prevents stale tool results from leaking into later turns.

The block class distinction is also important:

ObsBlockClass::Evidence
ObsBlockClass::Primer

That gives the lattice a way to keep proof-carrying tool output separate from support-only summaries or hints.

Good design:

- In-memory only.
- Bounded to 128 entries per thread.
- Strict epoch collection.
- Marker hints only emitted if the block is actually packed.
- Blocks have plane_hint and class.

This is the right seam for future tool outputs that should become prompt-visible.

6. Toolchains are deterministic macro-tools

ToolChainSpec is data only. It defines ordered steps, args templates, retry count, and continue-on-error. ToolChainExecutor executes the chain deterministically through ToolRunner, writes a toolchain_trace_v1, emits grouped activity, and appends a chain-level run index entry.

The useful feature is $ref argument resolution:

args.some_field
step.<step_id>.outputs.some.path
step.<step_id>.call_id
step.<step_id>.ok
step.<step_id>.activity_line

That gives toolchains deterministic dataflow without allowing the model to freestyle tool execution.

Good:

- Toolchain does not call an LLM.
- Each step still goes through ToolRunner.
- Step activity is grouped under chain run id.
- Standalone tool run-index entries are suppressed for chain steps.
- Chain trace captures capped step outputs.

Risk: $ref resolution is intentionally simple, which is good. Do not let this become a general expression language. Keep it declarative.

7. Planning helpers are the highest-risk part of this set

The planning folder is where this block gets more complex.

Some helpers are deterministic and safe:

plan_json_extract
plan_preflight
plan_store_local
plan_target_resolution
plan_apply_effects

But plan_rewrite_fallback can call the LLM to synthesize file contents or multi-file rewrite plans. That is powerful, but it must stay tightly guarded.

The current safety shape is decent:

- Explicit update goals are detected from mutation cues + file-like mentions.
- Target relpaths are normalized and resolved against included files.
- Existing target contents are read under root with root-containment checks.
- Single-file fallback produces fc.write_file_v1 overwrite plans.
- Multi-file fallback validates target parity.
- Preflight rejects ask_user for /apply.
- Preflight rejects duplicate mutation targets.
- Preflight requires write text for write tools.
- Preflight requires unified diff payload for patch tools.

This is strong, but it still has one conceptual risk:

Planner fallback is allowed to generate executable mutation content.

That is not wrong, but it means the validation/preflight layer must remain strict. The fallback must never become a “just make something up and apply it” path.

8. plan_target_resolution is practical but heuristic

plan_target_resolution collects included relpaths from project file context, normalizes planning relpaths, detects explicit update goals, extracts file-like mentions, resolves mentions against included rels, reads target files under root, and builds planner prompts.

Good safety:

- URL-ish paths rejected.
- Absolute/root paths rejected.
- `..` segments rejected.
- File reads check containment under root.
- Read cap enforced.
- Target context is explicit.

The heuristic target detection is useful but should not become canonical turn truth. It is fine for planning/tool execution assistance. The canonical answer target still belongs upstream in resolved turn state.

9. plan_preflight is a necessary hard gate

PreflightStoredPlanForApply is exactly the kind of guard /apply needs.

It blocks:

- empty plans
- non-object steps
- missing action kind
- ask_user steps
- missing tool names
- invalid repo-relative targets
- duplicate mutation targets
- write steps without text
- bad write modes
- patch steps without unified diff
- action JSON that RunController cannot parse

This is one of the main controls preventing a stored plan from becoming unsafe or vague at apply time.

Important: this file validates plan executability, not semantic correctness. That is the correct scope.

10. plan_apply_effects correctly separates mutation truth from planning hints

This file is especially important for recent-mutation continuity.

It explicitly says only explicit mutation transport surfaces contribute to recent-mutation truth, and that working_set_rels and planned_mutation_target_rels are continuity/debug anchors, not evidence that a mutation actually executed.

That directly protects against one of the earlier failures:

planned target ≠ executed mutation
working set ≠ mutation proof
debug anchor ≠ authoritative recent mutation

Good accepted mutation surfaces include:

mutated_files
written_files
updated_files
created_files
touched_files
successful_mutation_target_rels
executed_mutation_target_rels
authoritative_mutation_target_rels
apply_debug_successful_mutation_targets
apply_debug_authoritative_applied_mutation_targets

This is the right direction. Mutation truth should come from execution feedback, not planner intention.

The fallback path MergeActionJsonRecentMutationFallback still infers mutation for write/patch-like action JSON, but only when execution context is being aggregated. That is acceptable as a fallback, but the preferred route should be explicit tool output surfaces.

Architectural verdict for this set

This set is mostly healthy.

The key architecture is:

Tools execute in host/runtime.
Tool traces and activity are model-invisible.
Only ToolObservationStore blocks can become prompt-visible.
Toolchains compose tools deterministically.
Planning can synthesize plans, but apply/preflight/effects constrain execution truth.
Recent mutation truth is based on executed mutation surfaces, not planning intent.

That aligns with the broader CLE doctrine:

execution truth ≠ prompt truth
tool trace ≠ proof
activity feed ≠ evidence
planned mutation ≠ applied mutation
observation block included by lattice = potential packed truth
Main risks to carry forward
Risk 1 — Planner fallback can become too powerful

plan_rewrite_fallback calls the LLM to generate full updated file contents and executable write plans. This is useful, but it must stay under strict explicit-target and preflight constraints.

Carry-forward rule:

LLM-generated plans are proposals until preflight + RunController execution succeeds.
Risk 2 — Tool aliases can hide canonical-name drift

Aliases are good for compatibility, but planner prompts should still require canonical tool names.

Carry-forward rule:

Aliases are runtime tolerance, not planner truth.
Risk 3 — Tool outputs must not become claims unless packed

The architecture already handles this well, but it is the biggest future risk.

Carry-forward rule:

Tool result JSON, trace refs, and activity lines are not model proof.
Only packed ToolObservationStore blocks can become prompt-visible support/evidence.
Risk 4 — Mutation fallback from action JSON should remain secondary

MergeActionJsonRecentMutationFallback is useful, but explicit execution outputs should outrank action shape.

Carry-forward rule:

executed mutation feedback > action JSON fallback > planned targets.
Set 1 conclusion

Block 12 set 1 establishes a solid tool substrate.

The tool layer is not currently acting like an ungrounded oracle. It is mostly a deterministic host execution system with model-invisible traces and a separate observation seam for pack-eligible outputs. The planning layer is the main place to watch because it mixes LLM synthesis with executable file mutation, but the preflight, target-resolution, and recent-mutation aggregation rules are much better than the older shape.

The core principle is preserved:

Tools can act.
Tools can leave traces.
Tools can emit observations.
But tools do not automatically become user-facing truth.
Carry-forward summary
Block 12 set 1 covers the core tools substrate. ToolSpec/ToolRegistry/ToolRunner are correctly host-executed and model-invisible: tools are registered as specs+handlers, aliases resolve centrally, ToolRunner enforces tier_min, generates call_id, writes tool_trace_v1, emits ActivityFeed, appends RunIndex, and does not call the LLM or pack anything into prompts. NonTranscriptToolApi runs tools/toolchains asynchronously against CurrentEpoch without BeginTurn, preserving non-transcript semantics. ToolObservationStore is the key proof seam: only its epoch-keyed observation blocks are pack-eligible at Tier>=2; tool traces and activity lines must not be packed. Toolchains are deterministic macro-tools composed of ToolRunner calls with simple $ref dataflow and grouped step activity. Planning helpers are higher risk: target resolution and preflight are deterministic and useful; rewrite fallback can call the LLM to synthesize file contents/plans but is constrained by explicit target context, target parity, and apply preflight. plan_apply_effects is strong because it treats planned targets and working sets as anchors/debug only, while recent-mutation truth comes from explicit executed mutation surfaces.
Portfolio-facing wording
The tool layer is a deterministic host-execution substrate: tools run outside the model, traces and activity are model-invisible, toolchains compose host tools with explicit dataflow, and only explicitly packed observation blocks can become prompt-visible. This keeps execution, evidence, and user-facing claims separate.










Block 12 — Rest Analysis and Summary

This second Block 12 set covers the builtin tools, builtin toolchains, repo/file-context/workspace hands, retrieval/search/index/inventory/summary tools, artifact access, network connector, and connector secret boundary. The broad result is good: the builtin surface mostly preserves the distinction between host action, trace/debug output, recent mutation truth, retrieval freshness, and model-visible evidence.

1. Builtin registration shape is clean

builtin_tools.cpp gives one central registration entrypoint:

RegisterBuiltinTools(ToolRegistry& R)

and registers modules in deterministic order:

core
filectx
filectx_tools
filectx_user_ops
plan
filectx_write
search / retrieval / inventory / summary / index / toolchains
ws / net / repo / artifact

That is a good shape. It keeps ToolRegistry product-agnostic and avoids scattered builtin seeding.

Main note: registration order matters because aliases can overwrite earlier aliases. There are several repeated aliases like fc.scan, fc.embed_pending, fc.status, etc. That is not necessarily wrong, but alias ownership should be intentional. The later registration currently tends to make the more canonical tool win, which is fine, but should remain documented.

2. File-context tools are split into three roles

The file-context builtins are not one blob; they have different operational meanings.

Maintenance/status tools

Examples:

fc.scan_v1
fc.embed_pending_v1
fc.status_basic_v1
fc.status_v1
fc.ingest_progress_v1
fc.cancel_ingest_v1
fc.scan_folder_async_v1
fc.embed_pending_async_v1
fc.rescan_file_async_v1
fc.rescan_then_embed_async_v1
fc.list_files_v1
fc.read_file_v1
fc.get_abs_path_v1
fc.set_files_included_v1
fc.set_files_attached_v1

These are mostly retrieval/index/file-context maintenance tools.

Good: builtin_fc.cpp explicitly avoids emitting recent_mutation_present=false for scan/embed/status, because maintenance completion should not clear previous /apply mutation continuity. That was one of the exact failure modes we were worried about.

This is correct:

scan/embed/status = retrieval freshness / context maintenance
not mutation truth
not recent mutation absence proof
User-facing file-context ops

Examples:

fc.set_root_v1
fc.add_files_v1

fc.set_root_v1 has a strong reset posture. replace and clear wipe current file-context items, clear recent file-context anchors, cancel ingest, reset vector surface via FullSyncFromCanonicalSurface, refresh inventory, rebuild file search, and persist state best-effort. That is good because a root change is not just a UI flag; it changes the canonical retrieval surface.

fc.add_files_v1 is designed for chat-ready usage: it can cache bounded file contents, set thread anchors, update recent filectx rels, schedule auto-ingest, and persist file-context/project context. This is useful, but it is intentionally file-context continuity, not recent mutation truth.

Mutation tool
fc.write_file_v1

This is the important one. It writes under the active file-context root, updates/creates the FileContextItem, seeds contents immediately, marks it pending, emits before/after artifacts, schedules auto-ingest, and returns explicit recent mutation payload fields.

Good mutation outputs include:

recent_mutation_present = true
recent_mutation_files
recent_mutation_primary_rel
recent_mutation_source = file_write
recent_mutation_prior_version_artifact_exists
recent_mutation_exact_diff_artifact_exists
recent_mutation_structural_summary_available
effects.recent_mutation
active_hint_rel
working_set_rels

That gives downstream layers a real executed mutation surface instead of scraping generic fields like path.

3. Repo tools are strong, but one inconsistency should be watched

Repo tools are:

repo.read_v1
repo.write_v1
repo.apply_patch_v1
repo.list_v1
repo.mkdir_v1

Good:

- repo paths are resolved under fileContextRootFolder if set, else project folder.
- absolute paths, drive roots, URL-ish paths, and traversal are rejected.
- reads are bounded.
- writes are atomic.
- write/patch support expected_digest preconditions.
- patch application is strict unified-diff single-file logic.
- write/patch emit before/after/patch artifacts.
- write/patch emit explicit recent_mutation effect blocks.
- successful mutations notify file-context auto-ingest.

The repo mutation tools are doing the right thing: they produce explicit mutation truth and freshness handoff.

The inconsistency: non-mutating repo tools call setNoRecentMutation_(), which emits:

recent_mutation_present = false
recent_mutation_files_count = 0
recent_mutation_bundle = false
recent_mutation_source = none

That conflicts a little with the newer file-context maintenance posture, where absence fields are deliberately not emitted to avoid clearing prior mutation continuity.

Maybe this is safe if downstream extraction only treats explicit mutation surfaces as additive and never treats false as a clearing instruction. But based on our earlier failure family, I would flag this.

Recommended doctrine:

Read/list/status tools should generally omit recent_mutation fields entirely.
Only mutation tools should emit recent_mutation=true.
Clearing mutation continuity should require a dedicated clear event, not absence fields.

This applies to repo.read_v1, repo.list_v1, repo.mkdir_v1, and similarly ws.* non-mutation tools.

4. Workspace tools are correctly quarantined from repo truth

Workspace tools operate under .cle_workspace:

ws.read_v1
ws.write_v1
ws.mkdir_v1
ws.apply_patch_v1
ws.list_v1

Good design point: workspace writes do not masquerade as repo mutations unless the workspace file actually maps under the current file-context root.

That is exactly the right policy:

workspace mutation ≠ repo mutation
unless mapped into active file-context root

When mapped, ws.write_v1 / ws.apply_patch_v1 trigger auto-ingest and emit recent mutation against the file-context relpath. When not mapped, they emit no repo mutation truth.

Same caution as repo tools: setNoRecentMutation_() is explicit absence. I would prefer omission for non-mutating or non-mapped workspace actions unless there is a dedicated downstream rule that ignores false as a clear.

5. Plan tools are now the orchestration seam, not the heavy logic bucket

builtin_plan.cpp is still large, but the comment says the heavy planning logic has moved into core/tools/planning/*. That is the right direction.

plan.make_v1 does:

- collect planning context
- detect/augment explicit targets
- read explicit existing targets when required
- build deterministic Tier-3 planner request
- extract/parse planner JSON robustly
- canonicalize action/tool names and args
- use rewrite fallback for explicit update failures
- validate explicit target coverage
- preflight stored plan for apply
- save immutable + last plan

plan.apply_v1 does:

- load last plan
- canonicalize stored action names/args
- require confirmation for guarded steps
- preflight full plan
- execute via RunController
- aggregate anchors and recent mutation
- finalize run trace/run index
- emit authoritative mutation bundle truth

The most important improvement is multi-file mutation handling. plan.apply_v1 computes bundle scope from:

planned mutation targets
executed/aggregated recent mutation files
working-set anchors

Then if that scope is multi-file, it suppresses single-file primary promotion:

no active_hint_rel
no recent_mutation_primary_rel
primary_rel_suppressed = true
recent_mutation_multi_file_bundle = true

That directly protects against the old “bundle collapsed into one preferred primary” problem.

This is a high-value repair.

6. Toolchains are useful deterministic operator paths

builtin_toolchains.cpp registers several practical chains:

fc.ingest_v1
fc.ingest_pending_v1
fc.add_files_and_ingest_v1
fc.rescan_then_embed_v1
fc.write_then_ingest_v1

Good distinction:

fc.ingest_v1 = legacy scan-only async
fc.ingest_pending_v1 = sync embed pending + best-effort semantic refresh + graph status + status
fc.write_then_ingest_v1 = write + sync embed + semantic refresh + graph status + status

That gives you an explicit “finish it now” path separate from background auto-ingest.

This matters because auto-ingest is helpful, but tests/evals often need deterministic completion. A chain like fc.write_then_ingest_v1 is a good eval/operator tool because it makes the mutation + retrieval freshness sequence explicit.

7. Search/retrieval tools are diagnostic, not proof by default

Search/retrieval tools:

project.search_v1
retrieval.probe_v1
inv.snapshot_v1
sum.ensure_meta_v1
sum.read_project_summary_v1
idx.graph_status_v1
idx.graph_rebuild_v1

These are mostly diagnostics and maintenance.

Good:

project.search_v1 returns bounded snippets.
retrieval.probe_v1 returns file-context counts plus top dense/hybrid hits.
inv.snapshot_v1 returns included-domain inventory preview.
sum.ensure_meta_v1 schedules semantic summary maintenance.
idx.graph_status_v1 / idx.graph_rebuild_v1 return stats only, not large graph dumps.

This means they are safe operational tools, but their outputs should not automatically become answer proof unless routed through ToolObservationStore or explicitly packed as evidence/primer.

Important distinction:

search tool result = diagnostic output
packed retrieval plane = evidence candidate
tool trace = audit

Do not let project.search_v1 output become a user-facing claim unless the prompt/lattice explicitly includes it.

8. Artifact tool is fine as a bounded trace reader

artifact.read_v1 reads thread artifacts by ref with offset and max bytes, returns metadata, text preview, and base64. It validates refs through ArtifactStore::ReadBytes.

This is useful for UI/debug and for follow-up tool workflows.

Again, artifact read is not automatically proof. It is a way to inspect an artifact. If the model is going to claim from artifact contents, the content must be packed or otherwise explicitly surfaced.

9. Net connector has the right security posture

net.fetch_v1 is host-owned and connector-gated:

- ConnectorRegistry must enable net.fetch_v1.
- Host must be allowlisted.
- Token bucket rate limit is enforced per tool|host.
- Dangerous headers are filtered.
- Optional auth is injected through SecretStore using auth_secret_key.
- Response body is capped.
- Full response is saved as a trace artifact.

This is a good boundary. The model does not get arbitrary network access merely because a tool exists.

ConnectorRegistry defaults to disabled / deny-all. SecretStore is in-memory only, sorted key listing, no persistence. That is the correct early-stage security posture.

One concern: SecretStore::ListKeys() exists. That is okay for UI/admin use, but model-facing tools should not expose secret keys unless deliberately filtered. The current net tool uses a secret by key but does not expose the secret value, which is good.

10. Main architectural risks
Risk A — explicit recent_mutation_present=false may reintroduce clearing bugs

builtin_fc.cpp has comments saying scan/status/embed should not emit false mutation absence because downstream may interpret it as clearing previous /apply mutation continuity.

But repo.read_v1, repo.list_v1, repo.mkdir_v1, and ws.* no-op/non-mapped cases do emit false mutation absence.

This is the biggest thing I would audit next.

Preferred rule:

Only mutation tools emit recent_mutation fields.
Non-mutating tools omit the recent_mutation namespace.
Dedicated clear tools may emit explicit clear semantics.
Risk B — alias overwrites need ownership

fc.scan, fc.embed_pending, fc.status, etc. are registered in multiple places. The final alias probably points to the intended canonical tools due to registration order, but this should be made explicit.

Preferred rule:

One canonical alias owner per alias.
Compatibility aliases can exist, but alias overwrites should be logged or avoided.
Risk C — repo root uses fileContextRootFolder when set

This is probably intentional, and comments say external roots are allowed. But it means repo.* is really “active repo/file-context root” rather than immutable project folder.

That is fine, but naming/documentation matters.

Preferred wording:

repo.* operates under active code root:
fileContextRootFolder if set, else Project::folderPath.
Risk D — search/retrieval tools could be mistaken for proof

The code mostly prevents this by keeping tool traces model-invisible. But if future layers pack raw tool outputs casually, this can drift.

Preferred rule:

Diagnostics are not evidence unless admitted into a proof-carrying plane.
Architecture verdict

This part of Block 12 is strong.

The builtin surface gives you practical “hands”:

file-context hands
repo hands
workspace hands
planning hands
toolchain hands
retrieval/search/index/summary diagnostics
network connector
artifact reader

And most of them preserve the key truth boundary:

host action ≠ user-facing claim
tool trace ≠ evidence
activity line ≠ proof
maintenance ≠ mutation
planned target ≠ applied target
workspace mutation ≠ repo mutation

The highest-value positive is plan.apply_v1’s bundle handling. It explicitly prevents multi-file applies from collapsing into a fake single-primary file. That is exactly the kind of repair we needed.

The highest-value concern is the remaining explicit recent_mutation_present=false emitted by some non-mutating tools. I would standardize that away unless downstream is proven to ignore it.

Carry-forward summary
Block 12 rest covers builtin tools, builtin toolchains, repo/file-context/workspace hands, retrieval/search/index/inventory/summary tools, artifact access, network connector, and connector secret boundary. The builtin registration surface is centralized through RegisterBuiltinTools with deterministic module order. File-context tools are split into maintenance/status, user-facing root/add operations, and mutation via fc.write_file_v1. Maintenance tools correctly avoid emitting recent-mutation absence fields; fc.write_file_v1 emits explicit executed mutation payloads, before/after artifacts, active_hint_rel, working_set_rels, and auto-ingest handoff. Repo tools are safe and bounded under active code root, with write/patch emitting explicit mutation effects and artifacts. Workspace tools are quarantined under .cle_workspace and only become file-context mutation truth if the workspace file maps under the active file-context root. plan.make_v1 and plan.apply_v1 now act as orchestration seams around planning helpers, with robust JSON extraction, canonical tool/action normalization, explicit-target rewrite fallback, preflight, deterministic RunController execution, and strong multi-file mutation bundle handling that suppresses fake primary promotion. Toolchains provide deterministic operator paths such as write_then_ingest and ingest_pending. Search/retrieval/index/summary tools remain diagnostic/maintenance surfaces, not proof unless explicitly packed. Net connector is appropriately disabled-by-default, allowlist/rate-limit guarded, and uses in-memory SecretStore for host-owned auth injection.
Most important follow-up
Audit and standardize recent_mutation absence semantics. Some file-context maintenance tools deliberately omit recent_mutation_present=false to avoid clearing prior mutation continuity, but repo/ws non-mutating paths still emit false absence fields. Prefer: mutation tools emit recent_mutation=true; non-mutating tools omit the namespace; only an explicit clear operation can clear mutation continuity.
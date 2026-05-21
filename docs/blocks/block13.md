Block 13 — Analysis and Summary

Block 13 is the telemetry/eval/debug spine. It is not part of the model’s authority surface by default. Its job is to make every turn, tool call, plan/apply run, lattice pack, retrieval trace, marker truth, and postprocess decision inspectable after the fact without accidentally feeding private/debug artifacts back into model-visible evidence.

1. ActivityFeed: UI-visible, model-invisible progress stream

activity_feed.* is a compact event feed for the UI. It writes JSONL to:

traces/threads/<thread>/runs/activity.jsonl

and keeps an in-memory bounded ring per (pid, thread).

Important properties:

- model-invisible
- user/UI-facing
- one-line sanitized text only
- async disk write through TaskEngine IO lane
- per-thread serialization key
- warm-loadable from disk
- no UI folder scanning required

This is good because it gives the user/operator a “terminal-like” progress view without making activity lines proof. Activity lines are status, not evidence.

2. ArtifactStore: content-addressed private artifacts

artifact_store.* is a thread-scoped artifact helper under:

runs/artifacts/fnv1a64_<hex>.<ext>

It is content-addressed, atomic, idempotent, exact-ref-only, and rejects traversal/unsafe filenames.

The key doctrine is in the header:

Artifacts are MODEL-INVISIBLE by default.
Tools return only refs/metadata.
No directory scanning.

That is exactly right. It means packed prompts, raw outputs, retrieval debug JSON, lattice traces, tool outputs, and before/after file snapshots can be saved for debugging without becoming accidental model context.

This is one of the most important privacy/eval boundaries in the whole system.

3. RunTrace: Tier-4 agentic run spine

run_trace.* records agentic/Tier-4 runs as:

runs/agentic/run_<run_id>.json

The schema captures:

- pid / tid / epoch_begin / epoch_end
- tier
- run_id
- origin_kind
- caps snapshot
- planner calls
- executed actions
- final output refs/digests
- termination reason/message

The implementation keeps active runs in memory, writes through coalesced async flushes, and drops finalized runs after a successful flush.

This gives you a replay/debug spine for plan.apply, agentic execution, and future task resumes. The important thing is that this spine stores what happened, not what the model may claim.

4. RunIndex: thread-local catalog of runs

run_index.* is the UI’s stable run catalog:

traces/threads/<thread>/runs/run_index.json

It indexes tool, chain, and agentic runs with:

run_id
kind = tool | chain | agentic_run
ok
start/end/duration
activity_line
details_ref
tool_name / chain_name

The key design win: the UI does not have to scan directories. It reads one index file or the warm-loaded in-memory cache.

This is exactly the right pattern for a professional debug UI:

ActivityFeed = live terminal stream
RunIndex = navigable run catalog
RunTrace / ToolTrace / ChainTrace = detail artifacts
5. ToolExporter and tool traces

tool_exporter.* writes per-tool traces to:

runs/tools/tool_<tool_name>_<call_id>.json

The trace includes:

- tool name / call id
- timing
- args digest
- outputs digest
- ok/error fields
- activity_line
- args JSON
- outputs JSON
- artifact refs

It also caps inline args/outputs. If they are too large, it writes the full args/outputs as content-addressed artifacts and replaces the inline JSON with a preview object.

That is a strong design. It gives full forensic/debug visibility while keeping trace files bounded and safe for UI panels.

Important: tool_trace_v1.h explicitly says tool traces are MODEL-INVISIBLE. So a tool result only becomes prompt-eligible through the explicit model_blocks path in tool_result_v1.h, not because a tool trace exists.

That distinction is critical:

outputs_json = debug/trace truth
activity_line = UI truth
model_blocks = pack-eligible model truth
6. ToolCall / ToolResult schemas separate identity from model-visible output

tool_call_v1.h pins invocation identity:

call_id
tool_name
args
args_canon
args_digest
idempotency_key
tier
category
determinism_declared
tool_spec_hash
preconditions

tool_result_v1.h pins outcome identity:

ok/error
tool_impl_id
tool_spec_hash
args_digest
idempotency_key
determinism_actual
snapshot
outputs_json
model_blocks
artifacts
debug_json_ref
activity_line
user_summary

The crucial line is conceptual:

model_blocks are the only pack-eligible outputs.

This is exactly the right boundary. Tool outputs can be huge, private, misleading, or diagnostic. The packer must not dump all outputs_json into the model. It must only admit explicitly constructed model blocks.

7. CLE exporter: per-turn bundle and thread manifest

cle_exporter.* is the main CLE observability exporter. It writes one bundle per turn and a thread manifest.

Typical layout:

.cle_traces/
  turns/
    pidX_tidY_epochZ.json
  threads/
    pidX_tidY.json
  artifacts/
    fnv1a64_<hex>.<ext>

The exporter has two phases:

Stage1:
  writes initial turn bundle, user text, included relpaths,
  retrieval debug JSON, lattice trace JSON, packed prompt markdown,
  policy truth, packed truth, prompt markers, route data.

Finalize:
  merges stage1 prompt markers with finalize markers,
  writes raw/final outputs,
  applies validator/postprocess/runtime/recent-mutation truth,
  updates thread manifest.

This is very strong because it lets you compare:

what the turn thought it was
what was packed
what the model produced
what validator/postprocess changed
what final output was committed

That is exactly what private evals need.

8. Stage1 dedupe is useful but worth watching

Stage1 has identity dedupe based on a fingerprint of bundle + extra payload. It suppresses same-identity pending/written duplicates and supports explicit updates when a newer identity appears for the same (pid, tid, epoch).

That solves a real async export issue: multiple telemetry writes may race for the same turn.

One caution: the identity includes large debug strings like retrieval debug, lattice trace, packed prompt, included relpaths, and user text. That is okay for dedupe, but if high-frequency small changes occur, Stage1 may write multiple explicit updates for one turn. That is probably acceptable for debugging, but eval readers should treat the final bundle as authoritative.

9. Marker truth export is the key eval layer

cle_exporter_marker_truth.* is the core of this block.

It parses prompt markers and transport markers into structured truth:

policy_truth
packed_truth
recent_mutation
validator
runtime_truth
semantic_truth
packed_plane_disambiguation
postprocess report

This is exactly the layer that lets private evals ask:

Did policy classify the turn correctly?
Was the required plane allowed?
Was the plane actually packed?
Was it support-only or authoritative?
Did validator repair the answer kind?
Was this a host command turn?
Was recent mutation real?
Was a multi-file bundle treated as a bundle?
Did final output mode match canonical policy?

The most important exported distinction is per-plane truth:

allowed != packed != support_only != authoritative

That aligns with your broader CLE doctrine. It prevents “the plane was allowed” from being confused with “the plane was packed,” and prevents “supporting context existed” from being promoted into claim authority.

10. Host command disambiguation is handled properly

Marker truth has explicit host-command detection:

slash_command
agent_flow.command_truth_local
host.command_kind
runtime.command_kind
command.kind
validator.runtime_command_kind
route.label = deterministic host command variants

When it detects a host/control turn, it forces the semantic family toward host command behavior:

control_turn = true
repo_answer_turn = false
scope = general_no_repo
repo_mode = no_repo
authority = host_control
requires_source_evidence = false
requires_diff_plane = false
suppress_nmc_by_truth = true

That is very good. It prevents /tool scan, /plan, /apply, etc. from being evaluated as if they were ordinary repo-answer turns.

11. Packed plane disambiguation fixes a subtle drift class

The marker parser distinguishes strict runtime plane markers from legacy fallback markers:

validator.packed_recent_mutation_plane_present
validator.packed_plan_plane_present
validator.packed_apply_plane_present
validator.packed_decision_plane_present

Only if strict disambiguation markers are absent does it fall back to older markers like:

plane.recent_mutation
plane.plan
plane.apply
plane.decision

This matters because legacy plane markers can be ambiguous. Without this, a stale or generic plane.plan=true can be mistaken as proof that the runtime packed the exact plan/apply/recent-mutation plane that the validator expected.

This is a high-value repair.

12. Trace helpers are correctly labelled as non-authoritative

trace.* includes helpers for retrieval/lattice debug persistence and EvidenceHit extraction. The comments repeatedly state:

These helpers are for telemetry/UI/reporting reconstruction.
They are NOT the source of evidence authority.
Upstream retrieval / lattice packing / validator logic owns authority.

That is exactly the right posture. A persisted retrieval trace can help a UI show “what was retrieved,” but it must not become proof that a final answer was grounded.

This protects against a major failure mode:

retrieval trace exists
=> exporter/UI shows it
=> model/validator acts as if it was authoritative evidence

The code’s comments explicitly reject that promotion.

13. Internal artifact suppression matters

This block makes the reason clear.

The system stores many sensitive or misleading-by-context artifacts:

raw model output
final model output
packed prompt markdown
retrieval debug JSON
lattice trace JSON
tool args
tool outputs
before/after file snapshots
plan/apply run traces
activity lines

These are essential for private evals and debugging, but unsafe as automatic prompt input.

Why?

- raw output may contain hallucinations before postprocess/validator repair
- packed prompt may expose private/system internals
- retrieval debug may contain candidates that were not packed or not authoritative
- tool outputs may include large or sensitive data
- activity_line is a UI summary, not proof
- before/after artifacts are audit data, not automatically source evidence
- trace artifacts can be stale relative to final canonical truth

So the correct rule is:

Telemetry is inspectable.
Telemetry is not automatically model-visible.
Only explicit pack-eligible surfaces become model context.

Block 13 largely honors that rule.

14. Main risks / things to audit
A. Marker truth parser is powerful but can become compensatory

cle_exporter_marker_truth.cpp is doing a lot of inference and fallback. That is useful for eval robustness, especially across schema versions, but it must not become a semantic repair engine that hides upstream drift.

Good use:

exporter reconstructs truth for debugging/eval

Bad use:

exporter “fixes” bad policy truth and makes evals look green

Recommendation: evals should compare raw upstream markers against parsed/exported truth, not only consume the final parsed truth story.

B. Legacy marker fallback should keep shrinking

The parser supports many legacy aliases and suffix fallbacks. That is useful now, but long term it can mask stale writers.

Recommended direction:

Keep legacy fallback readable.
But add diagnostics showing which canonical marker was absent and which legacy key supplied the value.

You already have packed-plane disambiguation metadata. Similar diagnostics for answer kind/output mode/recent mutation would help.

C. write_alias_files = true can leak readable raw artifacts

CLE exporter can write alias files like:

*_packed_prompt.md
*_raw_output.txt
*_final_output.txt
*_retrieval_debug.json

This is useful for local debugging, but it is more leak-prone than content-addressed refs. For private eval/dev, fine. For production or shared workspaces, consider disabling aliases by default.

Recommended config posture:

development/eval: write_alias_files=true
production/customer: write_alias_files=false
D. FNV-1a is stable, not cryptographic

FNV-1a is fine for deterministic local artifact identity and eval correlation. It should not be treated as tamper-proof or collision-secure.

That is acceptable here, as long as the language stays “digest/fingerprint,” not “security hash.”

Architecture verdict

Block 13 is strong and important.

It gives you a proper private eval/debug stack:

ActivityFeed: live UI events
RunIndex: thread run catalog
RunTrace: Tier-4 execution spine
ToolTrace: per-tool forensic record
ToolChainTrace: chain-level record
ArtifactStore: private content-addressed blobs
CLE exporter: per-turn truth bundle
MarkerTruth: structured semantic/policy/packed/runtime truth extraction
Trace helpers: retrieval/lattice UI reconstruction

The best part is the repeated model-invisible doctrine. The code largely understands that telemetry is not evidence. That protects the main CLE architecture from a nasty class of bugs where debug artifacts become user-facing claims.

The most important review finding is this:

Telemetry should explain what happened.
It should not decide what was allowed to be claimed.

Block 13 mostly follows that principle.

Carry-forward summary
Block 13 defines the telemetry and private-eval/debug spine. ActivityFeed provides a model-invisible UI progress stream in JSONL plus an in-memory ring. ArtifactStore writes exact-ref, content-addressed, thread-scoped artifacts under runs/artifacts and keeps them model-invisible by default. RunTrace records Tier-4 agentic execution spines with planner calls, actions, caps, final output refs, and termination. RunIndex provides a thread-local catalog of tool/chain/agentic runs so the UI does not directory-scan. ToolExporter writes per-tool traces, caps inline args/outputs, offloads large payloads to artifacts, and preserves the boundary that tool traces are model-invisible. ToolCall/ToolResult schemas separate invocation identity, replay digests, outputs_json, UI summaries, artifacts, and model_blocks; only model_blocks are pack-eligible. CLE exporter writes per-turn bundles and thread manifests in Stage1/Finalize, storing large blobs as artifacts and merging prompt markers across stages. MarkerTruth parses prompt/transport markers into policy_truth, packed_truth, validator, runtime_truth, recent_mutation, semantic_truth, and packed-plane disambiguation. It correctly distinguishes allowed vs packed vs support-only vs authoritative planes, detects host command turns, and prevents legacy plane markers from silently substituting for strict runtime plane truth. Trace helpers persist retrieval/lattice debug snapshots and reconstruct UI evidence rows, but explicitly do not own evidence authority.
Highest-value follow-up
Audit eval readers to ensure they do not treat telemetry artifacts, retrieval traces, activity lines, tool outputs, or alias files as proof. The only valid proof surfaces should be canonical policy/packed/evidence truth and explicitly pack-eligible model blocks. Telemetry should explain and diagnose the turn, not silently become authority for claims.
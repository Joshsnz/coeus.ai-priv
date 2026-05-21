Block 11 — Set 1 Analysis and Summary

This first Block 11 set covers the runtime/orchestration substrate more than final answer validation/postprocess. The validation/postprocess part likely belongs in set 2 unless those files are still coming.

The core runtime flow is now:

AgentFlow::RunTurn
  → PrepareRuntimeTurn
      → TurnContext
      → TurnPlan
      → ResolvedTurnExtras
      → ResolvedTurnState
      → RetrievalOptions
  → deterministic command / missing-target bypasses
  → retrieval
  → keyed context + prompt options
  → PromptBuilder / lattice prompt
  → profile/provider wiring
  → Stage-1 CLE export
  → LLM or agentic loop
  → PostCommit returned
  → EphemeralStateStore applied later

That shape is good. Runtime is acting mostly as an orchestrator and truth transporter, not as a new semantic classifier.

1. agent_flow.cpp is the central runtime seam

AgentFlow::RunTurn is the main turn orchestrator. It prepares the turn, resolves the canonical policy state, mirrors canonical fields into RunResult, handles deterministic command turns, handles missing-repo-target turns, gates retrieval, builds prompt options, calls buildUnifiedPrompt, wires profile/provider settings, exports Stage-1 telemetry, and returns either a prompt request or a served deterministic answer.

The best thing in this file is that it keeps canonical fields explicitly available in RunResult:

answer_kind
effective_output_mode
nmc_allowed
direct_answer_preferred
requires_source_evidence
requires_diff_plane
suppress_needs_more_context_by_truth
answer_contract_reason
canonical_target_file

That gives downstream validation/export/postprocess a stable runtime view instead of forcing them to parse prompt text again.

2. Runtime TURN_STATE transport is being protected from stale truth

The strongest anti-drift mechanism in this set is:

stripAgentFlowOwnedTurnStateLines_
appendCanonicalTransportOverrideLines_
upsertCanonicalTransportMarkers_

buildPromptBuilderTurnStateText_ strips older policy.*, resolved.*, answer.*, recent_mutation.*, mutation.*, and plan.* lines before writing current canonical markers and override lines. This means stale transport truth cannot accidentally outrank the current ResolvedTurnState.

That directly supports the doctrine:

resolved truth owns policy/answer semantics
prompt/runtime transport mirrors it
transport must not become a competing source of truth

This is one of the most valuable controls in the whole block.

3. Retrieval is gated by planner/resolved truth, not by runtime guesswork

applyPlannerRetrievalPolicyToOptions_ uses planner packing policy to set retrieval flags, then applies resolved-turn hard forbids. It disables retrieval/global scan/quick summaries/inventory gist for missing-repo-target turns, inventory-list turns, and conversation-recall authority.

That is correct ownership:

TurnPlan decides retrieval admission shape.
ResolvedTurnState may forbid retrieval for terminal/deterministic answer families.
Runtime only applies those decisions to RetrievalOptions.

Runtime should not decide “this looks source-led” or “this needs retrieval.” It should enforce already-resolved policy.

4. Deterministic slash commands now get explicit local truth

Command turns bypass the normal LLM path, but they no longer bypass truth. buildDeterministicCommandTruth_ creates an explicit ResolvedTurnState for command turns. Inventory commands become InventoryListing; ordinary host-control commands become GeneralChat/NoRepo-style host-control truth. Then markers are appended saying this is local command truth and not reused repo state.

That is exactly the right move. A deterministic command cannot leave the exporter/validator guessing what answer family it was. Even if no prompt was built, there is still canonical runtime truth.

The important subtle fix is recent-mutation preservation on command turns. The code explicitly carries the pre-command recent-mutation snapshot before applying command post-commit effects, so maintenance commands with no mutation effect do not falsely clear the prior /apply bundle.

5. Recent mutation state is separated from working-set convenience

This block clearly fixes an earlier class of bugs: recent-mutation truth is not clipped by the working-set cap. The comment states that working_set is a small convenience/focus surface, while recent_mutation is a truth surface used by recent-update/exporter semantics and must preserve the full batch under a larger safety cap.

Current shape:

working_set
  small focus/convenience surface

recent_mutation
  truth surface for changed-file bundle
  max cap = 64
  preserves multi-file apply batches
  no synthetic single primary for multi-file bundles unless justified

That is important because recent-update answers and bundle coverage depend on the full changed-file set, not on the small UI working set.

6. PostCommit is the state-application boundary

AgentFlow::PostCommit is the transport shape for post-turn state effects: active hint, working set, recent mutation files, primary, preferred primary, source, and artifact flags. AgentFlow::ApplyPostCommit eventually applies it via EphemeralStateStore.

This is good layering:

RunController derives effects.
AgenticRunner / AgentFlow merges effects.
AgentFlow returns PostCommit.
EphemeralStateStore applies state.

The key invariant is: execution layers do not directly mutate thread continuity state. They emit effects. Runtime owns composition. State store owns persistence/application.

7. ActionRequestV1 is correctly host-executed, not model-executed

action_request_v1.* defines the model/host boundary clearly: the model emits an ActionRequestV1; the host parses, validates, and executes it deterministically. Supported actions are tool, toolchain, ask_user, final, and stop. The parser is intentionally lenient, accepting JSON object form, fenced JSON, and one-line directives, but validation still rejects missing names/text or non-object args.

This is the right compromise:

lenient input parsing
strict normalized action shape
host-side validation
host-side caps
host-side execution

Potential risk: because parsing is lenient, malformed planner outputs can be “rescued” into valid actions. That is okay for UX, but for high-trust execution the normalized action should always be logged/digested, which it is via Digest.

8. RunController is the execution substrate, not the planner

RunController owns deterministic execution of ActionRequestV1 actions. It runs tools/toolchains, writes/updates RunTrace when run_id is present, returns curated feedback, and derives structured RecentMutationEffectV1 from successful tool outputs. The header explicitly says it does not do planner LLM calls, and that recent-mutation effects are execution-layer summaries, not applied state.

This is exactly the right seam:

RunController
  executes action
  returns ActionExecResult
  returns RecentMutationEffectV1
  writes action trace
  does not apply continuity state
  does not classify conversational semantics

The recent-mutation derivation is carefully schema-first: it looks for explicit recent_mutation/mutation_effect data, then uses narrow compatibility keys. It also avoids synthesizing a primary for multi-file bundles. That matters for preventing a bundle from collapsing into a fake single-file target.

9. AgenticRunner is a host-controlled Tier-4 loop

AgenticRunner performs the planner → act → observe loop. It begins a RunTrace, enforces max steps/tool calls/wall time, calls the planner, parses one ActionRequestV1, gates toolchain usage, enforces confirmation for network/repo/workspace writes/toolchains, executes via RunController, feeds bounded feedback back to the planner, and stops on terminal actions or failed tool actions.

This is good because the model is not “agentic” in the unsafe sense. The host is agentic:

model proposes one action
host validates
host confirms if needed
host executes
host observes
model gets bounded observation

The comments in agentic_runner.h are also correct: tool traces and ActivityFeed remain model-invisible; planner output must be JSON-only; confirmation gates are host-side.

10. PlanStore is model-invisible state

PlanStore is thread-scoped and writes plans/last_plan.json plus content-addressed plans/plan_<hex>.json. The header explicitly says this is model-invisible state, like TaskStore, and that higher-level parsing happens elsewhere.

That is correct. Plans are durable host artifacts, not prompt truth. They can be packed later, but storage alone does not make them model-visible or proof-bearing.

11. SprintPlanV1 normalizes planner drift

sprint_plan_v1.h accepts actions[] or steps[], normalizes common action/name drift, and converts entries into canonical ActionRequestV1-compatible action_json. It also puts terminal action payloads like ask_user, final, and stop at top level because ActionRequestV1 expects terminal text/reason outside args.

This is useful, but it is also a drift-control hotspot. It intentionally rescues non-canonical planner outputs. That is fine for Tier 3 planning as long as the normalized output is stored and executed, not the raw planner text.

Semantic drift map
Controlled well
1. AgentFlow does not classify raw text after policy finalization.
2. AgentFlow strips stale TURN_STATE keys before appending canonical override lines.
3. Retrieval options are gated by TurnPlan + ResolvedTurnState.
4. Deterministic command turns get explicit local ResolvedTurnState.
5. Recent mutation truth is not clipped to working-set cap.
6. RunController returns effects; it does not apply continuity state.
7. AgenticRunner enforces host-side caps and confirmation.
8. PlanStore is model-invisible.
9. ActionRequestV1 creates a host-validated action boundary.
Remaining risk points
1. Runtime still has some local semantic special cases

buildDeterministicCommandTruth_ and buildMissingRepoTargetServedText_ are local semantic paths. They are acceptable because they are deterministic terminal paths, but they need to stay narrow. Runtime should not keep accumulating answer-family logic.

2. Marker fallback for recent mutation is useful but risky

mergeRecentMutationFromMarkersIntoPostCommit_ can recover mutation data from prompt markers if structured effects are absent. That is a practical fallback, but structured effects should be the primary route. Marker fallback should remain defensive, logged, and eventually rare.

3. SprintPlanV1 alias normalization can hide planner contract failure

The parser’s tolerance is good for UX, but it can also make bad planner output look valid. For evaluation, we should record whether normalization rescued the plan so we can distinguish “planner followed contract” from “host repaired drift.”

4. AgentFlow owns a lot

agent_flow.cpp is doing orchestration, deterministic command truth, TURN_STATE transport construction, retrieval gating, post-commit merge, recent-mutation marker handling, Stage-1 export, and agentic handoff. It is coherent, but it is becoming the central pressure point. Further growth should be split into helper modules, not added inline.

Main conclusion

Block 11 set 1 is strong. The runtime layer is mostly doing what we want: it consumes canonical turn truth, transports it into prompt/export/telemetry surfaces, executes deterministic bypasses safely, keeps tool execution host-owned, and routes post-commit continuity through a structured effect boundary.

The most important positive pattern is:

semantic truth is resolved before runtime execution
runtime mirrors and enforces it
execution produces effects
post-commit applies effects
telemetry records both policy truth and packed truth

The highest-value thing to verify in set 2 is whether validation and answer postprocess consume these same canonical fields instead of parsing answer text, prompt markers, or old transport lines as a competing truth source.

Carry-forward summary
Block 11 set 1 reviews runtime orchestration. `AgentFlow::RunTurn` is the central turn orchestrator: it prepares TurnContext, TurnPlan, ResolvedTurnExtras, ResolvedTurnState, retrieval options, prompt options, and Stage-1 telemetry. It is mostly a policy consumer, not a semantic classifier. The strongest anti-drift control is that AgentFlow strips stale policy/resolved/mutation/plan TURN_STATE lines before appending canonical override lines from the current ResolvedTurnState. Deterministic slash-command and missing-repo-target turns now get explicit local resolved truth and no longer leave validation/export to guess. Retrieval is gated by TurnPlan plus ResolvedTurnState. Recent mutation truth is separated from working-set convenience and preserved up to a larger cap. RunController executes host-owned ActionRequestV1 actions and returns structured recent-mutation effects without applying state directly. AgenticRunner runs the Tier-4 planner-act-observe loop with host-side caps and confirmation gates. PlanStore stores model-invisible thread-scoped SprintPlan JSON. Remaining drift risks are local deterministic semantic special cases in AgentFlow, marker fallback for recent mutation recovery, tolerant SprintPlan normalization hiding planner drift, and AgentFlow becoming too large as an orchestration pressure point.
Portfolio-facing wording
The runtime layer acts as a compiled-turn orchestrator: it consumes resolved turn policy, mirrors canonical truth into prompt/export markers, gates retrieval and deterministic bypasses, executes tool requests through host validation, and applies continuity only through structured post-commit effects. This keeps model-visible prompts, execution traces, recent-mutation continuity, and telemetry aligned without letting any one support surface silently become canonical truth.




Block 11 — Rest Analysis and Summary

This completes Block 11 by covering the commit layer, ephemeral state, task persistence, output validation, and postprocess. Overall, this set is much stronger than earlier architecture versions because it clearly separates:

conversation commit lifecycle
runtime/post-commit continuity
validator contract enforcement
presentation cleanup
model-invisible task state

The main result: validation/postprocess now mostly consumes canonical truth instead of reclassifying the turn.

1. ConversationManager is correctly reduced to commit lifecycle

ConversationManager is now explicitly “frozen” and its header says the removed responsibilities are routing, TurnPlan, retrieval, prompt packing, evidence extraction, ActiveHint/WorkingSet/epochs, deterministic shortcuts, and summary scheduling. Its remaining role is narrow: append user message + placeholder, call AgentFlow::RunTurn, run the model only when needed, validate/postprocess once, replace the placeholder, apply post-commit after commit, then finalize export.

That is the correct shape. ConversationManager no longer owns semantics. It owns conversation mutation and lifecycle.

The important ordering is:

append user + placeholder
→ AgentFlow::RunTurn
→ model call or deterministic served text
→ OutputValidator::ValidateAndPostprocess
→ replace assistant placeholder
→ save conversation
→ ApplyPostCommit
→ EnqueueFinalizeExport

This is good because post-commit continuity is applied after the final answer is committed, not before. That prevents a failed/cancelled turn from mutating continuity state.

Cancellation is also handled cleanly: cancelled turns replace the placeholder with [Cancelled] and log post_commit_applied=0. That is the right safety posture.

2. EphemeralStateStore is now a real continuity authority

This file is one of the strongest parts of the set.

It owns active hint, working set, recent mutation continuity, epoch progression, TTL expiry, durable epoch bridging, and strict post-commit application. BeginTurn increments the epoch, reads durable thread epoch, expires stale hint/working-set/recent-mutation state, and returns a snapshot for that turn. ApplyPostCommit requires a non-zero epoch and strict equality with the current thread epoch before applying anything.

That strict equality rule is very important:

post_commit.epoch == current_thread_epoch

It prevents old async completions or stale agentic runs from applying continuity mutations after a newer turn has begun.

The recent-mutation semantics are also correct:

source alone does not make mutation present
preferred_primary alone does not make mutation present
single-file bundle may synthesize primary_rel
multi-file bundle must clear primary_rel
preferred_primary remains metadata only
recent mutation cap = 64
working set cap = 8

That directly addresses the previous drift class where a stale or synthetic primary could make a multi-file apply look like a single-target mutation.

3. TaskStore is safely model-invisible

TaskStore persists long-running task records per (pid, thread_id), including status, goal, steps, refs, and optional next_action_json. Its header explicitly says it is model-invisible and must not be packed into prompts. Validation of next_action_json happens later in RunController before execution.

That is the right boundary:

TaskStore stores host workflow state.
RunController validates/executes next_action_json.
Prompt packing should not silently include task-store content.

The main small risk is that TaskStore sanitizes and preserves next_action_json as an object but does not validate it at write time. That is acceptable because execution validates, but for diagnostics it may be useful later to add “invalid next_action stored” markers when task creation/upsert happens.

4. AnswerPostprocess is mostly presentation cleanup, with contract enforcement

answer_postprocess.cpp states the correct rule directly: it consumes validator markers first, may consume TURN_STATE transport as secondary canonical source, does presentation cleanup only, and must not rebuild semantic truth from prompt-builder hints, legacy markers, or raw text.

It does still enforce output-shape contracts:

Patch mode
  → requires one fenced unified diff

Verbatim/dump mode with source evidence
  → requires provenance / allowed evidence paths

Tier-0/full-file formatting
  → repairs file headers and code fences

NMC
  → canonicalizes or rewrites based on nmcAllowed / suppressNeedsMoreContextByTruth

That is acceptable because these are shape and contract guards, not new answer-kind classification.

The best detail: postprocess distinguishes source evidence from semantic primer/session/tool planes, and dump/provenance enforcement is reserved for source-evidence turns only. That prevents support-only planes from being silently upgraded into proof.

5. OutputValidator is now contract-first, not semantic-repair-first

output_validator.cpp says the right thing in its header: it consumes authoritative Stage-1 merged marker truth, enforces contract/state constraints without semantic reconstruction, uses runtime diagnostics only from explicit packed planes/runtime markers, avoids a secondary semantic backstop after canonical truth is known, and does not treat route labels/request tags/loose metadata as authoritative truth.

The Stage-1 marker merge is important. The validator can load the Stage-1 turn bundle markers, merge them with current markers, dedupe by key, and then validate against the merged authoritative marker set. That helps avoid finalization using a weaker marker set than the prompt/export stage had.

The validation path is now:

merge Stage-1 prompt markers
→ upsert runtime truth markers
→ read canonical marker view
→ upsert canonical policy markers
→ build contract fallback only if canonical primary truth exists
→ sanitize forbidden NEEDS_MORE_CONTEXT
→ delegate once to AnswerPostprocess
→ rewrite packed-source-evidence contradictions
→ finalize export

That is a good contract-first validator shape.

6. OutputValidatorPolicy is the canonical marker reader

output_validator_policy.cpp is the main bridge from marker transport into validator truth. It reads TurnStateTransport first where usable, then falls back to explicit policy.*, resolved.*, and answer.* markers. It infers requires_source_evidence from canonical answer_kind only when the explicit field is missing, which is acceptable because this is contract alignment, not raw text reclassification.

It also clears legacy Phase-4 prompt-shaping fields and markers, which is valuable. The validator should not retain old “phase4_*” backstop semantics as hidden truth.

The strongest contract behavior:

MissingRepoTarget
  → deterministic answer

requires_diff_plane && missing diff
  → NMC or text fallback based on nmcAllowed

requires_source_evidence && missing evidence
  → NMC or text fallback based on nmcAllowed

suppressNeedsMoreContextByTruth / nmcAllowed=false
  → suppress or rewrite NMC

That is the right place for enforcement.

Semantic drift assessment
Strong anti-drift controls
1. ConversationManager no longer routes/classifies.
2. OutputValidator merges Stage-1 packed truth before final validation.
3. ValidatorPolicy reads canonical TURN_STATE / policy markers before fallbacks.
4. Postprocess consumes validator markers first, TURN_STATE second, raw text last.
5. EphemeralStateStore applies PostCommit only on exact epoch match.
6. Recent mutation continuity cannot be fabricated by source/preferred-primary alone.
7. Multi-file recent mutations cannot carry a fake single primary.
8. TaskStore remains model-invisible.
9. Deterministic/caller-owned contract turns suppress normal user-turn guard behavior.
Remaining risk points
1. Validator still has a lot of marker fallback logic

This is mostly necessary for compatibility, but it is still a broad surface. The safest long-term path is to ensure Stage-1 canonical markers are always present, so direct fallback parsing becomes rare.

2. Postprocess has shape enforcement plus presentation rewrites

The rewrites are useful, but this file is naturally dangerous because it touches final user-visible text. The current comments are correct: it must remain downstream-only. Avoid adding new semantic interpretation here.

3. TaskStore accepts invalid next_action_json until resume

Execution validates later, so this is safe, but stored invalid tasks can create confusing UX. A future low-risk improvement would be to store a validation status or warning when creating/upserting a task.

4. Source-evidence contradiction repair is a validator patch, not a model fix

The repair that softens “content not shown” when source evidence was packed is useful. But it is still compensating for model behavior. The better upstream fix remains: packed prompt guidance should make the model less likely to emit that contradiction in the first place.

Main conclusion

Block 11 is now coherent.

The completed runtime/commit/validation chain is:

TurnContext / TurnPlan / ResolvedTurnState
  → AgentFlow transports canonical truth
  → ConversationManager commits exactly one output
  → OutputValidator enforces canonical contract
  → AnswerPostprocess cleans presentation
  → EphemeralStateStore applies continuity only after commit
  → CLE export finalizes with merged authoritative markers

That is the architecture we were aiming for. The validator is no longer trying to become a semantic repair engine; it mostly checks whether the final output obeys canonical truth and packed-truth constraints. The store is no longer a vague continuity cache; it is an epoch-gated continuity authority. The conversation manager is no longer a routing brain; it is a commit lifecycle wrapper.

The most important remaining work is not to add more intelligence to these layers. It is to make sure upstream canonical truth and Stage-1 packed truth are always complete enough that the validator rarely needs fallback behavior.

Carry-forward summary
Block 11 completes the runtime/commit/validation review. ConversationManager is now reduced to lifecycle: append user + placeholder, call AgentFlow, run model only if needed, validate/postprocess exactly once, replace placeholder, apply PostCommit after final commit, and enqueue export finalization. EphemeralStateStore is the continuity authority: BeginTurn increments durable epoch, expires TTL-scoped hint/working-set/recent-mutation state, and ApplyPostCommit requires strict nonzero epoch equality before mutating state. Recent mutation semantics are strong: source/preferred-primary alone cannot fabricate presence; multi-file bundles clear primary_rel; working set stays capped at 8 while recent mutation tracks up to 64. TaskStore is model-invisible host state and stores task status/steps/next_action_json for RunController validation/execution later. OutputValidator is contract-first: it merges Stage-1 prompt markers, reads canonical markers via ValidatorPolicy, applies contract fallbacks for missing source/diff planes, suppresses forbidden NMC, delegates once to AnswerPostprocess, and finalizes export. AnswerPostprocess consumes validator markers first, TURN_STATE transport second, and raw text only for cleanup/shape handling. Remaining risks are broad compatibility fallback parsing, final-text rewrite complexity, invalid task action JSON persisting until resume, and source-evidence contradiction repair compensating for model behavior.
Portfolio-facing wording
The committed-output path is now a contract-first pipeline: the runtime resolves canonical turn truth, the conversation layer commits exactly one output, the validator enforces source/diff/NMC constraints from authoritative markers, postprocess performs presentation cleanup, and continuity is applied only through strict epoch-gated post-commit state. This prevents prompt text, summaries, task state, or stale continuity from silently becoming proof.
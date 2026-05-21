Block 9 Analysis — Turn Semantics / Policy Spine

This is the most important block so far because it defines the single-turn semantic compiler:

raw user turn
  → scoped/sanitized TurnContext
  → weak heuristic/boundary facts
  → canonical TurnPlan
  → ResolvedTurnExtras
  → TargetSeed
  → ResolvedTurnState
  → AnswerContract
  → claim/ownership/canonical target markers
  → retrieval / prompt / validator / exporter projection

The core result: the intended ownership boundaries are much cleaner than earlier versions. The block now explicitly states that TurnContext observes, TurnPlanner plans, turn_target_selection seeds targets, resolved_turn_contract derives answer contract truth, and resolved_turn_state finalizes the canonical resolved state exactly once.

1. Turn context construction is doing the right first-stage isolation

TurnContext::Build now acts like a turn-normalization and observation layer, not a policy owner.

It builds several text forms:

userSanitized
cleanedUserVisible
heuristicRaw
scopedSignal
retrievalText
modelVisibleUserText

The important move is policyScopedText: if the user pasted a path dump, code blob, prompt dump, trace, or large file block, the policy layer uses only the natural-language prefix before the dump/blob. That means a huge pasted code block or file list does not dominate intent classification. The scoping layer is explicitly limited to dump/blob detection and prefix extraction; it is not allowed to decide answer kind, authority mode, output mode, retrieval mode, or recent-update semantics.

It also normalizes ephemeral state at the beginning of the turn: active hint, working set, recent mutation files, recent mutation primary, and recent mutation artifact flags. Two good safeguards appear here: a single recent mutation can backfill the working set, but a multi-file recent mutation bundle can clear/debias the active hint so one stale active file does not collapse a bundle into a fake single-file target.

Turn context also resolves explicit mentions through several channels: existing attention files, snippet file hints, file-token extraction, inventory matches, recent file-context references, pinned focus, and synthetic active hints. But the key is that these are still observations: explicit mentions, unresolved mentions, focus, inventory matches, and semantic-family hints are logged and passed forward, not made final answer truth inside TurnContext.

2. Heuristic detection is mostly centralized, but still used in multiple stages

turn_context_heuristics.* is a weak-signal library. It owns reusable detectors for:

compare request
explicit diff request
recent update
explicit full-file request
current-file responsibility
conversation recall
source usage trace
source startup trace
source order trace
source runtime trace
source responsibility trace
architecture overview
cross-repo usage
analytic file ordering
inventory listing
multi-file planning
file-token extraction
snippet parsing
inventory match candidates

The header explicitly says this seam must not decide final answer kind, authority mode, output mode, retrieval mode, NMC admissibility, recent-update canonical truth, or canonical target/anchor truth. That is the right architectural posture.

The good repair here is that inventory-list detection is now guarded against swallowing analytic/source-led turns. The inventory predicate refuses file-scoped, source-led, runtime-flow, responsibility, and broad analytic architecture questions. This directly addresses one of the recurring eval failures: questions like “which files are central to runtime flow and why?” should not become “list files.”

The remaining risk is not that heuristics are missing; it is that TurnContext, TurnPlan, and ResolvedTurnExtras all call the same weak recognizers again. That is acceptable if they always call the centralized functions, but it still creates a drift surface if one layer starts adding local exceptions again.

3. Boundary intent is now properly weak

intent.h is a boundary recognizer only. It can detect obvious classes:

repo veto
general writing
general knowledge
list files
inventory state
show file
edit files
compare files
repo question

It explicitly must not decide answer kind, authority mode, output mode, current-file semantics, recent-update semantics, or NMC eligibility. This is exactly right: boundary intent is an admission/routing signal, not canonical policy.

It also has an important priority rule: explicit file-targeted turns beat broad inventory/list wording. Example from the comment: “list the constants defined in config.py” should be file-understanding, not repository inventory listing. That is a good guard against the old “list” keyword flattening everything into inventory.

4. Planner is now the canonical TurnPlan owner

turn_plan_builder.cpp is the main policy planner. Its ownership boundary is stated clearly: it may inspect TurnContext, consume weak heuristics, derive TurnPlan, derive packing admission, and build request tags. It must not finalize resolved state, derive answer contract truth, derive claim target truth, mutate resolved state, own /plan or /apply, or own final target seed truth.

The planner decides:

scope
repoMode
authorityMode
outputMode
inventoryMode
wantsFetch
hasExplicitAnchors
globalUnanchored
seedSource
seedFiles
lattice parameters
packing directives
claim policy
requestTag
traceLine

This is the right layer for those decisions.

The source-led classification is much stronger now. It distinguishes:

SourceLedIntent::Usage
SourceLedIntent::Responsibilities
SourceLedIntent::Order
SourceLedIntent::Startup
SourceLedIntent::Runtime

Then maps these to different authority modes. Usage is ImplementationStrict; source-led startup/order/runtime/responsibility are ImplementationStrict when broad, but can become FileUnderstanding when targeted by explicit ownership signals.

Most importantly, semantic primer admission is intentionally blocked for source-led turns. admitsSemanticPrimerPlanes_ refuses semantic primer for any non-None sourceLedIntent, keeping source-led usage/startup/order/runtime/responsibility answers evidence-first rather than summary-first.

5. Target seed selection is now its own seam

turn_target_selection.cpp is exactly the seam we wanted. It owns only:

operational target seed
current-file seed
anchor source
anchor strength

It explicitly must not derive answer kind, NMC policy, evidence requirements, or finalize resolved turn state.

Its target priority is clear:

compare pair
explicit mention
unresolved explicit mention
pinned focus
recent mutation
active hint
single-file working set
working set
recent mutation bundle
working set bundle
inventory match only/ranked/single
none

Anchor strengths are also explicit:

compare_pair             100
explicit_mention          90
explicit_missing_mention  88
pinned_focus              84
recent_mutation           80
recent_mutation_bundle    74
active_hint               72
single_file_working_set   68
working_set_bundle        60
working_set               56
inventory_match_ranked    42
inventory_match_single    34
inventory_match_only      22

This is a good hierarchy. It makes explicit user intent outrank continuity, and continuity outrank weak inventory matches.

The planner does still ask target selection for a preview seed and then performs plan-time inventory promotion if allowed. That is probably acceptable because the comment says planner may layer inventory-promotion gating on top of the preview, but it is the one place where target-ish truth leaks slightly back into planning. Keep an eye on that; it should remain plan-time gating only, not final claim target truth.

6. ResolvedTurnExtras is a pre-finalization fact layer

resolved_turn_extras_builder.* derives extra semantic facts from TurnContext and the canonical TurnPlan, including:

compare pair
historical diff
explicit full-file request
current-file responsibility
conversation recall
architecture overview
cross-repo usage
analytic ordering
responsibility trace
global cross-file question
recent update question
recent mutation targets
recent mutation artifact availability
source-led intent
source-led targeted
preferred current file

The header says this layer must not derive final answer contract truth, final claim-target truth, rewrite planner truth, or finalize state. That matches the architecture.

A good detail: it suppresses single-file fallback when the turn is global cross-file, a multi-file recent mutation bundle, or a multi-file working set without explicit anchors. That prevents active hints or one mutation primary from incorrectly narrowing broad/source-led turns.

7. ResolvedTurnState is now the canonical finalization pass

resolved_turn_state.cpp is the final compiler pass. It seeds from:

TurnContext
TurnPlan
ResolvedTurnExtras
TargetSeedResult

Then calls FinalizeResolvedTurnState exactly once. The file explicitly says it must not classify raw text, build plans inline, choose packing policy inline, do heuristic query-shape detection inline, or run a second policy pass after finalization.

Finalization does the heavy semantic correction:

normalizes all relpath lists
syncs compare pair fields
normalizes active/pinned/current/operational/claim targets
normalizes host command truth
recomputes recent mutation plane availability
clears active hint for multi-file mutation bundles
clears single-file targets for compare pairs
clears single-file targets for recent-update bundles
clears continuity-only targets for global/source-led broad turns
downgrades weak inventory promotions when they should not own a target
derives ownership anchor truth
derives answer contract
derives claim target truth
syncs legacy aliases
computes continuity/downgrade guard

This is the most important behavior in the block. It means stale continuity can still support routing, but it is cleared when it would fabricate a single-file target for broad/global/source-led turns.

The compare-pair and recent-update bundle behavior is especially important: compare-pair turns clear single-file target fields and force anchorSource=compare_pair; recent-update bundle turns clear single-file target fields and use recent_mutation_bundle or working_set_bundle as the anchor source. That prevents a pair/bundle from being collapsed into one “primary” file.

8. Answer contract is properly isolated

resolved_turn_contract.cpp owns only answer-contract truth:

answerKind
effectiveOutputMode
nmcAllowed
directAnswerPreferred
requiresSourceEvidence
requiresDiffPlane
suppressNeedsMoreContextByTruth
answerContractReason

It explicitly must not seed or clear targets, derive ownership anchors, re-run planner heuristics from raw text, or finalize resolved state.

The answer-kind mapping is now strong and specific:

InventoryListing
ConversationRecall
MissingRepoTarget
ExplicitFullFileRequest
CurrentFileResponsibility
CompareTargets
RecentUpdateSummary
StrictDiffRequest
ImpossibleDiffRequest
SourceUsageTrace
SourceResponsibilityTrace
SourceStartupTrace
SourceOrderTrace
SourceRuntimeTrace
ArchitectureOverview
DirectRepoAnswer
FileUnderstanding
GeneralChat

The crucial design is that most repo-grounded answer kinds set requiresSourceEvidence=true, while conversation recall and inventory listing do not. Architecture overview only requires source evidence when it is source-grounded rather than purely broad/conceptual.

This is aligned with the Block 8 rule:

source evidence proves
semantic primer or continuity only supports
9. Markers / transport preserve canonical semantics downstream

resolved_turn_state_io.cpp exports both policy.* and resolved.* markers for:

answer kind
answer family
trace shape
effective output mode
NMC flags
source/diff requirements
canonical target
operational target
ownership anchor
claim target
compare pair
source-led intent
allowed planes
recent mutation state
unresolved explicit targets

It also parses transport back into TurnStateTransport, but treats missing scalar tokens like -, none, and null as empty. This gives downstream layers a canonical transport vocabulary instead of asking prompt/validator/exporter to infer intent again.

This is the preservation layer: once ResolvedTurnState says answer_kind=source_startup_trace, canonical_target=-, compare_pair_present=0, requires_source_evidence=1, downstream code can consume that directly.

Overall assessment

Block 9 is a major improvement. The architecture now mostly matches the doctrine:

TurnContext observes.
Heuristics provide weak candidate facts.
Boundary intent gives weak admission cues.
TurnPlan decides scope/repo/authority/output/inventory/packing.
Target selection seeds operational/current/anchor truth.
ResolvedTurnState finalizes canonical state once.
Answer contract derives answer-kind/evidence/NMC/output contract only.
Markers preserve canonical truth downstream.

The important repair is that broad/source-led/global turns are no longer allowed to silently inherit a single-file target from continuity unless there is a real ownership basis. That directly attacks the old failure mode where active hint, recent mutation primary, or inventory match would fabricate a “current file” answer for a broad architecture/order/runtime question.

Remaining risks
1. Heuristic calls are centralized, but semantic derivation is repeated

TurnContext, TurnPlan, and ResolvedTurnExtras all call the same candidate helpers. That is much better than local duplicate heuristics, but there is still repeated classification work. The next hardening step would be to compute a WeakSemanticFamilyHints object once in TurnContext and pass it forward, rather than re-evaluating the same low-string multiple times.

2. Planner still does limited target-ish inventory promotion

The code comments say this is plan-time gating, not final target ownership. That is acceptable, but fragile. The safe invariant should be:

planner may preview/gate inventory target promotion;
resolved target seed + finalization own final operational/current/claim target truth.

Do not let inventory promotion become claim target truth without going through turn_target_selection and FinalizeResolvedTurnState.

3. buildRetrievalOptionsFromTurn rebuilds plan/extras/state

This is okay because it uses the canonical builders, but it means retrieval option construction replays the turn compiler from TurnContext. If TurnContext is immutable after build, this is fine. If any ephemeral fields mutate after TurnContext::Build, this can drift. The safe rule is: build TurnContext once, freeze it, derive all downstream projections from that same frozen object.

4. HostCommand depends on runtime explicitly setting answer kind

ResolvedTurnState has a strong host-command normalization path, but it only activates if answerKind == HostCommand. The visible block does not show where slash command runtime sets that. This needs verification in the runtime/command block.

5. /use parser is low-level and intentionally separate

The /use parser removes /use lines and extracts tokens, but the visible TurnContext::Build path mostly clears attachment tokens and relies on attention/focus/mentions/snippets/inventory. That may be intentional given your rule that /use transport should not participate in routing/target/validator truth, but it is worth confirming in the runtime/input block.

Carry-forward summary
Block 9 defines the turn semantic compiler. The pipeline is now: TurnContext observation/scoping → weak heuristic/boundary facts → canonical TurnPlan → ResolvedTurnExtras → target seed from turn_target_selection → ResolvedTurnState finalization → answer contract derivation → canonical markers/transport. Ownership boundaries are mostly clean: heuristics are weak only, planner owns scope/repo/authority/output/inventory/packing, target_selection owns target seed, resolved_turn_contract owns answer kind/evidence/NMC/output contract, and resolved_turn_state finalizes canonical truth exactly once.

The strongest improvement is target de-biasing: broad/global/source-led turns no longer automatically inherit active hints, recent mutation primaries, or inventory matches as single-file claim targets. Compare pairs and recent-mutation bundles clear single-file target fields and preserve pair/bundle semantics. Source-led turns are evidence-first and generally do not admit semantic-primer planes. The remaining watch points are repeated heuristic evaluation across layers, plan-time inventory promotion, retrieval options rebuilding state from TurnContext, and verifying where HostCommand truth is set.
Portfolio-facing wording
Coeus includes a deterministic turn-policy compiler that converts each user turn into canonical semantic truth before retrieval or prompting. It separates weak observations from final policy: context construction observes files, focus, snippets, mutation state, and scoped text; the planner derives scope, authority, output mode, inventory mode, and packing permissions; target selection assigns operational/current anchors; and a final resolved-state pass derives ownership, claim target, answer kind, evidence requirements, and downgrade guards. This prevents downstream prompt, retrieval, and validator layers from reclassifying the user’s intent independently.
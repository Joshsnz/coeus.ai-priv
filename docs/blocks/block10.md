Block 10 Analysis — Prompt Assembly / Policy Projection

Block 10 is broadly aligned with the architecture we want. The prompt layer is now mostly a consumer of canonical turn truth, not a second semantic classifier. The intended flow is:

ResolvedTurnState / TURN_STATE transport
  → prompt_policy_projection
  → prompt_builder admission + guidance
  → context/keyed support blocks
  → lattice ContextIR / final prompt
  → Agent::Request

The most important line in this block is in prompt_builder.cpp: it states that PromptBuilder is a consumer of canonical prompt-policy projection and “must not rebuild answer-kind / authority / coverage truth from raw text.” That is exactly the correct doctrine.

1. PromptBuilder is now mostly orchestration

prompt_builder.cpp has been refactored into a coordinator:

runtime profile/model-cap resolution     → agent_profiles.*
prompt-context keyed block production    → context_prompt_service.*
prompt-policy projection                 → prompt_policy_projection.*
prompt assembly + request wiring         → prompt_builder.cpp
final prompt compilation                  → lattice

That is a good split. PromptBuilder no longer tries to own all prompt semantics. It now:

1. sanitizes user/plan text
2. detects caller-owned internal contracts
3. resolves canonical TURN_STATE projection
4. derives output guidance from projection
5. derives authority/admission from projection
6. injects locked tier contract
7. injects tone-only profile prompt
8. prunes/adopts semantic primer and recent mutation blocks
9. adds source-evidence discipline blocks when actual source evidence is packed
10. sends everything into lattice prompt compilation
11. appends prompt markers

The strong improvement is that it distinguishes source evidence expected from source evidence actually packed. When packed source evidence exists for a source-required/source-only turn, it emits a model-visible discipline block telling the model not to say the file is “not shown/provided” and not to hedge with “likely/presumably” for claims directly supported by evidence. That directly addresses the “model claims not shown despite packed evidence” failure mode.

2. PromptPolicyProjection is the main prompt truth adapter

prompt_policy_projection.* is the right seam. Its comment is clear: it parses canonical TURN_STATE transport, projects it into prompt-facing structures, derives structural prompt contracts from canonical truth, derives summary admission from canonical truth, and emits helper blocks/markers from canonical truth. It explicitly must not infer policy from raw user text, recover answer kind from prose, repair canonical target truth from prompt blocks, upgrade keyed blocks into policy truth, or rewrite planner admission from local prompt heuristics.

The key output structures are:

CanonicalTurnProjection
ParsedCanonicalPolicy
StructuralPromptContract
Phase4PromptingHints
AuthorityAdmissionPolicy
Phase3SummaryAdmission
RecentMutationPromptSignal

This gives the prompt layer its own vocabulary without letting it become a second policy engine. It maps resolved answer kinds into prompt-facing shape:

answer family
answer shape
evidence posture
coverage requirement
compare symmetry
source-led flags
recent mutation bundle flags
canonical target
compare targets

That is good because prompt building needs these concepts, but it should derive them from resolved state, not from user text.

The remaining risk: prompt_policy_projection has fallback logic like effectiveParsedAnswerKindForPolicy_, which infers an answer kind from canonical fields if answerKind is missing. That is acceptable as compatibility, but long-term the canonical answerKind should always be present. The fallback should stay defensive, not become normal operation.

3. Semantic primer and recent mutation are now treated as support planes

context_prompt_service.* is doing the right thing with keyed blocks. The header says keyed blocks are support-plane material only, not canonical policy truth, and downstream projection may read their hints but must not let them rewrite TURN_STATE transport truth.

BuildSemanticSummaryBlocks is also correctly scoped. It says it is a semantic-summary materializer only. It may consume planner packing directives, materialize project/folder/file summaries, and pick stable seed files for lookup. It must not decide semantic-primer admissibility, re-derive source-led/compare/recent-update/strict-file truth, or suppress summaries using a second local answer-family model.

That is the right ownership:

Planner says semantic primer may be materialized.
ContextPromptService materializes if available.
PromptPolicyProjection / PromptBuilder decide final admission/pruning from canonical truth.
Lattice packs what survives.

Recent mutation blocks are also explicitly labeled as support context, not proof. The recent mutation plane includes files, primary, source, prior-version artifact flag, exact-diff artifact flag, structural-summary flag, epoch, and bundle guidance. But it is marked with support_only.recent_mutation=1. That is good: it can orient change-summary answers, but it should not prove exact code or exact diff by itself.

4. Attention/context packing is deterministic, but still a watch point

attention_builder.cpp resolves file mentions, durable focus, neighbors, and attention edges. It normalizes paths, resolves ambiguity, applies source priorities, merges duplicates, and caps attention files. It also carefully prevents ephemeral sources like active hint, attachment, working set, neighbor, or unknown from masquerading as user mentions or persistent focus.

That is mostly good. It helps the system create a stable attention set:

UserMention      strongest
Attachment       explicit but not user mention
PersistentFocus  durable focus
WorkingSet       continuity
ActiveHint       weaker continuity
Neighbor         low-weight helper
Unknown          lowest

The risk is conceptual: attention_builder lives under prompting but still contributes to TurnContext. It must remain an input observation builder, not final target or answer truth. In Block 9, the final target truth is owned by turn_target_selection and FinalizeResolvedTurnState. So this is safe as long as attention is treated as observation/support, not claim truth.

5. Locked prompts and templates are now cleanly separated

locked_prompts.* is the read-only catalog for hard contracts:

locked.tier0_policy
locked.tier0_contract
locked.tier1_contract_user_turn
locked.cle_contract_user_turn
locked.tier2_contract_user_turn
locked.tier3_sprint_contract
locked.tier4_planner_contract
locked.cle_contract_active

The important distinction is that locked.cle_contract_active is only a compatibility selector for ordinary user-turn contracts. Tier 3 and Tier 4 use dedicated planner contracts and should not route through the ordinary user-turn selector.

prompt_templates.* is now mostly compatibility wrappers plus tone-only built-ins and summary prompts. It explicitly says locked contracts/policy are owned by LockedPrompts, not PromptTemplates. Built-in profile prompts are tone-only and must not override hard contracts.

That separation is correct:

LockedPrompts / tier contracts = hard rules
PromptTemplates built-ins      = tone/work-mode only
PromptStore custom prompts     = tone-only custom prompts
SummaryPrompts                 = internal summarizer prompts
6. Tiered prompt contracts look coherent

The tier model is now reasonably clean:

Tier 0
  no-repo / conversation-only contract
  strong generated-file formatting discipline
  no repo claims unless content is in conversation

Tier 1
  repo-aware user turn
  TURN_STATE authoritative
  packed planes only
  source evidence vs semantic primer vs inventory vs memory separation

Tier 2
  tool-informed, proof-carrying
  tool outputs only visible if packed
  policy permission does not prove tool execution or evidence existence

Tier 3
  SprintPlanV1 JSON-only planner contract
  caller-owned internal contract

Tier 4
  ActionRequestV1 / full agentic planner contract
  caller-owned internal contract

Tier 1 is especially important. It encodes the core evidence distinction:

SOURCE EVIDENCE = proof / content-visible
SUMMARIES = semantic primer only
INVENTORY = existence/breadth only
SESSION MEMORY = continuity only
RECENT MUTATION = change continuity only
TOOL_OBSERVATIONS = support unless source-backed

That matches the broader CLE doctrine and should reduce prompt-induced hallucination.

Main conclusion

Block 10 is in good shape. The prompt layer is no longer the main source of semantic drift. It mostly consumes canonical policy truth from TURN_STATE, projects it into prompt-facing structures, prunes/adopts support planes, injects the correct locked contract, and emits discipline blocks when actual source evidence is packed.

The best signs are these markers/behaviors:

prompt_builder.policy_rewrite=0
prompt_builder.policy_consumer=1
prompt_builder.output_guidance_from_canonical_truth=1
prompt_builder.summary_admission_from_canonical_truth=1
prompt_builder.phase4_from_canonical_truth=1

That is exactly what we want.

Remaining risks
1. PromptPolicyProjection still has fallback answer-kind inference

This is okay for compatibility, but it should become rare. The invariant should be:

If TURN_STATE is present, resolved.answer_kind / policy.answer_kind should always be present.
Prompt projection fallback should only protect against stale/legacy transports.
2. buildCanonicalPromptingHintsFromProjection_ partly duplicates structural derivation

prompt_policy_projection.cpp already has StructuralPromptContract, but prompt_builder.cpp has its own canonical guidance builder. It still derives from projection, not raw text, so it is not a severe problem. But eventually this should probably collapse into one projection-owned API so PromptBuilder only consumes final Phase4PromptingHints.

3. Greenfield detection still uses raw user text

There are two greenfield helpers that inspect raw text for .py, main.py, settings.py, “full code”, “starter”, etc. This is a controlled special case, but it is still local prompt-layer heuristic behavior. It is probably acceptable for Tier 0/no-repo generation, but it should not expand into repo-policy semantics.

4. Semantic summary seed fallback can pick graph/index/included files

BuildSemanticSummaryBlocks may fall back to graph-index/nav-start or first included file if seed files are empty. Because this is only semantic primer materialization and downstream admission can prune it, this is acceptable. But it must remain support-only and never become target truth.

Carry-forward summary
Block 10 defines the prompt layer as a canonical policy consumer. PromptBuilder now orchestrates runtime profile resolution, locked contract injection, prompt-policy projection, support-plane admission, source-evidence discipline, marker emission, and lattice prompt compilation. PromptPolicyProjection parses TURN_STATE transport into prompt-facing structures and derives answer family, answer shape, evidence posture, coverage requirements, compare symmetry, summary admission, and recent mutation admission from canonical truth. ContextPromptService materializes keyed support blocks only; semantic summaries and recent mutation planes are not canonical policy truth. LockedPrompts owns hard contracts; PromptTemplates/PromptStore own tone-only work-mode/persona prompts and summary worker prompts. Overall, the layer mostly consumes policy truth rather than reclassifying, with remaining watch points around fallback answer-kind inference, duplicated projection/guidance logic in PromptBuilder, and raw-text greenfield helpers.
Portfolio-facing wording
Coeus uses a prompt assembly layer that consumes canonical TURN_STATE rather than reinterpreting user intent. The prompt builder projects resolved turn policy into prompt-facing guidance, admits or prunes support planes, injects locked tier contracts, and distinguishes source evidence that is expected from source evidence actually packed into the prompt. This prevents semantic summaries, inventory, recent mutation continuity, or profile prompts from silently becoming proof or overriding resolved turn truth.
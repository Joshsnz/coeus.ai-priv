# Prompting, Contracts, and Context Packing

## Purpose

Prompting in Coeus should not be a free-form string assembly layer. It should be a projection of canonical turn truth, retrieval/source context, selected summaries/memory, and answer contracts into a model-visible context package.

The key rule:

> The prompt layer should project resolved policy truth; it should not independently decide what the user meant.

## Key files

```text
core/agent/prompting/prompt_builder.cpp
core/agent/prompting/prompt_policy_projection.cpp
core/agent/prompting/context_prompt_service.cpp
core/agent/prompting/attention_builder.cpp
core/agent/prompting/agent_profiles.cpp
core/agent/prompting/prompt_templates.cpp
core/agent/prompting/prompts_store.cpp
core/agent/prompting/locked_prompts.cpp
core/agent/prompting/summary_prompts.cpp
core/agent/prompting/tier0_contract.cpp
core/agent/prompting/tier1_contract.cpp
core/agent/prompting/tier2_contract.cpp
core/agent/prompting/tier3_contract.cpp
core/agent/prompting/tier4_contract.cpp
```

## Prompt builder

The prompt builder assembles the final request text or message blocks. It must respect:

- selected profile
- locked prompts
- prompt templates
- turn policy projection
- packed context blocks
- source evidence
- summaries/memory where admitted
- answer contract
- tool/plan context if applicable
- token budget

## Prompt policy projection

`prompt_policy_projection` is one of the most important files. Its job is to translate canonical turn state into prompt-visible rules and markers.

Good architecture:

```text
ResolvedTurnState / TurnPlan / AnswerContract
→ PromptPolicyProjection
→ prompt instructions + markers + allowed planes
```

Bad architecture would be:

```text
PromptPolicyProjection guesses intent from raw user text again
```

The source review suggests the projection layer is intended to preserve policy truth and prevent reclassification drift.

## Attention builder

The attention builder likely selects or orders context. It may help decide what context is prominent and how blocks are prioritized. This matters in large projects where everything cannot be included.

## Context prompt service

This service likely combines active project context, retrieved sources, summaries, memory, and user context into prompt-ready material.

The review lens:

- Does it keep source evidence distinct from summaries?
- Does it respect retrieval admission rules?
- Does it pack recent mutation or plan planes only when allowed?
- Does it avoid internal trace details unless deliberately asked/debug-mode?

## Profiles, templates, and locked prompts

Agent profiles, templates, prompt store, and locked prompts define reusable policy/persona/instruction material.

Public explanation should be high-level. Private review can inspect details. Do not expose private prompt strategy publicly unless sanitized.

## Tiered contracts

The presence of `tier0_contract` through `tier4_contract` suggests a tiered prompt/behavior contract system. Without overclaiming exact semantics, it is safe to describe it as:

> tiered prompt contracts that help differentiate simple/direct turns, repo/context turns, tool/routing turns, and agentic/plan-style turns.

The important point is not the exact tier labels; the point is that the prompt/routing system is stratified rather than one-size-fits-all.

## Context packing planes

A useful mental model:

| Plane | Meaning |
|---|---|
| source evidence | actual current project file/chunk content |
| inventory | file/project existence and structure |
| summary/gist | compressed project/file memory |
| user context | user/session/task context |
| tool observation | selected host-tool output allowed for model use |
| recent mutation | previous apply/write/patch context |
| telemetry markers | runtime/debug facts, usually not user-facing evidence |
| answer contract | output shape and claim constraints |

The model should only use planes that were packed or deliberately made model-visible.

## Packed truth vs available truth

This is central:

- A file may exist, but if it was not retrieved/packed, the model cannot claim its content.
- A trace may exist, but traces are private/model-invisible by default.
- A summary may exist, but it may not be enough for exact implementation claims.
- A tool may have run, but only explicitly model-visible blocks should influence the model.

## Relation to evals

RepoGrounding evals test whether the model can answer from project context, avoid false missing-context refusals, avoid hallucinating absent features, avoid internal artifact leaks, and avoid semantic-to-diff misrouting.

Prompting/contracts are where those tests become enforceable:

- policy markers tell the model what mode it is in
- packed source context gives evidence
- answer contract constrains output shape
- validator checks for leaks/misroutes/failures
- telemetry exports marker truth for eval harnesses

## Reviewer answer: how does context packing work?

> Coeus separates context into different planes—source evidence, inventory, summaries, memory, tool observations, recent mutation state, and policy markers. The turn compiler decides what the current turn is allowed to use, retrieval selects relevant source context, and the prompt builder packs only the admitted context into the model request. This prevents summaries or internal traces from accidentally becoming authoritative proof.

## Risk and audit points

Watch for:

- prompt code doing intent classification instead of projection
- summaries packed in a way that sounds like source evidence
- tool traces packed wholesale
- too many markers creating confusing model instructions
- token-budget truncation removing critical source evidence
- private prompt details leaking into public docs

## Public/private split

Publicly say:

> Coeus uses a policy-first context assembly pipeline that separates turn semantics, source evidence, summaries, tools, and validation.

Privately document:

- specific prompt/tier files
- marker categories
- contract details
- packer/attention behavior
- risks and refactor notes

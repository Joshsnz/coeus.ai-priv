# Coeus / CLE Project Q&A Prep

Private reference notes for explaining the Coeus / CLE architecture in interviews, demos, portfolio discussions, technical reviews, or repo documentation.

---

## 1. What is this project?

**Q: What is Coeus / the CLE agent harness?**

Coeus is a repo-native LLM assistant architecture built around a Code-Linked Environment, or CLE. The goal is not just to chat with an LLM, but to give the model a controlled, deterministic view of a project: what files are visible, what evidence is packed, what kind of answer is allowed, and whether the model is allowed to make source-backed claims.

The core idea is that every turn is compiled through a policy pipeline before the model sees anything. The system decides whether the user is asking a general question, a repository question, a file-specific question, a diff request, a recent-update question, a comparison, or a command. Then it builds the prompt from explicit evidence planes rather than letting the model guess from conversation history.

A simple way to explain it:

> It is an LLM coding/repo assistant where the host application owns truth, evidence, retrieval, and validation, instead of letting the model freely infer everything from chat history.

---

## 2. What problem does it solve?

**Q: Why build this instead of just using a normal chatbot with RAG?**

Normal RAG systems often blur together three things: what the user asked, what was retrieved, and what the model is allowed to claim. That creates semantic drift. The model may answer from stale summaries, infer files that were not actually shown, or treat memory as proof.

This project separates those concerns. It distinguishes between:

```text
policy truth      = what kind of turn this is and what evidence is allowed
packed truth      = what context was actually shown to the model
retrieval truth   = what is genuinely searchable/indexed
continuity truth  = prior conversation or recent state, useful but not proof
claim truth       = what the final answer is allowed to assert
```

That separation makes the system much more reliable for codebase reasoning, project inspection, and private evaluation.

---

## 3. How does retrieval work?

**Q: How does your retrieval system work at a high level?**

Retrieval starts from the current project surface. The system tracks which files are included in the active file context, normalizes their relative paths, chunks their content, and indexes those chunks into both dense and sparse retrieval structures.

At query time, the turn policy decides whether retrieval is even allowed. For example, a general chat message should not scan the repository. A source-grounded architecture question should. A missing-file request should not perform misleading retrieval if the target is already known to be absent.

When retrieval is allowed, the system uses a hybrid approach:

```text
dense vector retrieval + BM25/sparse retrieval + path normalization + rank fusion
```

The result is not just “some context.” Retrieved chunks are classified into evidence classes, mainly:

```text
source evidence     = content-visible, can support exact claims
semantic primer     = summary-level context, useful for orientation only
inventory facts     = file existence / breadth, not content proof
```

The prompt layer then packs only the planes allowed by the canonical turn policy.

---

## 4. What is hybrid retrieval in your system?

**Q: When you say hybrid retrieval, what do you mean specifically?**

Hybrid retrieval means the system does not rely only on embeddings. It combines dense vector search with sparse keyword/BM25-style search, then normalizes and fuses candidates.

The reason is that codebase retrieval needs both semantic similarity and exact lexical matching. Embeddings are good for conceptual questions like “where is authentication handled?” but sparse search is often better for file names, function names, class names, config keys, route names, and exact identifiers.

The retrieval layer fuses these candidates into one candidate set and applies path normalization so the dense index, sparse index, and active file manifest are talking about the same surface. One important engineering goal was avoiding silent mismatches, such as a sparse result pointing to an invalid or stale file row.

---

## 5. How do you prevent stale retrieval?

**Q: How do you make sure retrieval doesn’t return stale files or old chunks?**

The system treats the active file surface as canonical. Files are normalized, chunked, hashed, and tracked through a manifest. The dense vector index and sparse/BM25 index are kept aligned with that surface.

There are two important sync paths:

```text
full sync from canonical surface
delta sync from file changes
```

A full sync rebuilds/aligns the index from the current included file surface. A delta sync applies specific file changes after edits or ingest events. Stale vector rows are pruned, invalid sparse IDs are dropped, and path normalization is used so the file manifest, vector memory, and sparse index describe the same active surface.

The point is that retrieval should not silently return evidence from files that are no longer active, renamed, removed, or out of sync.

---

## 6. What are summaries used for?

**Q: How do summaries work in the system?**

Summaries are used as a semantic primer, not as proof. They help orient the model around project structure, folder roles, file responsibilities, and high-level meaning. They are useful when answering broad architecture questions or when deciding what evidence might be relevant.

But the system is strict about what summaries are allowed to prove. A summary can help the model understand that a file is probably related to authentication or retrieval, but it cannot support exact claims like “this function calls X on line Y” unless source evidence is also packed.

So the model is guided to treat summaries like this:

```text
summary = orientation
source evidence = proof
inventory = existence/breadth
conversation memory = continuity
recent mutation = change continuity
```

That distinction is one of the core design decisions.

---

## 7. What is source evidence?

**Q: What do you mean by source evidence?**

Source evidence is actual content from the repository that has been packed into the prompt for the current turn. It can be file slices, verbatim excerpts, or source chunks. If source evidence is packed, the model can make grounded claims about that content.

The system deliberately distinguishes source evidence from weaker planes. For example:

```text
A file in source evidence:
  The model can discuss its actual contents.

A file in inventory only:
  The model can say it exists, but not what it contains.

A file in summaries only:
  The model can describe high-level meaning, but not exact code.

A file only in conversation memory:
  The model can use it for continuity, but not proof.
```

That separation prevents the model from pretending it saw code that it did not actually see.

---

## 8. How does the system decide what to retrieve?

**Q: How does the system know when to retrieve files?**

The system first builds turn context. It looks at the latest user message, explicit file mentions, current file focus, pinned focus, working set, recent mutations, inventory matches, and whether the request is repository-aware or general chat.

Then the planner creates a turn plan. The plan decides things like:

```text
is this a repo question?
is this a file-specific question?
is this asking for architecture?
is it asking for a diff?
is retrieval allowed?
is global scan allowed?
should inventory be included?
should summaries be included?
should recent mutation context be included?
```

After that, the resolved turn state becomes the canonical policy for the turn. Retrieval consumes that policy instead of making its own independent decision.

That is the important point: retrieval is downstream of turn semantics. It should not invent a new interpretation of the user’s request.

---

## 9. What is TURN_STATE?

**Q: What is TURN_STATE?**

TURN_STATE is the host-generated canonical description of the current turn. It is a compact transport of the system’s decision about the turn: answer kind, output mode, authority mode, target file, whether source evidence is required, whether a diff plane is required, whether needs-more-context is allowed, and what context planes are admissible.

It exists because the model should not be responsible for deciding all of that from raw conversation text. The host application resolves those semantics once, then passes the result downstream.

A good public explanation:

> TURN_STATE is the compiled instruction state for a turn. It tells the prompt layer and validator what the host has decided this turn actually is.

---

## 10. How do you avoid semantic drift?

**Q: How do you prevent different parts of the system from interpreting the user request differently?**

The project is built around a “derive once, consume downstream” principle.

Earlier versions had risk of drift because the planner, prompt builder, retrieval layer, validator, and postprocessor could each infer slightly different semantics. The current architecture pushes toward one canonical resolved turn state.

The flow is roughly:

```text
TurnContext
→ TurnPlan
→ ResolvedTurnExtras
→ TargetSeed
→ ResolvedTurnState
→ prompt/retrieval/validator consume that state
```

Prompt building should not reclassify the turn. Retrieval should not reclassify the turn. Validation should enforce the canonical contract, not invent a new one.

That is one of the main engineering achievements of the project.

---

## 11. How does target selection work?

**Q: How does the system know which file the user is talking about?**

The target selection layer considers multiple signals in priority order. The strongest signal is an explicit file mention in the latest user message. If the user says `agent_flow.cpp`, that wins.

If the user says “this file” or “that file,” the system can use host-owned anchors such as:

```text
current file target
pinned focus target
active hint target
working set
recent mutation primary file
inventory matches
```

But it does not blindly guess. If the target is ambiguous, the canonical turn state can mark the target as unresolved or missing. That allows the assistant to ask for clarification or return a deterministic missing-target response instead of hallucinating.

---

## 12. What is the difference between policy truth and packed truth?

**Q: You mention policy truth and packed truth. What is the difference?**

Policy truth says what is allowed. Packed truth says what was actually included in the prompt.

For example, a turn may allow source evidence, but retrieval might return no source chunks. In that case:

```text
source evidence allowed = true
source evidence packed  = false
```

The model must not act as if source evidence exists just because it was allowed. This is important because many RAG systems confuse “retrieval was permitted” with “retrieval succeeded.”

The system explicitly tracks that distinction so the validator and postprocessor can detect when an answer claims unsupported evidence.

---

## 13. What is the context lattice?

**Q: What is the context lattice?**

The context lattice is the packing layer. It takes different context planes and assembles them into the final prompt under a token budget.

The planes can include:

```text
TURN_STATE
conversation tail
user context
source evidence
semantic primer
inventory
recent mutation
plan/apply/decision continuity
tool observations
```

Each plane has a role. The lattice decides what gets included, in what priority order, under the current policy. It also records markers about what was packed, so the validator and telemetry can later understand the actual context used for the turn.

The key point is that context is not just thrown into the prompt. It is structured, prioritized, and tagged.

---

## 14. What are prompt markers?

**Q: What are prompt markers?**

Prompt markers are internal metadata emitted by the host pipeline to describe what happened during a turn. They can record things like answer kind, packed planes, source evidence count, canonical target, recent mutation status, and validator-relevant facts.

They are not meant to be user-facing. They are for debugging, telemetry, validation, and private evals.

A public-safe way to explain it:

> Prompt markers are structured internal breadcrumbs that let the system audit what policy was selected, what context was packed, and what the answer was allowed to claim.

---

## 15. How does the prompt builder work?

**Q: What does the prompt builder actually do?**

The prompt builder consumes canonical policy projection from the resolved turn state. It does not decide the turn from scratch. Its job is to assemble the final model request.

It does several things:

```text
resolves runtime profile/model/token settings
adds locked system contracts
adds output guidance based on canonical answer kind
controls which context planes are admitted
passes source evidence and summaries into the lattice
adds source-evidence discipline when evidence is actually packed
attaches markers for validator/export/debugging
```

It also handles special cases, such as caller-owned contracts for internal planner/toolchain calls, where the normal user-turn contract should not be injected.

---

## 16. What are locked prompts?

**Q: What are locked prompts / tiered contracts?**

Locked prompts are stable system contracts used to constrain the model. They define how the model should treat context, evidence, output modes, and unsupported claims.

The system has tiered prompt contracts because not every turn is the same. A basic general chat turn should not have the same contract as a repo-grounded implementation turn or an agentic planning turn.

The purpose of the tiers is to keep behavior predictable:

```text
lower tiers = simpler direct responses
repo-aware tiers = source/evidence discipline
planner tiers = internal planning/tool orchestration
```

The important design principle is that these prompts enforce host policy; they should not create new policy.

---

## 17. How does validation work?

**Q: How does the validator fit in?**

The validator checks the answer against the canonical contract. It looks at the resolved turn state, packed evidence markers, output mode, source evidence requirements, diff requirements, and whether the answer is allowed to ask for more context.

The validator is not supposed to be a second planner. It should not reinterpret the whole turn. Its role is to enforce the contract already selected upstream.

For example, it can catch situations like:

```text
the model says source was not provided when source evidence was packed
the model gives a patch when patch mode was not allowed
the model claims exact file contents without source evidence
the model asks for more context when the host decided a direct answer is valid
```

---

## 18. What is answer postprocessing?

**Q: What is answer postprocessing used for?**

Postprocessing is final cleanup. It aligns the final answer with the contract and can fix presentation-level problems. For example, it can suppress inappropriate “needs more context” behavior when the canonical truth says a direct answer is valid.

But postprocessing should not become the semantic brain of the system. One major architecture goal is to avoid relying on postprocess to fix upstream classification mistakes. The correct answer contract should be established before prompting.

---

## 19. What is recent mutation?

**Q: What is the recent mutation plane?**

Recent mutation is continuity about recent changes made by tools, plan application, patch application, file writes, or other runtime actions. It tracks which files changed, whether there is a primary changed file, and whether artifacts like prior versions or diffs exist.

It helps answer questions like:

```text
what changed?
summarize the recent update
what did the last apply touch?
continue from the file we just edited
```

But it is support-only unless backed by source evidence. It can tell the model that a file changed, but it should not allow the model to invent exact diff hunks unless exact diff/source artifacts are packed.

---

## 20. How do tools integrate?

**Q: How do tools work in the system?**

Tools are registered through a tool registry and executed through a runner/controller layer. Tool execution is host-owned, not model-owned. The model can propose or request actions, but the host validates and executes them under caps and structured schemas.

Tool outputs are converted into curated feedback. Large raw tool outputs can be stored as artifacts or traces, while the model receives only controlled summaries or structured observations when allowed.

Tool execution can also produce recent mutation effects, such as:

```text
file written
patch applied
plan applied
working set updated
active hint updated
recent mutation files updated
```

Those effects are applied through post-commit state rather than letting tools directly mutate conversation truth in an uncontrolled way.

---

## 21. How does agentic mode work?

**Q: Does the system have an agentic planner mode?**

Yes. There is a higher-tier agentic path where the system can run a planner/action loop. The planner can request tools or toolchains, execute steps through the run controller, collect feedback, and produce a final response.

The important part is that tool traces and raw execution details remain host-controlled. The model does not get unrestricted access to everything. It gets curated feedback that is safe and relevant for the next planning step.

This makes the agentic path more disciplined than a basic “LLM calls tools freely” setup.

---

## 22. What are toolchains?

**Q: What are toolchains?**

Toolchains are predefined multi-step workflows. Instead of asking the model to independently decide every low-level tool step, the system can run a known chain like “write file then ingest” or “rescan then embed.”

This is useful because common workflows can be made deterministic and safer. The model can request a higher-level action, while the host executes a known sequence with consistent tracing and post-commit effects.

---

## 23. What is the role of telemetry?

**Q: What is telemetry used for?**

Telemetry supports debugging, private evaluation, and traceability. It records what happened during a turn: policy decisions, packed planes, tool calls, run traces, source evidence coverage, outputs, and postprocess behavior.

The important part is that telemetry is for development and evaluation, not for normal public release. In release mode, internal trace artifacts and raw packed prompts should be disabled or suppressed.

The telemetry helped identify issues such as:

```text
semantic drift between planner/prompt/validator
retrieval surface mismatches
recent mutation state inconsistencies
summary timing lag
source evidence not being packed when expected
validator compensating for upstream mistakes
```

---

## 24. Why did you disable trace artifacts for release?

**Q: Why not leave all the internal logs and traces on?**

Because they are useful for private evaluation but unnecessary and noisy in a release build. They may expose internal file paths, prompt contracts, policy names, packed prompts, trace schemas, or implementation strategy.

They are not enough to recreate the system, but they are still unnecessary operational leakage.

The release posture is:

```text
console logging off
CLE trace export off by default
packed prompt artifacts off
alias files off
trace persistence off unless explicitly enabled for private eval/debug
```

That gives a cleaner and safer product while keeping the debug/eval machinery available when needed.

---

## 25. How does headless mode relate to the GUI?

**Q: Why does the project have headless mode?**

Headless mode allows the same core agent pipeline to run without the GUI. That is valuable for automated evaluation, integration tests, CI-style tests, scripted scenarios, and future non-GUI surfaces.

The GUI is one interface. The headless runtime is the integration/eval surface.

This matters because a project like this should not only be tested through manual UI interaction. Headless mode makes it possible to run deterministic scenarios and verify whether the system retrieves the right files, packs the right evidence, and produces the right answer style.

---

## 26. How do private evals work?

**Q: How do you evaluate the system?**

The project uses private eval scenarios that run through headless mode. A scenario loads a fixture repository, runs a sequence of user prompts or commands, and checks whether the system behaves correctly.

The evals are mainly designed to test hard retrieval and grounding behavior, such as:

```text
did it identify the correct file?
did it avoid unsupported claims?
did it pack source evidence when required?
did it preserve recent mutation state?
did it avoid using summaries as proof?
did it answer directly when context was sufficient?
did it avoid asking for more context unnecessarily?
```

The public-facing eval reports should show outcomes and methodology, but not raw private traces, packed prompts, or internal contracts.

---

## 27. What makes this more advanced than a normal RAG app?

**Q: What is sophisticated about this architecture?**

The sophistication is not just “it has embeddings.” The key is the control architecture around the LLM.

A normal RAG app usually does:

```text
user query → retrieve chunks → stuff prompt → answer
```

This project does something closer to:

```text
user query
→ build scoped turn context
→ derive plan
→ resolve canonical answer contract
→ select target
→ determine retrieval authority
→ sync active retrieval surface
→ fetch hybrid evidence
→ classify evidence planes
→ pack lattice under policy
→ run model
→ validate against canonical truth
→ export/evaluate trace privately
→ apply post-commit continuity
```

That is a much more disciplined system.

---

## 28. How do you handle “I don’t know” or missing context?

**Q: When does the assistant ask for more context?**

The system tries not to overuse missing-context responses. If the canonical turn state says a direct answer is valid, the assistant should answer directly.

But if the user asks for something that truly requires missing evidence, such as an exact diff, exact full-file output, or exact code quote without source evidence, the system can return a controlled needs-more-context response.

The important distinction is:

```text
high-level explanation may be possible from summaries/inventory
exact code claims require source evidence
exact patches require source or diff evidence
```

---

## 29. How do you prevent hallucinated file claims?

**Q: How do you stop the model from inventing files or code?**

The system makes the model distinguish between visibility states:

```text
content-visible       = source evidence packed
existence-visible     = inventory only
semantically visible  = summary only
not visible           = absent from packed planes
```

The prompt contract tells the model not to claim file contents unless the file is content-visible. The validator also checks whether the output is consistent with the evidence posture.

This is especially important for codebase Q&A, because models often confidently describe files they have not actually seen.

---

## 30. How does the system handle broad architecture questions?

**Q: If the user asks “explain the architecture,” what happens?**

The system classifies whether it is a broad conceptual architecture question or a source-led architecture question.

A broad architecture overview may use summaries, inventory, and some source evidence depending on policy. A source-led architecture question requires stronger evidence coverage and should distinguish files that are actually content-visible from files only listed in inventory.

The answer should avoid pretending the whole codebase was read if only a subset was packed.

A good phrasing:

> For broad architecture questions, the system can use summaries to orient the answer, but it still separates summary-backed structure from source-backed implementation claims.

---

## 31. How does the system handle compare questions?

**Q: What if the user asks to compare two files or components?**

The turn state can represent compare targets as a pair. The prompt builder and validator then preserve both sides of the comparison. The goal is to avoid answering only one side or pretending both sides had equal evidence when they did not.

A good answer should say things like:

```text
For file A, source evidence was packed.
For file B, only summary/inventory evidence was available.
So the comparison is stronger on A than B.
```

This is another example of packed truth being separate from policy truth.

---

## 32. How do commands fit into the system?

**Q: How are slash commands or deterministic commands handled?**

Some commands are handled deterministically without an LLM call. For example, inventory listing, plan/apply commands, tool commands, or task commands can be routed through the command dispatcher.

When a command is handled deterministically, the system still emits canonical markers and turn state so telemetry and post-commit continuity stay coherent. But it does not unnecessarily ask the model to answer something the host can answer directly.

This keeps command behavior predictable.

---

## 33. How does the system handle file edits?

**Q: What happens after a file is edited or written?**

After a tool or command modifies files, the runtime produces a post-commit effect. That effect can update:

```text
recent mutation files
recent mutation primary file
working set
active hint
artifact availability flags
```

The file context/retrieval layer can then rescan or update the index so future retrieval reflects the updated state.

The design goal is that edits affect continuity and retrieval in a structured way rather than just being buried in chat history.

---

## 34. How does the system handle conversation memory?

**Q: Does the assistant remember previous messages?**

Yes, but conversation memory is treated carefully. Conversation tail and summaries can support continuity, but they do not prove repository contents.

The system uses recent conversation when the user is clearly referring back, such as “continue,” “that file,” “what did we change,” or “previous step.” Otherwise, the latest user request remains primary.

This avoids a common LLM failure where old context hijacks the current request.

---

## 35. What is the role of inventory?

**Q: What is inventory used for?**

Inventory is a factual list or digest of the included project surface. It can prove that a file exists in the active surface or give a sense of breadth, folders, and file counts.

But inventory does not prove file contents.

So inventory can support:

```text
this file exists
these folders exist
these are the included files
the project appears to have these broad areas
```

It cannot support:

```text
this function does X
this file imports Y
this class calls Z
```

Those require source evidence.

---

## 36. How do you explain the security posture?

**Q: Are internal traces exposed to users?**

In a release posture, they should not be. The GUI should show user-facing answers and selected useful UI evidence, not raw packed prompts, prompt contracts, trace markers, or full debug artifacts.

Internal telemetry is useful for private debugging and evals, but it should be opt-in. The project was adjusted so trace exports, prompt artifacts, alias files, and console logs can be disabled by default for release.

The principle is:

```text
debug visibility for developers
minimal internal exposure for users
```

---

## 37. What would you say if someone asks if the logs were a big IP leak?

**Q: If logs were exposed, can someone recreate the system?**

No, not realistically. Logs may reveal names and concepts, but not the full source, control flow, invariants, bug history, eval failures, or implementation decisions. Someone could create a surface-level imitation, but not the carefully engineered system.

Still, logs should be disabled for release because they expose unnecessary operational details. It is not catastrophic, but it is unprofessional and avoidable.

---

## 38. What are the biggest engineering challenges you solved?

**Q: What was the hardest part?**

The hardest part was preventing semantic drift across subsystems. It is easy to build retrieval. It is much harder to make turn classification, target selection, retrieval, prompt packing, validation, telemetry, and post-commit continuity all agree about what happened.

Specific challenges included:

```text
keeping retrieval indexes aligned with the active file surface
preventing summaries from being treated as proof
separating source evidence from inventory and memory
handling recent file mutations without fabricating exact diffs
ensuring prompt builder consumes canonical policy instead of reclassifying
stopping validator/postprocess from becoming semantic repair engines
maintaining headless eval compatibility while supporting GUI usage
```

---

## 39. How do you describe the architecture in one minute?

**Q: Give me the one-minute technical overview.**

Coeus is a deterministic, repo-native LLM assistant architecture. A user turn is first compiled into a canonical turn state that defines the answer kind, output mode, evidence requirements, and target file or scope. Retrieval is policy-driven and uses a hybrid dense plus sparse search over the active file surface. Retrieved context is separated into source evidence, summaries, inventory, memory, and recent mutation planes. A context lattice packs only the allowed planes into the model prompt under budget. The model response is then validated against the canonical contract so it cannot freely claim unsupported code details. Headless mode and telemetry support private evals, while release mode suppresses internal traces and prompt artifacts.

---

## 40. How do you describe retrieval in one minute?

**Q: Give me the one-minute retrieval explanation.**

Retrieval is based on the active included project surface. Files are normalized, chunked, hashed, embedded, and indexed into both dense vector memory and sparse/BM25 search. At turn time, the canonical policy decides whether retrieval is allowed and what kind of evidence is required. The retrieval controller fetches hybrid candidates, fuses dense and sparse results, filters stale or invalid entries, and classifies results as source evidence or semantic primer. The lattice then packs those results according to policy. Source evidence can support exact implementation claims; summaries can only orient the answer; inventory can only prove existence or breadth.

---

## 41. How do you describe summaries in one minute?

**Q: How do summaries fit without causing hallucinations?**

Summaries are treated as semantic primer, not proof. They help the model understand project structure, file responsibility, or where to look, but they cannot support exact code claims. Exact claims require source evidence. This is enforced through the prompt contract, packed-plane markers, and validator logic. The system deliberately tells the model to separate content-visible files from inventory-only or summary-only files.

---

## 42. How do you describe the validator in one minute?

**Q: What does the validator do?**

The validator enforces the canonical answer contract. It checks the selected answer kind, output mode, evidence requirements, source evidence presence, diff requirements, and needs-more-context rules. It is not supposed to reinterpret the user request. It ensures that the answer matches what the host decided and what the prompt actually contained.

---

## 43. What limitations would you honestly mention?

**Q: What are the current limitations?**

The project is sophisticated but still evolving. The main limitations are:

```text
some telemetry/debug systems are private and should remain disabled in release
summary freshness can lag if async producers have not caught up
large repositories require careful surface/index management
broad architecture answers still depend on what evidence was packed
tool execution needs strict caps and safety boundaries
the GUI/release posture needs ongoing hardening
```

A good honest framing:

> The hard part is not making it answer once. The hard part is keeping every layer aligned under many turns, commands, edits, and eval scenarios. That is where the project has focused most of its engineering.

---

## 44. What makes it portfolio-worthy?

**Q: Why is this a strong portfolio project?**

It shows more than API usage. It demonstrates systems thinking, retrieval architecture, deterministic runtime design, prompt governance, validation, telemetry, GUI integration, headless evaluation, and release hardening.

It is a project about building an LLM product substrate, not just calling an LLM.

Portfolio framing:

```text
Built a C++ desktop/agent runtime for repo-grounded LLM interaction, with deterministic turn policy, hybrid retrieval, structured context packing, source-evidence discipline, validation, tool execution, telemetry, and headless eval support.
```

---

## 45. What is the safest public summary?

**Q: What should I say publicly without oversharing?**

Use this:

> I built a C++ LLM assistant runtime that works over a local project repository. It uses a deterministic turn-policy system to decide whether a request is general chat, file-specific, architecture-level, recent-change-related, or command-driven. Retrieval combines dense and sparse search over the active file surface, and context is packed into explicit planes such as source evidence, summaries, inventory, and conversation continuity. The model is constrained to distinguish actual source evidence from summaries or file listings, and responses are validated against the turn contract. I also built headless evaluation support and private telemetry for debugging, while keeping internal traces disabled in release builds.

---

# Best concise answers to memorize

## How does retrieval work?

It indexes the active included file surface into dense and sparse search. The turn policy decides whether retrieval is allowed and what evidence is required. The retrieval controller fetches hybrid candidates, fuses them, filters stale entries, and classifies results as source evidence or summary/semantic primer. The lattice then packs only the allowed evidence planes. Source evidence supports exact claims; summaries only orient.

## How do summaries work?

Summaries are semantic primer. They help with architecture and orientation, but they are not treated as proof. Exact code claims require source evidence. That prevents the model from turning a summary into a hallucinated implementation claim.

## What makes this different from normal RAG?

Normal RAG retrieves chunks and stuffs them into a prompt. This system compiles each turn into a canonical contract first, then retrieval, prompt packing, validation, and telemetry all consume that same truth. It separates what is allowed, what was packed, and what can be claimed.

## How do you prevent hallucinations?

By separating context planes. Source evidence is proof. Inventory is existence only. Summaries are orientation only. Conversation memory is continuity only. Recent mutation is change continuity only. The prompt and validator enforce those boundaries.

## What was the hardest part?

Preventing semantic drift. The challenge was making planner, retrieval, prompt builder, lattice, validator, postprocess, telemetry, and runtime continuity all agree on the same canonical turn meaning.

## Is it production ready?

The core architecture is advanced and testable, but release hardening is still important. Internal traces, packed prompts, and debug artifacts should be disabled by default. The private eval and telemetry systems remain valuable, but they should not be exposed in a public build.

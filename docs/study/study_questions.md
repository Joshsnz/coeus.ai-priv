Below is a technical interview study pack for Coeus / Context Lattice Engine. The goal is to help you answer deep-dive questions without exposing proprietary internals.

1. Core positioning answer

Use this as your default project explanation.

Coeus AI is a native C++ project-aware AI workspace. The main technical idea is that each user turn is treated like a compiled unit, not just a chat message. The system first derives a canonical contract for the turn: what kind of answer is allowed, what evidence is required, what target or scope applies, and what output shape is valid. Retrieval, prompt packing, model execution, validation, and telemetry then consume that compiled truth instead of each layer reinterpreting the user request independently.

The short version:

Coeus is not just RAG over files. It is a compiled-turn runtime for repository-aware AI assistance. Retrieval is only one input. The system separates policy truth, retrieval truth, packed prompt truth, continuity truth, and claim truth, then validates the final answer against the compiled contract.

That framing is strong because it explains the novelty without needing to reveal source.

2. “Is the turn compiler inside the Context Lattice Engine?”

Good answer:

Yes. The turn compiler is the front half of the Context Lattice Engine. The lattice is not just a visual stack of context blocks. It is the structured result of a compiled turn.

A clean breakdown:

Turn compiler
Converts raw user input into canonical turn semantics.
Retrieval policy
Decides which context types are allowed based on that compiled truth.
Lattice packing
Packs admitted context planes into the prompt budget.
Execution
Either serves a deterministic command/tool result or sends a model request.
Validation
Checks whether the answer satisfied the contract.
Export
Writes traceable artifacts for debugging/evaluation.

A good sentence:

The turn compiler decides what the turn means; the context lattice decides what is allowed to enter the model request; validation checks whether the answer obeyed the original compiled contract.

3. “How does the lattice resolve conflicts between planes?”

This is one of the most important answers.

Good answer

The lattice resolves conflicts by separating authority. Different planes can inform the answer, but they do not all have equal proof value. Source evidence can prove source-code claims. Semantic summaries can orient the model, but they are support-only unless explicitly admitted. Continuity helps route follow-up turns, but continuity is not proof. Tool observations only become model-visible when policy admits them. The packed prompt truth records what actually entered the request.

Stronger version

The engine avoids a common failure mode where chat history, summaries, retrieved chunks, and tool outputs all blur into one “context blob.” Instead, each plane carries a role:

Plane   Role    Can it prove code claims?
Policy truth    Defines answer kind, scope, output mode, evidence requirements  No, it defines the contract
Source evidence Actual source slices / file content Yes
Semantic primer Summaries, inventory, architecture map  Usually support-only
Continuity  Prior turn state, recent mutation state, plan/apply state   No, unless admitted as evidence
Tool observations   Tool outputs    Only when admitted
Packed prompt truth What actually entered the prompt    It proves what the model saw, not necessarily source truth
Claim truth Final answer obligations    Must be checked against contract/evidence
Example answer

If a summary says a file is responsible for X, but source evidence does not show that, the answer should not claim X as proven. The summary can guide retrieval or orient the model, but proof-bearing claims need source evidence. That is why I separate semantic primer from source evidence.

What to avoid saying

Do not say:

The lattice just ranks all context and picks the best chunks.

That undersells it. Better:

The lattice is authority-aware context packing, not just ranking.

4. “How is validation implemented conceptually?”
Good answer

Validation is contract-first. It does not act as a second planner. It consumes the canonical turn state and prompt markers, then checks whether the final answer satisfies the required output contract.

Validation checks things like:

Was this a repo-answer turn or a general chat/control turn?
Did the turn require source evidence?
Was source evidence actually packed?
Was the evidence contract satisfied?
Was a diff/patch plane required?
Was a full-file or patch output expected?
Was a canonical target file involved?
Was it a compare-pair turn?
Was a recent mutation bundle involved?
Did the answer leak internal artifacts, trace paths, vector-memory details, or project storage internals?
Should “needs more context” be allowed or suppressed?
Stronger version

The validator is deliberately not responsible for reinterpreting the user request. Earlier layers derive the canonical answer contract. The validator checks that the answer obeys that contract. That reduces semantic drift, because prompting, validation, and post-processing all consume the same resolved truth instead of each making its own classification.

Important nuance

Validation can strongly enforce structure, source-evidence discipline, route discipline, and artifact suppression. It cannot magically prove every semantic claim is correct in the mathematical sense. Your honest answer:

The validator does not make hallucination impossible. It reduces specific failure classes by enforcing evidence visibility, output-mode discipline, and contract alignment. The evals test whether this actually works on repository-grounded scenarios.

That sounds mature.

5. “What edge cases did you design for?”

Use this section to sound practical.

Edge case package
Named file question

Example:
“List the constants in config.py.”

Expected behavior:

Compile as a targeted source-grounded question.
Focus retrieval on config.py.
Pack source evidence from the target file.
Validate source evidence is present.
Answer only from visible file content.
Project-wide usage question

Example:
“Where is player health used across the project?”

Expected behavior:

Compile as source usage trace.
Use global project scope.
Broaden retrieval.
Prefer source evidence only.
Suppress support-only semantic summaries.
Use scan evidence when necessary.
Validate exhaustive/source-trace posture.
Startup/runtime flow question

Example:
“Explain how the program starts and reaches the game loop.”

Expected behavior:

Compile as source order trace.
Use implementation-strict posture.
Pack source evidence from multiple files.
Require ordered reasoning grounded in visible source.
Negative grounding

Example:
“Where is persistence implemented?”

If absent:

Do not invent a persistence subsystem.
Search current project files.
State that no implementation is visible.
Mention nearby state-bearing objects only as evidence of absence, not proof of hidden implementation.
Compare-pair request

Example:
“Compare player.py and world.py.”

Expected behavior:

Compile compare pair truth.
Require both files to be represented.
Avoid turning semantic comparison into a diff/patch request unless the user asks for code changes.
Command/control turns

Example:
/tool scan .

Expected behavior:

Do not invoke normal model reasoning.
Treat as host-control command.
Return deterministic result.
Export trace showing served without LLM.
Recent mutation follow-up

Example:
“What changed?” after an apply/edit.

Expected behavior:

Use recent mutation plane only if real mutation evidence exists.
Do not fabricate a preferred primary file from vague metadata.
Require diff/patch evidence when relevant.
Stale summary or missing summary

Expected behavior:

Summaries are optional support planes.
Fresh source evidence wins.
Correctness should not depend on a summary being available.

Good concise answer:

The system is designed so that missing summaries degrade quality, not correctness. Source evidence remains the authority for source-code claims.

6. “What is novel here?”

Use this carefully. Do not sound like you are claiming a mathematically new invention unless you want to defend that. Say “novel runtime abstraction” or “original architecture.”

Good answer

The novelty is not any single component like BM25, vector retrieval, or a prompt template. The novelty is the runtime abstraction: treating an AI turn as a compiled unit with separately preserved truth surfaces.

Key points:

The system derives one canonical answer contract.
Retrieval does not own turn semantics.
Prompting is projection, not reclassification.
Validation checks the same contract rather than guessing intent again.
Telemetry exports what happened as a traceable unit.
The headless eval harness tests those invariants.

Strong sentence:

The core idea is that project-aware AI assistance needs a runtime contract, not just a bigger context window.

7. “Is this just RAG?”
Good answer

No. It uses retrieval, but retrieval is only one subsystem. Traditional RAG often retrieves chunks and lets the prompt/model decide how to use them. Coeus first compiles the turn contract, then retrieval and packing obey that contract.

Differences:

Normal RAG  Coeus CLE
Query → retrieve chunks → prompt    Turn contract → retrieval policy → lattice packing → execution → validation
Retrieved text often treated uniformly  Context has authority classes
Summaries/history may blur with source  Source evidence, primer, continuity separated
Prompt often owns behavior  Prompt consumes compiled truth
Validation often weak/post-hoc  Validation checks canonical contract
Debugging is ad hoc Trace bundles export policy/packed/validation state

Good one-liner:

RAG answers “what chunks should I show the model?” Coeus also asks “what kind of answer is this allowed to be, what counts as proof, and did the final answer obey that contract?”

8. “How do you prevent hallucinations?”

Be precise. Do not overclaim.

Good answer

I do not claim hallucinations are impossible. The system reduces specific hallucination paths by making source-evidence requirements explicit, suppressing unsupported context planes, validating evidence presence, and testing negative grounding.

Mechanisms:

Source-required turns must have source evidence packed.
Semantic primers can be disabled for strict/source-led turns.
Continuity cannot silently become proof.
Tool observations are only admitted under policy.
Internal artifacts are suppressed from final answers.
Negative-grounding evals test absent-feature behavior.
Validator checks whether source evidence obligations were met.

Good sentence:

The goal is not “the model can never hallucinate.” The goal is to make unsupported claims harder to produce, easier to detect, and easier to debug.

9. “How does the system handle conflict between summaries and source?”
Good answer

Source evidence wins. Summaries are treated as semantic primers unless a specific policy admits them for a given turn. For source-code claims, summaries can guide attention, but visible source content is the proof-bearing plane.

Example:

If a file summary says the system has persistence, but the current source files do not show persistence, the answer should say persistence was not found in the current project files.

This maps directly to your negative-grounding eval claims.

10. “How does the retrieval policy work?”
Good answer

Retrieval policy is derived from the compiled turn state. It decides whether retrieval is allowed, whether global scan is allowed, whether semantic primers are allowed, whether source evidence is required, and whether the turn should be targeted, focused, or broad.

Common retrieval shapes:

Strict file: user asks about one named file.
Focused set: user mentions multiple files or compare pair.
Focused hybrid: named file plus broader context allowed.
Hybrid/global: project-wide usage, order, runtime-flow, architecture trace.
Inventory listing: list files / project structure.
No retrieval: host command or general chat.

Good sentence:

Retrieval is not a free-for-all. It is an execution strategy selected under the compiled contract.

11. “How does lattice packing work?”
Good answer

Packing chooses which admitted planes actually enter the prompt under a budget. It records packed prompt truth: what was included, whether it was support-only or authoritative, how many blocks/chunks were packed, and which source files were visible.

Important distinction:

Allowed means policy permits the plane.
Available means there is data for that plane.
Packed means it actually entered the prompt.
Authoritative means it can support final claims.
Support-only means it can orient but not prove.

Good answer:

A plane can be allowed but not packed because of budget or because it is not useful for the turn. A plane can be packed but support-only. That distinction matters for validation.

12. “How does prompt assembly avoid semantic drift?”
Good answer

The prompt layer consumes canonical turn state. It should not independently classify the user request. Prompt assembly projects the resolved policy into model instructions, selected context planes, source discipline, and output guidance.

Good sentence:

The prompt builder is a projection layer. It should not be a second planner.

If they ask what happens if prompt classification conflicts:

That is exactly the class of bug I designed against. The canonical resolved state is the source of truth, and downstream layers are expected to consume it rather than recompute it.

13. “What is the role of telemetry and export?”
Good answer

Telemetry is how the runtime proves what happened. Each meaningful turn can export markers showing the policy contract, packed planes, evidence posture, runtime path, validation result, and final-output reference.

Public-safe explanation:

It supports debugging.
It supports private evals.
It helps explain why a turn passed or failed.
It proves whether the model saw source evidence.
It detects route drift and artifact leaks.

Good sentence:

The trace is not just logging. It is the audit surface for the compiled turn.

14. “What does the headless eval harness prove?”
Good answer

It proves that the core runtime can be exercised without the GUI and that repository-grounding behavior is repeatable under scripted scenarios. It does not prove production readiness or SWE-bench-level autonomous repair.

What it tests:

Project bootstrap.
File scan/embed.
Source-grounded QA.
Runtime/source-order tracing.
Exact symbol questions.
Negative grounding.
Multi-turn continuity.
Semantic comparison vs diff routing.
Artifact suppression.
Route retention.

What it does not claim:

Full autonomous software engineering.
Broad benchmark generalization.
Production deployment readiness.
Perfect hallucination prevention.
SWE-bench performance.

Good sentence:

The evals are scoped honestly: they test repository-context behavior and route discipline, not autonomous issue repair.

15. “Why C++ / native desktop?”
Good answer

I wanted control over runtime architecture, local project state, GUI surfaces, telemetry, and eventual headless/embedded reuse. C++ also forced clear ownership boundaries and gave me a path toward native desktop performance and systems-level integration.

Possible points:

Native GUI with Skia/SDL2.
Local-first project workspace.
Deterministic runtime and task orchestration.
Headless mode can reuse core logic.
Better fit for future desktop/robotics/edge directions.
Strong signal of systems engineering ability.

Balanced caveat:

The cost is higher development complexity. But for this project, the runtime architecture was the point, not simply shipping a web chat UI.

16. “Why not LangChain / existing frameworks?”
Good answer

Frameworks are useful, but this project was about controlling the runtime semantics. I wanted explicit ownership over turn policy, prompt packing, retrieval, validation, telemetry, and tool execution. Existing frameworks can help orchestrate agents, but they often hide or blur the exact boundaries I wanted to make explicit.

Good sentence:

I was not trying to build the fastest prototype. I was trying to build a runtime where every layer has clear authority and traceability.

17. “What are the biggest trade-offs?”

Use this to sound grounded.

Trade-off   Benefit Cost
Native C++ stack    Control, performance, systems-level design  More implementation work
Proprietary core    Protects original IP    Harder for reviewers to inspect quickly
Contract-first runtime  Less semantic drift More upfront architecture
Strict evidence discipline  Better grounding    Sometimes less flexible / may refuse or narrow answers
Headless eval harness   Repeatable testing  More infrastructure to maintain
Multiple truth surfaces Better debugging    More complex mental model
Summaries as support planes Better orientation  Freshness/timing complexity

Strong sentence:

The main trade-off is complexity. I accepted that complexity to make the runtime more inspectable and less dependent on prompt luck.

18. “What failure modes did you find?”

Use your actual architecture language, but keep it public-safe.

Common failure families:

Policy-origin mistakes
The turn is classified incorrectly upstream.
Retrieval routing mistakes
Retrieval is too narrow or too broad.
Prompt-plane omission
The right evidence exists but is not packed.
Summary/inventory timing lag
A support artifact is not fresh or available yet.
Continuity overreach
Previous state is treated as proof.
Mutation truth fabrication
The system infers a changed file without real mutation evidence.
Validator compensation
Validator starts “repairing” semantic intent instead of checking contract.
Artifact leakage
Final answer exposes internal trace/vector/cache/project paths.

Good answer:

A lot of the engineering work was not adding features. It was removing ambiguity between layers so that the same turn could not mean different things to the planner, retriever, prompt builder, validator, and exporter.

19. “What did you do to control those failures?”

Map failure → control.

Failure Control
Policy drift    Canonical resolved turn state
Retrieval overreach Retrieval authority modes and scope rules
Summary over-trust  Semantic primer/support-only distinction
Missing source evidence Source evidence required flags and validator checks
Prompt reclassification Prompt policy projection from canonical state
Recent mutation fabrication Strict recent mutation presence rules
Compare-pair misses Coverage backstops and pair diagnostics
Artifact leaks  Postprocess/validator artifact suppression
Eval ambiguity  Headless trace bundles and public scorecards

Good sentence:

My fix pattern was to identify where two layers owned the same truth, then collapse that truth into one canonical source and make downstream layers consume it.

20. “What metrics should we look at?”

You can separate scale metrics, eval metrics, and behavioral metrics.

Scale metrics
364 application files analyzed.
181,781 total lines.
144,110 code lines.
36,717 reported complexity.

Say:

I use those as codebase-scale evidence, not as a quality score.

Eval metrics

RepoGrounding Core:

12 / 12 stable scenarios.
13 / 13 repository QA turns.
13 / 13 context retention.
13 / 13 artifact suppression clean.
13 / 13 diff-route discipline clean.

RepoGrounding Plus:

12 / 12 stable scenarios.
14 / 14 repository QA turns.
14 / 14 context retention.
14 / 14 artifact suppression clean.
14 / 14 diff-route discipline clean.
Behavioral metrics
Did source-required turns actually receive source evidence?
Did negative-grounding turns avoid hallucinating absent features?
Did semantic comparison avoid patch/diff misrouting?
Did repo follow-ups retain project scope?
Did the output avoid internal artifact leakage?

Good sentence:

The most important metric is not just pass/fail. It is whether the trace shows the right contract, the right packed evidence, and the right validation result.

21. “How would you walk through selected internals without exposing everything?”

Good answer:

I would show a controlled architecture/code walkthrough, not the entire repository by default. The useful review path is to walk through one turn end-to-end.

Suggested private review path:

Show a small eval fixture.
Run a source-grounded query.
Show the compiled turn state.
Show retrieval policy result.
Show packed lattice summary.
Show model/deterministic execution path.
Show validation result.
Show export bundle.
Then selectively open the relevant source modules.

Files/subsystems you can mention at a high level:

Turn context construction.
Turn planner.
Resolved turn state.
Retrieval controller.
Hybrid search / BM25 / vector memory.
Lattice/prompt packing.
Prompt policy projection.
Agent flow / run controller.
Output validator.
Answer postprocess.
CLE exporter / run traces.
Headless protocol/eval runner.

Good sentence:

I can show enough internals to prove I built it and understand it, without disclosing the full proprietary implementation publicly.

22. “Why not open source the whole thing?”
Good answer

Because the core runtime method, prompts, validator logic, trace schema, and implementation structure are original IP. The public portfolio gives enough signal to assess the project: architecture, demo binary, evaluation reports, compiled-turn examples, diagrams, and private-review availability. Full public source disclosure is not necessary for a portfolio.

Balanced addition:

I am not trying to hide the work from serious review. I am separating public demonstration from controlled technical disclosure.

Good sentence:

For a hiring process, I would be happy to do a live walkthrough or provide selected internals under an appropriate review context.

23. “How do you avoid looking too secretive?”

Say this:

I keep the public material specific enough to be evaluated: concrete architecture, diagrams, eval results, demo release, codebase metrics, and examples from traces. I keep only the parts private that would expose proprietary implementation details or secrets.

This is important because employers might worry about collaboration.

Good answer:

I am protective of the IP, but not vague about the engineering. I can explain the architecture, trade-offs, failure modes, and evaluation method clearly.

24. “What would you improve next?”

Good future roadmap:

Improve public demo polish and onboarding.
Add more visual trace summaries.
Expand eval fixtures.
Add broader multi-language repositories.
Improve source citation UX in the GUI.
Harden summary freshness and async timing.
Add better regression dashboards.
Improve private review package.
Explore safe plugin/tool API.
Add controlled local-only mode for sensitive repos.

Good sentence:

The next step is less about adding flashy features and more about making the runtime easier to inspect, test, and present.

25. “What are the limitations?”

Be honest. This builds trust.

Limitations:

Public repo does not include full source.
Eval fixtures are scoped, not universal.
It is not a SWE-bench claim.
Native desktop packaging needs polish.
Source grounding reduces hallucination risk but cannot eliminate it completely.
Large-codebase behavior needs broader external validation.
Some features are prototype/research-grade.
Summaries and semantic artifacts are support systems, not always required.
The private headless harness is not public.

Good sentence:

The project demonstrates a serious runtime architecture and working eval evidence, but I would not claim it is a finished commercial coding assistant yet.

26. Likely rapid-fire technical questions
Q: What is the Context Lattice Engine?

A compiled-turn architecture that separates turn semantics, retrieval, context packing, execution, validation, and telemetry into traceable stages.

Q: What is the turn compiler?

The component chain that converts raw user input into canonical turn state: answer kind, scope, target, evidence contract, output mode, and routing constraints.

Q: What is the lattice?

The authority-aware packed context structure. It contains admitted planes like source evidence, semantic primers, continuity, tool observations, inventory, and packed prompt truth.

Q: What is policy truth?

The canonical contract for the turn.

Q: What is retrieval truth?

What the retrieval/index/search layer actually found.

Q: What is packed truth?

What actually entered the model request.

Q: What is continuity truth?

State from previous turns, recent mutations, or plan/apply workflows. Useful for routing but not proof by itself.

Q: What is claim truth?

What the final answer is allowed to assert after validation.

Q: What makes source evidence different from semantic primer?

Source evidence is proof-bearing file content. Semantic primer is support context like summaries or inventory.

Q: Why does that matter?

Because summaries can be stale or incomplete. Code claims should be grounded in visible source evidence.

Q: What happens when retrieval misses?

The system should either broaden retrieval if policy allows, inject coverage backstops for required targets, or avoid unsupported claims.

Q: What happens when no evidence is available?

The answer should ask for more context or state that the evidence is not available, depending on the contract.

Q: How do you test route discipline?

With headless scenarios that check whether semantic questions stay semantic, patch/diff requests route correctly, repo follow-ups retain scope, and internal artifacts are suppressed.

Q: How do you test negative grounding?

Ask about absent features in small repos with distractor symbols, then check whether the system refuses to invent implementation.

Q: How do you know the model saw the right files?

Packed prompt truth and export markers record visible source files/chunks.

Q: What is served without LLM?

Host-control commands like scan/embed can be deterministic command turns.

Q: What does the validator do if the answer violates the contract?

Depending on the violation, postprocess/validation can suppress, repair presentation, require safer output, or flag the trace. The important point is that validator checks the compiled contract rather than inventing a new one.

27. The “deep-dive” answer for the lattice resolving conflicts

Use this if someone asks directly.

The main conflict problem is that several layers can contain useful but different kinds of truth. A previous conversation may imply one thing, a summary may imply another, retrieval may find different source files, and the model may want to generalize. Coeus handles that by not flattening everything into generic context. The turn compiler derives a canonical contract first. Then each context plane is admitted with an authority role. Source evidence can support code claims. Semantic primers can orient but are not automatically proof. Continuity can preserve state but cannot silently prove current source behavior. Tool observations are only admitted when policy allows. Finally, packed prompt truth records what actually entered the request. The validator then checks the output against the original contract and the packed evidence posture. So the conflict is handled structurally: authority is assigned before prompting, not improvised by the model after retrieval.

That is probably one of your best answers.

28. The “deep-dive” answer for validation

Validation is contract-first. It consumes the canonical turn state and prompt/export markers. It checks whether the turn was supposed to be general chat, a source-grounded architecture answer, a source usage trace, a source order trace, a patch/diff, a full-file response, or another answer family. Then it checks whether required planes were present: source evidence, diff plane, recent mutation evidence, compare-pair coverage, and so on. It also checks user-facing discipline, such as suppressing internal trace artifacts. The validator is intentionally not the planner. If validation has to reinterpret the user request, that means upstream policy truth has already drifted. The design goal is that planner, retrieval, prompt builder, validator, postprocess, and exporter all consume the same canonical contract.

29. The “deep-dive” answer for evals

The headless harness lets me run the same core assistant behavior without the GUI. Each scenario creates a project, sets a root, scans files, embeds/indexes them, then sends source-grounded user turns. The logs show whether the turn was classified correctly, what retrieval mode was used, what source files were packed, whether semantic primers were suppressed or admitted, whether the answer required source evidence, and whether validation was satisfied. The public RepoGrounding reports summarize those results without exposing the private trace schema or harness code.

30. The strongest final interview framing

Use this near the end of a technical screen:

The project is valuable because it shows I can build beyond prompt engineering. I designed a native app, a project context system, retrieval, tool execution, prompt packing, validation, telemetry, and a headless eval loop. The core lesson is that reliable AI tooling needs runtime architecture: contracts, evidence boundaries, state management, validation, and traceability. Coeus is my attempt to build that architecture from first principles.

That is the main story.
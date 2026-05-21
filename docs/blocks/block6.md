Block 6 Summary — LLM Provider Registry, Remote/Local Runners, and Model-Call Boundary

Block 6 defines the LLM provider substrate. This layer is intentionally lower-level than policy, retrieval, prompt assembly, validation, and runtime orchestration. It owns provider configuration, model lists, runner selection, request transport, response parsing, and local-process execution.

The strongest reviewer-facing point is:

Coeus separates model transport from turn semantics. The LLM layer does not decide what the user asked, what files to retrieve, what evidence is valid, or how prompts should be assembled. It receives an already-built LlmRequest and executes it through either an HTTP JSON runner or a local process runner.

1. Batch-level role

Block 6 has four layers:

llm.h
  router header

provider_manager.*
  persistent provider/model registry
  remote/local provider config
  runner factory

runner_spec.h / ilm_runner.h
  shared request/response structs
  common runner interface

http_json_runner.*
  remote HTTP JSON model call path

local_process_runner.*
  local executable model call path

The architecture is:

Agent/runtime/prompt layer
  → builds LlmRequest

Provider / Runner layer
  → selects transport
  → performs call
  → returns LlmResponse

Downstream runtime
  → validates/postprocesses/records response

This is a clean boundary. The LLM layer is a transport abstraction, not the brain of the agent.

2. File-by-file role map
core/llm/llm.h

Role: Small router header.

It includes:

#include "core/llm/provider_manager.h"
#include "core/llm/runners/runner_spec.h"

This is just a convenience import surface for provider and runner definitions.

core/llm/provider_manager.h/cpp

Role: Persistent provider/model registry and runner factory.

This is the main config layer for model providers.

Provider model

The provider system supports two provider kinds:

Remote
  RemoteCfg
    apiKeyName
    apiKeyValue
    baseUrl

Local
  LocalCfg
    exePath

A Provider contains:

providerID
displayName
ProviderCfg
RunnerSpec
models[]

Each model can be either legacy/simple:

"gpt-4.1-mini"

or object-shaped with caps:

{
  "modelName": "gpt-4.1-mini",
  "contextWindow": 1047576,
  "maxOutputTokens": 32768
}

This gives the app a way to record per-model context/output limits without hardcoding everything into runtime.

Provider persistence

Providers are loaded from and saved to:

PathUtils::ProvidersJson()
build/config/providers.json

The expected JSON shape is:

{
  "providers": [
    {
      "id": 1,
      "displayName": "OpenAI",
      "kind": "remote",
      "baseUrl": "https://api.openai.com",
      "apiKeyName": "OPENAI_API_KEY",
      "apiKeyValue": "",
      "models": ["gpt-4.1-mini"],
      "runner": {
        "kind": "HttpJson",
        "endpoint": "https://api.openai.com/v1/chat/completions",
        "maxContext": 128000
      }
    }
  ]
}
Runner factory

Provider::createRunner() creates:

RunnerSpec::Kind::HttpJson
  → HttpJsonRunner

RunnerSpec::Kind::LocalProcess
  → LocalProcessRunner

This is the main remote/local boundary.

Endpoint helpers

LlmEndpoint builds conventional endpoints:

JoinChatCompletionsEndpoint()
JoinEmbeddingsEndpoint()

It handles base URLs like:

https://api.openai.com
https://api.openai.com/v1
https://api.openai.com/v1/chat/completions
Safety / sanitization

The manager sanitizes:

provider display names
base URLs
model names
runner endpoints
request templates
API key names/values

It also strips CR/LF from header-sensitive values and drops placeholder API keys like:

your_api_key
openai_api_key
${openai_api_key}
<openai_api_key>
changeme
replace_me
Missing/invalid config behavior

The header explicitly states:

No silent provider bootstrapping on failed load:
missing/invalid file -> empty registry, UI/runtime decide what to do next.

That is important. createDefaultProviders() exists, but the supplied load path does not silently call it when config is missing or invalid.

What this proves:
Coeus has a persistent provider registry with remote/local provider support, model metadata, runner specifications, endpoint normalization, and safe empty-registry behavior.

Reviewer-facing wording:
“ProvidersManager is the persistent model-provider registry. It loads and saves remote/local providers, model lists, optional model caps, runner specs, and endpoints, and it creates the appropriate runner without owning policy, retrieval, or prompt semantics.”

core/llm/runners/runner_spec.h

Role: Shared request/response and runner configuration contract.

Defines:

RunnerSpec
LlmMessage
LlmRequest
LlmResponse
RunnerSpec
kind: HttpJson | LocalProcess
endpoint
requestTmpl
maxContext
LlmRequest
RequestTag tag
baseUrl
apiKey
model
temperature
maxTokens
messages[]
LlmResponse
role
content
toolCall

The important design note:

RequestTag is LOG-ONLY. It must never affect serialized JSON payload.

This is a clean trace/correlation boundary. The model call can be correlated in logs without affecting prompt contents.

core/llm/runners/ilm_runner.h

Role: Common runner interface.

Defines:

class ILlmRunner
{
public:
    virtual std::string name() const = 0;
    virtual LlmResponse call(const LlmRequest& in) = 0;
    virtual int maxContext() const { return 32000; }
};

This is the common abstraction for both remote HTTP and local process execution.

What this proves:
Model invocation is behind a simple polymorphic interface. The runtime does not need to know whether the model is remote HTTP or local executable once it has a runner.

core/llm/runners/http_json_runner.cpp/h

Role: Remote HTTP JSON chat-completions runner.

This runner takes an LlmRequest, builds an OpenAI-style chat JSON body, sends it to an HTTP endpoint with libcurl, and parses the response.

Request construction

It serializes:

{
  "model": "...",
  "temperature": 0.7,
  "max_tokens": 1024,
  "messages": [
    { "role": "system", "content": "..." },
    { "role": "user", "content": "..." }
  ]
}

It sanitizes role/content/model fields before serialization.

Endpoint composition

It supports:

spec endpoint
per-call baseUrl override
base URL or full endpoint override

The endpoint composer avoids common duplication like:

https://host/v1 + /v1/chat/completions
HTTP behavior

It uses libcurl with:

POST JSON
10s connect timeout
30s total timeout
HTTP/2 TLS where available
SSL peer and host verification enabled
redirects limited to 3
TCP keepalive
Expect: disabled to avoid 100-continue stalls
Bearer authorization header if an API key is supplied
Response parsing

It handles:

OpenAI-style choices[0].message.content
completion-style choices[0].text
error.message
unknown response shape fallback to raw sanitized JSON

It also captures:

tool_calls
function_call

into LlmResponse::toolCall.

Logging

It logs:

request correlation prefix from RequestTag
endpoint and payload byte count
HTTP status
response byte count
response head preview
final content preview

It does not intentionally log the API key.

What this proves:
The remote runner is a concrete HTTP transport implementation, not a policy engine. It receives model/messages and returns parsed content/tool-call fields.

Reviewer-facing wording:
“HttpJsonRunner is the remote model transport. It converts LlmRequest into an OpenAI-style chat-completions JSON payload, sends it through libcurl with TLS verification and timeouts, parses common response shapes, and returns an LlmResponse.”

core/llm/runners/local_process_runner.cpp/h

Role: Local executable runner.

This runner executes a local command/process as an LLM backend.

Behavior

It:

Takes the last message content as the prompt.
Writes that prompt to a temp input file.
Runs the configured endpoint command as a child process.
Redirects stdin/stdout/stderr through temp files.
Waits up to 30 seconds.
Kills the process on timeout.
Reads stdout/stderr with byte caps.
Returns stdout if present, otherwise stderr, otherwise a structured error string.
Cleans temp files in a RAII destructor.
Platform behavior

On Windows:

uses CreateProcessW
creates temp file handles
redirects stdio
runs with CREATE_NO_WINDOW

On POSIX:

uses fork
redirects file descriptors
uses execvp
waits with timeout and SIGKILL
Bounds

Reads are bounded:

stdout: 512 KiB
stderr: 128 KiB
timeout: 30 seconds
Important limitation

Unlike HttpJsonRunner, which serializes all messages, LocalProcessRunner sends only the last message content to the subprocess:

const std::string prompt = in.messages.back().content;

So local runners are prompt-text process backends, not full chat-message transport unless upstream packages the full prompt into the last message.

What this proves:
Coeus supports local model/process execution as a separate runner path, with bounded IO, timeout behavior, and platform-specific process management.

Reviewer-facing wording:
“LocalProcessRunner allows a configured local executable to act as a model runner. It writes the final prompt to stdin, captures bounded stdout/stderr, enforces a timeout, and returns sanitized process output as the model response.”

3. Provider / runner control flow

The model-call path looks like this:

Runtime / agent flow
  → selects provider + model
  → Provider::createRunner()
      if remote:
        HttpJsonRunner
      if local:
        LocalProcessRunner
  → builds LlmRequest
      baseUrl
      apiKey
      model
      temperature
      maxTokens
      messages
      RequestTag for logs only
  → runner.call(request)
  → LlmResponse
      role
      content
      toolCall

Provider config flow:

build/config/providers.json
  → ProvidersManager::loadProviders()
      → Provider[]
      → model list/caps
      → runner specs
  → Provider editor / runtime selection
  → ProvidersManager::saveProviders()
4. Remote vs local boundary
Remote provider

Remote provider fields:

baseUrl
apiKeyName
apiKeyValue
runner kind = HttpJson
endpoint = chat completions endpoint

Runtime call:

LlmRequest
  → HttpJsonRunner
  → HTTPS POST JSON
  → parse JSON response

Best for:

OpenAI-compatible APIs
cloud providers
local HTTP-compatible servers
Local provider

Local provider fields:

localExePath
runner kind = LocalProcess
endpoint = executable/command path

Runtime call:

LlmRequest.messages.back().content
  → temp stdin file
  → child process
  → stdout/stderr
  → LlmResponse.content

Best for:

local model wrappers
CLI-based inference tools
experimental local processes
Shared contract

Both implement:

ILlmRunner::call(const LlmRequest&)
ILlmRunner::maxContext()
ILlmRunner::name()

So the agent runtime can treat them similarly after construction.

5. Isolation from policy/retrieval

This block shows good isolation.

What the LLM layer does
loads provider config
stores provider/model metadata
creates HTTP/local runners
sends already-built messages
parses model responses
reports max context
logs correlated transport events
What the LLM layer does not do
does not classify user intent
does not choose retrieval targets
does not inspect project inventory
does not decide evidence eligibility
does not build context lattice planes
does not validate source claims
does not postprocess answers
does not mutate project state

The only policy-adjacent type in this block is RequestTag, but its comment is explicit:

LOG ONLY; never serialized.

That means correlation metadata does not leak into prompt payload.

This supports the architecture principle:

Provider/runner code should be downstream of policy and prompt assembly. It is transport, not turn semantics.

6. Secrets/config boundary

This block is very important for private review packaging.

Sensitive files

The main sensitive config file is:

build/config/providers.json

It may contain:

remote provider base URLs
apiKeyName
apiKeyValue
local executable paths
custom model names
runner endpoints
request templates

The private review package should exclude real provider config or include a redacted/sample version.

API key handling

The code supports two concepts:

apiKeyName
  e.g. OPENAI_API_KEY

apiKeyValue
  actual key value, if stored directly

provider_manager sanitizes placeholder values and CR/LF injection, but real apiKeyValue can still be persisted if configured.

So the review boundary should say:

Never include real providers.json if it contains apiKeyValue.
Use providers.example.json with placeholders.
Runner logs

HttpJsonRunner logs endpoint and response previews, not the bearer token. That is good. But logs may still contain:

endpoint URLs
model names
response text
error messages
prompt/response previews

So logs should also be considered private unless curated.

Local process config

Local provider config may contain:

local executable path
command-line arguments
machine-specific paths

These should also be redacted or replaced in public/private review samples if they reveal local machine details.

Request templates

RunnerSpec::requestTmpl is persisted and sanitized. In this block, HttpJsonRunner does not appear to consume it. But it can still contain custom payload templates or provider-specific details, so treat it as config to review/redact.

7. What this proves about Coeus
Proven: provider abstraction exists

Safe claim:

Coeus has a persistent provider registry supporting remote and local providers, model lists, optional model caps, and runner specs.

Proven: remote/local execution is abstracted

Safe claim:

Model calls are routed through an ILlmRunner interface, with HTTP JSON and local process implementations.

Proven: model transport is isolated from policy/retrieval

Safe claim:

The provider/runners receive an already-assembled LlmRequest; they do not own retrieval, turn classification, evidence, or validation decisions.

Proven: config is runtime-editable/persistent

Safe claim:

Providers and models are loaded from and saved to providers.json, with support for remote base URLs, local executable paths, and model metadata.

Proven: safety/robustness considerations exist

Safe claim:

The HTTP runner sanitizes payload fields, strips unsafe header newlines, uses TLS verification and timeouts, and parses failure responses. The local process runner uses bounded temp-file IO and timeouts.

8. Architecture strengths
Clean ILlmRunner boundary

The interface is small and clear:

name
call
maxContext

That makes remote/local execution swappable.

Provider registry avoids hardcoded runtime config

Providers are persisted in config and can be edited by UI/runtime. This supports user-configurable model backends.

No silent provider creation on bad config

This is good for professional behavior: bad/missing providers.json leaves the registry empty and lets UI/runtime handle it.

Transport logs are correlated

RequestTag gives a shared log prefix for runner-owned logs without affecting serialized model payloads.

Defensive input/output sanitization

The layer uses Utf8::Sanitize broadly and strips CR/LF from header values.

9. Complexity / risk notes
apiKeyValue can be persisted

Even though placeholder keys are dropped and headers are sanitized, the design allows real API keys to live in providers.json. That is fine for local-first software, but for review packaging it is a hard exclusion/redaction point.

requestTmpl is persisted but not used by HttpJsonRunner

RunnerSpec supports requestTmpl, and Provider::toJson() saves it, but the supplied HttpJsonRunner builds a fixed OpenAI-style JSON body. If custom templates are expected, they are not proven by this block.

Professional wording:

“The runner spec reserves a request-template field, but the provided HTTP runner currently uses a fixed chat-completions-style payload.”

Local runner receives only the last message

The local process runner sends in.messages.back().content, not the full message list. If local backends need full chat context, upstream must flatten/package the prompt into the final message.

Local process endpoint is sensitive and potentially powerful

A local provider can execute a configured command. That is expected for local-process model execution, but the config must be trusted and excluded/redacted in review packages.

No streaming in this block

Both runners appear request/response based. Streaming may exist elsewhere, but it is not proven here.

Response parsing is OpenAI-ish

HttpJsonRunner supports OpenAI-style chat responses, completion-style fallback, and error objects. It does not prove broad provider-specific protocol support beyond compatible JSON shapes.

10. Reviewer questions and suggested answers
Q: How does Coeus support different model providers?

A:
Providers are stored in a persistent registry. Each provider can be remote or local, has a model list, optional model context/output caps, and a runner spec. The provider creates either an HTTP JSON runner or a local process runner.

Q: Are provider calls mixed with retrieval logic?

A:
No. The LLM layer receives an already-built LlmRequest containing model parameters and messages. Retrieval, prompt packing, policy, and validation happen upstream/downstream in other subsystems.

Q: What is the remote runner?

A:
HttpJsonRunner sends an OpenAI-style chat-completions JSON payload over HTTPS using libcurl and parses common response shapes into LlmResponse.

Q: What is the local runner?

A:
LocalProcessRunner runs a configured local command, sends the final prompt through stdin/temp files, captures bounded stdout/stderr, enforces a timeout, and returns the process output.

Q: Where are secrets stored?

A:
Provider config is loaded from build/config/providers.json. It can contain API key names and, if configured, API key values. Real provider config should be excluded from review packages or replaced with redacted examples.

Q: Does the request tag affect the model prompt?

A:
No. RequestTag is explicitly log-only and must never be serialized into the JSON payload.

11. Notes to carry into CODEBASE_OVERVIEW.md
The LLM layer provides a provider and runner substrate rather than owning agent semantics. `ProvidersManager` loads and saves a persistent provider registry from `providers.json`, supporting remote providers, local process providers, model lists, optional model context/output caps, and runner specs. Providers instantiate a common `ILlmRunner` interface with either `HttpJsonRunner` for HTTP chat-completions-style APIs or `LocalProcessRunner` for CLI/local executable backends. The runner request type carries model parameters, messages, base URL/API key fields, and a log-only request tag. This layer performs transport and response parsing only; it does not classify turns, retrieve source context, decide evidence eligibility, build prompts, or validate claims.
12. Notes to carry into ARCHITECTURE_NOTES.md
Block 6 LLM architecture

Provider registry:
  ProvidersManager
    loads/saves build/config/providers.json
    remote providers:
      baseUrl
      apiKeyName
      apiKeyValue
    local providers:
      exePath
    models:
      modelName
      optional contextWindow
      optional maxOutputTokens
    runnerSpec:
      kind
      endpoint
      requestTmpl
      maxContext

Runner interface:
  ILlmRunner
    name()
    call(LlmRequest)
    maxContext()

Shared request:
  LlmRequest
    RequestTag tag       // log-only, not serialized
    baseUrl
    apiKey
    model
    temperature
    maxTokens
    messages[]

Remote path:
  HttpJsonRunner
    builds OpenAI-style JSON
    libcurl POST
    TLS verify
    timeout
    parse choices/message/content/tool_calls

Local path:
  LocalProcessRunner
    sends last message content to local process stdin
    captures stdout/stderr
    timeout + bounded reads
    returns output as content

Boundary:
  upstream policy/retrieval/prompt builder produces messages
  provider/runner only transports them
  downstream runtime/validator handles response discipline
13. Private review boundary notes

Add this to PRIVATE_REVIEW_BOUNDARY.md:

Exclude or redact provider configuration.

Do not include real `build/config/providers.json` in the review package if it contains API keys, private base URLs, local executable paths, custom endpoints, request templates, or machine-specific provider settings. Provide a `providers.example.json` instead.

Sensitive fields include:

- `apiKeyValue`
- private `baseUrl`
- local `exePath`
- custom `runner.endpoint`
- `runner.requestTmpl`
- private model/provider names if they reveal vendor/client details

Runner logs should also be treated as private unless curated, because they may include endpoints, model names, request/response previews, and provider error messages.
14. Portfolio-facing summary
Block 6 defines Coeus AI’s model-provider substrate. Providers are stored in a persistent registry and can represent either remote HTTP APIs or local executable model backends. Each provider has model metadata and a runner spec, and model calls go through a shared `ILlmRunner` interface. The HTTP runner sends chat-completions-style JSON through libcurl with TLS verification and timeouts, while the local runner executes a configured process with bounded temp-file IO and timeout handling. This layer is deliberately isolated from turn policy, retrieval, prompt assembly, evidence semantics, and validation.

Stronger version:

The Block 6 source shows that Coeus separates model transport from agent intelligence. Provider configuration, model lists, remote HTTP calls, and local process execution live behind a runner abstraction, while policy, retrieval, source-grounding, prompt packing, and validation remain outside the LLM transport layer. This keeps the model provider system swappable without letting provider code become another source of turn semantics.
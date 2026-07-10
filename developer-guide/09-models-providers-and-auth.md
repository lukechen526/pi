# Models, providers, streaming, and authentication

Pi separates four concerns that are often conflated in smaller applications:

```text
model metadata   -> what a model can do and how much it costs
provider config  -> where/how requests are sent
credentials      -> how a provider is authenticated
API adapter       -> how normalized Pi context becomes provider protocol
```

The separation lets the coding agent present one model selector and one agent
loop while supporting OpenAI-compatible servers, Anthropic, Google, Bedrock,
OAuth, proxies, and extension-defined providers.

Key source files are `packages/ai/src/types.ts`, `src/compat.ts`,
`src/models.ts`, `packages/coding-agent/src/core/model-registry.ts`, and
`src/core/auth-storage.ts`.

## The `Model` contract

A model is a data record consumed by the agent and provider layer. Its important
fields include provider identifier, model ID, API type, input capabilities,
reasoning/thinking support, context window, maximum output tokens, cost, base
URL, headers, and compatibility flags.

The model's `api` field chooses an adapter such as `openai-completions`,
`openai-responses`, `anthropic-messages`, `google-generative-ai`, or
`bedrock-converse-stream`. The provider name is not enough: two provider names
can use the same wire protocol, and one provider can expose models with
different API requirements.

Use `input: ["text", "image"]` only when a model can actually receive image
content. That metadata drives UI/tool behavior and protects against sending an
image to a text-only model.

## Model registry assembly

`ModelRegistry` begins with generated built-in provider/model data, applies
`models.json` provider and per-model overrides, merges custom models, applies
OAuth-provider model modifications where required, and finally applies dynamic
extension provider registrations.

```text
built-in provider catalogs
  -> models.json provider overrides
  -> models.json per-model overrides
  -> custom models merged/upserted
  -> OAuth model adjustment
  -> extension registerProvider configuration
  -> current ModelRegistry.models
```

The registry preserves built-in models even when `models.json` is invalid; it
records the load error for the interactive application to display. That is a
useful resilience property: a typo in a local model file should not make all
known providers disappear.

`refresh()` resets dynamic API/OAuth registrations, reloads disk config, then
reapplies registered extension providers. This prevents stale registrations
from surviving a model configuration reload.

## Selecting an initial model

At session construction, `createAgentSession()` first tries the model recorded
in a nonempty session, then consults scoped/default models through
`findInitialModel()`. It restores or chooses a thinking level and clamps it to
the selected model's capability. A missing configured model becomes a helpful
fallback message rather than an unexplained null choice.

The selection is persisted with `model_change` and `thinking_level_change`
entries. Resume is therefore not simply "use today's global default"; it
attempts to continue the model choice that created historical context.

## Thinking levels are Pi vocabulary

Pi's model-facing thinking levels are `off`, `minimal`, `low`, `medium`,
`high`, `xhigh`, and `max`. Providers use different APIs and vocabulary. A
model's `thinkingLevelMap` maps Pi levels to provider-specific values and can
set a level to `null` when unsupported.

```json
{
  "id": "custom-reasoner",
  "reasoning": true,
  "thinkingLevelMap": {
    "minimal": null,
    "low": null,
    "medium": null,
    "high": "default",
    "xhigh": null,
    "max": "maximum"
  }
}
```

This is more precise than a boolean `reasoning` flag. A selector can hide
unsupported levels, and the session can clamp a requested level instead of
serializing a value the upstream provider does not understand.

## Custom models in `models.json`

The conventional file is `~/.pi/agent/models.json`. A minimal local
OpenAI-compatible provider looks like this:

```json
{
  "providers": {
    "local": {
      "baseUrl": "http://localhost:11434/v1",
      "api": "openai-completions",
      "apiKey": "local-placeholder",
      "models": [{ "id": "local-coder" }]
    }
  }
}
```

The placeholder key is deliberate for local services that ignore it. Pi uses
configured-auth status when deciding whether to offer a model, so a provider
with no auth source would otherwise be unavailable even if its server accepts
unauthenticated requests.

A full model definition can set human name, `reasoning`, inputs, context window,
maximum tokens, cost rates, and compatibility behavior. Cost rates are per
million tokens; tiers can replace the whole request's rate set above an input
threshold. This metadata affects footer/session accounting, not the provider's
bill itself.

## Provider overrides and model overrides

There are two different ways to change a built-in provider:

| Change | Configuration | Effect |
|---|---|---|
| Route existing models through a proxy | provider `baseUrl`/`headers`, no `models` | Keep built-in catalog, change request configuration |
| Add/replace custom models | provider `models` array | Merge/upsert with built-ins for models.json behavior |
| Adjust a known model only | `modelOverrides` keyed by model ID | Preserve catalog, change selected metadata |
| Replace dynamic extension provider catalog | `pi.registerProvider(..., { models })` | Replace models for that dynamic provider |

The last distinction is easy to miss: the extension `registerProvider` API
treats supplied `models` as a replacement for that provider's current models,
while `models.json` can merge custom models into a built-in provider. Use the
mechanism whose semantics you intend.

## Config values and secrets

Provider `apiKey` and header values can be literals, `$ENV_VAR`/
`${ENV_VAR}` interpolation, or a leading `!command` whose stdout becomes the
value. `$$` escapes a dollar and `$!` escapes a leading bang.

```json
{
  "apiKey": "$TEAM_PROVIDER_KEY",
  "headers": {
    "x-tenant": "$TENANT_ID",
    "x-token": "!security find-generic-password -ws team-provider"
  }
}
```

Command values resolve at request time. Pi intentionally does not guess a
cache/TTL policy for arbitrary credential commands. If a command is costly or
rate limited, provide a wrapper with the caching semantics your environment
needs. Do not assume checking model availability runs the command; auth-status
checks deliberately avoid executing command-backed values.

## Credential precedence and storage

`AuthStorage` persists API-key and OAuth credentials in `auth.json` under the
agent directory. Its file backend creates parent directories with restrictive
permissions and writes the file mode as `0600`. It uses a lock around reads and
writes so multiple Pi processes do not race during token refresh.

Credentials may come from several places. The registry first asks `AuthStorage`
for a provider key without its broad fallback, then uses configured provider
API-key values when available. Runtime `--api-key` overrides are in memory
only. Provider-scoped environment values can override process environment for
config expansion.

The result is not merely a string. `getApiKeyAndHeaders()` produces either an
error or an API key, merged provider/model headers, and provider-scoped
environment values. If `authHeader: true` is configured, the registry adds an
Authorization Bearer header from the resolved API key and fails clearly when it
is absent.

## Provider headers and request layering

Request headers are layered in practical order:

1. Model-level static headers.
2. Resolved provider headers from configuration.
3. Resolved model-specific request headers.
4. Coding-agent attribution/session/request headers.
5. Extension `before_provider_headers` mutations.
6. Provider adapter defaults and transport-specific handling.

Later sources can override earlier sources; a null header value can suppress a
provider default where the adapter supports it. Avoid logging this merged map:
it can contain secrets as well as useful tracing IDs.

## The unified streaming API

`packages/ai/src/types.ts` defines a provider-independent `Context` and an
`AssistantMessageEventStream`. `StreamOptions` carries abort signal, key,
transport preference, cache retention, session ID, headers, timeout, retry,
payload/response hooks, metadata, and provider-scoped environment.

`streamSimple(model, context, options)` in `src/compat.ts` is a dispatch point:

```text
model selected by coding-agent
  -> built-in model dispatch, or API provider lookup by model.api
  -> provider.streamSimple(model, Context, options)
  -> normalized AssistantMessageEventStream
```

Individual API adapters serialize messages/tools/system prompt into their wire
format, parse streaming chunks, and emit normalized partial/final events. The
agent loop therefore does not need a branch for every vendor protocol.

## Payload and response hooks

`StreamOptions` has `onPayload` and `onResponse`. Coding-agent connects these
to extension events. `before_provider_request` sees the provider-specific
payload after an adapter serializes Pi context. `after_provider_response` sees
status/headers after HTTP response receipt and before body consumption when the
provider exposes them.

Use these for narrow instrumentation or compatibility experiments. A broad
payload rewrite is fragile because it takes ownership of a provider protocol.
If the endpoint genuinely differs from supported APIs, use a custom provider
stream implementation instead.

## Transport, timeout, and retry

Pi's shared `Transport` values are `sse`, `websocket`, `websocket-cached`, and
`auto`. Providers that do not support a choice ignore it. HTTP idle timeout,
WebSocket connection timeout, provider SDK retry count, and max retry delay are
separate settings because they govern different failures.

The coding-agent stream function applies settings and calls `streamSimple` with
an effective timeout. It treats a configured zero HTTP idle timeout as an
effectively disabled finite timeout for SDKs that interpret zero as immediate
expiry. Agent-level retry happens later in `AgentSession`, after it receives a
final failed assistant message, and is separate from SDK/provider retries.

## Custom providers from an extension

`pi.registerProvider(name, config)` can override endpoint/headers for an
existing provider without model replacement, register a new provider with model
definitions, register OAuth callbacks for `/login`, or register a custom
`streamSimple` implementation for a nonstandard API.

Provider config validation rejects incoherent registration: models require a
base URL and either API-key config or OAuth; each model must resolve an API
type. An async extension factory is the correct place for remote model
discovery because Pi waits for it before startup/model listing continues.

When unregistering, the registry refreshes and reapplies remaining registered
providers so built-in models/protocol behavior previously overridden by that
provider are restored.

## Provider debugging workflow

When a model does not appear or a request fails, work in this order:

1. Inspect `ModelRegistry.getError()` for a `models.json` parse/load error.
2. Confirm provider/model ID and API type.
3. Check `getProviderAuthStatus()` rather than printing secrets.
4. Confirm `baseUrl`, `compat`, input capabilities, and thinking map.
5. Use `before_provider_request` only to inspect a minimal sanitized payload.
6. Check `after_provider_response` status/headers where supported.
7. Distinguish provider error/timeout from agent-level retry/compaction behavior.

This preserves Pi's architecture: the model registry owns configuration and
credentials; the adapter owns protocol translation; the agent owns turn flow.

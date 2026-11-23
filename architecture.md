# TinLynk Architecture

Components, responsibilities, protocol, and relationships between Chat, Connect, Protocol, SDK, and .github.

> Canonical names & IDs
> - Platform: TinLynk
> - Apps: TinLynk Chat (“Chat”), TinLynk Connect (“Connect”)
> - Packages: @tinlynk/protocol, @tinlynk/sdk
> - Repos (GitHub org: tinlynk): chat, connect, sdk, protocol, .github
> - Service IDs: dev.tinlynk.connect

This document distinguishes the **target architecture** from **verified implementation**. Mocked tests cover local mappings; live provider and harness compatibility remains unverified.

## 1) High-level flow

```text
User
  │
  ▼
TinLynk Chat ── @tinlynk/sdk ── TinLynk Connect ── provider adapter ── provider
                     │                 │
                     └── @tinlynk/protocol contracts ──┘
```

- **Chat** is the end-user app and agent runtime.
- **Connect is the required gateway** for app model traffic. The SDK sends requests to Connect; it does not call provider APIs directly.
- **Connect** owns provider credentials, upstream calls, and provider-specific adapters/mappings.
- **Protocol** defines the TinLynk app-facing contract. The initial target is a text-and-tools subset, with room to expand.
- **.github** provides shared CI/CD workflows and policies.

## 2) Roles and dependency rules

| Component | Responsibility | Target dependency |
|---|---|---|
| **Chat** (`chat`) | Conversations, history, agent configuration, tool consent, and rendering | `@tinlynk/sdk`, `@tinlynk/protocol`; model traffic through Connect |
| **Connect** (`connect`) | Required local gateway, provider credentials, model configuration, and provider adapter ownership | `@tinlynk/protocol`; provider integrations remain behind Connect |
| **@tinlynk/sdk** (`sdk`) | Typed app client, Connect transport, and tool invocation support | `@tinlynk/protocol` |
| **@tinlynk/protocol** (`protocol`) | Versioned app-facing types and wire contracts | none |
| **.github** (`.github`) | Shared build, test, lint, release, and security workflows | n/a |

Target dependency rules:
- `protocol → none`
- `sdk → protocol`
- `connect → protocol` (not SDK; provider integration belongs in Connect)
- App clients such as Chat → SDK and protocol; all app model requests go via Connect.

These are target boundaries. Current package dependencies and behavior are summarized in [Implementation status](#8-implementation-status).

## 3) TinLynk protocol: `@tinlynk/protocol`

The protocol is the canonical contract between apps, SDK, and Connect. It is not a provider API shape. The initial target covers text and application-executed tools only; multimodal content is explicitly outside the initial contract.

### Protocol message shape

Option B uses **ordered typed content parts**. The protocol package exports the following contract:

```ts
type Role = "system" | "user" | "assistant" | "tool";

type ContentPart =
  | { type: "text"; text: string }
  | {
      type: "tool_call";
      callId: string;
      name: string;
      arguments: unknown;
    }
  | {
      type: "tool_result";
      callId: string;
      result: unknown;
      isError?: boolean;
    };

interface Message {
  id: string;
  role: Role;
  content: ContentPart[];
  timestamp: string;
  metadata?: Record<string, unknown>;
}

const messages: Message[] = [
  {
    id: "a1",
    role: "assistant",
    content: [
      { type: "text", text: "I'll check that." },
      {
        type: "tool_call",
        callId: "call-1",
        name: "fs.readFile",
        arguments: { path: "README.md" },
      },
    ],
    timestamp: "2026-11-23T12:00:00Z",
  },
  {
    id: "t1",
    role: "tool",
    content: [
      { type: "tool_result", callId: "call-1", result: { text: "..." } },
    ],
    timestamp: "2026-11-23T12:00:01Z",
  },
  {
    id: "a2",
    role: "assistant",
    content: [{ type: "text", text: "Here is the summary." }],
    timestamp: "2026-11-23T12:00:02Z",
  },
];
```

Part ordering preserves text and multiple tool interactions within messages; `callId` correlates calls and results. Tool descriptors and request/event contracts also belong in the protocol.

Adding content types in the future requires explicit protocol versioning and rules for unknown parts. It does not imply support for images, audio, video, or other multimodal data in the initial contract. Unsupported content or provider-specific continuation state must not be silently discarded.

## 4) SDK: `@tinlynk/sdk`

The SDK is the app-facing client for TinLynk contracts. Its target model transport sends requests to Connect and receives normalized responses/events. It must not expose direct provider calls as an app model path or require provider credentials in apps.

Tool handlers may be local to the app/runtime: the app applies its consent policy, executes requested tools, and sends correlated results back through the SDK/Connect flow. The SDK implements a repeated tool-call loop: it executes allow-listed tool calls, appends correlated results, and requests the next model turn.

## 5) Connect: `connect`

Connect is the required local gateway for app model traffic (service ID `dev.tinlynk.connect`). It owns provider credentials and upstream provider communication. Provider-specific request, response, and streaming differences are handled behind Connect's adapter boundary; apps consume the TinLynk protocol instead.

Connect also owns configured model aliases and provider settings. Credentials stay in Connect and are not part of Chat or SDK configuration.

### OpenAI-compatible compatibility surface

Connect exposes `/v1/chat` as the **canonical TinLynk protocol** and keeps `/v1/chat/completions` as an OpenAI-compatible interoperability facade. The two wire formats remain separate.

### External harness compatibility (target)

Connect is the shared gateway for external harnesses too: model aliases, provider routing, and credential management live in Connect. Harnesses use their native wire format through standard facades, not the TinLynk SDK protocol:

| Standard facade | Wire format | Status |
|---|---|---|
| `/v1/chat/completions` | OpenAI Chat Completions | Implemented for compatible providers |
| `/v1/responses` | OpenAI Responses | Implemented as OpenAI-only same-format forwarding |
| `/v1/messages` | Anthropic Messages | Implemented as Anthropic-only same-format forwarding |

Target harnesses and their expected formats:

- **Copilot CLI** supports custom OpenAI-compatible providers, including the Responses wire option, and Anthropic providers using Messages.
- **Copilot in VS Code** is a separate target: Copilot Chat has a built-in custom-endpoint provider for Chat Completions, Responses, or Messages; an extension is optional.
- **Codex CLI** requires the Responses facade.
- **Claude Code** uses the Messages facade and may route only to Claude-compatible upstreams.

These facades are implemented with provider-type checks and preserve the matching provider wire format. The canonical SDK endpoint is `POST /v1/chat`, versioned with `X-TinLynk-Protocol-Version`; OpenAI Responses and Anthropic Messages are also used as native upstream APIs by the canonical adapter. Live Copilot, Codex CLI, and Claude Code compatibility is not verified.

Adapters must preserve each wire format's tool and streaming semantics. If a requested feature cannot be represented or supported, the adapter should return an explicit error rather than silently dropping it. Use the standard facade paths above; no `/v1/tinlynk` namespace is planned for external harness compatibility.

## 6) Chat: `chat`

Chat is the end-user app for conversations and agent management. Its product scope includes sessions/history, prompt and agent configuration, model selection through Connect, tool opt-in/consent, and conversation rendering. Chat uses the SDK and TinLynk protocol; it does not call providers directly or hold provider credentials.

## 7) .github: reusable CI/CD

Centralized GitHub Actions and policies.

Workflows may include build/test, lint, release, artifact provenance, and security checks. Shared workflows are consumed by package repositories; their exact availability is maintained in the `.github` repository.

## 8) Implementation status

The current packages are a baseline, not a production-complete gateway-backed agent/provider stack.

- **Protocol:** The package exports ordered `text`, `tool_call`, and `tool_result` parts; JSON Schemas reject unknown part variants. Focused schema and TypeScript tests pass. The package remains unpublished, so no external legacy migration is required.
- **SDK:** `ConnectTransport` calls `/v1/chat`, sends and checks the protocol-version header, and parses non-stream messages and SSE events. `ModelClient` requires a transport. The SDK also retains an OpenAI-compatible Chat Completions adapter with tool mapping and SSE deltas, and `Agent` runs repeated allow-listed tool calls with correlated results. SDK build and focused tests pass.
- **Connect:** `/v1/chat` validates protocol version and supported content parts, maps canonical requests to OpenAI Responses, Anthropic Messages, or Chat Completions, and returns a canonical assistant message or event stream. Standard `/v1/chat/completions`, `/v1/responses`, and `/v1/messages` facades preserve the matching provider format; model routes include OpenAI-shaped `/v1/models` and TinLynk `/v1/catalog/models`. Connect build and server integration tests pass with loopback access; live API and harness compatibility is unverified.
- **Security and operations:** Connect binds only to loopback and requires a named client token that can be revoked independently. Provider secrets are managed with the CLI and an OS credential store. Upstream calls have a total deadline and retry explicit rate-limit responses only before returning a response; cancellations propagate to upstream. See the Connect README for platform support and migration requirements.

The canonical gateway, ordered typed-parts contract, provider mappings, and SDK-driven tool loop are implemented. Production hardening and live harness validation remain outstanding.

## 9) End-to-end target flow

1. User input arrives in Chat, which constructs protocol messages with ordered content parts.
2. The SDK sends the request to Connect using the configured model alias; it never sends the app request directly to a provider.
3. Connect resolves the alias, obtains credentials from its own configuration/secret handling, and routes through its provider adapter.
4. Connect returns the TinLynk response or stream through the SDK to Chat.
5. If the assistant requests tools, Chat/runtime applies consent and executes the requested tool.
6. The tool result is correlated by `callId` and submitted in a subsequent request through the SDK and Connect.
7. Chat renders the final response.

The local mocked flow is implemented. No live provider calls or harness runs have validated end-to-end behavior.

## 10) Security and policy

- Connect is responsible for provider credentials; apps and SDK clients do not store provider keys.
- Chat/runtime applies tool consent before executing tools.
- Connect API clients authenticate using a local token; provider keys remain in the credential store or provider environment.
- Canonical request and message metadata are app-local and are not forwarded to providers. Provider-native facade payloads are passed through as native wire formats.
- Live harness compatibility and security validation remain outstanding; do not treat local access controls as a defense against a compromised user account.

## 11) Repository layout recap

- `protocol` → `@tinlynk/protocol` (types/schemas; no runtime dependencies)
- `sdk` → `@tinlynk/sdk` (app clients/transports; depends on protocol)
- `connect` → TinLynk Connect (gateway, CLI, credentials, provider adapters)
- `chat` → TinLynk Chat (app; uses SDK/protocol and reaches providers through Connect)
- `.github` → shared workflows and policy

## 12) Contract compatibility and versioning

- Protocol changes follow semver; breaking changes increment the major version.
- The SDK pins a compatible protocol range, and apps pin a compatible SDK range.
- Connect currently requires an exact X-TinLynk-Protocol-Version match on the canonical route.
- Future content-part variants require explicit versioning; unknown parts and unsupported canonical fields are rejected.

## Validation

Protocol and SDK build/tests pass. Connect tests cover authorization, deadlines, retries, cancellation, metadata isolation, local IPC, and loopback HTTP integration. No live provider or harness compatibility tests were run.

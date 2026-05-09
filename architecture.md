# TinLynk Architecture

Components, capabilities, APIs, and relationships between Chat, Connect, Protocol, SDK, and .github.

> Canonical names & IDs
> - Platform: TinLynk
> - Apps: TinLynk Chat (“Chat”), TinLynk Connect (“Connect”)
> - Packages: @tinlynk/protocol, @tinlynk/sdk
> - Repos (GitHub org: tinlynk): chat, connect, sdk, protocol, .github
> - Service IDs: dev.tinlynk.connect

## 1) High-level: Agent-centric view

User
│
▼
TinLynk Chat (app)
│   (uses @tinlynk/sdk; typed by @tinlynk/protocol)
▼
@tinlynk/sdk  ──────────────►  Model Providers (OpenAI/OSS/etc)
│
└────────► TinLynk Connect (profiles, routing, secrets, model catalog)

- **The AI Agent** lives inside Chat as a configurable runtime using `@tinlynk/sdk`.
- The Agent routes model calls via **Connect** (discovery, profiles, secrets, routing) and coordinates tools through SDK-defined contracts.
- Types and schemas are defined in **@tinlynk/protocol**.
- CI/CD, versioning, and policies are orchestrated by workflows in **.github**.

## 2) Roles & Capabilities (at a glance)

| Component | Primary Function | Key Capabilities | Depends On |
|---|---|---|---|
| **Chat** (`chat`) | Desktop/web app for conversations | Sessions, history, attachments, prompt presets, agent configs, multi-model, tool opt-in | `@tinlynk/sdk`, `@tinlynk/protocol`, Connect |
| **Connect** (`connect`) | Local daemon + CLI + service registry | Model discovery, profiles, routing, token/secret vault, policy enforcement | `@tinlynk/sdk` (optional), `@tinlynk/protocol` |
| **@tinlynk/sdk** (`sdk`) | Client library for apps/CLIs | Model client, tool client, auth, transports, router, retries, streaming | `@tinlynk/protocol` |
| **@tinlynk/protocol** (`protocol`) | Types + JSON Schemas | Messages, tool contracts, events, error types, versioning (semver) | none |
| **.github** (`.github`) | Reusable GitHub Actions | Build, test, lint, release, SBoM, provenance, publish (NPM/Docker), PR gates | n/a |

**Dependency rules (enforced):**
- `protocol → none`
- `sdk → protocol`
- Apps/CLIs (`chat`, `connect`) → `sdk`
- Shared workflows live in `.github`

## 3) Protocol: `@tinlynk/protocol`
Source of truth for types, JSON Schemas, and wire contracts.

### Capabilities
- Versioned schemas (semver); `x-protocol-version` header on HTTP/WebSocket.
- Message contracts for: chat, tool calls, function arguments, events, errors.
- MCP-compatible tool descriptors.

### Example Types (TypeScript)
```ts
// @tinlynk/protocol

export type Role = "system" | "user" | "assistant" | "tool";

export interface Message {
  id: string;
  role: Role;
  content: string;            // plain or markdown
  toolCall?: ToolCall;        // when assistant invokes a tool
  timestamp: string;          // ISO
  metadata?: Record<string, unknown>;
}

export interface ToolDescriptor {
  name: string;               // e.g. "fs.readFile"
  title?: string;
  description?: string;
  inputSchema: JSONSchema;    // strict input contract
  outputSchema?: JSONSchema;
}

export interface ToolCall {
  name: string;               // must match ToolDescriptor.name
  arguments: unknown;         // validated against inputSchema
  callId: string;             // for streaming/result correlation
}

export interface ChatRequest {
  model: string;
  messages: Message[];
  tools?: ToolDescriptor[];
  stream?: boolean;
  temperature?: number;
  metadata?: Record<string, unknown>;
}

export interface ChatStreamEvent {
  type: "token" | "tool_call" | "tool_result" | "done" | "error";
  data: unknown;
}
```

## 4) SDK: @tinlynk/sdk

A lightweight, typed client for models, tools (MCP), and routing.

Capabilities
- Transports: HTTP(S), WebSocket, process/IPC, and Connect bridge.
- ModelClient: chat/completions embedding; SSE/WebSocket streaming.
- ToolClient: MCP handshake, list, call, streaming tool I/O.
- Router: profile-aware model selection and failover.
- Auth: bearer tokens, local Connect session, provider keys.
- Observability: structured logs, hooks, request IDs.

Minimal Usage

```ts
import { ModelClient, ToolClient, Router, type ChatRequest } from "@tinlynk/sdk";

const router = new Router({
  // Delegates to Connect if available, else direct
  strategy: "connect-first",
});

const model = new ModelClient({ router });
const tools = new ToolClient({
  invoke: async (call) => ({ callId: call.callId, result: { ok: true } }),
});

const req: ChatRequest = {
  model: "openai:gpt-4.1-mini",
  messages: [{ id: "m1", role: "user", content: "Summarize repo structure." , timestamp: new Date().toISOString()}],
  tools: await tools.list(), // pull live tool descriptors
  stream: true,
};

for await (const evt of model.stream(req)) {
  if (evt.type === "tool_call") {
    const result = await tools.call(evt.data as any);
    await model.sendToolResult(req, result); // continue the run
  }
}
```

## 5) Connect: connect

Local daemon (and CLI tlconnect) that centralizes model profiles, secrets, routing.

Capabilities
- Profiles: named sets of provider keys, defaults, rate caps.
- Model catalog: discovered from providers and local backends.
- Routing: choose best model by policy (cost, latency, size, allowlist).
- Token/secret vault: per-profile secure storage.
- HTTP API for apps/SDK; service id: dev.tinlynk.connect.

Example API (HTTP; local only by default)
- GET /v1/models → list known models with metadata.
- GET /v1/profiles → list profiles.
- POST /v1/route → choose target { model, transport } for a given request.
- POST /v1/secrets/put → store provider key (scoped).
- POST /v1/policy/estimate → tokens/cost estimate.

Example profile JSON:

```json
{
  "name": "dev",
  "defaults": { "model": "openai:gpt-4.1-mini" },
  "providers": {
    "openai": { "apiKeyRef": "secret:openai" },
    "ollama": { "endpoint": "http://localhost:11434" }
  },
  "routing": {
    "rules": [
      { "if": { "maxInputTokens": 8000 }, "use": "openai:gpt-4.1-mini" },
      { "if": { "offline": true }, "use": "ollama:llama3.1:8b" }
    ]
  }
}
```

## 6) Chat: chat

End-user app for conversations and agent management.

Capabilities
- Sessions, message search, attachments, prompt libraries.
- Agent configs (system prompts, tools on/off, profile, safety).
- Multi-provider model selection via Connect.
- Tool consent UI (per tool or per session).
- Export/import (JSONL), offline fallback to local models.

Agent Config (YAML)

```yaml
name: "TinLynk Default Agent"
profile: "dev"
model: "openai:gpt-4.1-mini"
tools:
  - "fs.readFile"
  - "git.status"
  - "http.fetch"
guardrails:
  max_tokens: 2000
  deny_tools_without_consent: true
memory:
  enabled: true
  scope: "per-agent"
```

## 7) .github: reusable CI/CD

Centralized GitHub Actions and policies.

Workflows
- build-node.yml: matrix builds for Node, caches, tsc, tests.
- release-npm.yml: version bump, changelog, provenance/SBoM, publish to @tinlynk/*.
- release-docker.yml: build and push ghcr.io/tinlynk/*.
- lint.yml: ESLint, TypeScript, markdownlint, license headers.
- security.yml: CodeQL/SAST, dependency review, secret scanning.

Usage in a repo (.github/workflows/ci.yml)

```yaml
name: CI
on: [push, pull_request]
jobs:
  build:
    uses: tinlynk/.github/.github/workflows/build-node.yml@main
    with:
      node-version: '22'
```

## 8) End-to-end Agent flow

1.	User input arrives in Chat.
2.	App constructs a ChatRequest (typed by @tinlynk/protocol).
3.	SDK Router queries Connect for profile-based routing.
4.	Model call begins; streaming tokens flow back to app via SDK.
5.	If the assistant requires a tool, it emits a ToolCall (per Protocol).
6.	App or SDK invokes the configured tool handler.
7.	Tool result streams back; SDK resumes the model run with tool output.
8.	Final response is rendered; logs/telemetry recorded.

## 9) Security & Policy

- Connect holds provider secrets and enforces routing/policies.
- Chat exposes tool toggles and per-session consent.
- SDK adds request IDs, redacts secrets, and supports signed attestations.

## 10) Minimal Agent bootstrap (Node)

```ts
import { Agent } from "@tinlynk/sdk/agent";
import { ToolClient, ModelClient, Router } from "@tinlynk/sdk";

const tools = new ToolClient({
  invoke: async (call) => ({ callId: call.callId, result: { ok: true } })
});
const router = new Router({ connectUrl: "http://127.0.0.1:8730" });  // Connect
const model = new ModelClient({ router });

const agent = new Agent({
  model,
  tools,
  policy: { allowTools: ["fs.readFile", "http.fetch"] }
});

const out = await agent.run({
  model: "openai:gpt-4.1-mini",
  messages: [{ id:"u1", role:"user", content:"Read README.md and summarize.", timestamp:new Date().toISOString() }]
});

console.log(out.text);
```

## 11) Repository layout recap

- protocol → @tinlynk/protocol (types/schemas; no runtime deps)
- sdk → @tinlynk/sdk (clients/auth/transport/router; depends on protocol)
- connect → “TinLynk Connect” (daemon/CLI; depends on sdk)
- chat → “TinLynk Chat” (app; depends on sdk)
- .github → shared workflows and policy

## 12) Contract compatibility and versioning

- Protocol is semver: breaking changes increment major.
- SDK pins a protocol range, apps pin an SDK range.
- Connect exposes its own API version and declares supported protocol ranges.

# librechat

> Snapshot date: 2026-04. Upstream: <https://github.com/danny-avila/LibreChat>

A self-hostable, multi-user **chat front-end** that targets feature
parity with the major commercial chat products while keeping every
provider key on your own server. The same browser UI lets a user
switch mid-conversation between Anthropic, OpenAI, Gemini, Bedrock,
Vertex, Mistral, Groq, DeepSeek, OpenRouter, and any
OpenAI-compatible local endpoint, and the conversation history is
stored in your own MongoDB so account portability is not a vendor
concern.

In the catalog it occupies the **"chat UI for a team"** slot:
[`open-webui`](../open-webui/) is its closest neighbor (also a
self-hosted, multi-user web chat), but `librechat` skews toward
*managed deployments with auth* (LDAP, OIDC, social login, per-user
quotas, admin panel) while `open-webui` skews toward *power-user
features for a single host* (built-in pipelines, Python tools,
function-calling UI). It is **not** an agent loop, **not** a
workflow canvas — it is a chat surface with auth and storage.

## 1. Repo facts

- Repository: <https://github.com/danny-avila/LibreChat>
- Latest tag: **v0.8.5** (no GitHub Releases feed; tag is the
  authoritative version marker on `main`)
- Primary language: TypeScript
- License: **MIT** — see
  [`LICENSE`](https://github.com/danny-avila/LibreChat/blob/main/LICENSE)
  (blob SHA `5ee6463dee826517754e332444f0acb7b3d106a0`). Permissive,
  commercial-use friendly.

## 2. Why it matters

- **Auth + multi-user are real, not bolted on.** Email/password,
  OIDC, OpenID, LDAP, GitHub, Google, Discord, Apple, plus
  rate-limit and quota knobs per user. You can hand the URL to a
  team without exposing a shared API key.
- **Provider switching is conversation-level.** A single chat thread
  can route turn N to Claude and turn N+1 to GPT without losing
  context, which is the cheapest way to A/B providers on real
  traffic.
- **MCP is wired into the agent surface.** As of the 0.8 line the
  Agents builder consumes MCP servers as tool sources, so anything
  you ship for `claude-code` or `goose` works here too.

## 3. Install

The supported deploy is Docker Compose; there is no single-binary
CLI.

```sh
git clone https://github.com/danny-avila/LibreChat.git
cd LibreChat
cp .env.example .env   # add at least one provider key
docker compose up -d
# UI at http://localhost:3080
```

## 4. Example invocation

After bringing the stack up and registering a user via the web UI,
the same backend exposes an OpenAI-shaped REST API per user token:

```sh
curl -X POST 'http://localhost:3080/api/ask/openAI' \
  -H 'Authorization: Bearer <user-token>' \
  -H 'Content-Type: application/json' \
  -d '{
    "text": "summarize the diff between v0.8.4 and v0.8.5",
    "model": "gpt-4o-mini",
    "endpoint": "openAI"
  }'
```

## 5. When to pick something else

- You want a **single-user power tool** with built-in pipelines and
  Python function calling → [`open-webui`](../open-webui/).
- You want a **workflow canvas** with datasets and eval, not just
  chat → [`dify`](../dify/) or [`langflow`](../langflow/).
- You want a **terminal** chat client, not a web UI →
  [`aichat`](../aichat/), [`mods`](../mods/), or [`oterm`](../oterm/).
- You want to **edit code on disk** from chat — wrong tool entirely;
  see [`opencode`](../opencode/), [`aider`](../aider/), or
  [`claude-code`](../claude-code/).

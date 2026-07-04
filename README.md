# AI Routing Discovery

A draft specification for a `.well-known/ai-routing-discovery` document — a
machine-readable metadata document that lets clients and SDKs discover an AI
gateway's capabilities (completion endpoints, model catalog, MCP servers,
auth methods, streaming/tooling support, etc.) at runtime instead of via
hardcoded configuration.

This is the same pattern as [`.well-known/openid-configuration`][oidc-disc]
(OpenID Connect Discovery) or [`.well-known/oauth-authorization-server`][rfc8414]
(RFC 8414), applied to the AI gateway / inference-provider space.

## Why

Today, every app that talks to an LLM gateway hardcodes:

- The chat/completion endpoint URL and API version
- Which models are available, their context windows, and their capabilities
  (vision, tool calling, JSON mode, streaming, etc.)
- Which MCP servers are reachable and what tools/resources they expose
- Auth scheme details (API key header name, OAuth token endpoint, scopes)

This is copy-pasted, provider-specific glue in every codebase and SDK. When a
provider adds a model, deprecates one, rotates an endpoint, or exposes a new
MCP server, every downstream integration has to be updated by hand.

Discovery documents solve this class of problem elsewhere (OIDC, OAuth AS
metadata, `security.txt`, `robots.txt`). This proposal applies the same idea
to AI gateways so client SDKs can do:

```ts
const gateway = await AiGateway.discover("https://api.example.com");
const res = await gateway.chat.completions.create({ ... });
```

instead of every app baking in `https://api.example.com/v1/chat/completions`,
a model allowlist, and per-provider auth logic.

## Contents

- [`rfc/draft-ai-routing-discovery-00.md`](rfc/draft-ai-routing-discovery-00.md)
  — the RFC-style draft specification, readable as Markdown
- [`rfc/draft-ai-routing-discovery-00.xml`](rfc/draft-ai-routing-discovery-00.xml)
  — the same draft in official IETF Internet-Draft form (xml2rfc v3 /
  RFC 7991 vocabulary). Fill in the `TODO` placeholders (author name,
  organization, address, draft name) and run it through `xml2rfc` to
  produce a submission-ready `.txt`/`.html` with correct IETF
  boilerplate. See the comment block at the top of the file for exact
  steps.
- [`schema/ai-routing-discovery.schema.json`](schema/ai-routing-discovery.schema.json)
  — JSON Schema for the discovery document
- [`examples/`](examples/) — example discovery documents:
  [`public-example.json`](examples/public-example.json) (anonymous
  caller — sensitive fields gated, per Section 5.1),
  [`full-example.json`](examples/full-example.json) (authenticated
  caller — every field), [`minimal-example.json`](examples/minimal-example.json)
  (smallest conforming document), and
  [`models-endpoint-response.json`](examples/models-endpoint-response.json)
  (the OpenAI-compatible shape returned by `models_endpoint`)

## License

The specification text (`rfc/`) is licensed under the
[Community Specification License 1.0](LICENSE), which includes patent
grants for implementers. See [`Governance.md`](Governance.md) for how
a draft becomes an Approved Specification, [`Scope.md`](Scope.md) for
what the patent grant covers, and [`Notices.md`](Notices.md) for the
exclusion/withdrawal/acceptance notice log required by the license.

The JSON Schema and example files (`schema/`, `examples/`) are source
code and are separately licensed under [MIT](LICENSE-CODE), per
Section 4 of the Community Specification License.

## Status

**Draft — v00.** Not yet implemented anywhere. This is a proposal meant to be
picked apart, argued with, and iterated on before anyone writes an SDK
against it. See the [Open Questions](rfc/draft-ai-routing-discovery-00.md#20-open-questions)
section for what's still unresolved.

## Quick example

```
GET https://api.example.com/.well-known/ai-routing-discovery HTTP/1.1
Accept: application/json
```

```json
{
  "issuer": "https://api.example.com",
  "ai_routing_discovery_version": "0.1",
  "chat_completion_endpoint": "https://api.example.com/v1/chat/completions",
  "models_endpoint": "https://api.example.com/v1/models",
  "mcp_servers_endpoint": "https://api.example.com/v1/mcp_servers",
  "auth_methods_supported": ["api_key", "oauth2"],
  "token_endpoint": "https://api.example.com/oauth/token",
  "streaming_protocols_supported": ["sse"]
}
```

This is what an **anonymous** caller gets: routing information only.
`models_endpoint` points at an OpenAI-compatible models list (see
[`examples/models-endpoint-response.json`](examples/models-endpoint-response.json)),
and `mcp_servers_endpoint` points at an endpoint that itself requires
authentication before it discloses any MCP server topology — the
gateway never advertises its internal server-stack architecture to an
unauthenticated request. See
[`examples/public-example.json`](examples/public-example.json) for
the full anonymous response and
[`examples/full-example.json`](examples/full-example.json) for what
an authenticated, authorized caller receives instead (including the
inline `mcp_servers` array).

[oidc-disc]: https://openid.net/specs/openid-connect-discovery-1_0.html
[rfc8414]: https://www.rfc-editor.org/rfc/rfc8414

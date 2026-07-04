    AI Routing Working Group (proposed)                        K. (author)
    Internet-Draft                                                Personal
    Intended status: Informational                              2026-07-04
    Expires: 2027-01-04

               AI Routing Discovery 1.0
               draft-ai-routing-discovery-00

## Abstract

This document defines a discovery mechanism for AI gateways (services
that expose LLM chat/completion, embeddings, and related inference
capabilities, optionally alongside Model Context Protocol (MCP)
servers) that mirrors the pattern established by OpenID Connect
Discovery 1.0 and RFC 8414 (OAuth 2.0 Authorization Server Metadata).
An AI gateway that implements this specification publishes a JSON
metadata document at a `.well-known` URI. Clients fetch this document
once and use it to locate endpoints, enumerate available models and
their capabilities, discover reachable MCP servers, and determine
supported authentication methods — removing the need to hardcode this
information in application or SDK code.

## Status of This Memo

This is a draft proposal, not a ratified standard, and is not
affiliated with IETF, OpenID Foundation, or any model provider. It is
published to solicit feedback before any implementation work begins.
Distribution of this draft is unlimited.

## Copyright Notice

This document is a Draft Specification and is subject to the
Community Specification License 1.0 (see `LICENSE` at the root of
this repository), including the copyright and patent grants and the
Scope defined in `Scope.md`. Implementations of this specification
(as opposed to reproductions of the specification text itself) do not
require attribution; see `LICENSE` Section 1.2.

## Table of Contents

1. Introduction
2. Terminology
3. Goals and Non-Goals
4. Discovery Document Location
5. Discovery Document Metadata
6. The Model Object
7. The MCP Server Object
8. The Rate Limit Object
9. Authentication Methods
10. Versioning and Extensibility
11. Vendor Extensions
12. Caching and Freshness
13. Multi-Tenant and Per-Key Discovery
14. Relationship to Existing Specifications
15. Security Considerations
16. Privacy Considerations
17. IANA Considerations
18. Client Implementation Guidance
19. Examples
20. Open Questions
21. References

Appendix A: JSON Schema

---

## 1. Introduction

Applications that call LLM providers or self-hosted AI gateways
today hardcode a surprisingly large amount of provider-specific
trivia: the completions URL, which API version to send, which
models exist and what each one supports (context window, vision,
tool calling, JSON mode, prompt caching), how to authenticate, and —
increasingly — which Model Context Protocol (MCP) servers are
reachable and what they expose.

This information already changes independently of application code:
providers ship new models, deprecate old ones, rotate endpoints
between API versions, and add capabilities. Every one of those
changes today requires a code change, a redeploy, or an SDK version
bump in every downstream consumer.

The OAuth and OpenID Connect ecosystems solved an analogous problem
with discovery documents: instead of hardcoding an authorization
server's endpoints and capabilities, a client fetches
`/.well-known/openid-configuration` (or
`/.well-known/oauth-authorization-server`, RFC 8414) once and derives
everything else at runtime. SDKs like `openid-client` exist almost
entirely to wrap that one HTTP call.

This document proposes the same pattern for AI gateways: a
`.well-known/ai-routing-discovery` JSON document that a gateway
publishes, and that client SDKs fetch (and cache) to configure
themselves, instead of requiring the developer to hardcode a base
URL, a model ID string, and a mental model of what that model can do.

### 1.1. Non-goals

This is not a proposal for a new wire protocol for chat completions.
It does not attempt to unify the OpenAI, Anthropic, and Google
request/response shapes. It assumes gateways continue to speak
whatever request format they already speak (OpenAI-compatible,
Anthropic Messages API, etc.) and simply describes *where* those
endpoints are and *what* they support.

## 2. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in RFC 2119.

- **AI Gateway**: An HTTP service that exposes one or more AI
  inference capabilities (chat completion, embeddings, image
  generation, etc.), whether that service fronts a single model
  provider, multiple providers, or self-hosted models.
- **Discovery Document**: The JSON metadata document described in
  this specification.
- **Issuer**: The base URL that identifies the AI gateway and
  under which the discovery document is published.
- **Model Object**: A JSON object describing a single model's
  identity and capabilities (Section 6).
- **MCP Server**: A Model Context Protocol server, as defined by the
  [Model Context Protocol specification][mcp-spec], reachable through
  or alongside the gateway.
- **Client**: An application, SDK, or library that consumes a
  discovery document.

## 3. Goals and Non-Goals

Goals:

- Let clients locate all relevant endpoints (completions,
  embeddings, files, fine-tuning, batch, models, health) from one
  fetch.
- Let clients enumerate available models and their capabilities
  programmatically instead of via out-of-band documentation.
- Let clients discover MCP servers associated with a gateway the
  same way OIDC discovery lets clients discover a `jwks_uri`.
- Let clients determine supported auth methods without
  documentation spelunking.
- Be transport- and vendor-neutral: this must work equally well for
  an OpenAI-compatible gateway, an Anthropic-compatible gateway, or
  an internal company-run LLM proxy.
- Be incrementally adoptable: every field except `issuer` and
  `ai_routing_discovery_version` is OPTIONAL, so a gateway can
  publish a minimal document on day one and grow it over time.

Non-goals:

- Standardizing the request/response body of a chat completion call.
- Replacing model cards, evals, or benchmark data — this describes
  routing and capability *flags*, not model quality.
- Billing, quota, or pricing negotiation (a `pricing_endpoint`
  pointer is provided, but pricing data itself is out of scope).

## 4. Discovery Document Location

Given a gateway identified by issuer URL `https://gateway.example.com`,
the discovery document MUST be retrievable via an HTTP GET request to:

```
https://gateway.example.com/.well-known/ai-routing-discovery
```

This follows RFC 8615 (Well-Known Uniform Resource Identifiers). The
response:

- MUST be served over HTTPS in production (plain HTTP MAY be used
  only for local development, e.g. `http://localhost:*`).
- MUST have a `Content-Type` of `application/json`.
- MUST NOT require authentication to fetch the *shape* of the
  document, though field values MAY be tailored per-caller (see
  Section 13) if the request is authenticated.
- SHOULD be served with a `Cache-Control` header (Section 12).

If the issuer URL contains a path component (e.g.
`https://gateway.example.com/tenant-42`), the well-known suffix is
inserted after the host, mirroring RFC 8414's handling of path-based
issuers:

```
https://gateway.example.com/.well-known/ai-routing-discovery/tenant-42
```

Clients MUST reject a discovery document whose `issuer` field does
not exactly match the URL the client expected to discover (scheme,
host, port, and path), to prevent a compromised or misconfigured
intermediary from redirecting a client to attacker-controlled
endpoints.

## 5. Discovery Document Metadata

The discovery document is a single JSON object. Field names use
`snake_case` for consistency with OIDC discovery and most LLM
provider APIs.

| Field | Required | Type | Description |
|---|---|---|---|
| `issuer` | REQUIRED | string (URL) | The gateway's canonical base URL. MUST match the URL used to fetch this document. |
| `ai_routing_discovery_version` | REQUIRED | string | Spec version this document conforms to, e.g. `"0.1"`. |
| `chat_completion_endpoint` | RECOMMENDED | string (URL) | Chat/message completion endpoint. |
| `completion_endpoint` | OPTIONAL | string (URL) | Legacy non-chat text completion endpoint, if supported. |
| `embeddings_endpoint` | OPTIONAL | string (URL) | Embeddings endpoint. |
| `responses_endpoint` | OPTIONAL | string (URL) | Stateful "responses"-style endpoint, if supported (e.g. OpenAI Responses API equivalents). |
| `images_endpoint` | OPTIONAL | string (URL) | Image generation/editing endpoint. |
| `audio_endpoint` | OPTIONAL | string (URL) | Speech-to-text / text-to-speech endpoint. |
| `moderations_endpoint` | OPTIONAL | string (URL) | Content moderation / safety classification endpoint. |
| `files_endpoint` | OPTIONAL | string (URL) | File upload endpoint (for fine-tuning, batch, or RAG inputs). |
| `batch_endpoint` | OPTIONAL | string (URL) | Asynchronous batch job submission endpoint. |
| `fine_tuning_endpoint` | OPTIONAL | string (URL) | Fine-tuning job endpoint. |
| `models_endpoint` | RECOMMENDED | string (URL) | Endpoint returning the live/paginated model list (the mirror of `models_supported` for large catalogs). |
| `models_supported` | RECOMMENDED | array of [Model Object](#6-the-model-object) | Inline model catalog. SHOULD be omitted (in favor of `models_endpoint`) if the catalog is large or changes frequently. |
| `mcp_servers` | OPTIONAL | array of [MCP Server Object](#7-the-mcp-server-object) | MCP servers associated with this gateway. |
| `auth_methods_supported` | RECOMMENDED | array of string | One or more of `api_key`, `oauth2`, `mtls`, `none`. |
| `token_endpoint` | OPTIONAL | string (URL) | OAuth 2.0 token endpoint, present if `oauth2` is supported. |
| `authorization_endpoint` | OPTIONAL | string (URL) | OAuth 2.0 authorization endpoint, for user-delegated flows. |
| `scopes_supported` | OPTIONAL | array of string | OAuth scopes relevant to gateway access. |
| `api_key_header` | OPTIONAL | string | Header name expected for API-key auth, e.g. `"Authorization"` or `"X-Api-Key"`. |
| `api_key_scheme` | OPTIONAL | string | Value prefix for the header, e.g. `"Bearer"`. |
| `streaming_protocols_supported` | OPTIONAL | array of string | One or more of `sse`, `websocket`, `grpc`. |
| `tool_calling_formats_supported` | OPTIONAL | array of string | One or more of `openai-tools`, `anthropic-tools`, `mcp`, `gemini-function-calling`. |
| `rate_limits` | OPTIONAL | [Rate Limit Object](#8-the-rate-limit-object) or string (URL) | Inline rate-limit policy, or a URL to fetch it dynamically (since limits are often per-key). |
| `health_endpoint` | OPTIONAL | string (URL) | Health/status check endpoint. |
| `status_page` | OPTIONAL | string (URL) | Human-readable incident/status page. |
| `pricing_endpoint` | OPTIONAL | string (URL) | Machine-readable pricing data. |
| `documentation_url` | OPTIONAL | string (URL) | Human documentation entry point. |
| `terms_of_service_url` | OPTIONAL | string (URL) | ToS document. |
| `privacy_policy_url` | OPTIONAL | string (URL) | Privacy policy document. |
| `data_residency` | OPTIONAL | array of string | ISO region codes where inference/storage occurs, e.g. `["eu-west-1"]`. |
| `jwks_uri` | OPTIONAL | string (URL) | JSON Web Key Set used to verify signed responses (e.g. signed model cards or usage receipts), mirroring OIDC's `jwks_uri`. |
| `contact` | OPTIONAL | array of string | Contact URIs (`mailto:`, `https://`) for the gateway operator. |
| `signature` | OPTIONAL | string (JWS) | Optional JWS signature over the document body for integrity verification (Section 15). |

Fields whose value is unknown or not applicable MUST be omitted
rather than set to `null`, so that clients can use simple key
presence checks.

## 6. The Model Object

Each entry in `models_supported` (or returned by `models_endpoint`)
is an object:

| Field | Required | Type | Description |
|---|---|---|---|
| `id` | REQUIRED | string | Model identifier as accepted by the completion endpoints. |
| `aliases` | OPTIONAL | array of string | Other identifiers that resolve to the same model (e.g. a floating `"latest"` tag). |
| `display_name` | OPTIONAL | string | Human-readable name. |
| `context_window` | OPTIONAL | integer | Max input tokens. |
| `max_output_tokens` | OPTIONAL | integer | Max output tokens per request. |
| `modalities_supported` | OPTIONAL | array of string | One or more of `text`, `image`, `audio`, `video`. |
| `capabilities` | OPTIONAL | array of string | One or more of `tool_calling`, `parallel_tool_calling`, `streaming`, `structured_outputs`, `json_mode`, `vision`, `prompt_caching`, `extended_thinking`, `fine_tunable`. |
| `deprecated` | OPTIONAL | boolean | Defaults to `false`. |
| `deprecation_date` | OPTIONAL | string (date) | ISO 8601 date after which the model may stop being served. |
| `successor_id` | OPTIONAL | string | Recommended replacement model `id`, if deprecated. |
| `knowledge_cutoff` | OPTIONAL | string (date) | Training data cutoff. |

Clients SHOULD treat unrecognized `capabilities` values as
informational and ignore them rather than failing, since this list
is expected to grow (Section 10).

## 7. The MCP Server Object

| Field | Required | Type | Description |
|---|---|---|---|
| `name` | REQUIRED | string | Short identifier for the server, unique within this document. |
| `url` | REQUIRED | string (URL) | MCP server endpoint. |
| `transport` | REQUIRED | string | One of `streamable-http`, `sse`, `stdio-remote-proxy`. |
| `description` | OPTIONAL | string | Human-readable summary of what the server exposes. |
| `auth_required` | OPTIONAL | boolean | Defaults to `false`. |
| `auth_methods_supported` | OPTIONAL | array of string | Same vocabulary as the top-level `auth_methods_supported`, if it differs from the gateway default. |
| `capabilities` | OPTIONAL | array of string | Subset of MCP capabilities the server declares, e.g. `tools`, `resources`, `prompts`, `sampling`. |

This lets a client resolve, e.g., "the filesystem MCP server for
this gateway" without a separate registry — analogous to how OIDC
discovery exposes a `jwks_uri` rather than requiring keys to be
distributed out of band.

## 8. The Rate Limit Object

`rate_limits`, when present as an object rather than a URL, has the
shape:

```json
{
  "requests_per_minute": 500,
  "tokens_per_minute": 200000,
  "concurrent_requests": 50,
  "scope": "per_api_key"
}
```

`scope` is one of `per_api_key`, `per_ip`, `global`. Because limits
are frequently tier- or key-specific, gateways SHOULD prefer
publishing a `rate_limits` URL that the client can fetch with its
credentials attached, rather than baking a single limit into the
public, unauthenticated discovery document.

## 9. Authentication Methods

`auth_methods_supported` enumerates the schemes a gateway accepts.
This document does not invent new auth flows; it exposes enough
metadata for a client to select and configure an existing one:

- `api_key`: static bearer token or header-based key. Combined with
  `api_key_header` / `api_key_scheme`.
- `oauth2`: standard OAuth 2.0 client-credentials or authorization-code
  flow. Combined with `token_endpoint`, `authorization_endpoint`,
  `scopes_supported`. Gateways that support OAuth SHOULD also publish
  a standard RFC 8414 authorization-server-metadata document and MAY
  simply reference it via an `oauth_authorization_server` field
  pointing at that document's URL instead of duplicating fields here.
- `mtls`: mutual TLS; no additional discovery fields are defined
  by this spec (certificate provisioning is out of band).
- `none`: unauthenticated access (e.g. a local dev gateway).

## 10. Versioning and Extensibility

`ai_routing_discovery_version` follows `MAJOR.MINOR`. Clients MUST
ignore fields they do not recognize. Additive, backward-compatible
changes (new optional fields, new enum values) bump `MINOR`.
Breaking changes (removing a field, changing a field's type or
semantics) bump `MAJOR` and MUST use a new well-known path segment,
e.g. `.well-known/ai-routing-discovery-v2`, so that old and new
clients can coexist against the same gateway during a migration
window.

## 11. Vendor Extensions

Gateways MAY include vendor- or deployment-specific fields as long as
they are namespaced with an `x_` prefix followed by a short vendor or
org token, e.g. `x_acme_internal_routing_tier`. Clients MUST ignore
`x_`-prefixed fields they do not understand. Namespacing prevents
collisions with future core spec fields and keeps custom fields
visually distinguishable from standard ones.

## 12. Caching and Freshness

Discovery documents change infrequently relative to request volume.
Gateways SHOULD send `Cache-Control: max-age=3600` (or similar) and
an `ETag`. Clients SHOULD cache the document for the duration
indicated and MAY use conditional GET (`If-None-Match`) to refresh
cheaply. Clients MUST NOT refetch the discovery document on every
inference request.

If a gateway needs to rotate an endpoint or retire a model with
short notice, it SHOULD keep the old value valid (e.g. the old
completion URL still resolving, or the model still present but
`deprecated: true`) for at least one full cache TTL period after
publishing the change, so already-cached clients do not break
mid-TTL.

## 13. Multi-Tenant and Per-Key Discovery

Some gateways expose different models or MCP servers to different
customers or API keys (e.g. an enterprise reseller). Such gateways
MAY serve a different discovery document body to authenticated
requests than to anonymous ones, using standard HTTP content
negotiation on the `Authorization` header. The `issuer` field MUST
remain identical regardless of authentication, since it is the
identity anchor clients validate against (Section 4).

Gateways with this need SHOULD still serve a reasonable
default/anonymous document (at minimum, endpoints and public model
capabilities) so that unauthenticated discovery — e.g. for
documentation generators or SDK setup wizards — still works.

## 14. Relationship to Existing Specifications

- **OpenID Connect Discovery 1.0** and **RFC 8414** (OAuth 2.0
  Authorization Server Metadata) are the direct structural
  precedent: a `.well-known` JSON document, an `issuer` identity
  check, optional fields with sane defaults, a `jwks_uri` pattern
  (mirrored here for signed responses).
- **RFC 8615** (Well-Known URIs) governs the `.well-known` registration
  this spec relies on; see Section 17.
- **Model Context Protocol (MCP)** defines the transport and
  semantics of the servers listed in `mcp_servers`; this spec only
  describes *how a client finds* those servers, not how they behave
  once connected.
- This spec deliberately does not compete with or replace
  provider-specific "list models" endpoints (e.g. `GET /v1/models`);
  `models_endpoint` is meant to point at exactly that kind of
  existing endpoint where one already exists.

## 15. Security Considerations

- **Issuer validation.** As in OIDC, failing to validate the
  `issuer` field against the expected discovery URL allows a
  man-in-the-middle or compromised CDN to redirect a client's
  inference traffic (and any embedded credentials) to an
  attacker-controlled endpoint. Clients MUST perform this check.
- **HTTPS required.** Discovery documents fetched over plain HTTP
  are trivially tamperable. Production clients MUST refuse to
  process a discovery document fetched over an insecure connection,
  localhost excepted.
- **Document integrity.** The optional `signature` field allows a
  gateway to publish a JWS-signed document (verifiable via
  `jwks_uri`) for clients that want end-to-end integrity even
  through an untrusted intermediary or cache. This is OPTIONAL
  because many deployments will rely on TLS alone, but is
  RECOMMENDED for gateways distributed through public CDNs or
  mirrors.
- **Endpoint allowlisting.** Because this document can point a
  client at arbitrary URLs, clients operating in security-sensitive
  environments SHOULD allow operators to pin or allowlist expected
  hosts rather than blindly trusting every URL in a fetched document.
- **MCP server trust.** Discovering an MCP server through this
  document does not imply it is safe to grant it tool-execution
  authority without the same scrutiny any MCP server integration
  requires. This document is a *routing* mechanism, not a trust or
  capability grant.
- **Denial of service via large catalogs.** Gateways with very large
  or frequently changing model catalogs SHOULD use `models_endpoint`
  rather than inlining thousands of `models_supported` entries into
  a document that's meant to be cheaply cacheable.

## 16. Privacy Considerations

Per-key or per-tenant discovery documents (Section 13) can leak
information about a customer's plan tier or entitlements to anyone
who can present that customer's credentials to the discovery
endpoint. Gateways implementing tenant-specific discovery SHOULD
apply the same access controls to the discovery document as to the
underlying APIs it describes.

## 17. IANA Considerations

This document requests registration of the following entry in the
"Well-Known URIs" registry defined by RFC 8615:

| URI Suffix | `ai-routing-discovery` |
|---|---|
| Change Controller | AI Routing Working Group (proposed) / this document's editors |
| Reference | This document |
| Status | Provisional — pending broader review |

No new media type is requested; discovery documents are served as
`application/json`.

## 18. Client Implementation Guidance

A conforming client SDK SHOULD:

1. Accept an issuer URL (or an API key that implies one) at
   construction time, rather than a hardcoded base URL.
2. Fetch and cache the discovery document per Section 12.
3. Resolve endpoint URLs, model capability checks, and MCP server
   connections from the cached document instead of from constants
   baked into the SDK.
4. Fail closed with a clear error if a requested capability (e.g.
   `structured_outputs` for a given model) is absent from the
   discovery document, rather than sending the request and hoping.
5. Expose an escape hatch (explicit endpoint override) for gateways
   that have not yet implemented discovery, so adoption can be
   incremental.

This is the same shape as `openid-client`, `passport-openidconnect`,
or Kubernetes' discovery-based `client-go` REST mapper: one runtime
fetch replaces a large surface of hardcoded, provider-specific
configuration.

## 19. Examples

See [`examples/full-example.json`](../examples/full-example.json) for
a document exercising every field, and
[`examples/minimal-example.json`](../examples/minimal-example.json)
for the smallest conforming document:

```json
{
  "issuer": "https://gateway.example.com",
  "ai_routing_discovery_version": "0.1",
  "chat_completion_endpoint": "https://gateway.example.com/v1/chat/completions",
  "models_supported": [
    { "id": "example-model-1" }
  ]
}
```

## 20. Open Questions

These are unresolved and called out explicitly rather than papered
over:

1. **Naming.** Is `ai-routing-discovery` the right well-known name,
   or should this be scoped more narrowly (e.g.
   `llm-gateway-discovery`) or more broadly (e.g. covering
   non-chat AI services like vector search)?
2. **Model object granularity.** Should capability flags be a flat
   string array (current proposal) or a nested object
   (`capabilities: { tool_calling: { parallel: true } }`)? The flat
   array is simpler but harder to extend with parameters.
3. **Relationship to `GET /v1/models`.** Most providers already
   have a models-list endpoint with a different (usually
   OpenAI-shaped) schema. Should `models_endpoint` require that
   response to conform to the Model Object shape in Section 6, or
   is it acceptable for it to point at the existing
   provider-native shape, with `models_supported` as the only
   place the spec's own shape is guaranteed?
4. **MCP auth delegation.** Should MCP server auth be assumed to
   flow through the same credential as the gateway's own API
   (single sign-on to all listed MCP servers), or fully independent?
5. **Should this be one document or two?** Splitting
   "endpoints + auth" (rarely changes) from "model catalog"
   (changes often) into two well-known documents would let clients
   cache each with different TTLs, at the cost of a second round
   trip for full discovery.
6. **Governance.** This draft has no working group behind it. Before
   any SDK depends on it, it needs either adoption by an existing
   body (IETF individual submission, a neutral consortium) or at
   least multi-vendor buy-in so it isn't read as one company's
   private format.

## 21. References

- RFC 2119 — Key words for use in RFCs to Indicate Requirement Levels
- RFC 8615 — Well-Known Uniform Resource Identifiers (URIs)
- RFC 8414 — OAuth 2.0 Authorization Server Metadata
- OpenID Connect Discovery 1.0 — openid.net/specs/openid-connect-discovery-1_0.html
- Model Context Protocol Specification — [mcp-spec]

[mcp-spec]: https://modelcontextprotocol.io/specification

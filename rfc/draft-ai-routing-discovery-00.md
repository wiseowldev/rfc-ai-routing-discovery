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
- Not turn discovery into reconnaissance: fields that expose internal
  architecture (notably MCP server topology) are explicitly
  identified as sensitive and gated behind authentication by default
  (Section 5.1), rather than published to any anonymous caller.

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
| `models_endpoint` | RECOMMENDED | string (URL) | Primary discovery mechanism for models. SHOULD return an OpenAI-compatible model list shape; see Section 6. |
| `models_supported` | OPTIONAL | array of [Model Object](#6-the-model-object) | Inline model catalog, for small/simple gateways only. SHOULD be omitted in favor of `models_endpoint` (Section 6) once a gateway has more than a handful of models. |
| `mcp_servers` | OPTIONAL, SENSITIVE | array of [MCP Server Object](#7-the-mcp-server-object) | MCP servers associated with this gateway. Reveals internal service topology — see Section 5.1 and Section 15 before including this in an unauthenticated response. Prefer `mcp_servers_endpoint`. |
| `mcp_servers_endpoint` | OPTIONAL | string (URL) | Pointer to a separately-authenticated endpoint returning `mcp_servers`-shaped data. RECOMMENDED over inlining `mcp_servers` so that anonymous discovery does not expose server topology. |
| `auth_methods_supported` | RECOMMENDED | array of string | One or more of `api_key`, `oauth2`, `mtls`, `none`. |
| `token_endpoint` | OPTIONAL | string (URL) | OAuth 2.0 token endpoint, present if `oauth2` is supported. |
| `authorization_endpoint` | OPTIONAL | string (URL) | OAuth 2.0 authorization endpoint, for user-delegated flows. |
| `scopes_supported` | OPTIONAL | array of string | OAuth scopes relevant to gateway access. |
| `api_key_header` | OPTIONAL | string | Header name expected for API-key auth, e.g. `"Authorization"` or `"X-Api-Key"`. |
| `api_key_scheme` | OPTIONAL | string | Value prefix for the header, e.g. `"Bearer"`. |
| `streaming_protocols_supported` | OPTIONAL | array of string | One or more of `sse`, `websocket`, `grpc`. |
| `tool_calling_formats_supported` | OPTIONAL | array of string | One or more of `openai-tools`, `anthropic-tools`, `mcp`, `gemini-function-calling`. |
| `rate_limits` | OPTIONAL | [Rate Limit Object](#8-the-rate-limit-object) or string (URL) | Inline rate-limit policy, or a URL to fetch it dynamically (since limits are often per-key). Prefer the URL form; see Section 5.1. |
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

### 5.1. Field Sensitivity and Default Disclosure

A discovery document is, by design, fetchable without authentication
(Section 4). That is appropriate for fields whose only job is to
route a client to the right URL — but not every field is like that.
Some fields describe internal architecture in enough detail that
publishing them to anyone on the internet is itself an exposure, most
notably the full list of MCP servers a gateway wires up (their
hostnames, transports, and what tools/resources each one grants
access to).

This specification therefore distinguishes two tiers:

- **Public fields** — safe to return to any caller, authenticated or
  not. This includes `issuer`, `ai_routing_discovery_version`,
  `chat_completion_endpoint` and the other endpoint pointers,
  `models_endpoint`, `auth_methods_supported`, `token_endpoint`,
  `documentation_url`, and similar. These exist specifically so that
  unauthenticated tooling (SDK setup wizards, documentation
  generators) can bootstrap against a gateway.
- **Sensitive fields** — marked SENSITIVE in the table above
  (currently `mcp_servers`; the inline form of `rate_limits` is a
  softer case of the same concern). These reveal internal topology or
  capacity information rather than just routing information.

For sensitive fields, gateways:

1. **SHOULD NOT** include them in the default, unauthenticated
   discovery response. Simply omit the field, per the rule above —
   do not return an error, and do not return an empty array as a
   signal that the field exists but is being withheld (see the "fail
   closed, not loud" guidance in Section 15).
2. **SHOULD** return them only from an authenticated request (Section
   13), and MAY further restrict them to callers holding a specific
   scope or permission if the gateway already has a scope system
   (e.g. a `discovery.mcp` scope) — this specification does not
   mandate a particular scope name.
3. **SHOULD** prefer exposing a *pointer* (`mcp_servers_endpoint`)
   over inlining the sensitive data directly, so that a cached or
   leaked copy of the public discovery document does not itself
   contain the sensitive payload. The pointer target enforces its own
   authentication independently.

Gateways with no sensitivity concerns (e.g. an internal-only gateway
already behind a private network boundary) MAY inline `mcp_servers`
directly and skip `mcp_servers_endpoint` entirely — this tiering is a
recommendation for gateways exposed to the public internet, not a
hard requirement for every deployment.

## 6. The Model Object

### 6.1. Models Endpoint (Primary Mechanism)

`models_endpoint` is the RECOMMENDED way to discover a gateway's
model catalog. This deliberately aligns with the model-listing
endpoint shape already implemented by OpenAI and the large number of
gateways and proxies that describe themselves as "OpenAI-compatible."
A `GET` to `models_endpoint` SHOULD return:

```json
{
  "object": "list",
  "data": [
    { "id": "acme-large-2026-05", "object": "model", "created": 1750000000, "owned_by": "acme" }
  ]
}
```

Each entry in `data` is a Model Object (Section 6.2). The four
fields shown above (`id`, `object`, `created`, `owned_by`) are the
fields defined by the widely-deployed OpenAI models-list shape.
Gateways that already run such an endpoint satisfy this part of the
specification without modification. This specification's additional
capability fields (Section 6.2) are purely additive JSON keys on the
same objects — existing OpenAI-compatible clients that ignore unknown
fields are unaffected, while clients that understand this
specification get the extra capability data for free.

Gateways with a small, rarely-changing catalog MAY instead (or
additionally) publish `models_supported` as an inline array of Model
Objects directly in the discovery document, skipping the extra round
trip. `models_endpoint` SHOULD be preferred once a catalog is large,
paginated, or changes independently of the discovery document's own
cache lifetime (Section 12).

### 6.2. Model Object Fields

| Field | Required | Type | Description |
|---|---|---|---|
| `id` | REQUIRED | string | Model identifier as accepted by the completion endpoints. |
| `object` | RECOMMENDED | string | `"model"`, for alignment with the OpenAI models-list shape. |
| `created` | OPTIONAL | integer | Unix timestamp of model creation/availability, for OpenAI-shape alignment. |
| `owned_by` | OPTIONAL | string | Organization or provider that owns the model, for OpenAI-shape alignment. |
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

Only `id` is REQUIRED by this specification; `object`, `created`, and
`owned_by` are RECOMMENDED specifically to keep a `models_endpoint`
response usable by generic OpenAI-compatible client code that expects
those keys, even before that code is updated to understand this
specification's capability fields.

Clients SHOULD treat unrecognized `capabilities` values as
informational and ignore them rather than failing, since this list
is expected to grow (Section 10).

## 7. The MCP Server Object

> **Sensitivity note:** the array of MCP Server Objects described
> here is a SENSITIVE field (Section 5.1) whether it appears inline
> as `mcp_servers` or is fetched from `mcp_servers_endpoint`. Do not
> expose it to unauthenticated callers; see Section 15.

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

Gateways SHOULD set `url` to an address on the gateway's own public
domain (routed internally to the actual MCP server) rather than a raw
internal hostname, IP address, or private DNS name. Even to an
authorized caller, an internal hostname unnecessarily reveals
infrastructure detail (e.g. cluster naming conventions, internal
network segmentation) that has nothing to do with using the server.

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
default/anonymous document (at minimum, the public fields identified
in Section 5.1) so that unauthenticated discovery — e.g. for
documentation generators or SDK setup wizards — still works.

This is also the mechanism by which sensitive fields (Section 5.1),
most notably `mcp_servers`, are gated: an unauthenticated request
gets a document with those fields omitted, and only a request
carrying valid credentials (and, at the gateway's option, an
appropriate scope) gets the full document or a working
`mcp_servers_endpoint` response.

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
  describes *how a client finds* those servers (subject to the
  sensitivity gating in Section 5.1), not how they behave once
  connected.
- This spec deliberately builds on, rather than replaces,
  provider-specific "list models" endpoints (e.g. OpenAI's
  `GET /v1/models`); `models_endpoint` (Section 6) is designed to be
  satisfiable by exactly that kind of existing endpoint, extended
  additively.

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
- **Topology minimization.** A discovery document is easy to fetch
  anonymously and, being cacheable, easy to mirror or leak well
  beyond its intended audience. Fields that describe internal
  architecture rather than mere routing — in particular `mcp_servers`
  — are SENSITIVE (Section 5.1) and SHOULD NOT appear in the default,
  unauthenticated response. An attacker who can enumerate a
  gateway's full MCP server topology (internal tool names,
  transports, and what each one grants access to) has a
  reconnaissance map they should not get for free from a `.well-known`
  URI. Prefer `mcp_servers_endpoint` gated behind its own
  authentication over inlining `mcp_servers` on a public gateway.
- **Fail closed, not loud.** When a gateway withholds sensitive
  fields from an unauthorized caller, it SHOULD do so by omission
  (the field is simply absent, exactly as if the gateway never
  implemented it) rather than by returning an error such as `403` on
  the discovery endpoint itself, or an empty array where a populated
  one would otherwise appear. Either of those signals to a prober
  that privileged data exists and is being withheld, which is itself
  information leakage about the gateway's internal posture.

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
an *authenticated* response exercising every field, including the
sensitive `mcp_servers` array (Section 5.1);
[`examples/public-example.json`](../examples/public-example.json) for
what the same gateway returns to an anonymous caller (no
`mcp_servers`, `rate_limits` given as a URL instead of inline); and
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
3. **Relationship to `GET /v1/models`.** *Resolved in this draft* —
   `models_endpoint` (Section 6) is defined to be satisfiable by an
   existing OpenAI-compatible models-list endpoint, with this spec's
   capability fields layered on additively. Still open: whether that
   should eventually become a MUST rather than a SHOULD (see item 8
   below).
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
7. **Sensitive-field gating mechanism.** Section 5.1 says gateways
   MAY use a scope or permission to gate `mcp_servers` /
   `mcp_servers_endpoint` beyond plain authentication, but this draft
   does not standardize a scope name (e.g. `discovery.mcp`) or
   require one. Should a future revision mandate a specific scope so
   clients can request least-privilege discovery credentials
   portably across gateways?
8. **OpenAI-shape as MUST vs. SHOULD.** Section 6 requires only `id`
   and merely RECOMMENDs `object`/`created`/`owned_by` on
   `models_endpoint` entries, to avoid forcing gateways with a
   differently-shaped native models endpoint (e.g. one that isn't
   OpenAI-compatible at all) to add a second endpoint. Is that
   flexibility worth the interop cost, or should conformance require
   the OpenAI shape outright?

## 21. References

- RFC 2119 — Key words for use in RFCs to Indicate Requirement Levels
- RFC 8615 — Well-Known Uniform Resource Identifiers (URIs)
- RFC 8414 — OAuth 2.0 Authorization Server Metadata
- OpenID Connect Discovery 1.0 — openid.net/specs/openid-connect-discovery-1_0.html
- Model Context Protocol Specification — [mcp-spec]

[mcp-spec]: https://modelcontextprotocol.io/specification

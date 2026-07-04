# Scope

This document defines "Scope" for purposes of Section 9.13 and the
patent licensing commitments in Section 2 of the Community
Specification License 1.0 (`LICENSE`) covering this specification.

## In scope

Necessary Claims are within Scope to the extent they are infringed by
an Implementation of:

1. The discovery document format defined in
   `rfc/draft-ai-routing-discovery-00.md` (and any successor draft
   numbered `-01`, `-02`, etc., or any Approved Specification
   produced from it), including:
   - The `.well-known/ai-routing-discovery` retrieval mechanism
     (Section 4 of the draft).
   - The top-level discovery document metadata fields (Section 5).
   - The Model Object, MCP Server Object, and Rate Limit Object
     schemas (Sections 6–8).
   - The versioning and extension mechanisms described in Sections
     10–11.
2. The JSON Schema in `schema/ai-routing-discovery.schema.json`, to
   the extent it restates normative requirements of the draft rather
   than introducing new ones.

## Out of scope

The following are explicitly **not** within Scope, and no patent
license is implied for them by virtue of implementing this
specification:

1. The wire format of any underlying chat completion, embeddings, or
   other inference API that a discovery document merely points to
   (e.g. the OpenAI-compatible or Anthropic Messages API request/response
   shapes). This specification only describes how to locate those
   endpoints, not their contents.
2. The Model Context Protocol itself, which is defined and licensed
   independently by its own specification.
3. Any vendor extension field (`x_`-prefixed, per Section 11 of the
   draft) defined by a particular Contributor or implementer.
4. Example code, sample gateways, or SDK implementations in this
   repository, which are source code governed by `LICENSE-CODE`
   (Section 4 of the Community Specification License), not this
   patent Scope.

## Amending Scope

Per Section 9.13 of the License, changes to Scope do not apply
retroactively — a narrowing or broadening of Scope only affects
Contributions made after the change is merged. Amendments follow the
same process described in `Governance.md`.

# Governance

This document defines the governance process for the AI Routing
Discovery specification, as referenced by the Community Specification
License 1.0 (`LICENSE`).

## Status levels

- **Draft Specification.** Any version of the specification under
  active development, including everything currently in this
  repository (`draft-ai-routing-discovery-00.md` and later numbered
  drafts). Draft Specifications may change incompatibly at any time.
- **Approved Specification.** A version of the specification that has
  gone through the review process below and been formally marked as
  approved via a tagged release (e.g. `v1.0`) and a corresponding
  entry in `CHANGELOG.md`. Only Approved Specifications carry the
  stability expectations described in the draft's Section 10
  (Versioning and Extensibility).

There is currently no Approved Specification. This project has not
yet been adopted by a neutral standards body or multi-vendor working
group (see the draft's Section 20, "Open Questions," item 6).

## Decision process

Until a formal Working Group with multiple independent Contributors
exists, decisions on the Draft Specification are made by rough
consensus among active Contributors, with the repository maintainer(s)
as tie-breaker. This is intended as a starting point, not a permanent
arrangement — see "Evolving this document" below.

A change is proposed by opening a pull request against the relevant
draft file. A change may be merged once:

1. It does not silently break a MUST/MUST NOT requirement relied upon
   by an existing implementation, without a corresponding major
   version bump per Section 10 of the spec; and
2. At least one Contributor other than the author has reviewed it, if
   more than one active Contributor exists at the time.

## Path to Approved Specification

Before any draft is proposed for Approved status, the following must
be true:

1. At least two independent implementations (e.g. one gateway
   publishing the document, one client SDK consuming it) exist and
   have been tested against each other.
2. The "Open Questions" section of the draft has been resolved or
   explicitly deferred with rationale.
3. At least one entity other than the original author has joined as a
   Contributor per the License's definition.

Marking a draft Approved requires a tagged release and update to this
file recording the vote/consensus and participating Contributors.

## Evolving this document

This Governance.md may itself be amended by the same rough-consensus
process described above, and should be replaced wholesale if/when
this specification moves under a formal standards body (e.g. as an
IETF individual submission or into a dedicated multi-vendor working
group), at which point that body's governance rules take precedence.

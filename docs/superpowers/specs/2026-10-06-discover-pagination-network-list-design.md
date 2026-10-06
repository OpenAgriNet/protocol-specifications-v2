# Design: Discover API Pagination & Context.networkId List Support

**Status:** Approved for implementation planning
**Date:** 2026-10-06
**Scope file:** `api/v2.0.0/beckn.yaml`
**Branch:** `feature/discover-pagination-network-list`

> This is a lightweight internal design spec, not a conformant Beckn RFC. A
> full RFC satisfying `docs/RFC_Authoring_Guide.md` (Document Details,
> Motivation, Roles and Actors, Protocol Flows with sequence diagrams,
> Security/Privacy Considerations, Prior Art, Conformance Requirements, etc.)
> is deferred as a follow-up task and MUST be authored before this change can
> be considered for merge into `main`, per this repo's process (an RFC is a
> required companion artifact for any protocol change, not optional).

## 1. Problem Statement

Two independent gaps were identified in `api/v2.0.0/beckn.yaml`:

1. **No pagination on `/discover`.** `OnDiscoverAction` (the response schema
   for `/on_discover`, lines 2356-2379) returns a flat `catalogs` array with
   no way for a caller to request or receive a bounded page of results. An
   offset-based `Pagination` schema (`total`/`limit`/`offset`, lines
   4503-4532) already exists in the spec but is only wired into two
   deprecated Cataloging Service endpoints (`/catalog/subscription`,
   `/catalog/search`), not into the live `/discover` flow.

2. **`Context.networkId` only accepts a single network id.** `Context`
   (lines 2901-3026) defines `networkId` as a plain `string` (lines
   2985-2991), describing the Beckn subnet a transaction occurs within. A
   transaction MAY need to reference more than one subnet (e.g. resolving
   across federated/bridged networks), which the current schema cannot
   express without breaking existing single-id producers and consumers.

## 2. Design: Discover Pagination

### 2.0 Shared `PaginationParams` (reuse, not duplication)

The existing `Pagination` schema (response metadata: `total`/`limit`/`offset`)
is not a fit to `$ref` directly into a request, since `total` is
server-assigned output and makes no sense as caller input. Rather than
hand-duplicating `limit`/`offset` a third time (they were already duplicated
between `Pagination` and the deprecated `CatalogSearchAction`), a new shared
`PaginationParams` schema holds just `limit`/`offset`:

```yaml
PaginationParams:
  type: object
  description: >-
    Reusable offset-based pagination fields shared between paginated
    list requests and the Pagination response metadata that describes
    them.
  properties:
    limit:
      type: integer
      minimum: 1
      maximum: 100
      description: Maximum number of records in this page.
    offset:
      type: integer
      minimum: 0
      default: 0
      description: Zero-based index of the first record in this page.
```

`Pagination` is refactored to build on it (non-breaking — same resulting
JSON shape for existing consumers of the deprecated `/catalog/search` and
`/catalog/subscription` endpoints):

```yaml
Pagination:
  description: ... # unchanged
  allOf:
    - $ref: '#/components/schemas/PaginationParams'
    - type: object
      properties:
        total:
          type: integer
          minimum: 0
          description: Total number of matching records across the entire result set.
```

The deprecated `CatalogSearchAction`/`CatalogPullAction` request schemas are
left untouched — they keep their own inline `limit`/`offset` (with a
`default: 20` specific to that endpoint), per §4 Out of Scope.

### 2.1 Request — `DiscoverAction`

Add `limit`/`offset` properties that `$ref` the individual property
definitions in `PaginationParams`, keeping the flat top-level request shape
while fully reusing the shared constraints/description (no `allOf`, so no
interaction with `DiscoverAction`'s existing `additionalProperties: false`):

```yaml
properties:
  intent:
    $ref: '#/components/schemas/Intent'
  limit:
    $ref: '#/components/schemas/PaginationParams/properties/limit'
  offset:
    $ref: '#/components/schemas/PaginationParams/properties/offset'
```

### 2.2 Response — `OnDiscoverAction`

Add an optional `pagination` property that reuses the existing
`#/components/schemas/Pagination` schema:

```yaml
OnDiscoverAction:
  type: object
  required:
    - catalogs
  properties:
    catalogs:
      type: array
      description: 'Array of catalogs matching the discovery request.'
      items:
        $ref: '#/components/schemas/Catalog'
    pagination:
      $ref: '#/components/schemas/Pagination'
  additionalProperties: false
```

### 2.3 Backward compatibility

Both additions are optional. A `DiscoverAction` request that omits
`limit`/`offset` behaves exactly as today. A `BPP` that does not implement
pagination simply omits `pagination` in its `OnDiscoverAction` response,
returning the full result set as it does today. No existing payload becomes
invalid.

## 3. Design: `Context.networkId` List Support

### 3.1 Schema change

Change `networkId` from a plain `string` to a `oneOf` of a single string or
an array of strings:

```yaml
networkId:
  description: >
    The unique identifier(s) of the Beckn subnet(s) within which this
    transaction is occurring. Actors use this value to resolve the
    subnet's registry URL before performing identity and address lookups.
    The RECOMMENDED format is namespace_id followed by a slash followed
    by registry_id, for example acmenet.org/charge-net.
    MAY be a single string (single-network transaction) or an array of
    strings (multi-network transaction). When an array is provided, order
    is significant: actors MUST attempt registry resolution starting with
    the first entry, treating it as the primary network, and MAY fall back
    to subsequent entries if resolution against the primary fails.
  oneOf:
    - type: string
    - type: array
      items:
        type: string
      minItems: 1
```

### 3.2 List semantics

- The array form is **ordered**. The first entry is the primary network;
  actors MUST attempt resolution against it first.
- Actors MAY fall back to subsequent entries if resolution against the
  primary network fails.

### 3.3 Backward compatibility

- Existing producers sending a plain string remain valid — the `string`
  branch of `oneOf` is unchanged from today's schema.
- Existing consumers that only handle the string form will need a code
  change to branch on the array form; this is a schema-level
  backward-compatible addition (old payloads still validate against the new
  schema), not a consumer-transparent one. This distinction MUST be called
  out explicitly in the eventual RFC's Section 14 (Breaking Changes and
  Migration) and Section 11 (Schema Changes).

### 3.4 Naming collision check

A distinct, pre-existing `networkIds` field (plural, array of strings)
already exists in `CatalogSearchAction`/`CatalogSubscriptionAction`/
`CatalogPullAction` filters (e.g. lines 1674, 1965, 4419, 4465, 4565). That
field is an unrelated search-filter concept scoped to the (deprecated)
Cataloging Service endpoints, not `Context`. No rename of `Context.networkId`
is needed to avoid collision, since the two fields live in different
schemas and `Context.networkId` keeps its existing name.

## 4. Out of Scope

- Changing the deprecated `/catalog/search` and `/catalog/subscription`
  endpoints' request schemas or behavior. (Note: `Pagination`'s *internal*
  structure was refactored to build on the new shared `PaginationParams`,
  per §2.0 — this is a non-breaking, shape-preserving change; these
  endpoints' request schemas and response shape are unaffected.)
- Cursor-based pagination (offset/limit was chosen for consistency with
  existing precedent in this spec; see Open Questions).
- Renaming or consolidating `Context.networkId` with the unrelated
  `networkIds` filter field.
- Authoring the full conformant RFC (deferred follow-up, see header note).

## 5. Open Questions

1. Should a future conformance requirement enumerate what "resolution
   fails" means for the primary-then-fallback network resolution order in
   §3.2 (e.g. registry unreachable vs. registry returns no match)? Deferred
   to the full RFC's Normative Requirements section.
2. Should `/catalog/search`'s and `/catalog/subscription`'s existing
   `networkIds` filter field be renamed in a future cleanup to avoid reader
   confusion with `Context.networkId`, even though no functional collision
   exists today? Deferred — out of scope per §4.

## 6. Validation Plan

- Lint `api/v2.0.0/beckn.yaml` with the repo's existing OpenAPI validation
  tooling (e.g. `@redocly/cli lint`) after edits.
- Add/update example payloads for `DiscoverAction` (with `limit`/`offset`),
  `OnDiscoverAction` (with `pagination`), and `Context` (with array-form
  `networkId`), and validate each against the updated schema.
- Confirm existing examples using string-form `networkId` and
  `DiscoverAction`/`OnDiscoverAction` without pagination fields still
  validate unchanged.

# Design: Discover API Pagination & Context.networkId List Support

**Status:** Approved for implementation planning
**Date:** 2026-10-06
**Scope file:** `api/v2.0.0/beckn.yaml`
**Branch:** `feature/discover-pagination-network-list`
**Merge target:** `release-0.2.0` (not `main` — see the RFC gate below)

> This is a lightweight internal design spec, not a conformant Beckn RFC. A
> full RFC satisfying `docs/RFC_Authoring_Guide.md` (Document Details,
> Motivation, Roles and Actors, Protocol Flows with sequence diagrams,
> Security/Privacy Considerations, Prior Art, Conformance Requirements, etc.)
> is deferred as a follow-up task and MUST be authored before this change can
> be considered for merge into `main`, per this repo's process (an RFC is a
> required companion artifact for any protocol change, not optional).
>
> This change targets `release-0.2.0`, so the gate is not yet tripped — but
> the RFC MUST land before `release-0.2.0` is promoted to `main`.

## 1. Problem Statement

Two independent gaps were identified in `api/v2.0.0/beckn.yaml` (line
references below are to the file as it stood *before* this change):

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
server-assigned output and makes no sense as caller input.

`limit`/`offset` are currently hand-declared in **five** places: `Pagination`,
the deprecated `CatalogSearchAction` and `CatalogPullAction` request schemas,
and the `limit`/`offset` query parameters on `/catalog/subscription`. Rather
than making `/discover` a sixth, a new shared `PaginationParams` schema holds
just `limit`/`offset`:

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
      description: Zero-based index of the first record in this page.
```

`default: 0` is deliberately **not** in the shared base: `default` is a
request-side annotation, and on server-assigned response metadata it would be
meaningless. `DiscoverAction` re-declares it as a `$ref` sibling (§2.1), which
is valid in OpenAPI 3.1.1 / JSON Schema 2020-12.

`Pagination` is refactored to build on it. The resulting JSON *shape* is
identical for existing consumers of the deprecated `/catalog/search` and
`/catalog/subscription` endpoints, with one deliberate constraint change
noted below:

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

**Constraint delta on `Pagination` (the one non-cosmetic effect):**

| field | before | after |
|---|---|---|
| `total` | `minimum: 0` | unchanged |
| `limit` | `minimum: 1`, no maximum | `minimum: 1`, **`maximum: 100`** |
| `offset` | `minimum: 0` | unchanged |

The added `maximum: 100` on the *response* `limit` is a tightening, not a
pure refactor: a CS that reported `limit: 200` in a response would have
validated before and will not now. This is intentional and consistent — all
three request paths already cap `limit` at 100, so no conforming server can
legitimately report a page size above it. Called out here so the change is
not mistaken for a no-op.

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
    default: 0
```

The `default: 0` sibling alongside `$ref` is valid in OpenAPI 3.1.1 (JSON
Schema 2020-12 treats `$ref` as an ordinary keyword, so adjacent keywords
apply); it keeps the request-side default on the request schema without
leaking it into `Pagination`.

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

(List items sit at the parent key's column, matching the indentation style
used throughout `beckn.yaml`.)

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
  per §2.0. Their request schemas and the response JSON shape are
  unaffected; the single effective change is the added `maximum: 100` on
  the response `limit`, tabulated in §2.0.)
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

## 6. Validation

This repository has no committed OpenAPI lint tooling — there is no
`package.json`, `Makefile`, or CI workflow — so validation was performed
directly against the schema rather than via `@redocly/cli`. Anyone adding
lint tooling later should wire these same checks into it.

Performed, all passing:

- [x] `api/v2.0.0/beckn.yaml` parses as valid YAML and the `Pagination` /
      `PaginationParams` / `DiscoverAction` schemas resolve as intended.
- [x] Example payloads added and validated against their request schemas
      (JSON Schema 2020-12):
      - `/discover` → `paginatedDiscover` (string `networkId` + `limit`/`offset`)
      - `/discover` → `multiNetworkDiscover` (array-form `networkId`)
      - `/on_discover` → `paginatedOnDiscover` (response `pagination` metadata)
- [x] Regression sweep: all 8 examples in the file (5 pre-existing + 3 new)
      validate against their schemas, confirming existing string-form
      `networkId` payloads and pagination-free `DiscoverAction` /
      `OnDiscoverAction` payloads are unaffected.

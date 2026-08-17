---
name: coorpacademy-provision-users-scim
description: >-
  Provision, look up, update and deactivate Coorpacademy learners through the SCIM 2.0 API, scoped to a
  single brand. Use when connecting an identity provider (Entra ID, Okta, Ping) to a Coorpacademy
  learning portal, or when reconciling a workforce roster against the platform.
api: Coorpacademy SCIM API
base_url: https://api.coorpacademy.com/scim
spec: openapi/coorpacademy-scim-openapi.json
operations:
  - listUsers
  - onboardingPOST
  - recommendedCoursePOST
  - putUser
  - patchUser
generated: '2026-08-17'
method: generated
source: >-
  openapi/coorpacademy-scim-openapi.json (harvested 2026-08-17), plus a live unauthenticated probe of
  https://api.coorpacademy.com/scim/coorp/Users. No provider-published skill or AGENTS.md exists.
consequence: writes-identity-records
escalation: human-approval-required-for-writes
---

# Provision Coorpacademy learners over SCIM

Coorpacademy exposes SCIM 2.0 user provisioning, and it is the one part of the estate that follows a
standard properly — error bodies come back in the real RFC 7644 envelope. But the published surface is
narrow, so read the limits before you wire an IdP to it.

## Before you start

**Base URL is `https://api.coorpacademy.com/scim`.** The specification declares
`servers[0].url = https://api.coorpacademy.com` with paths of the form `/{brand}/Users`, which resolves
to a path that is not served. The `/scim` prefix is required. Verified 2026-08-17: a GET of
`https://api.coorpacademy.com/scim/coorp/Users` returns a SCIM error body; the spec-derived URL does not.
See `overlays/coorpacademy-scim-overlay.yaml`.

**Auth is a header-borne key.** The spec declares `apiKey` in a header named `token`. The live 400 body
reports `JWTError: Expecting type: string at key: authorization but instead got: undefined`, which says
the implementation reads a JWT from an `authorization` header. The contract and the behaviour disagree —
confirm with Coorpacademy which one is authoritative before you go live. There is no OAuth and no scopes
anywhere in this estate.

**`{brand}` is the tenant.** Every path is brand-scoped. You must know the brand identifier before any
call; there is no operation that lists brands from this service (that lives in the Platform API,
`brandsListGET`).

## The operations, and their real names

The published `operationId` values on two of these are wrong — copy-pasted from the unrelated email API.
Both names are given so you can match either the spec or a generated client.

| Step | Method + path | Published operationId | Corrected |
|---|---|---|---|
| List users | `GET /{brand}/Users` | `listUsers` | — |
| Create a user | `POST /{brand}/Users` | `onboardingPOST` | `createScimUser` |
| Read one user | `GET /{brand}/Users/{userId}` | `recommendedCoursePOST` | `getScimUser` |
| Replace a user | `PUT /{brand}/Users/{userId}` | `putUser` | — |
| Patch a user | `PATCH /{brand}/Users/{userId}` | `patchUser` | — |

`recommendedCoursePOST` on a GET operation is the one to watch: a code generator will emit a method
whose name says POST and whose noun belongs to a different product.

## Steps

1. **Confirm the brand and the auth header.** Get the brand identifier and the working key from
   Coorpacademy. Send one `listUsers` call and check you get a 200 rather than the SCIM 400.
2. **Reconcile before you write.** Call `listUsers` (`GET /{brand}/Users`) and diff against your source
   roster. Do this first, every time — see the idempotency warning below.
3. **Create missing users** with `onboardingPOST` (`POST /{brand}/Users`), body shape
   `IDPUserBodyRequest`. One call per user; there is no bulk operation.
4. **Read back** with `recommendedCoursePOST` (`GET /{brand}/Users/{userId}`) to confirm the record
   landed and to capture the platform's `userId`. Capture it: `userId` is the join key into every
   learner-data operation in the Progression and Review APIs, and nothing else resolves it.
5. **Update** with `putUser` for a full replacement or `patchUser` for a partial change.
6. **Deactivate** with `patchUser` setting the user inactive. **There is no DELETE operation in the
   published contract**, and the contract does not state which attribute deactivation uses. Confirm the
   de-provisioning path with Coorpacademy in writing before you rely on it for a leaver process.

## Rules you must follow

- **No idempotency, anywhere.** Zero matches for `idempoten` across all 155 operations in this estate.
  A retried `POST /{brand}/Users` after a socket timeout can create a duplicate person. Always
  re-run step 2 before re-attempting a create — reconcile, then write.
- **Errors come back in the SCIM envelope, minus the useful part.** Bodies are
  `{"schemas":["urn:ietf:params:scim:api:messages:2.0:Error"],"detail":"...","status":N}`. The `scimType`
  field is **omitted**, so you cannot distinguish `invalidValue` from `uniqueness` from `mutability`
  programmatically. Parse `detail` as a last resort and expect it to change without notice.
- **Only 200 and 500 are declared** on all five operations. No 400, 401, 403, 404 or 409 appears in the
  contract even though the live surface returns 400. Handle undocumented statuses.
- **No pagination or filtering is declared.** SCIM's `startIndex`, `count` and `filter` are absent from
  the contract, so you cannot tell whether an IdP's standard paged sync will work. Test with a real
  roster size before committing.
- **No rate limits and no rate-limit headers.** No 429 is declared and no `RateLimit-*` or `Retry-After`
  header is served. Self-throttle: serialise creates, add exponential backoff on 5xx.
- **No `/Groups`, `/ServiceProviderConfig`, `/Schemas` or `/ResourceTypes`.** Group-based provisioning is
  not supported by the published contract, and your IdP cannot self-configure by discovery — it will
  have to be configured by hand.
- **This operation handles personal data.** Treat every response as personal data under RGPD.
  Coorpacademy's own privacy notice (last updated 2020-04-14) names a DPO at
  4-6 boulevard Poissonnière, 75009 Paris.

## Cross-references

- `authentication/coorpacademy-authentication.yml` — the five different auth headers in this estate
- `errors/coorpacademy-problem-types.yml` — all three error envelopes
- `conventions/coorpacademy-conventions.yml` — idempotency, pagination, tenancy
- `overlays/coorpacademy-scim-overlay.yaml` — the corrections applied above
- `conformance/coorpacademy-conformance.yml` — SCIM 2.0 conformance assessment and its gaps

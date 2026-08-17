---
name: coorpacademy-configure-brand-sso
description: >-
  Inspect and configure enterprise single sign-on for a Coorpacademy brand (tenant) — read the brand,
  extract SSO settings from an IdP SAML metadata.xml, and update the SAML or OIDC configuration including
  the claim-to-user mapping. Use when onboarding a new corporate customer's identity provider or
  diagnosing a broken SSO login.
api: Coorpacademy Platform API
base_url: https://platform.coorpacademy.com/api/v1
spec: openapi/coorpacademy-platform-openapi.json
operations:
  - brandsListGET
  - brandsGET
  - brandsIdExistGET
  - brandsIdGET
  - brandsIdSSOPOST
  - brandsIdPUT
generated: '2026-08-17'
method: generated
source: >-
  openapi/coorpacademy-platform-openapi.json (harvested 2026-08-17), plus a live unauthenticated probe of
  https://platform.coorpacademy.com/api/v1/brands. No provider-published skill exists.
consequence: writes-tenant-authentication-configuration
escalation: human-approval-required
---

# Configure SSO for a Coorpacademy brand

A "brand" is a Coorpacademy tenant — a customer's own learning portal, with its own host, skin and
identity configuration. This skill covers the SSO half of brand management. It is the highest-privilege
surface in the estate: the brand payload contains the customer's IdP certificate and the mapping that
decides who a logged-in person becomes.

## Before you start

- **Base URL:** `https://platform.coorpacademy.com/api/v1`. Confirmed 2026-08-17: an unauthenticated
  `GET /api/v1/brands` returns 401 `{"code":"server_error","status":401,"success":false,"message":"Unauthorized","errors":[...]}`.
- **Auth header is `authentication`.** Same as the Progression API, different from every other service.
- **The spec is Swagger 2.0** (`info.version 1.412.0`), so it uses `host` + `basePath` rather than
  `servers[]`. That build number is high, which means this service is still actively shipped.
- **This is administrative.** Treat every write here as a change to a customer's authentication path.

## Steps

1. **Find the brand.** `brandsListGET` (`GET /brands/list`) for the list, or `brandsGET` (`GET /brands`)
   for full objects. `brandsIdExistGET` (`GET /brands/{id}/exist`) is a cheap existence check that returns
   a `Boolean` `{value}` and declares 404.
2. **Read the current configuration.** `brandsIdGET` (`GET /brands/{id}`) returns a `Brand`
   `{alias, host, payload}`. The `BrandPayload` carries the portal identity (`name`, `description`,
   `skin`, `moocName`, `teamName`, `teamEmail`, `assistanceEmail`, `contentCategoryName`, `sector`,
   `baseUrl`, `defaultTimePerChapter`, `language`), a `Slider` of onboarding slides, an `ImportPayload`,
   and the `sso` block.
   **Treat this response as a secret.** `SSOPayload.saml2.cert` is the customer's IdP signing
   certificate. Do not log it, do not persist it outside the customer's own scope.
3. **Extract settings from IdP metadata.** `brandsIdSSOPOST` (`POST /brands/{id}/metadatas`) —
   "extract brand sso config info from metadata.xml". Post the identity provider's SAML metadata document
   and the service parses it into the shape the brand needs. This is the intended onboarding path: use it
   rather than hand-assembling a `SAMLPayload`.
4. **Review the parsed result** against the `SAMLPayload` fields the contract declares:
   `authnRequestBinding`, `disableRequestedAuthnContext`, `identifierFormat`, `forceAuthn`,
   `skipRequestCompression`, `logoutUrl`, `idpIssuer`, `acceptedClockSkewMs`, `signatureAlgorithm`,
   `cert`, `entryPoint`, and `userMapping`. For OIDC the shape is `OIDCPayload` `{auth, userMapping}`.
5. **Get the claim mapping right.** `userMappingPayload` maps IdP claims onto
   `uniqueLogin`, `emails`, `name`, `provider`, `language`, `providerInfos`, `roles`, `organizations`.
   `uniqueLogin` is the identity anchor — get it wrong and you either merge two people into one account or
   split one person across two. Agree the source claim with the customer's identity team in writing.
6. **Apply** with `brandsIdPUT` (`PUT /brands/{id}`). `SSOPayload` also carries the switches
   `enabled`, `connectEnabled` and `enableDefaultRoute` — decide deliberately whether SSO is enforced or
   offered alongside a default route, because `enableDefaultRoute` is what determines whether a
   non-SSO login path remains open.
7. **Test with a real login before you leave.** There is no validation, dry-run or test-connection
   operation in the contract. The only verification is an actual SSO login against the brand host.

## Rules you must follow

- **Never call `brandsDELETE` or `brandsIdMigratePOST` from automation.** `DELETE /brands/{id}` deletes a
  tenant (declares 202, 400, 404, 409) and `POST /brands/{id}/migrate` migrates one between clusters
  (returns `BrandMigrationStatus` `{message, status, brand, cluster}`). Both are destructive at the
  customer level and neither has an undo. They are excluded from this skill's `operations` list on
  purpose.
- **`brandsPOST` creates a tenant.** Also excluded — creating a brand is a commercial act, not a
  configuration change.
- **No idempotency key.** A retried `PUT /brands/{id}` replaces the whole brand. Always re-read step 2
  immediately before the write and send back the full current payload with only your intended change
  applied, or you will silently blank fields.
- **409 on write** means `header brandName value is not the same as given id parameter` — a mismatch
  between the `brandName` header and the `{id}` path parameter, not a concurrency conflict.
- **`default` responses are typed as `Error`.** Six of the nine operations declare a `default`
  "Unexpected error" catch-all alongside their specific statuses, so handle unmapped statuses.
- **No rate limits and no rate-limit headers.** Not a concern at this call volume, but there is no signal.
- **This is a change to who can log in.** Require explicit human approval for steps 3 and 6. An agent
  must not reconfigure a customer's authentication unattended.

## Cross-references

- `data-model/coorpacademy-data-model.yml` — `Brand`, `SSOPayload`, `SAMLPayload`, `OIDCPayload`,
  `userMappingPayload` and how tenancy propagates through the estate
- `conformance/coorpacademy-conformance.yml` — SAML 2.0 and OIDC assessment (Coorpacademy is an OIDC
  relying party, not a provider: it serves no `/.well-known/openid-configuration`)
- `skills/coorpacademy-provision-users-scim.md` — the provisioning half of the same integration
- `security/coorpacademy-domain-security.yml` — TLS/HSTS/DNS posture of the hosts involved

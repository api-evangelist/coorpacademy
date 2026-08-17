---
name: coorpacademy-publish-certification
description: >-
  Author and publish a Coorpacademy certification through the edition → snapshot → consommation flow, and
  revert a bad edit. Use when building a certification programme in a brand's content repository, or when
  a change was made and learners still cannot see it.
api: Coorpacademy Content API
base_url: https://content.coorpacademy.com/api/v2
spec: openapi/coorpacademy-content-openapi.json
operations:
  - findCertifications
  - findOneCertification
  - upsertCertification
  - undoCertificationChange
  - findCertificationsConsommation
  - upsertCertificationConsommation
  - countCertificationConsommation
generated: '2026-08-17'
method: generated
source: >-
  openapi/coorpacademy-content-openapi.json (harvested 2026-08-17 from
  https://content.coorpacademy.com/api-docs), plus a live unauthenticated probe of
  https://content.coorpacademy.com/api/v2/notifications. No provider-published skill exists.
consequence: writes-learner-facing-content
escalation: human-approval-required-for-publish
---

# Publish a Coorpacademy certification

**The single most common way to get this API wrong: you write to the edition, get a 201, report success —
and learners see nothing.** The content API keeps authoring and consumption on separate path prefixes, and
only the consumption side is what learners get.

## The four states

Certifications (and custom skills, and custom playlists) exist in up to four parallel shapes:

| Shape | Path prefix | What it is |
|---|---|---|
| **Edition** | `/repository/{repository}/certifications/...` | The mutable draft you edit |
| **Diff** | same | The pending change set since the last publish |
| **Snapshot** | same, via `upsert` | The published immutable generation, keyed by `version` |
| **Consommation** | `/consommation/repository/{repository}/certifications/...` | The learner-facing projection |

`upsertCertification` is described in the spec as "create a certification diff **and** upsert a snapshot" —
so it does both the pending-change and the published-generation write. The `/consommation/` surface has its
own separate upsert. Confirm with Coorpacademy which of the two is authoritative for learner visibility in
your deployment before you automate this; the contract describes both and reconciles neither.

## Before you start

- **Base URL:** `https://content.coorpacademy.com/api/v2`. Note the spec lists a relative `/api/v2` and a
  `localhost:3700` server ahead of the production host — do not take `servers[0]`. See
  `overlays/coorpacademy-content-overlay.yaml`.
- **The spec is not where Coorpacademy's own Swagger UI says it is.** The explorer at
  `https://api.coorpacademy.com/` builds this document's URL as
  `https://content.coorpacademy.com/swagger.json`, which 404s. It is served at `/api-docs`. The Content
  API entry in the provider's own API explorer is broken.
- **Auth header is lower-case `authorization`.** An unauthenticated call returns 401
  `{"message":"Invalid or missing authorization key"}` — the bare-message envelope, not the Express one.
- **`{repository}` is the tenant scope.** Certifications are addressed by the composite key
  `(repository, ref, version)`. There is no object id, no UUID, no URN. `ref` is an author-chosen string
  you must know or choose.

## Steps

1. **Inventory** — `findCertifications` (`GET /repository/{repository}/certifications`) to list snapshots
   in the repository. Pass `states` to filter on the publication lifecycle
   (`published` / `draft` / `archived` / `deleted`) and `includeDeleted` if you need tombstones.
2. **Read the current generation** — `findOneCertification`
   (`GET /repository/{repository}/certifications/{ref}`). Capture the `version`: you will need it, and a
   write against a `(repository, ref, version)` that already exists returns **409**.
3. **Compose the change.** The body is `CertificationEdition` / `CertificationSnapshot`, carrying
   `CertificationLocales` (per-language labels), `CertificationMeta`, and a structure of
   `CertificationCondition` entries (each referencing content refs through
   `CertificationConditionItemsDecoder`) plus `CertificationRewards`, which fans out to
   `DiplomaRewards`, `ResourceRewards` and `StarsRewards`.
4. **Write** — `upsertCertification` (`POST /repository/{repository}/certifications/upsert`). Expect 201.
   A 409 means your `(repository, ref, version)` triple already exists — re-read step 2 and bump.
5. **VERIFY ON THE LEARNER SIDE.** This is the step people skip. Call `findCertificationsConsommation`
   (`GET /consommation/repository/{repository}/certifications`) and confirm your change is present there.
   `countCertificationConsommation`
   (`GET /consommation/repository/{repository}/certifications/count`) gives counts by state. If your change
   is in the edition and not in the consommation, learners cannot see it and you are not done. If your
   deployment requires an explicit consumption write, that is
   `upsertCertificationConsommation` (`POST /consommation/repository/{repository}/certifications/upsert`).
6. **Revert if wrong** — `undoCertificationChange`
   (`PUT /repository/{repository}/certifications/{ref}/undo`), described as "Revert modification since the
   last publish". This is a genuinely useful primitive and rarer than it should be. It reverts the
   **edition** to the last published snapshot; it does not roll back a snapshot that has already gone out.

## Rules you must follow

- **Never call step 4 without step 5.** Reporting a successful upsert without verifying the consommation
  surface is the failure mode this skill exists to prevent.
- **No idempotency key exists.** A retried upsert after a timeout can produce a 409 (best case) or a
  duplicate generation. Always re-read (step 2) before re-attempting.
- **409 is a uniqueness violation, not a transient error.** Do not back off and retry the same body —
  change the `ref` or the `version`.
- **Deletion is soft.** State moves to `deleted` rather than the record being removed. A resource that
  404s on a plain read may still exist behind `includeDeleted=true`.
- **Errors carry no useful code.** Two envelopes appear on this service — the Express
  `{id, code, status, success, message, errors[]}` and a bare `{message}` — and `code` was
  `"server_error"` on every live body observed. Branch on the HTTP status.
- **No rate limits, no headers, no 429 declared.** Serialise your writes.
- **Publishing changes what learners see.** Treat step 4 and any `/consommation/` write as requiring
  explicit human approval. Do not let an agent publish a certification unattended.
- **The same pattern applies to sibling entities.** Custom skills (`findCustomSkillSnapshots`,
  `upsertCustomSkillSnapshot`, `undoCustomSkillsChange`) and custom playlists (`findCustomPlaylists`,
  `upsertCustomPlaylist`, `undoCustomPlaylistChange`) work identically, each with its own
  `/consommation/` twin.

## Cross-references

- `data-model/coorpacademy-data-model.yml` — the edition/snapshot/consommation pattern in full, and the
  certification entity graph
- `conventions/coorpacademy-conventions.yml` — soft delete, composite keys, tenancy
- `errors/coorpacademy-problem-types.yml` — what each status means on this service
- `overlays/coorpacademy-content-overlay.yaml` — server correction, tag declarations, filled-in summaries

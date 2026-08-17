---
name: coorpacademy-ingest-external-content
description: >-
  Bring third-party learning content into a Coorpacademy brand — mint presigned S3 upload URLs, register
  external courses and contents, run a bulk ingest, and read the per-row error report. Use when loading a
  customer's own course library or a partner catalogue into their Coorpacademy portal.
api: Coorpacademy Content API
base_url: https://content.coorpacademy.com/api/v2
spec: openapi/coorpacademy-content-openapi.json
operations:
  - findExternalCourses
  - createExternalCourse
  - findExternalContents
  - createExternalContent
  - bulkExternalContentUPSERT
  - bulkExternalContentsGETAll
  - bulkExternalContentGET
  - generateMetadataReport
  - externalPOST
  - presignedUrlPOST
  - presignedBulkUrlPOST
generated: '2026-08-17'
method: generated
source: >-
  openapi/coorpacademy-content-openapi.json, openapi/coorpacademy-external-openapi.json and
  openapi/coorpacademy-scorm-openapi.json (all harvested 2026-08-17). No provider-published skill exists.
consequence: writes-learner-facing-content
escalation: human-approval-required-for-bulk
---

# Ingest external content into Coorpacademy

Two different jobs sit behind this: registering **metadata** about external courses and contents (Content
API), and getting the **bytes** into storage (the External, SCORM and Media services, each of which just
mints a presigned S3 URL). They live on different hosts with different auth headers.

## Which service does what

| Job | Service | Base | Auth header |
|---|---|---|---|
| Register courses/contents, run bulk ingest | Content | `https://content.coorpacademy.com/api/v2` | `authorization` |
| Presigned URL for arbitrary external content | External | `https://api.coorpacademy.com/external` | **not declared** |
| Presigned URL for a SCORM package (single/bulk) | SCORM | `https://api.coorpacademy.com/scorm` | `Authorization` |
| Media upload + resize | Media | `https://api.coorpacademy.com/api-service` | **not declared** |

The External and Media specs declare no `securityScheme` at all. They are still gated: an unauthenticated
call to a sibling path on `api.coorpacademy.com` returns 403 `{"message":"Missing Authentication Token"}`
(AWS API Gateway). Ask Coorpacademy for the header name — it is not published.

## Steps — single content

1. **Register the course.** `createExternalCourse`
   (`POST /repository/{repository}/external-courses`). Expect 201; 409 means that `ref` already exists in
   the repository. Check first with `findExternalCourses`
   (`GET /repository/{repository}/external-courses`).
2. **Get an upload target.** `externalPOST` (`POST /presignedUrl/{ext}` on the External service) for
   generic content, or `presignedUrlPOST` (`POST /presignedUrl` on the SCORM service) for a SCORM package.
   **Do not use the staging server from the External spec** — it is declared as
   `https://api-staging.coorpacademy.com/h5p`, the H5P service's path, copy-pasted. See
   `overlays/coorpacademy-external-overlay.yaml`.
3. **PUT the bytes to S3** at the returned URL. This is a direct S3 upload; Coorpacademy is not in the
   data path.
4. **Register the content against the course.** `createExternalContent`
   (`POST /repository/{repository}/external-courses/{ref}/external-contents`). Verify with
   `findExternalContents` (`GET /repository/{repository}/external-courses/{ref}/external-contents`).

## Steps — bulk ingest

1. **Get a bulk upload target.** `presignedBulkUrlPOST` (`POST /presignedBulkUrl` on the SCORM service),
   or `presignedS3UrlPOST` on the SCORM Content service — note that one is **the only operation in the
   whole estate that declares a 501 Not Implemented**, so treat it as not guaranteed available.
2. **Upload the bulk file** to the presigned URL.
3. **Create the job.** `bulkExternalContentUPSERT`
   (`POST /repository/{repository}/bulk-external-contents/upsert`).
4. **Poll for state.** `bulkExternalContentsGETAll`
   (`GET /repository/{repository}/bulk-external-contents`) or `bulkExternalContentGET`
   (`GET .../{ref}`). Filter with `states` and `withTotalFailedItems`. **There is no webhook and no
   callback** — polling is the only completion signal in this estate (see
   `asyncapi/coorpacademy-event-surface.yml`).
5. **Read the failures — this is the step that matters.** `generateMetadataReport`
   (`GET /repository/{repository}/bulk-external-contents/{ref}/metadata-report`) returns per-row detail.
   Each `ExternalContentForBulk` can carry `ErrorResource` `{type, fileName}` and `ErrorCSV`
   `{column, errorCode, description}` rows.
   **The `errorCode` values are not enumerated anywhere in the contract**, so you cannot branch on them
   programmatically — read `description` and expect the vocabulary to change without notice. Log the
   report verbatim before you interpret it.
6. **Add locales if needed.** `addLocale` (`POST /repository/{repository}/jobs/add-locale`) adds a new
   locale across the repository's resources. Repository-wide and blunt — human approval only.

## Rules you must follow

- **No idempotency key anywhere.** A retried bulk upsert can create a second job over the same file.
  Always list existing jobs (step 4) before re-creating one.
- **Presigned URL mints are cheap individually and unbounded in aggregate**, with no rate-limit signal to
  throttle against. Cap your own concurrency.
- **A 400 from the External or SCORM services is ambiguous.** Both specs describe their 400 as
  "Undefined/ Internal errors", so a 400 there is **not** reliably a client fault and must not be
  classified as permanently non-retryable.
- **409 on registration is a uniqueness violation** on `(repository, ref, version)` — change the ref, do
  not back off and retry.
- **External content is learner-facing.** A successful bulk ingest changes what a customer's employees
  see. Require human approval for step 3 of the bulk flow and for `addLocale`.
- **Do not take `servers[0]`** on the Content spec — it is the relative `/api/v2`, followed by
  `localhost:3700`. Production is `https://content.coorpacademy.com/api/v2`.
- **The Content spec is served at `/api-docs`, not `/swagger.json`.** Coorpacademy's own Swagger UI links
  the wrong URL and the Content API entry fails to load in its own explorer.

## Cross-references

- `data-model/coorpacademy-data-model.yml` — `ExternalCourse`, `ExternalContent`, `BulkExternalContent`,
  `ErrorCSV`, `ErrorResource`
- `errors/coorpacademy-problem-types.yml` — the ambiguous-400 problem and the undocumented error registry
- `conventions/coorpacademy-conventions.yml` — the auth-header table and the missing idempotency
- `overlays/coorpacademy-external-overlay.yaml` — the wrong-staging-host correction
- `asyncapi/coorpacademy-event-surface.yml` — why you have to poll

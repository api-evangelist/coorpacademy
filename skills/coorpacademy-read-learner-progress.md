---
name: coorpacademy-read-learner-progress
description: >-
  Read a named learner's Coorpacademy progress — completions, stars, slide counts, review state,
  in-flight progressions and the platform's next-content recommendation. Use when building an HR
  reporting extract, syncing completion data into an HRIS or LMS, or answering "has this person finished
  the compliance course".
api: Coorpacademy Progression API
base_url: https://progression.coorpacademy.com/api
spec: openapi/coorpacademy-progression-openapi.json
operations:
  - progressionsGET
  - progressionGET
  - actionsGET
  - completionFromDynamodbForUserGET
  - reviewCompletionFromDynamodbForUserGET
  - starsBySkillIForUserGET
  - slideCountFromDynamodbForUserGET
  - heroRecommendationFromDynamodbGET
  - getUserSkillsToReview
generated: '2026-08-17'
method: generated
source: >-
  openapi/coorpacademy-progression-openapi.json and openapi/coorpacademy-review-openapi.json (harvested
  2026-08-17), plus a live unauthenticated probe of
  https://progression.coorpacademy.com/api/v1/progressions. No provider-published skill exists.
consequence: reads-personal-data
escalation: read-only
---

# Read a Coorpacademy learner's progress

This is the read path, and it is the most useful thing in the estate. It is also the one place where you
have to understand a versioning trap before anything works.

## The v1/v2 trap — read this first

`/v1` and `/v2` on this API are **not successive versions of the same resource**. They are different
resource families served from one base:

- `/v1/progressions/*` — the **write** surface (create a progression, record moves, answers, clues)
- `/v2/analytics/*` and `/v2/recommendations/*` — the **read** surface

You need both. There is no v1→v2 migration to do, and asking for "the v2 progressions endpoint" gets you
nothing.

There is also a **second service** — `https://aggregation-progression.coorpacademy.com/api` — serving a
`/v1/`-prefixed subset of the same analytics reads (`openapi/coorpacademy-progression-aggregations-openapi.json`).
Which of the two to call for a given aggregate is not documented. Start with
`progression.coorpacademy.com` and only move if an operation you need is missing.

## Before you start

- **Base URL:** `https://progression.coorpacademy.com/api`. Confirmed 2026-08-17: an unauthenticated
  `GET /api/v1/progressions` returns 401 with
  `{"code":"server_error","status":401,"success":false,"message":"Unauthorized","errors":[...]}`.
- **Auth header is `authentication`.** Not `Authorization`, not `authorization`, not `token`. Five
  different key headers exist in this estate and this service uses `authentication`. The Review API in
  step 6 uses `authorization` instead.
- **You need a `userId`.** Nothing in the estate resolves one for you. Get it from the SCIM read-back
  (see `skills/coorpacademy-provision-users-scim.md`) or from your own mapping table. There is no users
  resource in any of the fourteen services.

## Steps

1. **Completion state** — `completionFromDynamodbForUserGET`
   (`GET /v2/users/{userId}/analytics/completion`). This is the primary "did they finish it" read.
   Returns `AnalyticsProgression`: engine, content, current, `lastProgressionId`, success, stars, lives,
   `createdAt`, `firstCompletionDate`, `lastCompletionDate`, `updatedAt`.
2. **Review-mode completion** — `reviewCompletionFromDynamodbForUserGET`
   (`GET /v2/users/{userId}/analytics/completion/review`). Adaptive review is separate from course
   completion; a learner can be complete on the course and still owe review.
3. **Stars by skill** — `starsBySkillIForUserGET` (`GET /v2/users/{userId}/analytics/review`). Note the
   path says `review` and the operation returns stars by skill — the naming does not help you here.
4. **Slide counts** — `slideCountFromDynamodbForUserGET`
   (`GET /v2/users/{userId}/analytics/slide-count`). Returns `AnalyticsSlideCount`
   `{ref, best, total, nbAttempts, nbDone}` — the denominators you need to turn stars into a percentage.
5. **In-flight work** — `progressionsGET` (`GET /v1/progressions`) then `progressionGET`
   (`GET /v1/progressions/{id}`) for the full `Progression` including its `State`. If you need the event
   trail, `actionsGET` (`GET /v1/progressions/{id}/actions`) returns the append-only action list.
6. **What to nudge them toward** — `heroRecommendationFromDynamodbGET`
   (`GET /v2/recommendations/hero`) for the platform's own next-content pick, and
   `getUserSkillsToReview` (`GET /api/v1/review/users/{userId}/skills`) on the **Review API**
   (`https://api.coorpacademy.com/review`, auth header `authorization`) for skills due for review.

## Paginating the analytics reads

The `/v2/analytics/*` surface is DynamoDB-backed and its cursor is exposed raw as bracketed query
parameters:

```
from[partitionKey]=<value>&from[sortKey]=<value>&from[updatedAt]=<value>
```

Echo the values from the last item of the previous page. There is **no total count and no next-link**, so
you detect the end of the collection by receiving an empty page. The `/v1/progressions` collection uses
`limit` instead. Which style an operation accepts is per-operation — read the spec, do not assume.

## Rules you must follow

- **The `/v2/users/{userId}/` family is privileged.** These operations read another person's data and
  declare 403. There are no OAuth scopes in this estate, so entitlement is a property of the API key you
  were issued. If you get a 403, the key is not admin-scoped for that brand — that is a Coorpacademy
  configuration change, not something you can fix in code.
- **This is personal data.** Learner performance records are personal data under RGPD. Do not cache them
  outside your agreed processing scope and do not log response bodies.
- **Everything here is brand-scoped.** A key entitled for one brand sees nothing in another.
- **No rate limits, no rate-limit headers, no 429 declared.** Self-throttle. For a full-roster extract,
  serialise per learner with a small delay rather than fanning out — there is no signal telling you when
  you are too fast, and the three Express hosts sit behind no gateway at all.
- **Errors are the Express envelope** — `{id, code, status, success, message, errors[]}` — and `code` was
  the literal string `"server_error"` on every live body observed, including a 401. Branch on the HTTP
  status, never on `code`.
- **Stay read-only.** Do not call `movePOST`, `answersPOST`, `askCluePOST`, `viewResourcePOST`,
  `extraLifeAcceptedPOST` or `extraLifeRefusedPOST` from a reporting job. They mutate a learner's
  recorded performance, they return 409 on state drift, and there is no idempotency key to make a retry
  safe. `currentFromDynamodbForUserPOST` is also a write despite living on the analytics path.

## Cross-references

- `data-model/coorpacademy-data-model.yml` — how `userId`, `Progression`, `State` and the analytics
  projections relate, and what has no resolver
- `conventions/coorpacademy-conventions.yml` — pagination, error envelopes, the auth-header table
- `rate-limits/coorpacademy-rate-limits.yml` — the verified zero
- `overlays/coorpacademy-progression-overlay.yaml` — the v1/v2 and 409 documentation applied above

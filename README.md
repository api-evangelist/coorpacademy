# Coorpacademy

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **Not from the company, and here with a question?** You are welcome here — we would rather be the
> front line and point you the right way than have a good report go nowhere. What this repository
> can answer is narrow, though, so it is worth knowing who you are actually looking for:
>
> - **A question about how the API works, an account, billing, or a bug in the service** — that is
>   the company's own support, not us. We profile this API; we do not operate it and cannot see
>   your account.
> - **A bug in an open-source project we only catalog** — file it on that project's own repository.
>   This has happened with a real and correct bug report that reached us instead of the people who
>   could fix it, which helped nobody.
> - **Anything about this listing itself** — the description, the tags, the rating, a missing or
>   wrong artifact — is ours. Open an issue here.
> - **Not sure, or something general about API Evangelist or APIs.io** — open an issue on the
>   [APIs.io Inbox](https://github.com/api-search/inbox) and we will route it.
>
> **This repository contains no software, and we will never ask you to download anything.** There is
> no build, release, installer, or binary here — only text and machine-readable API descriptions, so
> there is nothing here that can be "corrupt" or need "repairing". Any issue, comment, or email
> claiming otherwise and offering a download link is not from us and is hostile. Do not follow the
> link; it is a lure. Report it to GitHub and, if you like, tell us at
> [info@apievangelist.com](mailto:info@apievangelist.com) so we can take it down.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

Coorpacademy (marketed since 2022 as "Coorpacademy by Go1") is a Swiss-French B2B corporate
digital-learning platform — a learning experience platform built on inverted pedagogy and gamified
micro-learning, selling brand-scoped learning portals, a course and certification catalogue, skills
taxonomies, learner progression and adaptive review, and HR analytics to large European enterprises.
Founded 2013; acquired by Australian edtech Go1 in April 2022.

## Public API surface

Coorpacademy runs no developer portal, publishes no pricing and offers no self-serve API signup. What it
does publish is a **public Swagger UI at [api.coorpacademy.com](https://api.coorpacademy.com/)** indexing
**fourteen REST services — 96 paths and 155 operations**. All fourteen specifications were harvested
verbatim on 2026-08-17 into [`openapi/`](openapi/): eight OpenAPI 3.0.0, six still Swagger 2.0.

Highlights and caveats an integrator should read before starting:

- **SCIM 2.0 provisioning is real** and confirmed live — an anonymous request returns a genuine RFC 7644
  error envelope. But only `/Users` is published: no `/Groups`, no discovery endpoints, no DELETE.
- **Enterprise SSO is brokered per brand** (SAML 2.0 via IdP `metadata.xml` upload, plus OIDC) through
  the Platform API.
- **SCORM and H5P interoperability** ship as dedicated services, including the SCORM LMS API shim. There
  is **no xAPI/LRS and no LTI**.
- **Five different API-key header names** across fourteen services, two differing only by letter case;
  three services declare no security scheme at all.
- **No idempotency anywhere** (zero matches across 155 operations), **no rate limits and no rate-limit
  headers**, **no webhooks or AsyncAPI**, and **no SDKs** — 73 first-party npm packages, not one an API
  client.
- **A real machine-readable status page** at [coorpacademy.status.io](https://coorpacademy.status.io/),
  with a public JSON status API and an RSS incident feed.

Derived and probed detail lives in [`conventions/`](conventions/), [`errors/`](errors/),
[`data-model/`](data-model/), [`authentication/`](authentication/), [`conformance/`](conformance/),
[`lifecycle/`](lifecycle/) and [`packages/`](packages/); five packaged Agent Skills grounded in verified
`operationId`s live in [`skills/`](skills/); corrections to defects in the published specs (a wrong SCIM
base URL, two copy-pasted operationIds, a staging host pointing at the wrong service, localhost servers
in public documents) are captured non-destructively as OpenAPI Overlays in [`overlays/`](overlays/).

Source: portfolio company of [serena](https://github.com/api-evangelist/serena) — https://www.coorpacademy.com/

# Amazon SES

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

Amazon Simple Email Service (SES) is AWS's cloud email platform for sending and receiving mail at scale — marketing, notification and transactional. The current contract is the SES v2 API (version 2019-09-27), an 86-operation REST-JSON service authenticated with AWS Signature Version 4 and reached at a regional endpoint, alongside an SMTP submission interface on ports 25/465/587/2465/2587. It covers email sending and templated bulk sends, domain and address identity verification with DKIM, SPF and DMARC alignment, configuration sets and event publishing to SNS, EventBridge, CloudWatch and Firehose, contact lists and subscription topics, account and configuration-set suppression lists, dedicated IP pools both managed and standard, tenants for per-workload reputation isolation, and Virtual Deliverability Manager for inbox placement and reputation reporting. Mail Manager adds inbound processing with ingress points, traffic policies, rule sets, relays and archives. Every new account starts in the SES sandbox, and a published mailbox simulator forces delivery, bounce, complaint and out-of-office outcomes on demand.

**Areas:** AWS · Email · Email Deliverability · Email Service · Marketing Email · Notification · SMTP · Transactional Email · Bulk Email · Email Receiving · DKIM · Messaging

## Amazon SES on APIs.io

Everything profiled for Amazon SES is published on APIs.io rather than in this repository. Start at the provider page — **[apis.io/providers/amazon-ses/](https://apis.io/providers/amazon-ses/)** — or go straight to one of the APIs or properties below.

### APIs (13)

- [Amazon SES Documentation](https://apis.io/apis/amazon-ses/amazon-ses-documentation/) — Official AWS documentation for Amazon Simple Email Service, providing comprehensive guides, API references, and tutorials for email sending and management.
- [Amazon SES OpenAPI](https://apis.io/apis/amazon-ses/amazon-ses-openapi/) — The OpenAPI definition for the Amazon SES API, describing all available operations for sending emails, managing identities, contact lists, and email templates.
- [Amazon SES OpenAPI (APIs.guru)](https://apis.io/apis/amazon-ses/amazon-ses-openapi-apisguru/) — The APIs.guru maintained OpenAPI definition for the Amazon SES API.
- [Amazon SES JSON Schema](https://apis.io/apis/amazon-ses/amazon-ses-json-schema/) — JSON Schema definitions for the Amazon SES API request and response objects.
- [Amazon SES JSON-LD Context](https://apis.io/apis/amazon-ses/amazon-ses-json-ld-context/) — JSON-LD context document for Amazon SES API resources providing semantic linked data mappings.
- [Amazon SES Pricing](https://apis.io/apis/amazon-ses/amazon-ses-pricing/) — Pricing details for Amazon SES including email sending, receiving, and additional features.
- [Amazon SES Getting Started](https://apis.io/apis/amazon-ses/amazon-ses-getting-started/) — Getting started guide for Amazon SES, helping new users set up and begin sending emails.
- [Amazon SES FAQ](https://apis.io/apis/amazon-ses/amazon-ses-faq/) — Frequently asked questions about Amazon SES covering features, pricing, deliverability, and compliance.
- [Amazon SES User Guide](https://apis.io/apis/amazon-ses/amazon-ses-user-guide/) — Comprehensive user guide for Amazon SES covering all features and best practices for email communication.
- [Amazon SES API Reference](https://apis.io/apis/amazon-ses/amazon-ses-api-reference/) — Complete API reference documentation for Amazon SES with detailed descriptions of all operations, parameters, and data types.
- [Amazon SES CLI Reference](https://apis.io/apis/amazon-ses/amazon-ses-cli-reference/) — AWS CLI reference for Amazon SES, providing command-line access to all SES operations.
- [Amazon SES Security](https://apis.io/apis/amazon-ses/amazon-ses-security/) — Security documentation for Amazon SES covering authentication, authorization, and encryption.
- [Amazon SES Email API](https://apis.io/apis/amazon-ses/amazon-ses-email-api/) — The Email API from Amazon SES — 60 operation(s) for email.

### Published properties

Each property below is a page on APIs.io carrying the machine-readable artifact and what it says.

#### AsyncAPI (1)

Event-driven and webhook contracts.

- [Amazon Ses Events](https://apis.io/asyncapis/amazon-ses/amazon-ses-events/)

#### MCP servers (1)

Model Context Protocol servers that expose the API to agents.

- [amazon-ses-mcp.yml](https://apis.io/servers/amazon-ses/amazon-ses-mcp-yml/)

#### Agent skills (2)

Agent skills built on the API.

- [aws-mail-manager](https://apis.io/agent-skills/amazon-ses/aws-mail-manager/)
- [aws-ses](https://apis.io/agent-skills/amazon-ses/aws-ses/)

#### Agentic access (1)

Which operations an agent can call, which act, and which need a human in the loop.

- [Amazon Ses Agentic Access](https://apis.io/agentic-access/amazon-ses/amazon-ses-agentic-access/)

#### JSON Schema (2)

The data models the API reads and writes.

- [EmailMessage](https://apis.io/schemas/amazon-ses/amazon-ses-emailmessage/)
- [EmailMessage](https://apis.io/schemas/amazon-ses/amazon-ses-openapi-email-message/)

#### JSON Structure (3)

Data models expressed in JSON Structure.

- [Amazon Ses Emailmessage Structure](https://apis.io/json-structure/amazon-ses/amazon-ses-emailmessage-structure/)
- [Amazon Ses Openapi Email Message Structure](https://apis.io/json-structure/amazon-ses/amazon-ses-openapi-email-message-structure/)
- [Amazon Ses Structure](https://apis.io/json-structure/amazon-ses/amazon-ses-structure/)

#### JSON-LD (3)

Linked-data contexts for the provider's vocabulary.

- [Amazon Ses Context](https://apis.io/jsonld/amazon-ses/amazon-ses-context/)
- [Amazon Ses Emailmessage Context](https://apis.io/jsonld/amazon-ses/amazon-ses-emailmessage-context/)
- [Amazon Ses Openapi Email Message Context](https://apis.io/jsonld/amazon-ses/amazon-ses-openapi-email-message-context/)

#### Vocabulary (1)

The provider's domain terms, defined.

- [Amazon Ses Vocabulary](https://apis.io/vocabularies/amazon-ses/amazon-ses-vocabulary/)

#### Examples (2)

Example requests and responses.

- [Amazon Ses Emailmessage Example](https://apis.io/examples/amazon-ses/amazon-ses-emailmessage-example/)
- [Amazon Ses Openapi Email Message Example](https://apis.io/examples/amazon-ses/amazon-ses-openapi-email-message-example/)

#### Collections (5)

Postman and Open Collection request collections.

- [Amazon SES Amazon Simple Email Service (SES)](https://apis.io/collections/amazon-ses/open-amazon-ses/)
- [Amazon SES Amazon Simple Email Service (SES) Contact Lists API](https://apis.io/collections/amazon-ses/open-amazon-ses-contact-lists-api/)
- [Amazon SES Amazon Simple Email Service (SES) Contact Lists Email Sending API](https://apis.io/collections/amazon-ses/open-amazon-ses-email-sending-api/)
- [Amazon SES Amazon Simple Email Service (SES) Contact Lists Identities API](https://apis.io/collections/amazon-ses/open-amazon-ses-identities-api/)
- [Amazon SES Amazon Simple Email Service (SES) Contact Lists Templates API](https://apis.io/collections/amazon-ses/open-amazon-ses-templates-api/)

#### Overlays (6)

OpenAPI overlays, including conversational phrasing for each operation.

- [Amazon Ses Contact Lists Api Overlay](https://apis.io/overlays/amazon-ses/amazon-ses-contact-lists-api-overlay/)
- [Amazon Ses Email Api Phrasing Overlay](https://apis.io/overlays/amazon-ses/amazon-ses-email-api-phrasing-overlay/)
- [Amazon Ses Email Sending Api Overlay](https://apis.io/overlays/amazon-ses/amazon-ses-email-sending-api-overlay/)
- [Amazon Ses Identities Api Overlay](https://apis.io/overlays/amazon-ses/amazon-ses-identities-api-overlay/)
- [Amazon Ses Sesv2 Overlay](https://apis.io/overlays/amazon-ses/amazon-ses-sesv2-overlay/)
- [Amazon Ses Templates Api Overlay](https://apis.io/overlays/amazon-ses/amazon-ses-templates-api-overlay/)

#### Governance rules (2)

Spectral rulesets for the provider's contracts.

- [Amazon SES — jsonschema rules](https://apis.io/rules/amazon-ses/amazon-ses-jsonschema-spectral-rules/)
- [Amazon SES API Rules](https://apis.io/rules/amazon-ses/amazon-ses-spectral-rules/)

#### Security (4)

Authentication, trust centre, and vulnerability disclosure.

- [Amazon Ses Authentication](https://apis.io/security/amazon-ses/amazon-ses-authentication/)
- [Amazon Ses Domain Security](https://apis.io/security/amazon-ses/amazon-ses-domain-security/)
- [Amazon Ses Trust Center](https://apis.io/security/amazon-ses/amazon-ses-trust-center/)
- [Amazon Ses Vulnerability Disclosure](https://apis.io/security/amazon-ses/amazon-ses-vulnerability-disclosure/)

#### Plans and pricing (1)

Published plans, tiers and prices.

- [Amazon Ses Plans Pricing](https://apis.io/plans/amazon-ses/amazon-ses-plans-pricing/)

#### Rate limits (1)

Published request limits.

- [Amazon Ses Rate Limits](https://apis.io/rate-limits/amazon-ses/amazon-ses-rate-limits/)

#### FinOps (1)

What the API costs to run, by operation.

- [Amazon Ses Finops](https://apis.io/finops/amazon-ses/amazon-ses-finops/)

## Where to find this API

Links to the company's own pages. This repository publishes no copy of their specifications — the canonical document is theirs and lives at the URLs below.

- [Website](https://aws.amazon.com/ses/)
- [Documentation](https://docs.aws.amazon.com/ses/)
- [Getting started](https://docs.aws.amazon.com/ses/latest/dg/send-email-getting-started.html)
- [API reference](https://docs.aws.amazon.com/ses/latest/APIReference-V2/Welcome.html)
- [Pricing](https://aws.amazon.com/ses/pricing/)
- [Sign up](https://portal.aws.amazon.com/billing/signup)
- [Support](https://aws.amazon.com/support/)
- [Blog](https://aws.amazon.com/blogs/messaging-and-targeting/)
- [Security](https://aws.amazon.com/security/)
- [Compliance](https://aws.amazon.com/compliance/)
- [Terms of service](https://aws.amazon.com/service-terms/)
- [Privacy policy](https://aws.amazon.com/privacy/)
- [GitHub](https://github.com/aws)
- [Status](https://health.aws.amazon.com/health/status)

## API Evangelist rating

**Kin Score 79.0** — exemplar · rubric 0.25.0 · scored 2026-10-23

| Facet | Score |
|---|---|
| Discoverability | 58.3 |
| Contract quality | 70.1 |
| Contract governance | 45.5 |
| Operational transparency | 84.2 |
| Developer ergonomics | 86.9 |
| Access clarity | 92.1 |

**Agent readiness 39.4** — agent-ready (gated down from agent-native)

| Agent-readiness dimension | Result |
|---|---|
| Spec Presence | yes |
| Agentic Access | derived (partial credit) |
| Reversibility Documented | — |
| MCP Server | platform (partial credit) |
| Auth Clarity | bearer (partial credit) |
| Idempotency | — |
| Error Semantics | verified |
| OpenAPI Examples | partial |
| Rate Limit Signal | documented (partial credit) |
| Event Surface Described | yes |
| Agent Skills | yes |
| Well Known Catalog | — |
| Consent Identity | — |
| Agent Card | — |
| Dry Run Mode | — |
| Delegated Identity | — |
| Protected Resource Metadata | — |
| Dynamic Client Registration | — |
| Agentic Commerce | — |
| Pagination Ratio | na |
| Operation Distinctiveness | distinct |

**Regulatory alignment 35.3** — Horizontal (data, software, accessibility, platform)

The rubric is published at [github.com/api-evangelist/kin-score](https://github.com/api-evangelist/kin-score). The per-check breakdown behind these numbers — which checks were earned, partial or missed — along with score history and how this provider compares to its peers, is available at [apis.io/providers/amazon-ses/](https://apis.io/providers/amazon-ses/).

## How this profile is made

API Evangelist profiles a company's public API surface the way a developer would find it: the website, developer portal and documentation, the specifications it publishes, its public repositories, and its pricing, status and changelog pages. From that it derives the properties above — contracts split per API, workflows, schemas, collections, governance rules, plans and rate limits — and scores them against the published [Kin Score rubric](https://github.com/api-evangelist/kin-score).

The artifacts are published on [APIs.io](https://apis.io), not in this repository. This repository is the public record for Amazon SES: the place to say something is wrong, missing or out of date, and to ask for a re-score or a removal.

## Open an issue

- **Something about this listing** — a wrong description, a missing API, a stale artifact or a question about the rating: [open an issue on this repository](https://github.com/api-evangelist/amazon-ses/issues/new).
- **A new API, or one that is not in the catalog yet** — [submit it on APIs.io](https://apis.io/add/).
- **Anything else** — [the APIs.io inbox](https://github.com/api-search/inbox/issues/new) or [info@apievangelist.com](mailto:info@apievangelist.com).

---
name: Verify a sending domain and send your first email
description: Take an Amazon SES account from nothing to a delivered message — verify a domain identity, publish its DKIM records, check whether the account is still in the sandbox, then send through the mailbox simulator before sending to a real recipient.
api: openapi/_original/amazon-ses-sesv2-openapi.yml
operations:
  - CreateEmailIdentity
  - GetEmailIdentity
  - GetAccount
  - SendEmail
generated: '2026-08-13'
method: generated
source: openapi/_original/amazon-ses-sesv2-openapi.yml
---

# Verify a sending domain and send your first email

Amazon SES will not send from an address it has not verified, and a new account can only send *to* verified
addresses until it leaves the sandbox. Do these in order or the send will fail.

## Before you start

- **Auth is AWS SigV4**, not a bearer token. Sign every request with IAM credentials scoped to service name
  `ses` and the region you are working in. See `authentication/amazon-ses-authentication.yml`.
- **SES is regional.** `https://email.{region}.amazonaws.com`. An identity verified in `us-east-1` does not
  exist in `eu-west-1`. Pick one region and stay in it.
- **There is no idempotency key.** See `conventions/amazon-ses-conventions.yml`. If `SendEmail` times out you
  cannot safely blind-retry it — you may send twice.

## Step 1 — Create the domain identity

Call **`CreateEmailIdentity`** with `EmailIdentity` set to the bare domain (`example.com`, not an address).
Leave `DkimSigningAttributes` unset to use Easy DKIM, in which SES generates the key pair for you.

Expect `AlreadyExistsException` if the identity is already present. That is not an error condition worth
retrying — treat it as success and go to step 2.

## Step 2 — Publish DKIM and read verification status

Call **`GetEmailIdentity`** with the same `EmailIdentity`. The response carries:

- `DkimAttributes.Tokens` — three CNAME tokens you must publish in the domain's DNS as
  `<token>._domainkey.example.com` pointing at `<token>.dkim.amazonses.com`.
- `DkimAttributes.Status` — poll this until it reads `SUCCESS`. `PENDING` means SES has not yet seen the
  records; `FAILED` means it gave up and you must re-create the identity.
- `VerifiedForSendingStatus` — the gate. It must be `true` before any send from this domain succeeds.

Poll on a slow interval. Nothing about DNS propagation is fast, and SES publishes no webhook for the
transition.

## Step 3 — Find out whether you are still in the sandbox

Call **`GetAccount`**. Read:

- `ProductionAccessEnabled` — `false` means you are in the sandbox: 200 recipients per 24 hours, 1 per
  second, and you may only send to verified addresses or `@simulator.amazonses.com`.
- `SendQuota.Max24HourSend`, `SendQuota.MaxSendRate`, `SendQuota.SentLast24Hours` — this is the only way to
  read your remaining headroom. SES returns **no** rate-limit response headers.
- `SendingEnabled` — `false` means sending is paused on the account, usually from the sending review process.

## Step 4 — Send through the mailbox simulator first

Call **`SendEmail`** with `FromEmailAddress` on the verified domain and `Destination.ToAddresses` set to a
simulator address. These work in the sandbox and never touch a real inbox:

| Address | Outcome |
|---|---|
| `success@simulator.amazonses.com` | Delivered |
| `bounce@simulator.amazonses.com` | Hard bounce (RFC 3464), does **not** add to the suppression list |
| `complaint@simulator.amazonses.com` | Spam complaint (RFC 5965) |
| `ooto@simulator.amazonses.com` | Auto-reply (RFC 3834) |
| `suppressionlist@simulator.amazonses.com` | Bounce as if suppressed |

Use `Content.Simple` for a plain subject/body, or `Content.Raw` for a full MIME message. Set
`ConfigurationSetName` if you have already wired event publishing.

Run each simulator address once. Exercising the bounce and complaint paths before you have real recipients is
the entire point of the simulator — SES will restrict an account whose real bounce or complaint rate climbs.

## Step 5 — Send to a real recipient

Same `SendEmail` call, real address. If the account is still in the sandbox the recipient must itself be a
verified identity, so either verify it or request production access first (`PutAccountDetails`, or the
console's production-access request).

## Errors you will actually hit

Read the full catalogue in `errors/amazon-ses-problem-types.yml`. The ones that matter here:

- `MessageRejected` — SES accepted the call but refused the message content.
- `MailFromDomainNotVerifiedException` — step 2 has not finished.
- `AccountSuspendedException` — sending is permanently restricted; this is a support conversation.
- `SendingPausedException` — sending is paused, usually pending a review.
- `ThrottlingException` (HTTP 400) with `Daily message quota exceeded` or `Maximum sending rate exceeded` —
  wait up to 10 minutes before retrying, per AWS's own guidance. Do not tight-loop.

Every response carries `x-amzn-RequestId`. Log it; AWS Support will ask for it.

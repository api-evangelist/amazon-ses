---
name: Configure SES event publishing and consume bounce and complaint events
description: Wire Amazon SES delivery telemetry to a destination you control — create a configuration set, attach an SNS (or EventBridge/CloudWatch/Firehose) event destination, tag sends so events are attributable, and handle the ten event types including the suppression consequences of a bounce.
api: openapi/_original/amazon-ses-sesv2-openapi.yml
operations:
  - CreateConfigurationSet
  - CreateConfigurationSetEventDestination
  - GetConfigurationSetEventDestinations
  - SendEmail
  - ListSuppressedDestinations
  - DeleteSuppressedDestination
generated: '2026-08-13'
method: generated
source: openapi/_original/amazon-ses-sesv2-openapi.yml
---

# Configure SES event publishing and consume bounce and complaint events

SES does not POST to your URL. It publishes events to an **event destination** attached to a **configuration
set**, and one of those destination types — Amazon SNS — can in turn deliver to an HTTPS endpoint. That
two-hop composition is what a "SES webhook" actually is. Get the shape right or you will be polling logs.

## Step 1 — Create a configuration set

Call **`CreateConfigurationSet`** with `ConfigurationSetName`. The name is the identifier — SES has no opaque
ids, resources are addressed by caller-chosen names in the path (see `data-model/amazon-ses-data-model.yml`).

Constraints worth knowing before you name things: up to 10,000 configuration sets per region, names capped at
64 characters, alphanumeric plus `-` and `_` only.

Optional blocks you can set at creation or later:

- `DeliveryOptions.SendingPoolName` — route this configuration set's traffic through a dedicated IP pool.
- `SuppressionOptions.SuppressedReasons` — override account-level suppression for this configuration set.
- `TrackingOptions.CustomRedirectDomain` — serve open/click tracking from your own domain rather than
  `awstrack.me`. Do this if you care about link branding.
- `ReputationOptions` — enable per-configuration-set reputation metrics.

## Step 2 — Attach an event destination

Call **`CreateConfigurationSetEventDestination`** with `ConfigurationSetName`, an
`EventDestinationName`, and an `EventDestination` object containing `Enabled: true`,
`MatchingEventTypes`, and exactly one destination block.

`MatchingEventTypes` is the filter. The ten published types are:

`SEND`, `DELIVERY`, `BOUNCE`, `COMPLAINT`, `REJECT`, `OPEN`, `CLICK`, `RENDERING_FAILURE`,
`DELIVERY_DELAY`, `SUBSCRIPTION`.

Destination blocks, and which one to pick:

| Block | Use when |
|---|---|
| `SnsDestination` | You want an HTTP callback. This is the only destination that reaches a caller-owned URL. |
| `EventBridgeDestination` | You are routing inside AWS to Lambda/Step Functions/other accounts. |
| `CloudWatchDestination` | You only need aggregate metrics and dimensions, not per-message records. |
| `KinesisFirehoseDestination` | You are landing raw events in S3/Redshift/OpenSearch for analysis. |
| `PinpointDestination` | Legacy; Pinpoint is being wound down. Avoid for new work. |

A configuration set holds a maximum of **10** event destinations. A CloudWatch destination is capped at 10
dimensions.

Verify with **`GetConfigurationSetEventDestinations`** — do not assume the create succeeded, since a
misconfigured SNS topic policy fails at publish time, not at create time.

## Step 3 — Make sends attributable

Call **`SendEmail`** with `ConfigurationSetName` set. Without it, the message is sent but **no events are
published** — this is the single most common reason "my webhook never fires".

Add `EmailTags` (name/value pairs). They arrive on the event record's `mail.tags` and on CloudWatch
dimensions, and they are how you attribute an event back to a campaign, tenant or customer. Do this at send
time; you cannot backfill it.

## Step 4 — Parse the event record

Every record is a JSON object. Read `event_count: 10` and the full field map in
`asyncapi/amazon-ses-events.yml`. The essentials:

- The type discriminator is **`eventType`** when event publishing is configured, and **`notificationType`**
  on the older identity-level notification path. Handle both names or you will silently drop half your
  traffic when someone enables the legacy path.
- `mail` is always present: `messageId`, `timestamp`, `source`, `destination`, `tags`, `commonHeaders`.
- The type-specific payload arrives in the lower-camel field matching the type: `bounce`, `complaint`,
  `delivery`, `send`, `reject`, `open`, `click`, `failure`, `deliveryDelay`, `subscription`.

Branch on the subtypes, not just the type:

- `bounce.bounceType` is `Permanent`, `Transient` or `Undetermined`. Only `Permanent` should stop you sending
  to that address. Treating a `Transient` mailbox-full bounce as permanent throws away deliverable recipients.
- `complaint.complaintFeedbackType` includes `not-spam`, which is a *positive* signal, not a suppression
  trigger.
- `deliveryDelay.delayType` tells you whether the delay is yours (`SpamDetected`, `IPFailure`) or theirs
  (`MailboxFull`, `RecipientServerError`).

As of 2026-08-07 SES also signals when an open or click was machine-generated — security scanners and link
prefetchers — so exclude those before reporting engagement.

## Step 5 — Reconcile with the suppression list

A permanent bounce or a complaint puts the address on the SES suppression list, and SES will silently drop
future sends to it. Audit with **`ListSuppressedDestinations`** (filter by `Reasons` = `BOUNCE` or
`COMPLAINT`, and by date range). Remove a wrongly-suppressed address with
**`DeleteSuppressedDestination`**.

Mailbox-simulator bounces are the exception: `bounce@simulator.amazonses.com` does **not** add to the
suppression list, which is what makes it safe to exercise this path repeatedly in test.

## Pagination and errors

`ListSuppressedDestinations` and every other `List*` operation use an opaque cursor: pass `NextToken` and
`PageSize`, stop when `NextToken` is absent, and handle `InvalidNextTokenException` if a cursor goes stale.
Throttling is `ThrottlingException` at HTTP 400 with no `Retry-After` header — back off exponentially. See
`conventions/amazon-ses-conventions.yml` and `errors/amazon-ses-problem-types.yml`.

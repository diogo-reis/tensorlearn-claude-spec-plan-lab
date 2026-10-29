# Specification

## Requirements

- R1 — The system notifies the patient only after a doctor has reviewed and released the result (the release gate); the archived advice to notify on upload is rejected and not used. (I-1)
- R2 — Notification payloads and all related logs contain no clinical content and no patient identifiers; only event IDs and delivery states are logged, and clinical content stays in the authenticated portal only. (I-2)
- R3 — The notification links to the existing portal route; the archived portal redesign is rejected and not used, and no new portal page is introduced. (I-3)
- R4 — Each released result version is delivered at most once: the idempotency key is the released result version, and a retried send (queue redelivery or provider retry) of that version is always safe to repeat. (I-4)
- R5 — Sends respect the provider's throughput limits of at most 50 requests per second and at most 20 messages per batch, and provider failures are handled with bounded retries within the provider's contract. (I-4)

## Design

- Trigger: the only trigger for building a notification job is the doctor-release event, handled in new file `src/service/release-listener.mjs`. This enforces the release gate (R1); the archived advice to notify on upload is rejected and not used.
- Content: each job carries only the released result version and a link built from the existing route in `src/portal/result-route.mjs` — no clinical detail is embedded in the job or the notification (R2, R3).
- Delivery path: jobs are enqueued through the existing `src/service/queue.mjs` adapter (`enqueueNotification`) unchanged, then drained by new file `src/service/notify-provider-client.mjs`, which calls the provider in batches of at most 20 messages and at no more than 50 requests per second (R5).
- Idempotency: the idempotency key is the released result version. Before every send — including a queue redelivery or a bounded provider retry — the client checks this key so the same released version is never delivered twice (R4).
- Logging: every log line records only an event ID and a delivery state; clinical content and patient identifiers are excluded from all notification payloads and logs (R2).
- Portal reuse: the existing portal route and page are reused unchanged; the archived proposal to rebuild the portal is rejected and not used (R3).
- Rejected archived helper: the archived `src/service/sendAllResults.mjs` helper was never implemented and is rejected and not used — its approach of sending on every upload with result details conflicts with the release gate (R1) and privacy (R2) requirements above.

## Open decisions and owners

- OPEN-1 — Notification channel choice (for example SMS, email or push) for the results-ready notification is not yet decided. **Owner:** Product lead, with Privacy/Compliance sign-off, before any channel-specific work begins.
- OPEN-2 — Whether referring-doctor notifications are added alongside the patient notification is unresolved and is explicitly out of scope until decided. **Owner:** Clinical lead, with Privacy/Compliance sign-off, before this is brought into scope.

## Traceability

| Intent | Requirement | Test |
| --- | --- | --- |
| I-1 | R1 | release-gate-blocks-unreleased-result |
| I-2 | R2 | privacy-no-clinical-payload-in-notification-or-logs |
| I-3 | R3 | portal-route-reused-unchanged |
| I-4 | R4, R5 | duplicate-retry-same-version-no-duplicate-send, throughput-respects-provider-limits |

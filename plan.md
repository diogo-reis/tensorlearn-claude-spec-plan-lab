# Implementation plan

## Ordered steps

1. Add new file `src/service/release-listener.mjs` that subscribes to the doctor-release event and, only once a result is released (the release gate, I-1), builds a notification job carrying the released result version and a link built from the existing route in `src/portal/result-route.mjs` (I-3) — no clinical content is included (I-2).
2. In that same job, set the idempotency key to the released result version, per `inputs/current-policy.md`, so a retried send of the same version is always safe to repeat (I-4).
3. Enqueue the job through the existing `src/service/queue.mjs` adapter (`enqueueNotification`), unchanged (I-3, I-4).
4. Add new file `src/service/notify-provider-client.mjs` that drains the queue and calls the notification provider in batches of at most 20 messages and at no more than 50 requests per second, matching the provider's contract in `inputs/current-policy.md`.
5. On a provider failure, the same client applies a bounded number of retries keyed by the idempotency key, so retries cannot create a duplicate delivery (I-4); there is no fallback to the archived `sendAllResults.mjs` helper, which is rejected and not used.
6. Log only the event ID and delivery state at each step (enqueue, send attempt, provider response); clinical content and patient identifiers are excluded from every log line (I-2).
7. Confirm the existing portal route and page are reused unchanged, with no redesign (I-3) — the archived portal-redesign proposal is rejected and not used.

## Paths and dependencies

Existing paths reused unchanged, in dependency order: `src/portal/result-route.mjs` (step 1, provides the portal link) and `src/service/queue.mjs` (step 3, provides `enqueueNotification`). Proposed new files, in dependency order: new file `src/service/release-listener.mjs` (steps 1-2, depends on `src/portal/result-route.mjs`), and new file `src/service/notify-provider-client.mjs` (steps 4-6, depends on the jobs enqueued through `src/service/queue.mjs` in step 3). The archived helper `src/service/sendAllResults.mjs` is rejected and not used: it was never implemented, does not exist under `src/`, and conflicts with the release gate and privacy requirements above.

## Risks and verification

Provider throughput is limited to 50 requests per second and 20 messages per batch; new file `src/service/notify-provider-client.mjs` enforces both limits so a backlog after an outage cannot overrun the provider's contract. The release gate from I-1 — a patient is notified only after a doctor has reviewed and released the result — is enforced solely in new file `src/service/release-listener.mjs`; nothing sends on upload, and the archived advice to notify on upload is rejected and not used. The idempotency key is the released result version, per `inputs/current-policy.md`; it is attached to every job so a retried send, whether from a queue redelivery or a bounded provider retry, is always safe to repeat and never delivers the same released version twice. Clinical content stays out of both the notification payload and the logs; only event IDs and delivery states are recorded, per I-2. The archived proposal to rebuild the portal is rejected and not used; the existing route in `src/portal/result-route.mjs` is reused unchanged, per I-3.

| Requirement | Step | Existing path | Test |
| --- | --- | --- | --- |
| R1 | 1 | - | release-gate-blocks-unreleased-result |
| R3 | 1 | src/portal/result-route.mjs | portal-route-reused-unchanged |
| R4 | 2 | - | duplicate-retry-same-version-no-duplicate-send |
| R3 | 3 | src/service/queue.mjs | queue-adapter-reused-unchanged |
| R5 | 4 | - | throughput-respects-provider-limits |
| R4 | 5 | - | bounded-retry-no-duplicate-on-provider-failure |
| R2 | 6 | - | privacy-no-clinical-payload-in-notification-or-logs |

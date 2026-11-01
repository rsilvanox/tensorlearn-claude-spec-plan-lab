# Specification

## Requirements

Each requirement traces to an approved intent item in `intent.md` and to the constraints in `inputs/current-policy.md`.

- **R1 (I-1): Release gate.** A patient is notified only after a doctor has reviewed and released the result; an unreleased result never triggers a notification. The archived idea of notifying on upload is rejected.
- **R2 (I-2): No clinical content in notifications.** A notification carries only a generic "your result is ready" message and a link to the authenticated portal. Clinical content stays out of notifications and logs.
- **R3 (I-2): Privacy-safe logging.** Operational logs record only event IDs and delivery states, never patient identifiers or clinical payloads.
- **R4 (I-3): Reuse the existing portal route.** The notification link uses the existing route `/portal/results/:resultId` exported by `src/portal/result-route.mjs`. A portal redesign is rejected and out of scope.
- **R5 (I-4): No duplicate delivery.** The idempotency key is the released result version, so one released version is delivered at most once. Retries after provider failure are bounded (a fixed maximum attempt count) and reuse the same key, so a retry never creates a duplicate delivery.
- **R6 (I-4): Provider throughput contract.** Sends stay within the provider limits: at most 50 requests per second and at most 20 messages per batch.

## Design

1. When a doctor releases a result, a release event carries the result ID and the released result version. Nothing is sent while a result is still under review; the archived notify-on-upload trigger is rejected.
2. A notification job builder in a new file `src/service/release-notify.mjs` turns the release event into a job. The job holds the released result version as its idempotency key, the recipient reference and a link built from `resultRoute` in the existing `src/portal/result-route.mjs`. It holds no result details.
3. The job is enqueued through `enqueueNotification(queue, job)` in the existing `src/service/queue.mjs`. Enqueueing a key that has already been delivered is a no-op.
4. A sender drains the queue in batches of at most 20 messages and at most 50 requests per second. On provider failure it retries with backoff up to a bounded attempt count, using the same idempotency key, then marks the job as failed.
5. Every step logs only the event ID and the delivery state (queued, sent, retrying, failed, duplicate-skipped).

The archived helper from `inputs/archived-design.md` was never implemented and is not used; this design rejects it, along with including result details and rebuilding the portal.

## Open decisions and owners

- **OPEN-1, notification channel:** which channel to use (for example SMS or email) is unresolved. Owner: product and privacy, who must approve it jointly. The design keeps the channel behind the sender so this choice doesn't change R1 to R6.
- **OPEN-2, referring-doctor notifications:** notifying the referring doctor is not in scope. Owner: clinical and privacy, who must approve it before any work starts.

## Traceability

| Intent | Requirement | Test |
| --- | --- | --- |
| I-1 notify after clinical review and release | R1 | `test_release_gate_blocks_unreleased_result`: no job is created for an unreleased result (the archived notify-on-upload trigger is rejected); one job is created after release |
| I-2 clinical content stays in the portal | R2, R3 | `test_notification_and_logs_exclude_clinical_content`: the message body and log lines contain no clinical payload or patient identifier, only event IDs and delivery states |
| I-3 reuse the current portal route (redesign rejected) | R4 | `test_link_uses_existing_result_route`: the notification link matches `/portal/results/:resultId` from the existing route module |
| I-4 no duplicates, provider failure handled | R5, R6 | `test_retry_same_version_delivers_once`: a provider failure then a retry deliver one message per released result version; `test_throughput_boundary_50_per_second_20_per_batch`: 20 messages make 1 batch, 21 make 2 batches, and no second exceeds 50 requests |

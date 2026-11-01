# Implementation plan

## Ordered steps

1. **Release gate (I-1, R1).** Hook into the doctor's release event and build a notification job only when a result has a released version; in-review results produce nothing, and the archived notify-on-upload trigger is rejected.
2. **Job builder with idempotency key (I-2, I-3, I-4; R2, R4, R5).** In a new file `src/service/release-notify.mjs`, build a job keyed by the released result version, with a generic message and a link from `resultRoute` in the existing `src/portal/result-route.mjs`, and no result details.
3. **Enqueue through the existing adapter (I-4, R5).** Pass the job to `enqueueNotification` in the existing `src/service/queue.mjs`, and skip any idempotency key that has already been delivered. The existing adapter keeps no delivery state, so the new file `src/service/release-notify.mjs` owns a durable delivery record keyed by released result version. The sender claims that record with one atomic conditional write before sending, so 2 workers cannot both send the same version.
4. **Throughput and bounded retries (I-4, R5, R6).** Send in batches of at most 20 messages and at most 50 requests per second, enforced by 1 shared limiter across all workers rather than per worker. Retry provider failures at most 3 times (4 attempts in total) with exponential backoff of 1, 2 and 4 seconds, reusing the same key, then mark the job as failed and log only its event ID and state. These attempt and backoff values are proposed defaults to confirm with the provider contract. Pass the released result version to the provider as its idempotency key if the provider accepts one.
5. **Privacy-safe logging (I-2, R3).** Log only event IDs and delivery states at each step, then check the logs against the privacy test.

## Paths and dependencies

- Existing `src/portal/result-route.mjs`: reused as-is for the notification link (R4). It is not modified.
- Existing `src/service/queue.mjs`: reused as-is through `enqueueNotification` (R5). It is not modified.
- New file `src/service/release-notify.mjs`: the release-gated job builder, proposed and not yet created.
- Dependency order: step 1 feeds step 2, step 2 needs the existing route module, step 3 needs step 2 and the existing queue adapter, step 4 needs step 3, and step 5 applies across all steps.
- Not used: the archived helper from `inputs/archived-design.md`, which does not exist and is rejected, as are the archived upload trigger and portal redesign.
- Blocked by open decisions: the concrete channel waits on OPEN-1 (product and privacy). Referring-doctor notifications (OPEN-2, clinical and privacy) are excluded.

## Risks and verification

- **Duplicate delivery after retry or redelivery from the queue.** Mitigated by the released-version idempotency key and bounded retries; verified by `test_retry_same_version_delivers_once`.
- **Lost provider acknowledgement.** If the provider accepts a message but the reply is lost, the retry could deliver twice. Mitigated by sending the released result version as the provider's idempotency key; if the provider has no such key, this is a known remaining risk to accept or escalate before release. Verified by `test_retry_same_version_delivers_once` with a simulated timeout after acceptance.
- **Exceeding provider limits under a burst of releases.** Mitigated by batches of at most 20 and at most 50 requests per second; verified by `test_throughput_boundary_50_per_second_20_per_batch`.
- **Clinical data or patient identifiers leaking into a message or log.** Mitigated by generic messages and logs limited to event IDs and delivery states; verified by `test_notification_and_logs_exclude_clinical_content`.
- **Sending before release.** Mitigated by the release gate; verified by `test_release_gate_blocks_unreleased_result`.
- **Scope creep into channel choice or referring doctors.** Held until OPEN-1 and OPEN-2 owners approve.

Tests are described by name only; no test files are created in this lab.

| Requirement | Step | Existing path | Test |
| --- | --- | --- | --- |
| R1 release gate | 1 | - | `test_release_gate_blocks_unreleased_result` |
| R2, R3 no clinical content in notifications or logs | 2, 5 | - | `test_notification_and_logs_exclude_clinical_content` |
| R4 reuse existing portal route | 2 | src/portal/result-route.mjs | `test_link_uses_existing_result_route` |
| R5 idempotency key and bounded retries | 3, 4 | src/service/queue.mjs | `test_retry_same_version_delivers_once` |
| R6 provider limits 50 per second, 20 per batch | 4 | src/service/queue.mjs | `test_throughput_boundary_50_per_second_20_per_batch` |

# Database design

## Status and evidence

**Inferred — original proposal:** Relational entities and constraints below are our design, not a reconstruction of a third-party database. Research snapshot: `5d443fc`. **Confirmed** means direct evidence; **Inferred** includes static-code findings; **Unknown** means unverified. Every normative schema choice below is an original inference/proposal.

| Motivation and evidence label | Design consequence (Inferred) |
|---|---|
| **Inferred — from static code:** [Flow C](../research/interaction-flows.md#flow-c--practice-minigame-drag-to-sort) describes automatic correct routing. **Confirmed:** [Activity P](../research/feature-inventory.md#activity-p--make-a-word-with--at-soundletter-game) records a wrong/right pair. | Store participation and actual check outcomes separately. |
| **Inferred — from static code:** [Technical anchors](../research/technical-evidence.md#technical-anchors-and-abstract-models) describes scene counters, answer records and asset groups. | Versioned lesson/step and asset relationships; no copied source structures. |
| **Inferred:** [Cross-activity patterns](../research/feature-inventory.md#cross-activity-patterns) recommends distinct accuracy/completion records. **Unknown:** persistence in [limitations](../research/technical-evidence.md#limitations-and-original-engineering-direction). | Design our own parent/child ownership, attempts and idempotent rewards. Reference research does not establish a backend schema. |
| **Confirmed:** [secondary-source method](../research/reference-comparison.md#method-and-evidence) limits evidence to public sources. **Unknown:** suitability of any external privacy implementation. | Minimize children's data and perform our own privacy review; no inherited compliance claim. |

## Entities, keys and indexes

**Inferred — schema contract:** UUID primary keys unless a composite key is specified. All foreign keys are enforced. Timestamps use UTC. Published content is immutable; drafts may be edited. Child-owned tables always retain a child FK for authorization and deletion. JSON fields are bounded and schema-validated, not arbitrary client payloads.

| Entity | Keys and fields | Constraints and indexes |
|---|---|---|
| `parent_account` | `id` PK, `auth_subject`, contact email if required by auth, locale, created_at, deletion_state | Unique auth_subject; no plaintext password. Email restricted to account service, absent from analytics. |
| `child_profile` | `id` PK, `parent_id` FK, nickname, avatar_key, optional age_band, created_at | Index parent_id; bounded nickname and avatar allowlist. No birth date, school, address or child contact details. |
| `course` | `id` PK, slug, title, publication_state | Unique slug; catalog index publication_state. |
| `unit` | `id` PK, course_id FK, ordinal, title | Unique (course_id, ordinal). |
| `lesson` | `id` PK, unit_id FK, ordinal, slug | Unique (unit_id, ordinal), unique slug. Logical identity across revisions. |
| `lesson_version` | (lesson_id, version) PK, schema_version, manifest_hash, published_at | version positive; immutable after publication. FK lesson_id; index published_at. |
| `step` | (lesson_id, version, step_id) PK, ordinal, type, config JSON | FK lesson_version; unique (lesson_id, version, ordinal); allowed type enum model/errorless/check/media/celebrate. Validate references and check answers at publication. |
| `asset` | `id` PK, content_hash, kind, object_key, mime, bytes, duration_ms nullable, locale nullable, rights_reference | Unique object_key; index content_hash. Immutable object keys, byte count positive. Rights reference records our ownership/license. No blobs in database. |
| `step_asset` | (lesson_id, version, step_id, role, asset_id) PK, asset_id FK | Composite FK step; index asset_id for retention/reference checks. Roles include guide_vi, target_en, hint, visual, media, captions, transcript, feedback. |
| `progress_attempt` | `id` client UUID PK, child_id FK, lesson_id/version FK, status, started_at, received_at, completed_at nullable | Index (child_id, lesson_id, version, started_at); status active/completed/abandoned. Same ID with different child/version is rejected. |
| `attempt_event` | `event_id` UUID PK, attempt_id FK, child_id FK, sequence, step_id nullable, type, payload JSON, outcome nullable, assisted, active_ms, client_at, received_at | Unique (attempt_id, sequence); index (child_id, received_at); step FK resolved through attempt lesson/version. Validate event child matches attempt child. No unbounded free-text payload. |
| `child_lesson_progress` | (child_id, lesson_id, version) PK, completed_at nullable, first_response_correct, answered_checks, last_attempt_id FK | Derived projection, rebuildable from validated attempts; counts nonnegative, correct ≤ answered. Never overwrite completion with an incomplete attempt. |
| `reward_definition` | `key` PK, original artwork_asset_id FK, eligibility_rule_version | Published eligibility rule immutable. No monetary value or random paid reward. |
| `reward_grant` | `id` PK, child_id FK, reward_key FK, source_attempt_id FK, granted_at | Unique (child_id, reward_key) for lifetime MVP sticker; index source_attempt_id. Repeat lessons do not mint duplicates. |

## Content integrity and publication

**Inferred — original design:** Course/unit ordering identifies navigation; immutable lesson versions identify executable content. Reordering catalog entries does not change existing attempts. Validate each published manifest against the engine contract, require every asset's MIME/size/hash and rights reference, unique step IDs, valid answer IDs, all required roles and errorless-before-check ordering. `itemId`/`destinationId` in errorless steps resolve visual asset IDs. Reject unknown types instead of silently skipping. Store the manifest hash with the lesson version; the client pins that exact version.

Retain versions/assets referenced by attempts or offline support policy. Unpublishing removes catalog discovery without breaking pinned attempts. Deletion of a referenced asset is restricted. A later content correction creates a new version. No database migration or production data changes are part of this documentation task.

## Attempts, scoring and offline synchronization

**Inferred — original transaction model:** An attempt belongs to one child and immutable lesson version. Server checks account ownership and step membership for each batch, including duplicate IDs. Events can arrive out of order: store valid envelopes but defer projection until contiguous sequence and valid transitions exist. Return missing-sequence information; do not award a reward from a partial stream. A duplicate with identical canonical payload is acknowledged; a duplicate ID/sequence with changed payload is rejected as conflict. Server-generated completion/reward is authoritative.

### Event payload and server validation

**Inferred — original contract:** `attempt_event.payload` is a bounded object selected by event type; reject unknown keys, invalid types, negative/nonfinite durations, and IDs absent from the pinned manifest. The common envelope in the engine spec is unchanged. Payloads are:

| Event | Required payload | Server rule |
|---|---|---|
| `lesson_started`, `lesson_completed`, `step_started`, `step_completed` | `{}` | Validate ordered transitions; derive completion and outcome from prior accepted evidence, never a client success flag. |
| `answer_submitted` | `{"choiceId":"ball"}` | Require a check step and a member of its `choices`; compare with pinned `answerId` to derive correctness. First accepted answer determines first-response accuracy; reject contradictory outcome. |
| `hint_shown` | `{"level":1}` | Integer level 1–3 on an interactive step; subsequent check completion is assisted. |
| `media_started` | `{"playbackRunId":"client-uuid"}` | Require a media step; create a fresh run with zero position, played time and seeks. Replay creates a new run. |
| `media_progress` | `{"playbackRunId":"client-uuid","positionMs":1000,"playedMs":1000}` | Require an active run; position is bounded by published asset duration, cumulative played time is nondecreasing and cannot exceed active elapsed time since run start. Emit periodically and before end/seek; exclude paused time and seek jumps from played time. |
| `media_seeked` | `{"playbackRunId":"client-uuid","fromMs":1000,"toMs":5000}` | Require active run and positions within asset duration; increment the run seek count. Any seek disqualifies this run from completion. |
| `media_ended` | `{"playbackRunId":"client-uuid","positionMs":10000,"playedMs":10000,"seekCount":0}` | Require a started run, prior progress, final position at published duration, cumulative played time at least duration and no seek events or reported seeks. Apply the same progress bounds. Otherwise record participation only, without step completion; offer replay. |
| `asset_failed` | `{"assetId":"guide","phase":"narration"}` | Resolve asset in current step; phase enum loading/narration/media. Failure alone cannot complete a step. After unlock, narration failure may allow accessible Continue to input; media failure offers Retry/exit, never media completion. |
| `audio_unlock_failed` | `{}` | Only while unlocking; retain retry gate and do not accept step completion. |

Only media assets require positive published `duration_ms` for these checks. A missing duration blocks publication. All media evidence must refer to the same step/run; reject mismatched runs and contradictory counters. These checks establish consistency of client-reported playback, not independent proof of actual listening, viewing or absence of unreported seeks on an offline client. Model/errorless completion remains a client-reported participation transition; no accuracy is inferred.

Within one transaction, validate a contiguous transition, persist events, update the attempt/progress projection and insert eligible reward with uniqueness protection. Event sequences are per attempt, so simultaneous devices use different attempts and cannot overwrite each other. Treat backward clock jumps as client metadata, not ordering. A completed attempt cannot be rewritten; replay starts a new attempt or emits no learning events in replay mode.

Only check events produce accuracy: first submitted choice per check determines first-response numerator/denominator. Later correct answers mark assisted completion where hints occurred. Model/errorless/media have null accuracy. A media-end event is participation unless the engine's unseeked completion rule and transition checks hold; even then it does not imply comprehension. Server validation limits accidental corruption, not proof that an offline client is tamper-proof.

## Privacy and lifecycle

**Inferred — original privacy requirements, not legal conclusions:** Collect parent authentication identity, optional contact email, child pseudonymous ID, bounded nickname/avatar and only the progress needed for the product. Optional age band is coarse and omittable. Avoid real child names in prompts and UI defaults. No microphone, camera, precise location, advertising identifier or behavioral replay. Do not put nicknames/email in event payloads, URLs, logs or CDN storage.

Apply parent ownership checks in the API and database access layer; analytics workers receive restricted fields. Encrypt transport and managed storage; separate operational credentials and restrict parent contact access. Publish retention/deletion policy before launch; exact durations require approval. Account deletion immediately tombstones authorization and rejects queued offline uploads, then cascades child profiles, attempts, events, projections and grants through an audited job. Preserve only nonidentifying aggregate metrics where approved. Content and reward definitions survive account deletion. Backups expire under a documented retention schedule and restored systems must reapply deletion tombstones; device-local private data is cleared on logout/deletion acknowledgement. Shared devices remain a privacy risk requiring parent-facing guidance.

## Technical risks

**Inferred:** Replayed or reordered events can inflate progress; unique IDs, contiguous validation and transactional reward grants mitigate this. Multiple offline versions increase storage and retention cost; define a supported-version window before launch. Complex JSON configs can bypass referential integrity; publication validation is mandatory. Deletion may race uploads and backup restores; tombstones and recovery procedures are required. Excessive analytics retention exposes children's behavior; minimize fields and approve durations before launch.

## Open questions / Unknowns

**Unknown:** expected event volume, retention windows, supported offline-version duration, whether rewards should reset by curriculum, account-sharing policy, deployment jurisdiction and applicable consent requirements. No third-party database, privacy guarantee or learning effect was verified by the research. [Research limitations](../research/technical-evidence.md#limitations-and-original-engineering-direction) explicitly leaves persistence and aggregation unverified. Resolve legal requirements with qualified review before production.

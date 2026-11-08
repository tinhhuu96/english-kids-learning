# System design

## Status and evidence convention

**Inferred — original proposal:** All components, APIs and requirements below describe a proposed MVP; none is a claim of implementation or measured efficacy. Research baseline is `5d443fc`. **Confirmed** denotes direct evidence, **Inferred** includes static-code findings, and **Unknown** denotes unverified behavior. Legacy C-code is treated as Inferred, never runtime observation.

## Architecture and decision traceability

| Research evidence | Proposed choice (Inferred) |
|---|---|
| **Confirmed:** [Activity R, observations](../research/feature-inventory.md#activity-r--zac-the-rat-learn-to-read-lesson) records a 390×292 canvas in portrait and missing semantic controls. | Responsive DOM buttons/text and SVG illustrations; canvas only for visual effects that need it, with equivalent DOM controls. Phone portrait is a primary layout, tablet uses extra space without shrinking controls. |
| **Confirmed:** [Flow A](../research/interaction-flows.md#flow-a--entry-and-start-gate) records Play entry. **Unknown:** real iOS behavior in [limitations](../research/technical-evidence.md#limitations-and-original-engineering-direction). | Large first-tap start gate attempts audio unlock; failure displays retry. Never assume autoplay support. |
| **Inferred — from static code:** [Technical anchors](../research/technical-evidence.md#technical-anchors-and-abstract-models) describes distinct asset groups. **Confirmed:** [performance definition](../research/technical-evidence.md#fallback-trigger-commands-and-performance-definition) reports substantial entry payloads in limited probes. | Immutable CDN assets, current/next-step loading, PWA package cache and parent-controlled media downloads. Research does not establish our performance budget. |
| **Inferred — from static code:** [Activity S](../research/feature-inventory.md#activity-s--wheels-on-the-bus-song--video) describes playback-position tracking; [Activity R](../research/feature-inventory.md#activity-r--zac-the-rat-learn-to-read-lesson) describes final-scene tracking. | Distinguish playback, participation, check attempts and lesson completion in analytics. |
| **Inferred:** [Cross-activity patterns](../research/feature-inventory.md#cross-activity-patterns) proposes separate participation/accuracy/completion. **Unknown:** server persistence in [limitations](../research/technical-evidence.md#limitations-and-original-engineering-direction). | Our own authenticated parent → child model, API and relational progress store; no inference about reference-site backend schemas. |

## Frontend and accessibility

**Inferred — original design:** A shared lesson shell interprets [typed step data](../product/lesson-engine-spec.md). State transitions are independent of rendering and audio adapters. DOM/SVG is the default; do not bake content into animation timelines. Use Pointer Events for optional drag, tap-select/tap-target and keyboard alternatives, minimum 64 CSS px hit areas, visible focus, labelled replay controls and no disabled browser zoom. Reflow to phone portrait without horizontal scrolling; use tablet columns only if target sizes remain intact. Test 320 CSS px width, landscape, text zoom and long Vietnamese labels.

Set UI language to Vietnamese and English target spans to English. Provide speech equivalents for visuals, text transcripts, synchronized media captions, non-color feedback, reduced-motion celebrations, and screen-reader announcements without repeated speech collisions. Keep pause/replay/exit reachable during narration. These are accessibility requirements, not conformance claims; research did not validate assistive-technology use.

## Audio and media delivery

**Inferred — original design:** A single audio coordinator serializes Vietnamese guide and English target playback, stops stale speech on navigation, and reports playback errors. First tap resumes the audio context and starts the first clip under that gesture. Audio-unlock rejection keeps the lesson at the retry gate and requires a fresh user gesture. After successful unlock, a failed narration clip offers Replay and an explicit accessible Continue that unlocks input without recording answer success; it never returns to the unlock gate or strands the step. Backgrounding pauses the lesson; return requires resume, not silent auto-advancement. Separate recordings enable independent replay and localization. Browser-format support is a device-test question, not a guarantee.

Serve only our licensed/original media from a CDN using versioned content-hash URLs, correct MIME and CORS headers, immutable public caching for nonpersonal assets, range support for media and a controlled origin. No child identifiers in URLs or cache keys. The publishing process validates manifests, referenced assets and licenses before atomically releasing a lesson version. Account/progress responses never enter public CDN caches.

## PWA and offline behavior

**Inferred — original design:** Service worker caches the app shell and immutable lesson packages; IndexedDB stores manifests and pending events. A parent may download a lesson after seeing estimated size. Cache status becomes ready only after all required assets pass hash checks. Incomplete packages cannot be advertised as offline-ready. Keep the current attempt pinned to its version and retain that version server-side for sync. Shell updates activate between attempts.

Online use streams optional video; an offline package either includes required media or remains unavailable offline. Storage denial/quota eviction produces a clear download/retry state. Progress queued locally is durable before Next; it is separate from the asset cache and cannot be evicted by our media policy. Offline access uses the last authorized local child profile, is device-local and expires on logout; no offline account creation or parental account changes. Reconnect rechecks authorization before syncing. Logout clears private IndexedDB/profile data and pending events only after warning the parent about unsynced work; account deletion rejects later uploads.

## Backend, identity and API

**Inferred — original design:** One API service and relational database are sufficient initially; worker jobs handle deletion and aggregate analytics. Parent login establishes a secure session; parent settings require reauthentication. A child profile is scoped to one parent account, selected locally from owned profiles. Child mode cannot change accounts, consent or payments. Enforce ownership on every request server-side, even when the client supplies a child ID. No child email, password or public profile.

| Proposed route | Contract |
|---|---|
| `GET /v1/catalog` | Public published course/unit/lesson metadata; versioned cache validator. |
| `GET /v1/lessons/{id}/versions/{version}` | Immutable lesson manifest; no personal data. |
| `GET/POST /v1/children` | Authenticated parent's profiles; bounded nickname/avatar enum. |
| `POST /v1/children/{id}/attempts` | Client UUID, pinned lesson version; idempotent creation. |
| `POST /v1/children/{id}/attempts/{attemptId}/events` | Bounded batch, stable event IDs and sequence; per-event accepted/duplicate/rejected results. Reject foreign ownership and invalid step transitions. |
| `GET /v1/children/{id}/progress` | Server aggregate plus cursor; no claim of mastery from media. |
| `DELETE /v1/account` | Reauthenticated parent request; deletion job and completion state, no fresh uploads after tombstone. |

Rate-limit writes, validate schemas and asset IDs, protect cookie mutations against CSRF, and return explicit retryable versus terminal errors. Reject immutable event payload changes under an existing ID. See [database design](database-design.md) for deduplication and transactions. Offline client timestamps are informational; server receipt time governs operational retention. Concurrent device attempts remain separate; completed lesson state is derived from validated completions rather than last-write-wins replacement.

## Analytics and operations

**Inferred — original design:** First-party events: `lesson_started`, `step_started`, `answer_submitted`, `hint_shown`, `step_completed`, `media_started`, `media_progress`, `media_seeked`, `media_ended`, `lesson_completed`, `asset_failed`, `audio_unlock_failed`. Allowlist dimensions: version, step, outcome, assistance, active duration and coarse device class. No voice recordings, free text, advertising SDKs or session replay. Learning events stay in the protected progress store; operational metrics omit child IDs. Deduplicate before aggregation. Reward grant is a transactional consequence of verified completion, not a client assertion.

Observe asset failure rate, first usable step time, unlock failures and sync queue age using bounded retention. Proposed 500 KB first-interaction budget is a target requiring throttled mobile measurements. Test offline restart, duplicate upload, reordering, logout and deletion during reconnect before launch. This is a design validation plan, not tests already run.

## Technical risks

**Inferred:** iOS audio policies/interruption may strand narration; require real-device checks and replay fallback. Weak networks make media-heavy packages unusable; progressive loading and download estimates help. Small screens make drag unreliable; mandatory tap equivalence helps. Offline events may conflict or replay; immutable versioning and server validation are required. Child-data leakage through logs/cache is possible; isolate private storage and audit allowlists. Deletion across backups requires a documented operational procedure.

## Open questions / Unknowns

**Unknown:** supported device/browser floor, real offline quota, target network profile, hosting/CDN provider, retention periods and deployment region, launch jurisdictions and consent obligations, parent authentication provider, accessibility test results. These require decisions/testing before production; this document makes no legal compliance or reference-site backend claim. [Research limitations](../research/technical-evidence.md#limitations-and-original-engineering-direction) similarly leaves persistence, iOS and assistive technology unknown.

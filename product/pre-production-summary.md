# Pre-production summary (board gate before production code)

**Status:** research and design phase complete; **no production code has been written**. This summary must be approved by the board before implementation starts. It condenses the nine deliverables; the linked documents hold the evidence and full contracts.

Labels follow the board's QA rule: **Confirmed** = direct runtime/REA observation or a public page (a publisher statement is confirmed only as a statement); **Inferred** = a conclusion from evidence, including anything read only from static HTML/JS; **Unknown** = not verified. Interaction flows read only from static code are labelled Inferred and are not claimed as analysed flows.

## 0. Research method and tooling

- **Confirmed:** REA CLI 6.8.0 (`rea capture-browser-scenario <scenario.json> --format json`) ran successfully on a Starfall activity: exit 0, completeness `complete`, 4/4 steps completed. Command, evidence ID and report path: [REA CLI verification](../research/technical-evidence.md#rea-cli-verification-2026-11-07-miu-miu).
- **Confirmed limitation:** REA `click` targets locators only, not canvas coordinates. Canvas drag/answer probes used a Playwright/CDP fallback, recorded in [fallback commands](../research/technical-evidence.md#fallback-trigger-commands-and-performance-definition). Long-term fix: a coordinate click/drag action in REA.
- **Sample:** four public Starfall activities (Letter A pilot, a song/video, a sound–letter game, a Learn-to-Read book), no login, nothing bypassed.
- **Secondary products (Lingokids, Duolingo ABC, Duolingo):** studied **only through public web pages and store listings** captured with REA. No app was installed and no account was created; nothing about their in-app behaviour is Confirmed beyond what the publisher states ([method](../research/reference-comparison.md#method-and-evidence)).
- No third-party code, lesson text, art, audio or screenshots are in the repository; raw REA output stays outside the repo.

## 1. Features researched

| Pattern | Evidence | Source |
|---|---|---|
| Large Play gate that also unlocks audio | **Confirmed** on all four Starfall samples | [Cross-activity patterns](../research/feature-inventory.md#cross-activity-patterns) |
| Real wrong/right feedback: wrong tile returns, correct advances | **Confirmed** (one wrong + one right attempt in the sound–letter game) | [Activity P](../research/feature-inventory.md#activity-p--make-a-word-with--at-soundletter-game) |
| Visual idle hint (pointing hand) | **Confirmed** in the pilot | [Flow B](../research/interaction-flows.md#flow-b--guided-lesson-4-pages-linear) |
| Semantic video player controls, end UI | **Confirmed** (end UI reached by seeking) | [Activity S](../research/feature-inventory.md#activity-s--wheels-on-the-bus-song--video) |
| Canvas shrinks to 390×292 in phone portrait; canvas choices lack semantics | **Confirmed** | [Activity R](../research/feature-inventory.md#activity-r--zac-the-rat-learn-to-read-lesson) |
| Rendering stacks (CreateJS up-front manifests; Video.js + MP4) | **Confirmed**; Phaser staged loading is **Inferred – from static code** | [Technical evidence](../research/technical-evidence.md) |
| Load size before first interaction ≈ 1.1–1.35 MB (single desktop runs) | **Confirmed** measurement, not a benchmark | [Measurements](../research/technical-evidence.md#fallback-trigger-commands-and-performance-definition) |
| Parent progress area, short lessons, offline play, badges/path progress | **Confirmed only as publisher statements** | [Reference comparison](../research/reference-comparison.md) |

## 2. Features not yet verified

- **Inferred – from static code (not exercised at runtime):** Letter A pages 2–4 and errorless auto-placement; completion rules of all four samples; separate instruction/target audio clips; hint timing/repeat limits; song completion threshold; later reading scenes.
- **Unknown:** full four-word reward path and exit in the sound–letter game; uninterrupted video viewing and captions; screen-reader and keyboard behaviour; iOS audio behaviour; how any reference product persists per-child progress; real in-app behaviour of Lingokids / Duolingo ABC / Duolingo.
- **Unknown for our own product:** Vietnamese voice comprehension by ages 3–6, effective lesson length and hint timing, number of choices per age, offline capacity on low-end devices, educational efficacy. Each has a verification method in [MVP requirements](mvp-requirements.md#features-not-yet-verified).

## 3. Proposed Lesson Engine architecture (Inferred — original design)

Full contract: [lesson engine spec](lesson-engine-spec.md); platform: [system design](../architecture/system-design.md); storage: [database design](../architecture/database-design.md).

- **Data-driven lessons:** versioned JSON (`schemaVersion`, immutable `version`, asset map, ordered typed `steps`). Content is never baked into animation timelines.
- **Five step types:** `model` (hear + tap target), `errorless` (participation only, no score), `check` (2–3 picture choices, answer compared by ID, gentle retry), `media` (original song/story; completes only on an un-seeked full run), `celebrate` (idempotent reward).
- **Audio coordinator:** Vietnamese guide → English target → input ready; never overlapping; independent replay. Audio-unlock failure keeps the retry gate; a later clip failure offers Replay plus an accessible Continue so no step can get stuck.
- **State machines:** lesson `idle → unlocking → loading → active → completed` (with `paused`, `recoverable_error`, `abandoned`); step `preloading → narrating → ready → responding → feedback → completed`; a transition token drops stale callbacks.
- **UI:** DOM/SVG (not canvas), hit areas ≥ 64 CSS px, tap mandatory and drag optional, keyboard/focus, reduced motion, portrait and landscape.
- **Loading/offline:** load current step, warm only the next; first-interaction budget 500 KB; PWA shell plus parent-initiated offline packs verified by SHA-256.
- **Progress:** local-first append-only events with stable IDs and typed payloads, validated by the server; first-response accuracy kept separate from participation and assisted answers; no mastery claims; parent account owns pseudonymous child profiles.

## 4. MVP feature list

Must (first release, curriculum units 1–2 from the [curriculum outline](curriculum-outline.md)):
1. Audio-led lesson entry with Vietnamese guide and separate English target audio.
2. Short ordered lessons: `model` → `errorless` → `check` → `celebrate`.
3. Picture-tap recognition check with calm retry; completion and first-answer accuracy recorded separately.
4. Phone-sized accessible controls (390×844 portrait/landscape, ≥ 64 px, keyboard, reduced motion).
5. Original Vietnamese voice recordings for all child guidance.
6. Resumable per-child progress without duplicate rewards; parent reset.

Should: parent practice summary, offline lesson pack, bounded hints, original song/story `media` step.
Later: sound–letter path (unit 3), multiple child profiles and caregiver controls.
Non-functional: load budget, offline survival, accessibility, child privacy (no ads, tracking, microphone or child email), Vietnamese UI voice. Acceptance criteria for every item: [MVP requirements](mvp-requirements.md).

## 5. Technical risks

| Risk | Mitigation (Inferred) |
|---|---|
| iOS/Safari audio unlock and interruptions strand narration | Fresh-gesture retry, Replay/Continue fallback, real-device testing early |
| Large media stalls on weak mobile networks | Step-level loading, poster-first media, explicit offline download with size shown |
| Drag is unreliable on small screens and masks recognition | Tap path is mandatory; drag optional and never scored when cancelled |
| Offline/duplicate event sync inflates progress or rewards | Stable event IDs, monotonic sequences, server validation, transactional rewards |
| Child data leaks through logs, caches or analytics | Pseudonymous IDs, field allowlist, no third-party SDKs, export/delete with tombstones |
| Complex lesson JSON breaks references | Publication-time validation of schema, asset refs and hashes |
| Offline storage quota on low-end devices | Evict unpinned media first, never pending events; measure on real devices |
| Content risk: unclear Vietnamese wording or English recordings | Language-educator review and moderated family sessions before launch |

Open decisions that need the board before production (not research blockers): device/browser floor, hosting/CDN and region, parent authentication provider, data-retention periods and launch jurisdictions (legal review), and sourcing of original voice and art assets (this costs money).

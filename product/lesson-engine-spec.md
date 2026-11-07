# Lesson engine specification

## Scope and evidence convention

**Inferred — original proposal:** This document defines our MVP contract, not an implemented engine or a claim of educational efficacy. All normative requirements and example values below are original design proposals. **Confirmed** means direct evidence; **Inferred** includes conclusions from static code; **Unknown** means unverified. Legacy research labels C-code and H are not promoted to Confirmed here. Research snapshot: `origin/research/integration` at `5d443fc`.

## Design rationale

| Evidence and label | Original decision (Inferred) |
|---|---|
| **Inferred — from static code:** [Flow B and Flow C](../research/interaction-flows.md#flow-b--guided-lesson-4-pages-linear) describe gated pages and automatic correct placement. | Ordered typed steps; errorless practice is participation, never accuracy. |
| **Confirmed:** [Activity P, observations](../research/feature-inventory.md#activity-p--make-a-word-with--at-soundletter-game) records an actual wrong/right pair. | Separate check steps with explicit answer comparison and gentle retry. |
| **Inferred — from static code:** [Activity P, observations](../research/feature-inventory.md#activity-p--make-a-word-with--at-soundletter-game) identifies separate instruction/target clips. | Separate Vietnamese guide and English target audio; do not copy those clips. |
| **Confirmed:** [Flow B](../research/interaction-flows.md#flow-b--guided-lesson-4-pages-linear) records an idle hint. | Bounded idle hints, with provisional timings below. |
| **Confirmed:** [Activity S, observations](../research/feature-inventory.md#activity-s--wheels-on-the-bus-song--video) records playback and end UI after seeking. **Inferred — from static code:** same section describes tracking thresholds. | Playback, completion and recognition are separate outcomes. |
| **Inferred — from static code:** [Technical anchors and abstract models](../research/technical-evidence.md#technical-anchors-and-abstract-models) describes staged loading. | Preload current step and warm the next, rather than an entire course. |

## Lesson envelope and common step contract

**Inferred — original contract:** A lesson has `schemaVersion:1`, stable `id`, immutable positive integer `version`, title, asset map and nonempty ordered `steps`. Step IDs are unique within a lesson version. Each step has `id`, `type`, `guideVi`, `targetEn`, `visual`, and optional `hintVi`; all asset references must resolve. Audio references identify separate recordings, never a mixed instruction/target track. Each visual has a spoken equivalent. Asset kinds include `audio`, `image`, `video`, `captions` and `transcript`; the required media `transcript` reference resolves a text asset with role `transcript`. Asset records require `kind`, relative `url`, MIME type, byte size and SHA-256 at publication; the example uses planned placeholders and cannot be published until metadata and original assets exist.

Audio sequence is Vietnamese guide, then English target, then input readiness. Replay controls can play either independently. Never overlap speech; pause cancels scheduled audio. Instruction locking applies only to task answers: replay, pause and parent exit remain available. Audio-unlock rejection keeps the lesson at the retry gate and requires a fresh user gesture. After successful unlock, a failed narration clip offers Replay and an explicit accessible Continue that unlocks input without recording answer success; it never returns to the unlock gate or strands the step. No microphone or speech scoring in this MVP.

All interactive hit areas are at least 64 CSS px at the rendered viewport, with spacing and visible focus. Tap selection is mandatory; drag is optional and uses one pointer, snap feedback and return-to-origin on outside drop. An outside drop is `cancelled_drop`, never incorrect. Keyboard activation and tap-select then tap-target provide equivalent actions. These dimensions are proposed requirements, not measured usability results.

## Typed step shapes

**Inferred — original contract:** Shapes below add fields to the common contract. Required assets include the common Vietnamese guide, English target and visual, plus those listed. `hintVi` is required for interactive steps. `feedback` contains original Vietnamese `successVi` and `retryVi` audio IDs for checks; practice requires `successVi`. No third-party scripts, wording or media are included.

| Type | JSON fields beyond common fields | Required assets | Input, feedback and completion |
|---|---|---|---|
| `model` | `{"type":"model","interaction":"tap","targetId":"ball"}` | Target illustration | Tap highlighted target replays English; no distractor or wrong state. Emit `step_completed` with `outcome:"participated"` after required tap. |
| `errorless` | `{"type":"errorless","interaction":"tap_or_drag","itemId":"ball","destinationId":"basket","feedback":{"successVi":"praise"}}` | Item and destination visuals; praise audio | Only valid destination offered. Snap any released item to it; tap equivalent available. Emit participation completion, score null. |
| `check` | `{"type":"check","interaction":"tap","choices":[{"id":"ball","visual":"ball"},{"id":"cup","visual":"cup"}],"answerId":"ball","feedback":{"successVi":"praise","retryVi":"retry"}}` | Each choice visual; success and retry audio | Compare choice ID. Wrong: gentle cue, keep item active, emit incorrect attempt, no penalty. Right: English replay + praise; emit correct attempt and completion. |
| `media` | `{"type":"media","media":"song","poster":"ball","captions":"song-captions","transcript":"song-transcript"}` | Original song/video with English targets, poster, synchronized captions and transcript | Large Play/pause/replay; no answer checking. Emit `media_started`, `media_ended`; completion requires actual end without seek in that run. Seeking may emit participation only; offer replay. |
| `celebrate` | `{"type":"celebrate","rewardKey":"first-ball"}` | Original celebration visual and bilingual audio in common fields | No accuracy; motion optional. After audio or fallback Continue, emit completion and request idempotent reward grant. Continue exits lesson. |

**Inferred — original feedback policy:** Errorless steps precede checks in published beginner lessons. No red failure screen, loss of reward or timer pressure. After 7 seconds of active idle time, glow; at next interval point; at next interval replay hint audio once, then stop until meaningful input. Pause, hidden tab, loading and narration suspend idle time. Two wrong answers show the hint immediately. A hinted correct answer remains correct but has `assisted:true`; do not report it as independent mastery. Media/celebrate offer replay cues rather than answer hints.

## State machines and scoring

**Inferred — original contract:** Lesson states: `idle → unlocking → loading → active → completed`. Loading failure enters `recoverable_error → loading` on retry; exit enters `abandoned`. `active → paused → active` retains current step; exit always remains available. Audio unlock rejection stays in unlocking with a visible retry button. Lesson completes only when every ordered step has a committed completion event.

Step states: `preloading → narrating → ready → responding → feedback → completed`. Incorrect check feedback returns to ready; a cancelled drop returns to ready without scoring. Model/errorless success and correct checks complete after feedback. Media uses responding during playback. Any active state may pause; resumption returns to ready and replays the interrupted instruction rather than continuing stale callbacks. Back revisits a completed step in replay mode; Next requires completion and next-step readiness. Replay never creates another score/reward. Use a transition generation token to ignore stale load/audio callbacks.

Events include `eventId`, `attemptId`, child pseudonymous ID, lesson ID/version, step ID, monotonic sequence, event type, elapsed active milliseconds, outcome, assistance and client timestamp. Each event also has a bounded, type-specific `payload`: `answer_submitted` carries `choiceId`; media events carry `playbackRunId`, with `media_progress` reporting `positionMs`/`playedMs`, `media_seeked` reporting `fromMs`/`toMs`, and `media_ended` reporting `positionMs`/`playedMs`/`seekCount`. A replay starts a new run; any seek disqualifies that run from completion. The database design defines validation and remaining payloads. Server adds received time. Check accuracy is first-response correct checks / answered checks; practice and media excluded. Retain final assisted completion separately. No mastery claim from lesson completion. Persist locally before advancing and retry upload with the same IDs. Detailed persistence is in [database design](../architecture/database-design.md).

## Asset loading

**Inferred — original contract:** Resolve/validate the manifest before playback. Load current-step required images and speech before entering narrating; warm only next-step essentials with bounded concurrency. Media loads poster and metadata first; full offline media download is a parent action. Loading failure offers Retry and parent exit, never records success. Verify content hashes before marking an offline package complete. Pin lesson version throughout an attempt; do not replace assets under a running step. Evict unpinned media first; never evict pending attempt events. A provisional first-interaction budget of 500 KB excludes optional video and requires device/network measurement.

## Original example lesson JSON

**Inferred — authored example:** “Ball in a basket” introduces one English noun. Asset URLs are future Min Labs recordings/art, not downloaded references. The manifest is a draft; publication must add MIME/bytes/hash metadata as specified above.

```json
{
  "schemaVersion": 1,
  "id": "ball-basket",
  "version": 1,
  "title": "Ball in a basket",
  "assets": {
    "guide": {"kind":"audio","url":"original/vi/listen-ball.aac"},
    "target": {"kind":"audio","url":"original/en/ball.aac"},
    "move": {"kind":"audio","url":"original/vi/put-ball-in-basket.aac"},
    "choose": {"kind":"audio","url":"original/vi/find-ball.aac"},
    "hint": {"kind":"audio","url":"original/vi/round-ball-hint.aac"},
    "praise": {"kind":"audio","url":"original/vi/you-found-it.aac"},
    "retry": {"kind":"audio","url":"original/vi/listen-and-try.aac"},
    "ball": {"kind":"image","url":"original/art/blue-ball.svg"},
    "cup": {"kind":"image","url":"original/art/yellow-cup.svg"},
    "basket": {"kind":"image","url":"original/art/basket.svg"},
    "song": {"kind":"video","url":"original/video/rolling-ball.mp4"},
    "song-captions": {"kind":"captions","url":"original/text/rolling-ball.vtt"},
    "song-transcript": {"kind":"transcript","url":"original/text/rolling-ball.txt"},
    "finish": {"kind":"audio","url":"original/vi/lesson-finished.aac"}
  },
  "steps": [
    {"id":"meet","type":"model","guideVi":"guide","targetEn":"target","visual":"ball","hintVi":"hint","interaction":"tap","targetId":"ball"},
    {"id":"move","type":"errorless","guideVi":"move","targetEn":"target","visual":"basket","hintVi":"hint","interaction":"tap_or_drag","itemId":"ball","destinationId":"basket","feedback":{"successVi":"praise"}},
    {"id":"find","type":"check","guideVi":"choose","targetEn":"target","visual":"ball","hintVi":"hint","interaction":"tap","choices":[{"id":"ball","visual":"ball"},{"id":"cup","visual":"cup"}],"answerId":"ball","feedback":{"successVi":"praise","retryVi":"retry"}},
    {"id":"watch","type":"media","guideVi":"guide","targetEn":"target","visual":"ball","media":"song","poster":"ball","captions":"song-captions","transcript":"song-transcript"},
    {"id":"finish","type":"celebrate","guideVi":"finish","targetEn":"target","visual":"ball","rewardKey":"first-ball"}
  ]
}
```

## Technical risks

**Inferred — proposed mitigations:** Audio unlock may fail on iOS: retry on a fresh gesture and test interruptions on real devices. Large media may stall: bounded prefetch and explicit offline downloads. Drag may mask recognition: equivalent tap path. Duplicate sync may overcount: stable IDs and server transactions. Idle cues may distract: bounded repetition and reduced motion. These are risks, not observed outcomes of our unbuilt engine.

## Open questions / Unknowns

**Unknown:** Effective lesson length, hint timing and Vietnamese wording for ages 3–6; real iOS behavior; low-memory offline capacity; caption comprehension; suitable independent-mastery threshold. Family and accessibility testing is required. [Research limitations](../research/technical-evidence.md#limitations-and-original-engineering-direction) also leaves child efficacy and iOS behavior unverified.

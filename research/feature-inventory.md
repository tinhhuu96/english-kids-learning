# Feature Inventory — Pilot

Scope: one Starfall activity ("ABCs → Letter A"), mapped onto the 7 research objectives.
Legend: **[C]** = confirmed, **[C-code]** = confirmed in code only, **[H]** = hypothesis. MVP relevance: **High / Med / Low**.

| # | Objective | Starfall pattern observed | Status | MVP relevance | Our original approach |
|---|---|---|---|---|---|
| 1 | Interactive lesson structure | Activity = guided lesson (4 linear pages) → practice minigame → completion. Pages are timeline frames. | C / C-code | High | Data-driven lesson = ordered list of typed *steps* (see future `product/lesson-engine-spec.md`). |
| 2 | Audio-first instructions | No on-screen text instructions. Narration per page; spoken directions in the minigame. Sound effect on every tap. | C-code | High | Vietnamese instruction VO + English target VO, as separate audio tracks per step. |
| 2b | Audio unlock | Big Play button gates the start (user gesture for audio). | C | High | Same idea, with our own design. Preload the first step's audio only. |
| 3 | Touch-friendly patterns | Tap-to-hear (scale + sound), drag-to-target with a generous snap, one drag at a time, `createjs.Touch`. | C-code | High | Pointer Events. Hit targets ≥ 64 px at the smallest viewport. Drag with snap + auto-return. |
| 3b | Hint system | Animated hand + sparkles on the next action after ~7 s idle, max ~4 repeats. | C | High | Idle hint with escalation: glow → mascot points → narration replay. |
| 4 | Correct / incorrect feedback | Correct: zoom + sound + slot fill. **Incorrect: impossible (errorless)**. Completion: praise VO + animation. | C-code | High | Errorless for "learn" steps; gentle-error for "practice" steps; never punitive. |
| 5 | Navigation and completion | Next appears only when the step is done; Back from page 2; Close always visible; completion ping to server. | C / C-code | High | Same gating. Plus a per-child progress record and a star/sticker reward (Duolingo-style), stored server-side. |
| 6 | Media loading / frontend | Adobe Animate canvas, CreateJS, 12 fps, per-activity manifest, ~2 MB and ~5 s up front, 2-year asset cache. | C | Med | Lazy load per step, sprite or SVG + Opus/AAC audio, service worker cache, a target of < 500 KB before the first interaction. |
| 7 | Responsive / accessibility | 4:3 letterbox scaling. On phone portrait the stage shrinks to 390×292 at the top and the close button is ~18 px. No ARIA on the canvas, no `lang`. Video alternative link in the footer. Zoom disabled. | C | High | Landscape-first layout that also works in portrait. DOM/SVG-based so it is accessible. Respect reduced motion. Large targets. |

## Patterns worth reusing (as ideas, not assets)

- Start gate → narrated intro → tap to explore → gated Next → errorless practice → celebration.
- Input lock while narration plays.
- Bounded idle hints.
- Every tap gives instant audio and visual feedback.
- One concept per activity (one letter).

## Patterns to avoid or improve

- Content baked into animation timelines, so it can't be localised or edited without an animator. We go data-driven.
- Front-loading every asset, so slow mobile starts suffer.
- Canvas-only UI, which gives poor accessibility and small touch targets on phones.
- No per-child progress.
- No real assessment, because the practice can't be failed.

## Not yet investigated (needs approval to expand)

- Other activity types: songs/videos, letter-sound games, the "Learn to Read" phonics series, math and colors.
- The Enhanced Accessible Index.
- The secondary references: Lingokids, Duolingo ABC, Duolingo. These mostly live in native apps, so web inspection is limited and research would rely on public docs and hands-on use.


## MIN-9 expansion — 2026-11-07

[C] Research below adds three public activities to the preserved pilot. Evidence IDs, measurement definitions and exact commands are in `technical-evidence.md`, MIN-9 sections. [H] Recommendations are original product proposals for Vietnamese children aged 3–6; they are not claims of measured learning outcomes. [C] The pilot's “Not yet investigated” list above describes the earlier scope, before this expansion.

### Activity S — Wheels on the Bus (song / video)

[C] Public example: https://www.starfall.com/h/motionsongs/vfs224390218/ ; discovered through the public kindergarten → Motion Songs index.

#### 1. Interaction flow

[C] Index link → poster and Play → video playback → end screen with green arrow. [C-code] Both Close and the green arrow use the configured Motion Songs index exit, rather than selecting a new lesson automatically.

#### 2. Observations (seven focus areas)

- [C] Structure: a continuous video, duration 159.42 s, rather than answer-driven steps.
- [C] Audio-first entry: a central Play control starts playback; a moving playback clock confirmed playback. [H] Whether children understand the English singing without support needs testing.
- [C] Touch/UI: a large Play control plus transport, progress, mute and speed controls; no drag task.
- [C] Feedback: no scored answer choices appear in the captured player. [C] Seeking near the end exposed an end screen and green arrow. [H] This observes navigation, not comprehension or full uninterrupted viewing.
- [C-code] Completion: tracking is configured at 25 s of playback position and the arrow at 10 s remaining; therefore a completion signal need not mean the child watched the entire song.
- [C] Loading: Video.js 8.16.1, MP4 and a poster, with media fetching before Play (`preload=auto`).
- [C] Accessibility: DOM buttons have text alternatives, progress has slider semantics, and HTML declares English; a transcript link is present. [C-code] `captionURL` is empty in this activity config. [H] A transcript link does not establish synchronized captions or screen-reader usability; those were not fully tested.

#### 3. Technical evidence

[C] REA captures `song.json` / `song-play.json` confirmed the page shell, controls and Play action; the fallback probe measured 18 requests / 1,115,370 completed encoded bytes when Play was available at 2.942 s. [C] The fallback observed playback time advancing after a real click and the end screen after a seek. [C-code] The shared video controller supplies threshold-based tracking and exit behavior.

#### 4. Observable data structures

[C-code] An activity config supplies video URL, poster URL, exit URL, tracking code, progress visibility, caption source and timing thresholds. [C] Runtime video state exposes current time, duration and paused state. [H] There is no evidence here of an individual child's comprehension record.

#### 5. Limitations / unknowns

[C] The end was reached by seeking, not watching the entire song; audio was not evaluated by a human listener. [H] Real mobile browser autoplay, captions, keyboard focus order and assistive-technology behavior remain unverified. [C] This sample provides no wrong-answer behavior to evaluate.

#### 6. Original implementation recommendation

[H] Use short original songs with Vietnamese spoken entry cues, English targets, a large replay icon and a parent-facing transcript. Separate “played,” “finished” and “practised” events; use a short picture-choice follow-up if recognition is the goal. Keep essential controls large in portrait and provide synchronized captions plus motion reduction outside the video.

### Activity P — Make-a-Word with -at (sound–letter game)

[C] Public example: https://www.starfall.com/h/ltr-sv-a/maw-at/ ; linked from the free Learn to Read index.

#### 1. Interaction flow

[C] Play → animated introduction → picture and incomplete word with four initial-letter tiles → drag a tile into the initial slot → retry on mismatch or blend animation on match. [C-code] Four shuffled picture items lead to a separate reward state and a Continue exit.

#### 2. Observations (seven focus areas)

- [C] Structure: an introductory animation precedes a recognition task, with randomized tile/picture order across runs.
- [C-code] Audio-first: separate instruction, phoneme, word, mismatch and success clips are loaded; the timeline calls their playback. [C] No written instruction is needed to locate the visual slot; understanding the spoken direction was not user-tested.
- [C] Touch: four large letter tiles support pointer dragging into a single visible slot. [C-code] Dragging is enabled/disabled by state and the drop must fall inside a defined rectangle.
- [C] Incorrect feedback exists: for a pictured target requiring `r`, dragging `b` into the slot returned it to its original position; item counter stayed 0 and animation frame stayed 5. [C-code] That branch plays `HONK`; perceived tone was not listened to. [C] The correct `r` disappeared from the choices and filled the slot; frame advanced to 26 after 1.8 s. This is genuine answer checking, unlike the pilot's auto-sort.
- [C-code] Navigation: Close uses the short-A index exit; after four successful words the reward state offers Continue. No per-item Next is required for a wrong answer.
- [C] Loading: Phaser CE 2.20.1, atlas JSON/PNG and individual MP3s. [C-code] Later asset groups are loaded after the initial game state, with loading flags used when progression needs them.
- [C] Accessibility: the activity DOM contains a canvas and footer links, with no semantic letter-choice controls and no HTML language declaration; viewport config disables zoom. [C-code] Mobile portrait triggers an orientation element in the scaling code. [H] This is not evidence of usable keyboard dragging or screen-reader access.

#### 3. Technical evidence

[C] REA `phonics.json` / `phonics-play.json` captured public entry and the post-Play state. [C] The fallback measured first game/Play state at 5.628 s (26 requests; 1,192,722 encoded bytes), then 58 requests / 1,659,480 bytes by the end of intro + wrong/right probes. [C-code] The retained application bundle compares the dragged letter with `currentWord.answerLetter`; it does not accept every letter.

#### 4. Observable data structures

[C-code] Abstract model: choice letters; shuffled word records `{pictureId, answerLetter, onsetAudio, rimeAudio, wordAudio}`; current-item counter; drag origin; drop rectangle; input-enabled flag; asset-group-ready flags; reward state. [C] Runtime state inspection was read-only; answers were submitted with mouse movement, not by invoking handlers.

#### 5. Limitations / unknowns

[C] Runtime verified one wrong and one right answer on the first item, not the whole four-item reward sequence. [C-code] Reward behavior and completion navigation were read in code. [H] The emotional effect of the mismatch sound, localization difficulty and independent use by preschoolers need testing. [C] Mobile orientation code was inspected, but this sample's portrait probe did not produce a timely completed result.

#### 6. Original implementation recommendation

[H] Offer both tap-choice and drag so motor difficulty does not masquerade as phonics difficulty. Replay Vietnamese instructions and English target audio independently. A wrong pick should preserve the item, return the tile gently and offer a visual hint after two misses; avoid copying the horn sound. Store attempts separately from completion and use a portrait layout with targets at least 64 CSS px.

### Activity R — Zac the Rat (Learn to Read lesson)

[C] Public example: https://www.starfall.com/h/ltr-sv-a/zac-the-rat/ ; the public Learn to Read index linked this interactive short-A book. [C] Entry and Play worked in a fresh browser context without a login, so the requested reading lesson did not require a substitute.

#### 1. Interaction flow

[C] Index → Play gate → animated short-vowel/title introduction. [C-code] The controller advances through eight scene states, exposes Back/Continue after scene signals, and exits to the short-A index after the final scene.

#### 2. Observations (seven focus areas)

- [C] Structure: an interactive canvas book, not the same unit as the separate Make-a-Word game beside it in the index.
- [C-code] Audio-first: scene animation and separate word/letter audio accompany highlighted reading interactions. [C] Runtime showed the vowel/title introduction after Play.
- [C-code] Touch: scene controls and oversized Play hit area are canvas objects; scene completion signals reveal navigation.
- [C-code] Feedback: reading interactions replay/animate content; no answer comparator was identified in this sampled book controller. [H] Do not treat page progression as a reading assessment.
- [C-code] Navigation/completion: scene counter 0–7, previous-scene action, gated Continue, final exit. Tracking is invoked upon entering scene 7, not after an externally verified comprehension check.
- [C] Loading: Phaser 2.10.5, separate scene sprite data, MP3 word clips and CanvasResizer. [C-code] First and later asset groups are distinct; progression can show a loader if a group is not ready.
- [C] Responsive/a11y: at 390×844 touch emulation, the canvas was 390×292 at the top and the rotation container was hidden. [C] Canvas choices/text are not semantic DOM controls; HTML lacks `lang`, zoom is disabled, and footer includes a separate Play Video alternative plus Accessibility Information.

#### 3. Technical evidence

[C] REA `book.json` / `book-play.json` confirmed anonymous entry and post-Play introduction. [C] Fallback desktop first game/Play state: 8.381 s, 45 requests / 1,347,188 encoded bytes; 118 requests / 2,895,693 bytes by the end of the intro/navigation probe. [C] Portrait first game state: 9.455 s, 45 requests / 1,347,170 bytes. [C-code] Code anchors: `sceneCounter`, `gotoNextScene`, `gotoPrevScene`, `goToNextPage`, `waitLoadingGroup02Assets`.

#### 4. Observable data structures

[C-code] Abstract model: scene index; first-play flag; current scene; sound/image groups; loading-ready flags; Back/Continue controls; scene signals for enabling navigation. [C-code] Lesson order and timings remain embedded in scene code rather than a portable lesson JSON schema.

#### 5. Limitations / unknowns

[C] Runtime covered entry/introduction, not every reading scene or final completion; the attempted lower-right click did not independently establish a page turn. [C-code] Later scenes, word interaction behavior and final navigation are code-only findings. [H] iOS audio unlock, narration quality, screen-reader operation and children's reading gains remain untested.

#### 6. Original implementation recommendation

[H] Build a short, original picture story using structured pages and per-word audio ranges; add Vietnamese voice guidance without translating away the English targets. Allow tap-to-hear and replay, accessible Back/Next buttons and a parent exit. Keep recognition practice separate from passive page completion, and lazy-load the next page while leaving the current page usable.

## Cross-activity patterns

| Dimension | Letter A pilot | Song S | Phonics P | Reading R |
|---|---|---|---|---|
| Structure | [C-code] Linear lesson + sort | [C] Video timeline | [C] Intro + choice task | [C-code] Eight scene states |
| Wrong answer | [C-code] Auto-routes correctly | [C] No scored choice in player | [C] Wrong tile returns; correct advances | [H] No assessment established |
| Entry | [C] Play gate | [C] DOM Play | [C] Canvas Play | [C] Canvas Play |
| Completion | [C-code] After sorting | [C-code] Playback-position threshold | [C-code] After four words | [C-code] Enter final scene |
| Rendering/loading | [C] CreateJS, up-front manifests | [C] Video.js + MP4 preload | [C-code] Phaser, staged groups | [C-code] Phaser, staged groups |
| A11y/phone | [C] Canvas shrinks in portrait | [C] Semantic player controls | [C] Canvas lacks choice semantics | [C] Canvas 390×292 in portrait |

[H] A shared shell can unify Play, replay, Close and return-to-index, while distinct activity types keep distinct success rules. The new samples show that the pilot's errorless feedback and entirely up-front loading must not be generalized to all Starfall activities. [H] For our MVP, use accessible DOM/SVG choices, original art/audio, Vietnamese spoken scaffolding and separate participation, answer-accuracy and completion records.

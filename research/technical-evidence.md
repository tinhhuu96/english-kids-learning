# Technical Evidence — Pilot: Starfall "ABCs → Letter A"

Status: **pilot** (one activity). Date: 2026-11-07. Researcher: Miu Miu (Min Labs).

Legend: **[C]** = confirmed (directly observed in network traffic, page source, or a rendered screenshot).
**[H]** = hypothesis (inferred, not verified at runtime).

## 0. Method and tooling

- **REA MCP was NOT available** in this runtime. The Paperclip connection catalog had no REA entry, and no REA tools were attached to the agent.
  As a fallback I used only public, unauthenticated access:
  - `curl` for HTML and JS of the public activity page (`robots.txt` only disallows `/cgi-bin/`).
  - Headless Google Chrome, driven over the Chrome DevTools Protocol by a small custom probe script (kept in scratch, not in the repo).
    It recorded network requests, page state and screenshots at 4 viewports.
- No login, paywall, or access restriction was touched. No Starfall media was downloaded into the repo or redistributed.
  Screenshots were only kept in temporary scratch space for analysis and are described in words here, not committed.
- Target URL: `https://www.starfall.com/h/abcs/letter-a/` (linked from the free ABCs index `https://www.starfall.com/h/abcs/`).

## 1. Page shell

| Observation | Evidence |
|---|---|
| [C] One HTML page with a single `<canvas id="canvas">` (initial size 1600×900) plus a DOM overlay used only for the loading spinner | page source |
| [C] Viewport meta turns off zoom: `user-scalable=no, maximum-scale=1` | page source |
| [C] Per-activity config is a global `SpecialDefinitions = { rateCode, exitURL, logoURL }` | inline `<script>` |
| [C] Entry is `<body onload="init()">` → `Main.init()` | page source |
| [C] The only DOM interactive elements are 3 footer links: "Play Video" (a video alternative), "Accessibility Information", "Back" | DOM query: 3 focusables |
| [C] No `lang` attribute on `<html>`; no ARIA on the canvas | DOM query |
| [C] HTML has `cache-control: no-store`; static assets have `max-age=63072000` (2 years) and cache-busting `?ds=YYYYMMDD` query strings | response headers |

## 2. Runtime stack

- [C] Adobe Animate (HTML5 Canvas) export running on **CreateJS 2015.11.26** (EaselJS, TweenJS, SoundJS, PreloadJS).
  Also **GSAP TweenMax 1.20.4** and **Proton** (particle system, used for sparkles).
- [C] An MVC-style split per activity:
  - `views/*.min.js` holds the Animate-exported timeline symbols (the "compositions").
  - `controllers/*.min.js` holds the hand-written game logic.
  - `BaseController` has `show()`, which adds the view to the stage, and `hide()`, which removes it.
- [C] The activity is made of **two compositions**, each with a controller:
  - `game`: the guided letter lesson, 4 pages.
  - `minigame`: a drag-and-sort practice game.
- [C] Stage: 1024×768 logical size, **12 fps** ticker (`RAF_SYNCHED`), `createjs.Touch.enable(stage)`, `enableMouseOver(10)`.
- [C] `CanvasResizer` scales the stage to fit the window while keeping the 4:3 ratio. It listens to `resize` and `orientationchange`.
  It can show a "rotate phone" overlay (`#rotate_icon_container`), but that overlay did not appear in any of our emulated viewports.
- [C] `activities.js` detects touch: `navigator.maxTouchPoints` plus `(pointer: coarse)` media queries.
  It toggles an `is-tablet-mode` class on `<html>`, which was set on the emulated phone and tablet.

## 3. Asset loading

- [C] Each composition declares a **manifest**, an array of `{src, id}` entries.
  `Main.loadComposition()` feeds it to a `createjs.LoadQueue` with the Sound plugin and `maxConnections = 10`.
  The loading spinner overlay stays up until the queue fires `complete`.
- [C] Sprite atlases: `ssMetadata = [{ name, frames: [[x,y,w,h], ...] }]`. One PNG atlas per composition, plus a few full-frame JPG/PNG backgrounds.
- [C] **Both compositions load up front.** The game composition loads first and is shown. Then `gameLoadCallback` starts loading the minigame in the background.
- [C] Audio is many small MP3 clips, one per utterance or sound effect, referenced by id: `playSound("<id>")`.
  - Game: 18 clips. Minigame: 9 clips.
  - Some clips are named after the voice line they hold (e.g. an instruction clip, a "that was great" clip), so voice lines are separate files from sound effects and music.
  - Audio is fetched with XHR.

### Measured load (headless Chrome, uncached, desktop broadband)

| Viewport | Time to spinner hidden | Requests | Transferred |
|---|---|---|---|
| Desktop 1366×768 | 4.7 s | 61 | ~2.0 MB |
| Phone portrait 390×844 (touch) | 5.3 s | 60 | ~2.0 MB |
| Phone landscape 844×390 (touch) | 8.5 s | 61 | ~2.0 MB |
| Tablet 1024×768 (touch) | 5.7 s | 61 | ~2.0 MB |

Breakdown (desktop):

| Type | Requests | Size |
|---|---|---|
| Audio (MP3) | 27 | ~800 KB |
| JS | 16 | ~620 KB |
| Images | 10 | ~520 KB |
| Fonts | 2 | ~52 KB |

The largest single file is a ~200 KB music loop.

[H] On a typical Vietnamese mobile connection (3G/4G, mid-range Android) the wait before first interaction will be clearly longer.
This approach front-loads everything and does not cache across activities, apart from the shared libraries.

## 4. Telemetry

- [C] `Tracking.insideFlash(rateCode)` sends a POST to `/cgi-bin/inside-flash-ok2?an=<rateCode>&<random>` when the activity is first shown. Observed in network traffic.
- [C] `Tracking.nRate(rateCode)` sends a POST to `/cgi-bin/flash-nratebook?an=...` once, when the minigame is completed (`checkForGameEnd`). Seen in code; not triggered at runtime in the pilot.
- [C] `Tracking.cntProgress(obj)` exists and posts arbitrary key/values to `/cgi-bin/rcn-progress`. It is not used by this activity.
- [H] There is no client-side per-child progress model for anonymous users. Completion is only an aggregate "activity finished" counter.

## 5. Observable data structures (abstracted, not copied)

```text
SpecialDefinitions  { rateCode: string, exitURL: url, logoURL: path }
compositions        { [name]: { id: animateCompositionId, controller: string, view: string } }
manifest[]          { src: relativePath, id: string }                 // images + sounds
ssMetadata[]        { name: atlasId, frames: [x, y, w, h][] }
lesson state        view.currentFrame = page index (0 = start, 1..4 = pages)
minigame state      capCount, smallCount (0..4 each), gameEnd: bool, canDrag: bool
```

All lesson content (words, picture order, audio timing) is baked into Animate timelines.
It is **not data-driven**: changing a word means re-exporting the animation.
This is the main architectural pattern we should **not** copy.

## 6. Limitations / unknowns

- Not observed at runtime, read from code only:
  - pages 2–4 of the lesson
  - the drag minigame
  - the completion celebration
  - exit to the index
- Audio content was not listened to: `--mute-audio` was on, and there is no human review.
  Voice-over timing relative to animation is only inferred from timeline `playSound` cues.
- Real-device behaviour (iOS Safari audio unlock, low-end Android performance) was not tested; we used emulation only.
- The Enhanced Accessible Index (an alternate accessible UI) was not examined.

## MIN-9 expansion — method, provenance and measurements

[C] Date: 2026-11-07; researcher: Tin Tin. The pilot above is preserved as historical evidence. [C] REA CLI 6.8.0 was available in this run; its absence in the earlier pilot does not apply to these captures. [C] Work was isolated in `$PAPERCLIP_SCRATCH_DIR/wt` on `research/starfall-expansion`, based on `origin/research/pilot-starfall-letter-a` (3f62ef4). [C] Only the three assigned Markdown files were edited; raw HTML, JavaScript, screenshots and response bodies remained in the run's scratch directory.

### Public sample selection

| Type | Public index | Activity |
|---|---|---|
| [C] Song/video | https://www.starfall.com/h/motionsongs/ | https://www.starfall.com/h/motionsongs/vfs224390218/ |
| [C] Sound–letter choice | https://www.starfall.com/h/learn-to-read/ | https://www.starfall.com/h/ltr-sv-a/maw-at/ |
| [C] Learn to Read | https://www.starfall.com/h/learn-to-read/ | https://www.starfall.com/h/ltr-sv-a/zac-the-rat/ |

[C] Index and activity pages were visited with no supplied cookies, local storage or credentials. Reading entry and Play were available without login. [C] No protected route was bypassed. [H] This establishes access for the sampled URLs/date, not universal access to the entire catalogue.

### Exact REA command record

[C] The following commands were run; all named JSON/HTML/image outputs are scratch-only. [C] `capture-web-page` was an unsupported command and printed generic help; it produced no page evidence.

```bash
rea --help
rea capture-browser-scenario --help
rea capture-web-page --help
rea capture-browser-scenario --schema
rea analyze-web-bundle --help
rea capture-browser-scenario '{"url":"https://www.starfall.com/h/"}' --format json
rea capture-browser-scenario '{"browser":{},"start_url":"https://www.starfall.com/h/","actions":[]}' --format json
rea capture-browser-scenario '{"browser":{"mode":"launch"},"start_url":{"url":"https://www.starfall.com/h/"},"actions":[{"type":"wait","ms":1000}]}' --format json
rea capture-browser-scenario '{"browser":{"mode":"launch","executable_path":"/usr/bin/google-chrome"},"start_url":{"url":"https://www.starfall.com/h/"},"actions":[{"action":"wait_for_timeout","timeout_ms":3000}]}' --format json
rea capture-browser-scenario '{"browser":{"mode":"launch","executable_path":"/usr/bin/google-chrome"},"start_url":{"url":"https://www.starfall.com/h/"},"actions":[{"action":"wait_for_timeout","step_id":"loaded","duration_ms":3000}]}' --format json
rea capture-browser-scenario '{"browser":{"mode":"launch","executable_path":"/usr/bin/google-chrome"},"start_url":{"url":"https://www.starfall.com/h/"},"actions":[{"action":"wait_for_timeout","step_id":"loaded","duration_ms":1000}],"capture":{"at_end":["dom","accessibility"],"events":["network"]}}' --format json
rea capture-browser-scenario "$PAPERCLIP_SCRATCH_DIR/reading-index.scenario.json" --format json
rea capture-browser-scenario "$PAPERCLIP_SCRATCH_DIR/kinder-index.scenario.json" --format json
rea capture-browser-scenario "$PAPERCLIP_SCRATCH_DIR/song-index.scenario.json" --format json
rea capture-browser-scenario "$PAPERCLIP_SCRATCH_DIR/phonics.scenario.json" --format json
rea capture-browser-scenario "$PAPERCLIP_SCRATCH_DIR/book.scenario.json" --format json
rea capture-browser-scenario "$PAPERCLIP_SCRATCH_DIR/song.scenario.json" --format json
rea capture-browser-scenario "$PAPERCLIP_SCRATCH_DIR/phonics-play.scenario.json" --format json
rea capture-browser-scenario "$PAPERCLIP_SCRATCH_DIR/phonics-play.scenario.json" --format json
rea capture-browser-scenario "$PAPERCLIP_SCRATCH_DIR/book-play.scenario.json" --format json
rea capture-browser-scenario "$PAPERCLIP_SCRATCH_DIR/song-play.scenario.json" --format json
```

[C] The first four scenario attempts failed input validation. The CLI schema exposed only an `inputJson` string, so validation errors were used to correct nested fields. [C] The first `phonics-play` attempt used `selector` and `position` directly on the action and failed validation (missing `locator`, unknown `selector`/`position`). The second used the valid locator below. [C-code] Inspection of the installed REA click adapter showed it forwards button/click count but not canvas coordinates; this limited the wrong-answer drag probe.

[C] File-based scenarios above used this input shape, substituting the exact URL from the sample-selection table (or the named index):

```json
{
  "browser": {"mode": "launch", "executable_path": "/usr/bin/google-chrome"},
  "start_url": {"url": "https://www.starfall.com/h/ltr-sv-a/maw-at/"},
  "actions": [{"action": "wait_for_timeout", "step_id": "loaded", "duration_ms": 5000}],
  "capture": {
    "at_end": ["dom", "accessibility", "screenshot"],
    "events": ["network"],
    "network": {"response_body": true}
  }
}
```

[C] Successful `*-play` scenarios set `response_body=false`, add `after_each_step: ["screenshot","dom"]` and append these actions (video selector `.vjs-big-play-button`; both Phaser activities selector `canvas`):

```json
[
  {"action":"click","step_id":"play","locator":{"kind":"css","selector":"canvas"}},
  {"action":"wait_for_timeout","step_id":"prompt","duration_ms":10000}
]
```

### REA evidence index and capture boundaries

| Sample / scratch output | Evidence ID | Initial capture duration | Recorded requests | Retained response-body bytes |
|---|---|---:|---:|---:|
| [C] Song `song.json` | `ev_009a5049a20dfbfdee2744ff1875ef3eb5ebc044651df56055948092c7cc9bf6` | 12,925 ms | 16 | 147,358 |
| [C] Phonics `phonics.json` | `ev_4f16b30422a620a39d7bd85e0b4c28803747b4878ca95b5d89ea597f81f09198` | 10,776 ms | 54 | 601,470 |
| [C] Reading `book.json` | `ev_56408f50927e21c81467fab5f323b5efeffd93140b0c3dca528a4043acba8a8d` | 9,735 ms | 71 | 712,262 |
| [C] Song Play `song-play.json` | `ev_3f9061a0f2065ad39b47be79e41e16815603039aea6c298141eb69a4051066fb` | — | — | Not requested |
| [C] Phonics Play `phonics-play.json` | `ev_f6024e3dde3cf19313ced2eddf2308116e69f4f5d46d377dedfdfd2ffad0fe8c` | — | — | Not requested |
| [C] Reading Play `book-play.json` | `ev_992692f37dadb648e5dd166d9383c2c29b29d346b07bad84aa8644ee56ec88e9` | — | — | Not requested |

[C] The initial captures mark completeness `incomplete`, `missing_sections=[events]`. Song had one unfinished request; reading had four; phonics had none, but still reported incomplete events. [C] Body retention has per-body and total budgets: retained bytes above are partial decoded/API response bytes, **not wire transfer totals**. [C] Capture duration includes the scripted wait and artifact work; it is **not time to first interaction**. [H] No full-load or performance conclusion should be drawn from those durations alone.

### Fallback trigger, commands and performance definition

[C] After REA validation failures and incomplete event captures, headless Google Chrome driven by Playwright/CDP was used only to fill coordinate-drag and encoded-byte/readiness gaps. REA remained the primary source for page selection, DOM, screenshots and entry/Play observation. [C] The fallback used the existing installed Playwright package at `/tmp/rea-run/node_modules/playwright/index.mjs`, fresh owned browsers and no persistent profile; it did not control the pre-existing shared browser.

[C] Exact fallback commands (helper sources and logs kept in scratch):

```bash
node "$PAPERCLIP_SCRATCH_DIR/probe.mjs" phonics https://www.starfall.com/h/ltr-sv-a/maw-at/
node "$PAPERCLIP_SCRATCH_DIR/probe.mjs" book https://www.starfall.com/h/ltr-sv-a/zac-the-rat/
node "$PAPERCLIP_SCRATCH_DIR/probe.mjs" song https://www.starfall.com/h/motionsongs/vfs224390218/
node "$PAPERCLIP_SCRATCH_DIR/deep.mjs" phonics https://www.starfall.com/h/ltr-sv-a/maw-at/
node "$PAPERCLIP_SCRATCH_DIR/deep.mjs" book https://www.starfall.com/h/ltr-sv-a/zac-the-rat/
node "$PAPERCLIP_SCRATCH_DIR/deep.mjs" phonics https://www.starfall.com/h/ltr-sv-a/maw-at/ phone
node "$PAPERCLIP_SCRATCH_DIR/deep.mjs" book https://www.starfall.com/h/ltr-sv-a/zac-the-rat/ phone
node "$PAPERCLIP_SCRATCH_DIR/deep.mjs" phonics https://www.starfall.com/h/ltr-sv-a/maw-at/
```

[C] The last phonics run corrected the readiness lookup from the unavailable Phaser game registry to the already-created sound manager's game reference. [C] Failed-registry runs waited through a timeout; their ~76 s result is excluded as a readiness measurement. [C] The earlier generic probe's fixed waits for canvas activities are also excluded; they were not measured readiness.

[C] Method: attach CDP `Network` before navigation; count `requestWillBeSent`, sum `loadingFinished.encodedDataLength`, and use elapsed wall time since navigation start. Requests include failed font attempts and browser-generated data URL requests; bytes include completed responses only, with headers/compression as reported by Chrome. [C] First interaction here means the Play/game state is present, not that all later assets have loaded or audio is decoded. Phaser readiness was checked by state `game`; video readiness by visible Play. [H] These are practical entry-readiness proxies, not standardized TTI/INP benchmarks.

| Sample / window | Requests | Completed encoded bytes | Time to first Play interaction state |
|---|---:|---:|---:|
| [C] Song, desktop at visible Play (`song-probe.log`) | 18 | 1,115,370 | 2,942 ms |
| [C] Phonics, desktop at game state (`phonics-deep2.log`) | 26 | 1,192,722 | 5,628 ms |
| [C] Reading, desktop at game state (`book-deep.log`) | 45 | 1,347,188 | 8,381 ms |
| [C] Reading, portrait touch at game state (`book-phone.log`) | 45 | 1,347,170 | 9,455 ms |
| [C] Phonics, through intro + wrong/right probes | 58 | 1,659,480 | Same run as phonics entry above |
| [C] Reading, through intro + attempted navigation | 118 | 2,895,693 | Same run as reading entry above |

[C] Desktop: 1280×720, Chromium 157.0.8103.38, fresh context, no deliberate throttling. Portrait: 390×844 with touch/mobile emulation. [C] Background loading continues after Play; the entry and later-window rows intentionally have different boundaries. [H] Single runs with concurrent browser work do not establish repeatable mobile performance or a comparative ranking against the pilot's differently measured spinner times.

### Technical anchors and abstract models

| Sample | Evidence / code anchors | Interpretation |
|---|---|---|
| [C] Song | DOM Video.js controls, English `lang`, video duration/currentTime; MP4/poster requests | Semantic transport controls exist; caption usability not established |
| [C-code] Song | `SpecialDefinitions`, `nRateInterval=25`, `greenArrowInterval=10`, empty `captionURL`, `sendNRate`, `showNextButton` | Config-driven media shell; position threshold is not comprehension |
| [C-code] Phonics | `GameConfig.WORDS_DATA`, `answerLetter`, drop rectangle, `Sounds.HONK`, `Sounds.WINK_BELL`, `currentWordCounter`, `reward` | Four answer records, actual comparator, retry distinct from outside drop |
| [C] Phonics | Wrong `b` for target `r`: counter 0/frame 5; right `r`: hidden choice/frame 26 | Runtime confirms branching; sound playback intent remains code-only |
| [C-code] Reading | `sceneCounter`, `gotoNextScene`, `gotoPrevScene`, `goToNextPage`, scene 7 tracking | Eight-scene reading sequence, gated navigation, final-scene participation event |
| [C-code] Both Phaser activities | `loadGroup02Assets`, loading flags, PNG/JSON atlases, separate MP3 audio | Staged loading rather than the pilot's two upfront compositions |

[C] DOM sources identify Phaser CE 2.20.1 for phonics and Phaser 2.10.5 for reading. [C] Both canvas shells disable zoom and lack HTML `lang`; footer links provide accessibility information, with a separate video alternative for the book. [C] Reading portrait canvas measured 390×292 and its rotation element was hidden. [C-code] Phonics contains mobile-orientation/scaling handlers; completed portrait behavior was not established by a valid readiness probe. [H] No WCAG conformance claim or real-device claim is supported by this study.

### Limitations and original engineering direction

[C] Runtime verified entry/Play for all three; one actual wrong/right pair for phonics; and video end UI after seeking. [C-code] Full phonics reward, book scenes beyond the introduction, final exits, and tracking semantics were inspected in retained source only. [C] No human listened to the audio and no child participated. [H] Progress persistence, server aggregation, lesson efficacy, assistive technology and iOS behavior remain unknown.

[H] Use separate original data schemas for media, recognition items and story pages under a common accessible shell. Include explicit attempt outcomes (`cancelled_drop`, `incorrect`, `correct`) and distinguish playback position from learning completion. Lazy-load the next unit, retain a usable current unit, and test original Vietnamese instructions and English target audio with families before treating these patterns as validated product requirements.

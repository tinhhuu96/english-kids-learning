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

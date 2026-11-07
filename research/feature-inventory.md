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

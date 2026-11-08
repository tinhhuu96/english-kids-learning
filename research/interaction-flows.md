# Interaction Flows — Pilot: Starfall "ABCs → Letter A"

**Evidence legend:** **Confirmed** = direct runtime/REA observation or a public page (publisher claims are confirmed only as statements); **Inferred** = conclusion from evidence, including anything read only from static HTML/JS; **Unknown** = not verified. Legacy mapping: `[C]` → Confirmed only for direct evidence; `[C-code]` → Inferred – from static code; `[H]` → Inferred for a reasoned conclusion or proposal, Unknown for an unverified fact.
Content (words, pictures, voice lines) is described generically on purpose; nothing is reproduced.

## Flow A — Entry and start gate

1. [Confirmed] The child taps the letter tile on the ABCs index. The activity page opens in the same tab.
2. [Confirmed] A full-screen loading animation shows until all assets of the first composition are ready (~5 s on desktop broadband).
3. [Confirmed] A start screen shows a large letter block and a big **Play** button. The only other control is a **Close (X)** in the top-right corner.
4. [Confirmed] Tapping Play plays a click sound and starts the lesson.
   This first tap also serves as the user gesture that browsers require before audio can play.

**Pattern:** "one big obvious button" start gate. It is needed for audio autoplay policy, and it also gives the child a clear first action.

## Flow B — Guided lesson (4 pages, linear)

For each page:

1. [Inferred – from static code] An animation introduces the target (for example, the capital and the lowercase letter appear) while narration plays.
2. [Inferred – from static code] Interactive elements are **disabled until the narration or animation finishes**. Then they are switched on (`btnControll(btn, 1)`).
3. [Inferred – from static code] Tapping a letter or a picture makes it grow (scale tween), plays the letter-sound clip, and shrinks it back when the clip ends.
   Then the next part of the page plays (for example a word animation).
4. [Inferred – from static code] When the page's required interaction is complete, a **Next arrow** appears bottom-right.
   It has a **sparkle effect** and an **animated pointing hand** next to it. The hand hint repeats after ~7 s idle, up to 4 times.
5. [Inferred – from static code] Tapping Next zooms a large letter outward with the letter sound, then advances to the next page.
   From page 2 on, a **Back** button is visible.
6. [Inferred – from static code] The last page ends with a short reward animation. Then Next appears and leads into the minigame.

**Key properties (Inferred – from static code for the full four-page flow)**
- Pacing is controlled by the system, not by the child: no skipping ahead, and taps are ignored during narration.
- No text instructions. Everything is cued by voice, motion, the hand pointer and sparkles.
- Every tap produces a sound plus motion, so there are no "dead" taps on active elements.

## Flow C — Practice minigame (drag-to-sort)

1. [Inferred – from static code] 8 draggable letter tiles (4 capital, 4 lowercase) pop in one by one, each with a pop sound.
2. [Inferred – from static code] A spoken direction plays while the matching target glows: first the capital target, then the lowercase target, each with its own voice clip.
3. [Inferred – from static code] An animated hand demonstrates dragging a tile to a target, looping until the child first touches a tile.
4. [Inferred – from static code] The child drags a tile (`mousedown` → `tick` move → `pressup`). Only one tile can be dragged at a time.
5. [Inferred – from static code] **On release, the tile always flies to its correct target**, whether or not it was dropped there.
   Dropping it directly on the correct target also counts.
   - **There is no wrong answer state.** This is *errorless learning*: the child cannot fail, they only practise the motion and hear the letter sound.
6. [Inferred – from static code] Each successful sort fills one slot (1 of 4 apples) on that target and plays the letter sound with a zoom.
7. [Inferred – from static code] After 4 capitals and 4 lowercase tiles, a "great job" animation and voice play, then a celebration.
   A completion ping is sent to the server. Next (with sparkles) appears.
8. [Inferred – from static code] Next goes back to the ABCs index. Back returns to the last lesson page.

## Flow D — Exit

- [Confirmed] Close (X) is visible. [Inferred – from static code] Its `exitURL` points to the ABCs index without a confirmation step; the exit was not exercised in the pilot.
- [Confirmed] The DOM footer offers Back, Play Video (a non-interactive alternative), and Accessibility Information.

## Feedback model summary

All rows below are **Inferred – from static code** for the full pilot flow; the later steps were not exercised at runtime.

| Event | Feedback |
|---|---|
| Tap active element | Scale-up + letter sound + scale-down |
| Step complete | Next button appears with sparkles; pointing-hand hint after idle |
| Correct drag | Tile snaps to target, slot fills, letter sound |
| Incorrect drag | **Not possible**: tile auto-routes to the correct target |
| Activity complete | Animated praise + voice + celebration, then Next |
| Idle | Repeating hand + sparkle hint (bounded to ~4 repeats) |

## Original implementation recommendations (for our product)

1. Keep the **start gate**. Its label should be a Vietnamese voice prompt plus an icon, not text.
2. Use **narration-locks-input** per step, but allow a parent setting or a long-press to replay narration.
   Do not hard-block a child who already knows the content.
3. Show **Next only after completion**, with a motion hint after idle. Use our own mascot hand or character, not a copy.
4. Start new content with **errorless practice**. Then move to **gentle-error** items: a wrong pick wobbles, plays a soft neutral sound, and the correct answer is highlighted after 2 misses.
   Gentle-error checks real recognition, which Starfall's sort does not.
5. Give **bilingual scaffolding**:
   - Instructions in Vietnamese voice-over.
   - Target words and letter sounds in English voice-over.
   - A toggle to fade out the Vietnamese as the child progresses.


## MIN-9 expansion — three activity flows

[Confirmed] URLs and evidence IDs are indexed in `technical-evidence.md`; pilot content above is preserved. [Confirmed] All captures used public pages in fresh, unauthenticated browser contexts.

### S — Wheels on the Bus

1. [Confirmed] Open Motion Songs index → public video link → poster and a large DOM Play button.
2. [Confirmed] Activate Play; the video clock advances and `paused=false`.
3. [Confirmed] The player supplies progress, mute and speed controls; it does not display a scored letter choice.
4. [Confirmed] Seek to two seconds before the end → wait → final still and green arrow remain on the activity URL. This deliberately tests end UI, not full viewing.
5. [Inferred – from static code] At playback position ≥25 s the controller sends its rating signal; within 10 s of the end it reveals the arrow. The arrow and Close return to the configured Motion Songs index.
6. [Inferred] Original flow: Vietnamese voice cue → original short song → replay/exit → optional picture check. Track viewing and correct recognition separately.

### P — Make-a-Word with -at

1. [Confirmed] Public Learn to Read index → Make-a-Word → canvas Play → introduction → picture, word ending and four draggable onsets.
2. [Confirmed] Read-only runtime state in the recorded wrong/right probe showed first target `r`; move `b` from its tile into the initial-letter slot using real mouse down/move/up.
3. [Confirmed] Wrong: tile returns to its original coordinates, remains visible, item counter stays 0, word frame stays 5; the task remains available. [Inferred – from static code] Mismatch invokes the `HONK` sound; no claim about its perceived loudness or tone is made.
4. [Confirmed] Drag `r` to the same slot. Its source tile hides, the initial letter fills in, and the blend animation advances to frame 26 after 1.8 s. [Inferred – from static code] The match branch invokes `WINK_BELL` and locks choices during the animation.
5. [Inferred – from static code] After the blend, increment current item and restore choices; after four items enter reward. Continue on reward exits to the short-A index. Close can exit independently.
6. [Inferred – from static code] An out-of-slot drop simply resets position; it differs from an in-slot wrong-letter answer. [Inferred] Our attempt model should distinguish motor cancellation from recognition error.
7. [Inferred] Original flow: spoken Vietnamese prompt + English target → tap or drag → gentle visual retry → optional hint → success and next item. Avoid a punitive sound and preserve the child's control of replay.

### R — Zac the Rat

1. [Confirmed] Public Learn to Read index → interactive short-A book → canvas Play → animated introduction.
2. [Inferred – from static code] Scene controller has states 0–7; scenes emit signals that reveal Continue/Back, and later asset groups can briefly block progression behind a loader.
3. [Inferred – from static code] Reading content uses word/letter audio and animation rather than an answer comparison. Page continuation is not proof of decoding accuracy.
4. [Inferred – from static code] Continue runs a transition into the next scene; Back decrements the scene index with a lower bound of zero. Final scene entry invokes tracking; continuing after scene 7 exits to the short-A index.
5. [Confirmed] Runtime captured only entry/introduction. A lower-right click was attempted, but a page turn was not established; remaining steps are explicitly code-only.
6. [Confirmed] Portrait emulation shrank the rendered canvas to 390×292; footer links remained separate DOM links. [Inferred] Original flow should keep story text, replay and Back/Next in accessible DOM with large controls even in portrait.

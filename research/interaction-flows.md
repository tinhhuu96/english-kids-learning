# Interaction Flows — Pilot: Starfall "ABCs → Letter A"

Legend: **[C]** = confirmed (code + runtime observation). **[C-code]** = confirmed in controller code but not observed at runtime. **[H]** = hypothesis.
Content (words, pictures, voice lines) is described generically on purpose; nothing is reproduced.

## Flow A — Entry and start gate

1. [C] The child taps the letter tile on the ABCs index. The activity page opens in the same tab.
2. [C] A full-screen loading animation shows until all assets of the first composition are ready (~5 s on desktop broadband).
3. [C] A start screen shows a large letter block and a big **Play** button. The only other control is a **Close (X)** in the top-right corner.
4. [C] Tapping Play plays a click sound and starts the lesson.
   This first tap also serves as the user gesture that browsers require before audio can play.

**Pattern:** "one big obvious button" start gate. It is needed for audio autoplay policy, and it also gives the child a clear first action.

## Flow B — Guided lesson (4 pages, linear)

For each page:

1. [C] An animation introduces the target (for example, the capital and the lowercase letter appear) while narration plays.
2. [C-code] Interactive elements are **disabled until the narration or animation finishes**. Then they are switched on (`btnControll(btn, 1)`).
3. [C-code] Tapping a letter or a picture makes it grow (scale tween), plays the letter-sound clip, and shrinks it back when the clip ends.
   Then the next part of the page plays (for example a word animation).
4. [C] When the page's required interaction is complete, a **Next arrow** appears bottom-right.
   It has a **sparkle effect** and an **animated pointing hand** next to it. The hand hint repeats after ~7 s idle, up to 4 times.
5. [C-code] Tapping Next zooms a large letter outward with the letter sound, then advances to the next page.
   From page 2 on, a **Back** button is visible.
6. [C-code] The last page ends with a short reward animation. Then Next appears and leads into the minigame.

**Key properties**
- Pacing is controlled by the system, not by the child: no skipping ahead, and taps are ignored during narration.
- No text instructions. Everything is cued by voice, motion, the hand pointer and sparkles.
- Every tap produces a sound plus motion, so there are no "dead" taps on active elements.

## Flow C — Practice minigame (drag-to-sort)

1. [C-code] 8 draggable letter tiles (4 capital, 4 lowercase) pop in one by one, each with a pop sound.
2. [C-code] A spoken direction plays while the matching target glows: first the capital target, then the lowercase target, each with its own voice clip.
3. [C-code] An animated hand demonstrates dragging a tile to a target, looping until the child first touches a tile.
4. [C-code] The child drags a tile (`mousedown` → `tick` move → `pressup`). Only one tile can be dragged at a time.
5. [C-code] **On release, the tile always flies to its correct target**, whether or not it was dropped there.
   Dropping it directly on the correct target also counts.
   - **There is no wrong answer state.** This is *errorless learning*: the child cannot fail, they only practise the motion and hear the letter sound.
6. [C-code] Each successful sort fills one slot (1 of 4 apples) on that target and plays the letter sound with a zoom.
7. [C-code] After 4 capitals and 4 lowercase tiles, a "great job" animation and voice play, then a celebration.
   A completion ping is sent to the server. Next (with sparkles) appears.
8. [C-code] Next goes back to the ABCs index. Back returns to the last lesson page.

## Flow D — Exit

- [C] Close (X) is always visible and goes back to the ABCs index (`exitURL`). No confirmation.
- [C] The DOM footer offers Back, Play Video (a non-interactive alternative), and Accessibility Information.

## Feedback model summary

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

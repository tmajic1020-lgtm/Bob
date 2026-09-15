# Iron Front

A single-file browser grand-strategy game. Everything lives in `index.html`
(~1.6MB, two inline `<script>` blocks). There is no build step: open the file.

## Shorthand

- **"E"** from the user means **make better graphics**. Nothing else.

## Working rules that have paid for themselves here

- **Measure before and after, and distrust a result that looks dramatic.**
  Most of the "bugs" found in this file have turned out to be faults in the
  measurement instead. Colour-classified land counted deep ocean as land; a
  border audit reported anything from 32% to 81% depending only on how far
  apart its two samples were taken; the two map images were compared at
  different pixels-per-degree, which alone accounted for a whole texture
  "regression". When a number is startling, vary the one parameter the method
  depends on and see whether the number moves with it.
- **A change that does not measure better gets reverted**, however good the
  argument for it. Several sound-sounding improvements are recorded in the git
  log as reverted for exactly this reason.
- **fps is not comparable across container restarts or between busy and idle
  containers.** Always A/B in the same run: stash, measure, pop, measure.
- **Unused code is not automatically code that should be used.** The
  lightness-ramp LUT was built per nation and never read; wiring it up scored
  worse than the flat paint it was meant to replace.

## Verification

Harnesses live in the session scratchpad, not the repo. The regression suites
are `final.js` (eras x graphics), `final2.js` (map modes and overlays),
`p10c.js` (panels against hostile game states), `save.js`, `repro5.js` and
`p8.js` (leaks). `check100.js` scores a render against a real HOI4 screen
capture on ~75 named checks; `zooms.js` reports frame rate.

## The embedded capture

`MAPPHOTO_SRC` is a screenshot of Paradox's Hearts of Iron IV, used as the map
ground behind a toggle (Settings -> MAP GROUND, or `setPhotoMap(false)`). It is
fine for private use and should not be published. Keep it toggleable.

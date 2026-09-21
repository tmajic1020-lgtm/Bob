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

## What the capture forecloses

`MAPPHOTO_SRC` is a screenshot taken at a FAR zoom, and that is baked in.
Measured against four captures of the real game at three distances, the share
of ground carrying strong colour (saturation > 0.25) falls away hard as you
come in: 77% with a continent in view, 38% at a country, 5% at a few
provinces — where the political colour has stopped being a fill at all and
survives only as a soft band along the frontiers, the ground itself carrying
the picture.

The embedded picture's own pixels measure 61.8% strongly coloured, and a
render of this map at close zoom measures 60.7%. Those are the same number:
the colour you see up close is the capture's own political wash, painted at
far-zoom strength and then magnified. It is not this renderer's wash — fading
that with zoom was tried and moved the figure by one point, because it was
never contributing it.

So the close view cannot be made to look like the real game's close view from
this picture, at any alpha, with any filter. A far-zoom photograph magnified
is a far-zoom photograph. The only route to terrain-up-close with colour only
at the frontiers is ground that is GENERATED at that zoom, with the capture
governing the far view where it is honest.

(One caveat on those numbers: the closest reference is a snow scene, and snow
lowers saturation by itself. The middle one is not, and disagrees with this
map by 25 points, so the trend is real well before the snow gets a say.)

## Where the close-zoom colour actually comes from

Peeled apart at zoom 14, over the same ground, share of pixels above
saturation 0.25:

    as shipped                         59.2%
    with the capture faded out         53.9%
    and the ownership wash off too     45.0%
    the real game                       5.2%

So the capture is worth 5 points and the wash 9, and the remaining 45 is the
GENERATED TERRAIN PALETTE itself — the greens, golds and ochres it paints
ground with. Anything that tries to reach the real game's close view by
adjusting the capture or the wash is arguing over 14 points of a 54-point gap.

Attempts on record, all reverted: fading the wash with zoom (moved 1 point);
a tone curve to match the brighter reference (matched its statistics, looked
like neon); pre-upscaling the capture 2x (worse at both high zooms); a third
detail octave (lost acutance); coloured frontier bands, which are what the
real game's close view actually has (moved close-zoom colour from 59.2 to
73.2 — fourteen points the WRONG way, because here they land on ground that
is already saturated rather than on neutral terrain).

What has worked is additive geometry, every time: railways, rivers brought out
from under the capture, conifers and ridge carets. They do not fight the
picture, they sit on top of it, and they are sharp at any magnification.

CAVEAT ON THE 5.2%: that reference is a snow scene and snow is white, which
costs saturation for free. The un-snowy middle reference gives 38.4% against
this map's 62.9%, and THAT is the honest target. Closing it means desaturating
the terrain palette, not touching the capture.

## The embedded capture

`MAPPHOTO_SRC` is a screenshot of Paradox's Hearts of Iron IV, used as the map
ground behind a toggle (Settings -> MAP GROUND, or `setPhotoMap(false)`). It is
fine for private use and should not be published. Keep it toggleable.

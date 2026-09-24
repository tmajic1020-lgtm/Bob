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
this map's 62.9%, and THAT is the honest target.

AND THE PALETTE IS NOT THE LEVER THIS SUGGESTED. Tried: jungle and tundra
re-set against the bands they actually land in, and the forest mottling raised
to the L=61 this file's own note had already measured off the reference. Total
error across the generated map's checks fell 269.1 to 260.7 -- 3%, real and
reproducible, but it flipped no check and the score stayed 60 of 75.

The dominant term is latitude band 5 (21.7N to 10N), 42 points of that 260 on
its own: the reference reads 143.5 there and this map 101.6. It is not bad
data -- that band holds the Mekong delta as marsh, the Sahel as desert and
Yucatan as jungle, all correct.

## Fitting the palette instead of choosing it

The twelve latitude bands are twelve equations in ten terrain colours. Measure
what fraction of each band each terrain covers BY AREA (through provPix, not by
counting provinces), weight each band by its pixel count, and solve. Worth
knowing before guessing again:

- The model explains what this map renders to within 4.5 luminance, so band
  brightness really is just area-weighted paint. The composition table looks
  self-contradictory -- plain is 0.59 of a band that needs to come DOWN 33 and
  0.35 of one that needs to go UP 20 -- but the solve handles that; eyeballing
  it does not.
- Effective on-screen forest was luminance 8.7 against a paint value of 76.
  The tree mottling, not the palette entry, is what sets a woodland band.
- Best achievable by any palette was 7.8 RMS against the 15.8 it sat at. A
  half-step in log space toward the solution, clamped to 45%, took the
  generated map 60 -> 61 and total error 269 -> 228.
- That brightened open ground under an unchanged dark mottle, so texture and
  local contrast OVERSHOT (p90 66.4 against 50.5) having been too smooth
  before. Raising the mottle to match fixed both and lifted the woodland
  bands: 61 -> 62, error 228 -> 196.
- A SECOND fitted step made it worse, 62 -> 61 and error 196 -> 220, and was
  reverted. The fit only models band luminance; it is blind to the saturation,
  warmth and texture checks the same change moves. One step, then stop.

## The embedded capture

`MAPPHOTO_SRC` is a screenshot of Paradox's Hearts of Iron IV, used as the map
ground behind a toggle (Settings -> MAP GROUND, or `setPhotoMap(false)`). It is
fine for private use and should not be published. Keep it toggleable.

## The standalone staff map

`staff-map.html` is a separate single file: a HOI4-styled 1936 theatre map
with a top status bar, counters and a clock. It shares nothing with
`index.html` and embeds no capture, so it is safe to publish.

Its first version drew Europe from hand-written polygons, about thirty
vertices a country, and was rightly called the worst thing in the repo.
The fix was not better hand-drawing:

- **The coastline data was already here.** `atlas50.json` in the scratchpad
  (the same encoding `_decRing` reads in `index.html`) holds 227 real
  country boundaries. Clipped to a Europe box and re-encoded at Q=24 --
  1/24 degree, about 3km, finer than a 1500px map resolves -- 51 countries
  and 7,833 vertices cost 17KB inline. Coastlines have not moved since
  1936, so this part is simply right rather than approximate.
- **1936 politics is a separate layer, clipped to that ground.** Most of
  the theatre is an exact union of modern states (Czechoslovakia = Czechia
  + Slovakia, Yugoslavia = the six republics + Kosovo, Romania + Moldova
  for Bessarabia). Only the borders that genuinely moved are hand-drawn --
  Germany's eastern provinces, the Kresy, Vilnius, Ruthenia, Bukovina --
  and those are all INLAND, the one place a few kilometres of error cannot
  be seen. A coast is where everybody can see it.
- **Stroking a multi-ring country strokes its internal edges too**, which
  ruled a line down the middle of Czechoslovakia at full national weight.
  Stroke at twice the width then fill the same path back over it: an
  internal edge is covered from both sides, the true boundary keeps its
  outward half, and what is left is the union outline.
- **A clip region is not a border.** The regions are closed polygons but
  only their leading `nb` points are a real frontier; the rest closes the
  shape out at sea. Stroking the whole polygon drew straight lines across
  the middle of Germany and out into the Baltic.

Measurement notes, in the spirit of the rest of this file:

- Classifying a painted pixel by nearest national colour DOES NOT WORK once
  the gradient and the dark veil are on it -- it put Warsaw in Lithuania and
  Minsk in Hungary. Repaint each nation a flat index colour and read the
  pixel exactly. Empty `CITIES` and `COUNTERS` first or the probe lands on a
  marker and reads the marker.
- Two "bugs" here were faults in the probe: Copenhagen "outside Denmark"
  was the probe asking at 12.6 when the coastline is at 12.583, and Danzig
  "floating in the bay" was me misreading a screenshot by 50 pixels. Both
  disagreed with a numeric check, and the numeric check was right both times.
- Audit city positions against the ring data rather than by eye. Of 46
  cities exactly one (Lisbon) was in the water, by 4km.

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

## The embedded capture is GONE

`MAPPHOTO_SRC` held a 215KB base64 screen capture of Paradox's Hearts of Iron
IV, used as the map ground. It has been DELETED at the owner's request, along
with the ~12KB decode-and-sharpen pipeline that existed only to process it.
Do not reintroduce it.

Nothing downstream broke, because every consumer already had to cope with the
picture failing to decode: `photoMapOn()` is `PHOTOMAP && !!MAPPHOTO`,
`MAPPHOTO` now stays null, and every branch that asked for the picture takes
the generated-terrain fallback it already had. Measured before and after the
deletion, the sea and the ground are pixel-identical to the pre-existing
"capture off" path, and `check100.js` still scores 62 of 75 -- the same as
before -- which is its own confirmation of what this file already recorded:
the capture was worth about five points of colour and the generated terrain
was carrying the picture.

The file went from 1,702,597 to 1,471,840 bytes.

CAVEAT: deleting it from the working file does not remove it from this
repository's git history. Earlier commits still carry it. If the point is to
be able to publish the repo, the history has to be rewritten separately.

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

## The staff map is terrain, not fill

Called out as still bad after the coastlines went in, and measuring it
against a capture of the real game at the same zoom said exactly why:

                      real game     flat-fill version
    land lightness      50 - 62         20 - 26
    internal detail   0.14 - 0.24     0.03 - 0.07
    sea saturation       0.34            0.67

Half the brightness, a third of the detail, a sea twice as blue as the
thing it was imitating. The real map is a TEXTURED TERRAIN MAP with a
pale political wash over it and a mesh of provinces ruled across it;
the political colour is a tint on ground, not the ground. What fixed
it, in order of how much each was worth:

- **A generated terrain ground**, baked from fBm and drawn under
  everything, with the nation colour dropped from an opaque gradient
  plus a dark veil to a wash at 0.44 alpha pushed 30% toward white.
  Land went 20-26 -> 45-49 lightness and 0.12-0.59 -> 0.18-0.20
  saturation.
- **A province mesh**: a jittered lattice, fixed in degrees, with about
  a fifth of its edges dropped by hash so cells merge into irregular
  ones. A complete lattice reads as graph paper. This is most of the
  detail figure: 0.03-0.07 -> 0.12-0.16.
- **Real mountain ranges.** Noise alone puts ridges nowhere in
  particular and the eye knows. Sixteen ranges as polylines, rasterised
  once through canvas into a coarse blurred field and sampled as an
  elevation bonus. Doing it as distance-to-segment per pixel would have
  been tens of millions of tests per bake.
- A lighter, greyer sea: 0.67 -> ~0.41 saturation against a target 0.34.

Reverted, each for failing to measure better:

- **Raising the fBm persistence** to get more detail when zoomed in.
  Moved 96 pixels of 988,000: over a hundredth of a degree the whole
  field varies by 0.005, so the terrain thresholds never notice.
- **Colour mottling at a zoom-tied frequency**, the same idea again.
  Measured very slightly WORSE on every figure it touched.
- **Baking terrain at half screen resolution when zoomed in** instead
  of a third. Changed no measured figure.

All three were chasing "the resolution fades when you zoom in", and the
premise was wrong. The real game's own close view measures 0.008-0.027
edge density and this map's close view was already 0.017, inside the
range, before any of it. ESTABLISH THE TARGET BEFORE OPTIMISING: two of
the three would never have been written.

More instrument faults, adding to the tally:

- **Frame timings in this container are worthless.** A bisection said
  baseline 6.5ms, then that REMOVING the province mesh cost 120ms --
  a physical impossibility. Repeated runs disagreed by 10x. The
  trustworthy measure was a counter, not a timer: terrain bakes during
  a pan and a zoom, which went 40 -> 0 and stayed there.
- **Caching on the camera needs like compared with like.** Keying the
  terrain cache on exact span re-baked every wheel step; the fix
  compared screen pixels-per-degree against BAKE pixels-per-degree,
  which differ by the subsampling factor, so the ratio was a constant 3
  and it re-baked every frame -- slower than what it replaced.
- The flat-index-colour ownership probe must also disable the province
  mesh and the country labels, or it reads those instead of the fill.

## Open data beats invented data

Asked whether the terrain could come off the internet. The answer splits
in two, and both halves matter:

- **Frames from videos and screenshots of the real game are still its
  art**, just laundered through another source, and worse quality for
  the trouble -- compressed, with UI over everything. Not a route.
- **Natural Earth is public domain and reachable**, and it is where the
  coastlines in this repo came from in the first place. Anything it
  holds is free to use and better than a guess.

The environment's network policy blocks naciscdn.org, NASA NEO and
NOAA's ETOPO endpoints, but raw.githubusercontent.com works, and the
whole Natural Earth vector set is mirrored at `nvkelso/natural-earth-vector`.
Fetched, clipped to the theatre and re-encoded the same way as the
coastlines:

- `ne_10m_rivers_europe` + the world layer for Turkey and the Levant:
  1,084 rivers, 30k points, carrying Natural Earth's scalerank so the
  Danube draws at every zoom and tributaries only when close.
- `ne_10m_lakes`: 118 lakes.
- `ne_10m_populated_places_simple`: 190 cities with real positions and
  populations, replacing about forty typed from memory.

Worth knowing for next time:

- **A present-day dataset on a 1936 map is an anachronism generator.**
  Natural Earth flags today's capitals, so Zagreb, Bratislava,
  Ljubljana, Sarajevo, Skopje, Pristina, Kiev, Minsk and Kishinev all
  arrived wearing a capital's star when every one was a provincial city
  of Yugoslavia, Czechoslovakia, the USSR or Romania. The star is now
  granted only to capitals in this map's own 1936 nation table. Names
  needed the same treatment in both directions: Koenigsberg, Danzig,
  Breslau, Stettin, Lwow, Memel, Allenstein and Wilno restored, and
  Dnipro, Kharkiv, Donetsk and Mykolaiv put back to their 1936 forms.
- **A population filter throws away exactly what a 1936 map needs.**
  Koenigsberg and Memel are small towns now. A keep-list fixed it --
  and then the 190-city cap silently dropped them anyway, because the
  sort ran before the cap. Selection must be keep-aware and the drawing
  order must be importance-first, or Wilno wins a label collision
  against Paris.
- **190 labels collide where 40 did not.** The first render produced
  "Viennaburg" and "Luxembourng" out of overlapping names. Labels now
  reserve a box and a later one is dropped.
- **The city-on-land audit cannot resolve better than the coastline.**
  It now flags six ports -- Lisbon, Helsinki, Istanbul, Genoa, Mersin,
  Latakia -- each at exactly 4km, against a coastline quantised to 3km.
  That is the instrument hitting its floor, not bad data; ports sit on
  the water's edge. Do not "fix" these by nudging them inland.

## The borders

Called out as bad, and there were three separate faults stacked on
each other.

**1. The geometry was too coarse.** The atlas was Natural Earth 50m on a
3km grid, so the Bohemian frontier -- which winds through the Ore
Mountains -- was about twenty straight segments with visible corners.
Replaced with `ne_10m_admin_0_countries` at ~600m: 31,690 points, 70KB,
facets under 2px even at maximum zoom.

**2. Douglas-Peucker broke the shared borders.** Simplifying each
country independently picks a DIFFERENT subset of vertices for each side
of a shared border, so Poland's edge and Russia's edge no longer
coincided and a sliver opened between them that belonged to neither.
Natural Earth's raw polygons DO share vertices exactly -- all 9 along
the Kaliningrad border -- so the fix is simplification that is a pure
function of local geometry:

- snap to the encoding grid (same coordinate in, same coordinate out);
- drop vertices collinear with their two neighbours, marking in one pass
  and removing afterwards, and testing by perpendicular distance, which
  is unchanged when the ring is traversed the other way. Both sides of a
  shared border then reach the same verdict and the border stays shared.

Same 70KB as the Douglas-Peucker version, and topologically sound.

**3. A country in two entries doubled its ring and flipped the clip.**
Germany holds parts of Poland under two entries, Silesia and East
Prussia, so the nation's ring list held Poland's rings TWICE. Under the
even-odd rule a doubled ring flips parity: Poland counted as INSIDE the
outward clip rather than outside it, so the frontier stroke along the
Kaliningrad-Poland border -- an internal edge that should have been
discarded -- survived as a full-weight line ruled across the middle of
German East Prussia. Deduplicate by country when building the list.

Also: the six hand-placed 1936 frontiers were seven-point straight lines
sitting next to 32,000 points of real coastline, which looks exactly as
wrong as it sounds. They are now splined and given a tapered fractal
wander, pinned to their placed endpoints. That displacement is a
STYLISATION, not data -- what it claims is only that the border was
irregular, which is true of every border and truer than a ruler.

### Four detectors in a row were wrong, and how the fifth worked

This hunt cost more in bad measurement than in code:

- **"Longest dark run along a screen row"** found 64px and said the line
  was not there. The line SLOPES, so it never occupies one row for long.
- **"Fraction of samples with a dark pixel within 4px"** read 100% --
  and would have read 100% anywhere, because rivers, province lines and
  terrain all put something dark within 4px. No control was run.
- **"Remove one political entry and re-measure"** showed nothing,
  because removing an entry also removes its FILL, which changes the
  contrast the detector was reading.
- Reasoning about the clip from first principles was wrong twice.

What worked, and is worth reaching for first next time:

- **Sample along the suspected feature and compare against two parallel
  control lines offset either side.** A real line is a dip against its
  own neighbourhood; an absolute threshold cannot tell a border from a
  river. This turned 98 into -1 and made the fix verifiable.
- **Colour-code the drawing phases.** Patching
  `CanvasRenderingContext2D.prototype.stroke` to recolour by strokeStyle
  and lineWidth named the guilty pass in one run, after four indirect
  experiments had failed.
- **Test the clip directly**: clip, fill the whole canvas red, then
  sample. That showed paint coming through on the Polish side and
  pointed straight at the parity bug.

## Engine core: states, supply hubs, division templates

Three layers added as foundations, in `index.html` just above `UTYPE`.

### The language and layout decision, with the number behind it

Asked which of Python, Java or C# suits a grand strategy engine. For
THIS engine the answer is none of them: it is 17,000 lines of
JavaScript in one file that runs by opening it, and that property is
worth more than a faster inner loop unless the tick is the constraint.
It is not:

    120 ticks   6.34 ms/tick    158 ticks/sec
    365 ticks   6.52 ms/tick    153 ticks/sec
    (251 provinces, 145 nations, 338 divisions, 54 fleets)

Roughly two orders of magnitude of headroom against the fastest speed a
player can pick. Per-tick cost also barely moved while divisions went
150 -> 338, so the tick is dominated by per-province and per-nation
work, not per-entity.

`tickProfile(days)` reports the per-phase breakdown. It found something
worth knowing: the NAMED phases together are about a fifth of a tick
(supplyTick 12%, everything else under 2% each). The rest is in tick()'s
own per-province loops. So a struct-of-arrays rewrite over divisions
would be optimising ~2% of the frame. The trigger to revisit the layout
is written into the code: a tick over 40ms, or the map past ~4,000
provinces.

### What was actually missing

- **States.** Factories and conscription are regional; provinces are the
  wrong unit for them. Derived at world build by deterministic BFS over
  provinces sharing a home owner, so no hand-authored table has to be
  redrawn for each of the nine eras. 103 states over 251 provinces.
- **Supply hub throughput.** `computeSupply` already picked depots and
  flood-filled distance from them, but a hub fed any number of divisions
  equally well. Hubs are now records with a capacity, the fill carries
  which hub feeds each province, and strain degrades supply.
- **Division templates.** The real gap. Infantry and armour were
  LITERALLY IDENTICAL -- both attack 1.00, defence 1.00 -- so armour
  differed by its icon. A division is now a composition of battalions
  and manpower, equipment draw, supply, speed, width, organisation,
  soft/hard attack, armour and piercing all derive from it.

### Migration by parity assertion, not by hope

The four stock templates are PURE (9 infantry, 9 medium tanks, and so
on) precisely so their averaged soft attack and defence reproduce the
old UTYPE numbers exactly. `assertStockParity()` checks it and returns
the failures, so a future edit to the battalion table that would
silently rebalance every existing army fails loudly. `u.tpl` is optional
throughout and `tplOf()` falls back to the stock template for `u.type`,
so old saves keep loading.

### Two tuning faults the measurements caught

- **Hub strain was binary.** `1/over^2` looked like a curve and measured
  as a switch: stacking divisions went x1.00 straight to the x0.35 floor
  between ten and twenty-five of them. `1/over` degrades properly
  (x1.00 -> x0.46 -> x0.35).
- **Armour came out 16.7x infantry.** The stock armour template is pure
  tanks, so hardness was 1.00 and infantry could only ever use its hard
  attack. No division is entirely tanks -- HOI4's own sit near 0.8 -- so
  hardness is capped at 0.85 at the point of use, the unpierced penalty
  softened 0.5 -> 0.65, and infantry given organic anti-tank
  (hard 0.12 -> 0.28). That lands at 3.97x, which is what a panzer
  division attacking plain infantry should look like.

### The balance change, measured

Two years simulated from the same start with the same seeded dice, the
only difference being `divCombat`:

    legacy flat atk/def   units 366  armoured 20  battles 176  killed  91
    armour + piercing     units 375  armoured 25  battles 262  killed 110

The world does not run away: the largest holdings are broadly the same
powers. Combat is more decisive (battles +49%, casualties +21%) and more
armour gets built, which is the point -- it is now worth building.
Regression suites all pass: `final.js` 14 era x graphics combinations,
`final2.js` every map mode and overlay, `p10c.js` all 13 panels against
hostile states, `save.js` differing only by pre-existing float rounding.


## Making the sea like the real game's

Asked to make the water match the reference. It was already close on spot
samples, so the honest test was the DISTRIBUTION of every sea pixel against
the same statistic on a lit capture:

                        before        reference (Z_mid)
    median luminance      27.4              30.7
    25th percentile       23.3              27.4
    median saturation     0.46              0.323

Darker and half again as blue. Two changes, both aimed at a measured gap:

- `THEMES.dark.sea` #30465d -> #3e515e: L 26.3 -> 30.5, S 0.484 -> 0.34.
- The abyss offset -25,-32,-31 -> -14,-18,-17. The deep had been darkened on
  an earlier bucket comparison and, on a fuller one, overshot.

After, measured over genuinely open ocean (mid-Pacific, Indian, South
Pacific) so it is like for like with the reference's open-water crop:
median 31.7-32.8 against 30.7, saturation 0.33 against 0.323, quartiles
within a point. `check100.js` holds at 62 of 75 and NOT ONE SEA CHECK FAILS
-- all thirteen remaining failures are land: latitude-band luminance,
warmth and texture.

### Two more instrument faults, both caught before acting on them

- **`landPath` is in MAP space, not screen space.** A "distance to land" test
  that called `isPointInPath` with screen coordinates was meaningless, so the
  first pass at classifying dark pixels as open water was worthless.
- **A world view is not an open-water crop.** Sampling the whole map put 28%
  of "sea" below luminance 24 against the reference's 2% -- but the world
  view is full of coastline, and the offending pixels measured rgb(13,28,38),
  which is exactly `THEMES.dark.coast`. It was coastline, not water. This is
  the same class of error as comparing two map images at different
  pixels-per-degree: fix the framing before believing the number.

## The rivers were invented; now they are real

The map's rivers were derived by flow accumulation over the GENERATED
heightfield -- every cell sends its water to its lowest neighbour, count
what passes through, call the busy ones rivers. The comment above that
code claimed the Nile, the Amazon, the Volga and the Mississippi "turn up
because the land makes them turn up". They did not: the heightfield is
noise, so what turned up were plausible rivers in the wrong places. The
province-level defence bonus they fed was therefore also wrong -- the
Rhine and the Vistula were not lines an army had to force.

Replaced with Natural Earth's, the same public-domain source as the
coastlines: scalerank <= 6 for the network, 0-5 drawn as major, 1,031
rivers and 453 lakes for 112KB, simplified to 0.1 degrees, which on a
2000-pixel world is half a pixel. The lakes it replaces were TWO
hand-drawn rings -- the Caspian, and one labelled "Great Lakes hint".

Verified by naming the rivers a player would notice rather than by
counting: Nile/Cairo, Volga/Moscow and Stalingrad, Dnieper/Kiev,
Danube/Vienna and Budapest and Belgrade, Tigris/Baghdad, Yangtze/
Shanghai, Ganges/Calcutta, Amazon/Manaus, Elbe/Hamburg, Vistula/Warsaw.
All fourteen correct.

Three faults found on the way, each by a check rather than by eye:

- **The ownership raster mis-attributed provinces.** The Danube passes
  0.2 world units from Vienna's own centre and Vienna still came back
  with no river, while Budapest and Belgrade on the same river were
  right. Dropped for nearest-province-centre, which is how this map
  identifies a place everywhere else: provinces are points, and with 251
  of them over a 2000x1000 world the mean spacing is 89 units, so 45 is
  the radius at which a river runs THROUGH somewhere rather than past it.
- **The major/minor line was in the wrong place.** At scalerank <= 4 the
  Vistula and the Elbe were excluded -- both are rank 5 -- which is
  exactly the rivers that matter militarily in Poland and Germany.
  Rank <= 5 fixes it, and takes major rivers from 472 to 637.
- **The old divisor turned a gradient into a switch.** Carried over as 3
  from the flow raster, where one cell was one step of the walk, it
  saturated instantly against real polylines: 145 of 150 river provinces
  at exactly 1.0, so 58% of the world drew the full +22% defence. Cells
  of major river per province actually measure min 1, p25 11, median 25,
  p75 47, p95 101, max 189. Sweeping the divisor: 3 -> 145 at full,
  10 -> 118, 20 -> 87, 25 -> 81, 35 -> 60 with the most even spread,
  60 -> 27 but two thirds barely counting. 35 sits between the median and
  the upper quartile, so a river has to run the length of a province to
  earn the full bonus and one that clips a corner still counts.

A note on sequencing: the geometry predicted the answer before the
measurement could be run (cells are 0.36 degrees, polylines simplified to
0.1, so a full crossing is 20-45 cells) and the measured median was 25.
That is the right way round -- the estimate was written down as
provisional and then checked, not asserted.

## The Time Machine

Settings -> TIME MACHINE recreates an earlier version by gating every
feature that came after it (`epochAtLeast(id)`), rather than shipping old
files. It had stopped at v5.3, still labelled "Current", while the game
went on to v23 -- and since every gate compared against an epoch no later
than 5.3, NOTHING from eighteen major versions was gated. "v4.0 -- The
Original Build" had weather, supply, trenches, four hull types, manpower,
the resource economy and armour.

Now 21 epochs, current = 24.0. Each added epoch gates what it introduced:

    5.5  peace conferences       openPeaceConf + the NEGOTIATE button
    5.8  Cold War, Civil War     era picker; setEpoch moves you to ww2
    9.0  artillery, anti-air     addUnit raises them as infantry; buttons hidden
    13.0 new nations rise        the riseNewNation roll
    16.0 stand in the line       setFPV (was gated to 5.0, the 3D view)
    19.0 ground/weather/supply   weatherAt -> clear, terrain defence 1,
                                 supplyTick leaves G.supply null, supMul 1
    20.0 width of a front        frontLine puts everyone in the front
    21.0 manpower                spendMP always succeeds
    22.0 resource economy        haveRes/spendRes true, eqpMul/steelMul 1,
                                 equipTick skipped
    23.0 real country colours    PRE23_COL restored in applyEra
    24.0 this build              divCombat -> roleMul, hubStrain 1,
                                 chargeHubs skipped

Things worth knowing before adding the next one:

- **`epochIdx()` maps an UNKNOWN id to the LAST epoch.** A gate written
  against a version that is not in EPOCHS silently reads as "current build
  only" rather than failing. Every gate must name a real epoch id.
- **The default epoch is stored, so moving "current" strands people.**
  `EPOCH` defaulted to "5.3" and was saved to localStorage the first time
  anyone touched Settings, so nearly every player had "5.3" stored. Adding
  newer epochs would have silently dropped them all into v5.3. A stored
  "5.3" without the `if_epoch_v24` marker is read as current once and
  rewritten; a deliberate 5.3 chosen afterwards is kept. EPOCH_CURRENT is
  now derived from the list, so next time only the marker needs bumping.
- **Test a gate on BOTH sides, and choose a probe that can tell.** Three of
  twelve first read "no change" and all three were the probe: armour
  attacking infantry is unchanged BY DESIGN (the stock templates keep parity),
  a unit in its own capital is fully supplied with or without a supply
  system, and resources live in `n.stock`, not `n.res`. The discriminating
  probes -- infantry attacking armour, whether a supply map exists, emptying
  the right store -- all flipped.
- **Gates remove systems, so play the old epochs, not just the new one.** A
  full war year at v4.0, 5.3, 9.0, 16.0, 19.0, 21.0, 23.0 and 24.0 runs with
  no page errors, and reads like the history: arm/inf only until 9.0, no
  supply map until 19.0, armies shrinking 439 -> 383 -> 312 as manpower and
  resources arrive.
- The rivers are NOT gated: the flow-accumulation code they replaced has
  been deleted, so there is nothing older to fall back to.

Build is now 24.0.0 (BUILD const and both .buildno spans), with a v24.0
changelog entry covering the Time Machine and this build's other work.

## Training

Experience was only ever earned in combat (+1 xp a round), so every
division arrived at zero and a country at peace could do nothing to
prepare. Training is a standing order: ARMY -> TRAINING, with TRAIN ALL
IDLE, STOP ALL, per army group, and "train the N at <selected province>".

- `trainTick()` runs after `battles()` and before `equipTick()`, so a
  division that entered battle today stops drilling today, and what the
  exercises wore out is drawn back from the stockpile the same day.
- +0.25 xp a day to a cap of 30 (TRAIN_CAP): past the first star at 15,
  short of seasoned at 45. Seasoned and elite remain combat-only.
- Costs 0.4% of a division's equipment a day, refilled from the national
  pool. A/B from the same seed with a 500-per-kind stockpile, 60 days:
  425.3 left without training, 377.3 with -- the cost is real, and it is
  the whole decision: a trained army or the reserve you go to war with.
- Marching, battle or leaving your own territory ends the order; being cut
  off only pauses it. Saved as unit field a[13]; a[14] now carries `u.tpl`
  too, which was not being saved at all (harmless until now, because
  nothing assigns a design template yet, but a latent loss).
- Gated at the v24.0 epoch like the rest of this build. Player-only: the AI
  does not train yet, which is a player advantage worth knowing about.

Faults caught on the way:

- **Drill must only ever add.** `xp = min(cap, xp + rate)` would pull a
  veteran who somehow held a training order DOWN to 30. A division already
  at or above the cap now simply stops training.
- **Two probes were wrong again.** "Any division over 30 without combat"
  counted divisions that trained and then went to war -- a war started
  inside the test window -- so it read true; tracking xp across trainTick
  itself found zero drill-driven overshoots. And the first equipment probe
  read Germany's stockpile at 0 before and after, which cannot show a cost;
  seeding the pool and A/B-ing did.
- **A Playwright click on this panel needs `openPanel("army")` and a paused
  clock.** Setting `curPanel` alone leaves the panel hidden, and an
  unpaused game re-renders it every tick, detaching the button mid-click.

## Briefings and credits

A campaign opens on a full-screen card: a line from a leader of the era
over a photograph, the clock stopped until the player continues (click,
any key, or 11 seconds; arrows cycle quotes). Settings -> BRIEFINGS turns
it off, shows one on demand, and takes the player's own photographs.
Settings -> CREDITS lists what the game is built from.

- **Every quote is tied to an occasion and a date** (`LEADER_QUOTES`).
  War quotations are some of the most misattributed lines there are --
  "an army marches on its stomach", "war is hell", nearly everything
  pinned on Genghis Khan -- so none of those are in. An era with nothing
  that meets the bar (the Mongols, the modern day) borrows from the
  others. No lines from the dictators: a loading screen is the wrong
  frame for a slogan.
- **Photographs: public domain or the player's own, nothing else.**
  `PD_PHOTOS` is for US federal works (Army, Navy, Coast Guard, White
  House) and expired-copyright photographs, each with its source and
  licence -- famous press photographs such as the Iwo Jima flag-raising
  (AP) are NOT public domain. It is EMPTY because this environment's
  network policy refuses commons.wikimedia.org, upload.wikimedia.org,
  www.loc.gov and catalog.archives.gov (403 at the proxy). Open those in
  the environment's Network access settings and the list can be filled.
- **Player photos** are downscaled to 1600px JPEG in the browser and kept
  in localStorage (`if_brief_photos`), newest first if the quota bites.
  They never leave the machine, so a private screenshot is fine there in
  a way it would not be in the shipped file.
- **Credits are generated from the same tables** -- the speakers from
  LEADER_QUOTES, the photographs from PD_PHOTOS -- so nothing can be shown
  that is not credited. They name Natural Earth with its requested line,
  TopoJSON (ISC) and the Firebase SDK (Apache 2.0), say plainly that the
  81 national colours come from Hearts of Iron IV's colour table, and that
  the game is inspired by it and not affiliated.
- Harnesses are unaffected: the card is DOM over the canvas, so
  `check100.js` (which reads pixels) still scores 62, and every suite
  passes. Multiplayer guests do not get it; the host runs the clock.

### Painted backdrops and the wider quote list

Asked to "search up images for the eras". Searched: every photo archive
is blocked from this environment (Wikimedia Commons and upload, LoC,
NARA's catalog, NASA, the Met, the Art Institute, Cleveland, SMK,
Rijksmuseum, Europeana, archive.org, Flickr). github.com clones DO work,
but no repository found held archive photographs with provenance good
enough to credit, and an uncredited photo cannot go in by this file's own
rule. So the briefing now paints its own:

- `paintScene(era,W,H,seed,lift)` draws an original silhouette scene per
  era -- sky, sun and halo, cloud banks, ridgelines into haze, smoke,
  then the era's figures from a small kit (`SceneKit`: man with seven kinds
  of headgear, horse and rider, tank, four aircraft, field gun, ship of the
  line, yurt, missile, radar, skyline, searchlights, wire, dead trees).
  The dice place everything, so no two are alike; arrowing to the next
  quote paints a new one. Player photos still take precedence.
- `lift` raises horizon and figures: at the default the action sat exactly
  where the caption goes and the caption's dark overlay hid it. Painted
  scenes also get a lighter overlay than photographs do.
- **Check helper names against the whole file.** The first cut defined
  `_mix`, which already exists at ~12940 with a different signature (it
  accepts rgb() strings). Function declarations hoist and the later one
  wins, so it silently replaced the map's colour mixer. Renamed to
  `_scMix`/`_scRgba`/`_scRng`/`SceneKit`. grep `function NAME` before adding
  any short helper.

Quotes went 22 -> 85, and no longer only leaders, as asked: Austen,
Douglass, Darwin, Dickinson, Cavell, Owen, Sassoon, Lou Gehrig, Einstein,
Murrow, Anne Frank, Salk, Gagarin, King, Ali, Armstrong, Sagan, Malala and
others. Same bar as before: each names its occasion and date, translations
name the translator, and recollections say so. The Mongol era, which had
none, now has four from its own century (Magna Carta, the Novgorod
Chronicle on the Mongols' arrival, Dante, Marco Polo) -- still nothing
pinned on Genghis Khan, and still no dictators.

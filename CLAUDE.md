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

### Thirty scenes, and the archives still shut

Asked again for real photographs. Re-tested: every archive and stock host
is still refused (also NPS, defense.gov, DVIDS, army/navy/af.mil, nist.gov,
picryl, rawpixel, Unsplash, Pexels, Pixabay, Kaggle, Zenodo, Hugging Face).
github.com clones, bitbucket, gitlab's API, S3 and GCS answer, but GitHub's
code search is refused (sessions are bound to their own repos), and web
searches turn up no repository holding archive photographs with provenance.

So `SCENE_ART[era]` is now a LIST of compositions, 4-6 per era, 30 in all,
and `paintScene(era,W,H,seed,lift,pick)` picks one. A composition may set
`mood` (night, snow, dawn, desert, storm -- palette overrides in
`SCENE_MOOD`, with stars and a moon at night, falling snow, rain) and `sea`
(open water from the horizon; `beach` adds a strip of shore in front).
New kit pieces: landing craft, beach obstacles, carrier, dreadnought,
ironclad, city walls with towers, trebuchet, the Berlin Wall and a
watchtower, palms and jungle, parachutes, Dakotas, trucks, an observation
balloon, campfires. Each paints in 2-9ms at 800x450.

Two faults found by looking at the renders, not by any check:
- a composition runs AFTER the foreground is drawn, so a backdrop element
  (the jungle wall) must not fill downwards or it paints over the ground
  the figures stand on;
- light shafts belong to a clear sun; in snow, storm or at night they read
  as searchlights from nowhere, so they are off for those moods.

## Releasable nations (WWII) and music

The owner sent a list of nations with colours, separatists included, and
said they are "for breakaway nations like in the actual game". So in the
WWII era (epoch 24.0) `RELEASABLES` holds the fifteen on the list that had
no state in 1936 AND have a province on this map: Scotland, Northern
Ireland, Catalonia, Basque Country, Portugal (Iberia is one Spanish block
here), Ukraine, Belarus, Slovakia, Moldova, Quebec, Kurdistan, Iraq
(Persian on this map), Pakistan, India, Korea. The rest of the list
(Belgium, Netherlands, Luxembourg, Kosovo, Transnistria, Syria,
Afghanistan, Kashmir, Taiwan, Tibet, Bougainville) has no province to stand
on in 251, and is recorded as such in the code comment.

- RELEASE: WARS panel lists any releasable with a core you hold; it becomes
  your puppet and ally with its cores as home ground.
- BREAK AWAY: `breakawayTick()` every 11 days -- a core whose holder is at
  war or under 45 stability, garrison <= 3, no holder division on it.
  Separatists 0.15%, others 0.10% a check, doubled when shaky or under
  foreign occupation. One a check at most.
- Keys are `S_r<ID>` so `clearShards()` removes them at the next new game
  like risen nations, and a save rebuilds them from name/flag/colour the
  way it already rebuilds risen ones.
- The existing nations' colours were NOT changed to the list's: they are
  the HOI4 colour table the owner asked for earlier. The list's colours are
  used for the releasables.

Music (`MUS_SLOTS`, nine moods: menu + allies/axis/comintern/neutral x
peace/war, side from ideology) -- the owner sent the HOI4 soundtrack on
YouTube. That is Paradox's and is not shipped, same rule as the art. Each
slot plays the player's own files from IndexedDB (`ironfront_music`), and
"add a whole soundtrack" sorts files by name; an empty slot plays an
original WebAudio score (`SCORE`, `_musBar`: strings, bass, brass/horn
tune as a seeded walk with cadences, timpani, march snare in war). Starts
on the first click or key (browser autoplay rules).

### Flags: modern drawn, releasables real

`drawFlagIcon` used device emoji for the whole modern era. Windows has no
flag emoji, so every modern country rendered as two letters ("US", "UA"),
and every other platform drew its own design. Every modern nation already
had a correct drawn `FLAG[k]` (checked on a rendered sheet of all 91), so
emoji are now used only for a nation with no drawn flag. Releasables got
`REL_FLAG` (Saltire, Ulster Banner, Senyera, Ikurrina, Fleurdelise,
Kurdish sun; the rest reuse the modern drawn flag), restored on load too.

## Foreign countries: new ground, every era, puppets and colonies

Owner asked for all three of: the eleven list countries that had no
province, releasables beyond WWII, and HOI4-style foreign puppets/colonies.

- **Eleven provinces** (`P.push` after LINKS): bxl, ams, lxm, prs, tsp,
  dam, kbl, srn, tpe, lhs, buk, with land/sea links and terrain. Owner for
  EVERY era is explicit in `NEWPROV_OWN` (merged into `ERAS[e].owners`), so
  nothing turns up where it never was -- verified per era: Napoleonic
  Brussels is French, Mongol Damascus Ayyubid, WWI Bougainville German.
- **Six new nations**: BEL, HOL, LUX, AFG (1861-2026 eras), SYR and TAW
  (Cold War and modern). WWII 1941 starts and WWI 1914 starts hand the Low
  Countries to Germany via yearOwners, as happened.
- **Releasables per era**: `REL_ERAS` (ww1 15, ww2 22, coldwar 18,
  modern 11) over one `RELEASABLES` table. A people appears only where it
  had no state in that era.
- **Restore**: a dead nation whose home provinces you hold -> WARS panel
  "Restore X as a puppet".
- **Colonies**: provinces on another continent from the capital AND not
  reachable overland through the owner's own provinces (`coloniesOf`),
  grouped by overland connection. Grant independence (puppet) or they
  break free in `breakawayTick` (0.05%/check x shakiness). Named from
  REBIRTH, which gained colonial names (Sudan, Gold Coast, Malaya...).

Measured: 4 simulated years produce 1-4 breakaways per game and several
are re-conquered; check100 still 62/75; all suites pass.

Fault caught: `REL_FLAG` entries written as `sV([...])` call the stripe
helper at definition time, ~6,000 lines before `const sV` exists -- a TDZ
error that stopped the whole script at boot. Wrap as `(u,v)=>sV(...)(u,v)`.
A boot-time error shows only as `__ifStage` stuck short of "booted", so
check that after any edit to a top-level table.

### Benelux showed in German colour

New nations need their real-country shapes routed too. The map paints each
Natural Earth country by `natOfCountry(name)` (NAME2NAT plus per-era
overrides); NAME2NAT still sent Netherlands/Belgium/Luxembourg to GER, so
the new provinces existed but the ground around them was painted German.
Routed to HOL/BEL/LUX (Napoleonic and Mongol overrides keep FEM/HRE). Owner
asked for clearer colours: Netherlands orange #ec7a1c, Belgium yellow
#e3b81c, Luxembourg blue #3d7fd6. When adding a nation, grep NAME2NAT and
every *_OVERRIDE for its country name.

### Germany's eastern frontiers (East Prussia, Pomerania, Silesia)

Owner: "Germany is supposed to have Prussia and Konigsberg". The Konigsberg
PROVINCE was German in 1936, but the paint routes each pixel by its modern
Natural Earth country: Kaliningrad is inside Russia's shape (-> painted
Soviet), Masuria/Pomerania/Silesia inside Poland's (-> Polish). Same fault in
1914, where the Posen PROVINCE was also Russian (SUBPARENT poz->war).

- `HIST_REGIONS` (ww2 = 1937 border, ww1 and civilwar = 1914 border, Memel
  from 1939): [lon,lat] polygons; a pixel of Poland/Russia/Lithuania inside
  one routes to the HOME of an anchor province (kon or ber), so it follows
  conquest like any other land. Checked against 20 cities: every German and
  Polish/Russian town on the right side in 1936, 1939 and 1914.
- `HIST_LINES`: the atlas outlines are always stroked, so the Oder-Neisse and
  Kaliningrad lines were ruled through 1936 Germany. Thin bands over just
  those modern stretches are clipped out (evenodd clip) and the historical
  frontier is stroked in their place at the same weight.
- poz owner: ww1 and civilwar now GER (in 1861 GER is Prussia).
- **The coast pass draws land frontiers too.** `coastTiles` strokes every
  country's outer ring, so the modern Oder-Neisse and Kaliningrad lines were
  drawn by the COAST pass (thick, T.coast) even after the border pass was
  masked. `_histClip()` now masks both. Bands are trimmed off the Szczecin,
  Vistula and Curonian lagoons so no real coastline is erased. Found by
  colour-coding strokes; the dark line left along the Oder is the river.
- Map scale: MW=2000 for 360 degrees, so zoom 6 still shows ~36 degrees.
  A close look at a frontier needs G.cam.z ~14 (set after startGame's own
  camera fit has run, ~2s).

### Poland's 1937 east, and the Baltic states in WWII

Owner: "give Poland its original borders like in ww2", Lithuania yellow,
Latvia dark red (not Soviet red), Estonia dark blue.

- `_KRESY` (HIST_REGIONS.ww2, anchor `war`): Lithuania/Belarus/Ukraine
  shape pixels west of the 1923 demarcation and the Riga line route to
  Poland. Checked: Wilno, Grodno, Brest, Pinsk, Baranowicze, Lwow, Rowne,
  Tarnopol Polish; Kaunas, Minsk, Kiev, Zhytomyr, Chernivtsi, Uzhhorod not.
- Hide bands for today's PL-LT, PL-BY, PL-UA, LT-BY and BY-UA lines;
  `_KRESY_EDGE` stroked as the 1937 frontier. The bands now OVERLAP, so the
  clip is one path per band applied in sequence (clips intersect = hides the
  union); a single even-odd path re-shows any overlap.
- LTU/LVA/EST exist in ww2 (eras ["ww2","modern"]); WW2_OVERRIDE routes their
  shapes to them. New province `kau` Kaunas = Lithuania's ww2 capital
  (ERAS.ww2.caps); `vil` renamed "Vilnius" and owned by POL in 1936-39,
  LTU in 1940, SOV from 1941 with the three Baltic states.
- Colours set in HOI4COL (which overrides NATIONS.c): LTU #e2c21e, LVA
  #9a3a3e (Soviet is #7d0d18), EST #1f3d7a.
- Also fixed on the way: ww2 bra (Slovakia) was POL in 1936-38 -> CZE;
  coldwar lwo POL -> SOV and bra POL -> CZE.
- Czech-Slovak line (`_CZSK`) hidden while Czechoslovakia is one country:
  ww2 starts before 1939 (`hidePre39`) and the whole coldwar era
  (HIST_LINES.coldwar). From 1939 it shows, as Slovakia had split off.
- **Never route a region by `home`.** In the 1936 data Vienna and Bohemia
  are OWNED by AUT/CZE but their HOME is still GER (yearOwners sets owner
  only), so "nearest GER-home province" sent all of Silesia to Vienna and
  painted it Austrian. Each HIST_REGIONS entry now lists `cands`, the
  provinces it may belong to, and takes the nearest one's owner.
- And a dead nation's soil (Austria after 1938) now goes to the provinces
  standing in its own shape (`byCountry`), not to the nearest province
  anywhere (Graz was going to Zagreb).
- `window.__terOwner` exposes the ownership grid for probes. Points within
  ~40km of a frontier read the neighbour (Innsbruck, Klagenfurt, Kosice):
  that is the 2-unit grid, not a bug -- check `countryNames[countryPix[i]]`
  before believing a border city.

## Click a country, see its provinces

Selecting any province outlines every province of the same owner in gold,
the clicked one brighter, and the province panel lists them all
("PROVINCES OF X (N)", capital starred) as buttons that `goProv(id)` to
select and centre on that province.

- `PROV_OUTLINE[id]` is a Path2D per province, built in `buildProvEdges`
  from the same `pidx` boundary runs as the mesh: land/land edges go to both
  sides, coast edges to the land side, all chamfered like the mesh.
- No second set of names: the map already labels provinces, and drawing
  them again collided. Also, `var(--x)` is NOT valid in a canvas font
  string -- the assignment silently fails and the previous font (the huge
  country-label font) stays, which is what the first cut showed.

## Smaller provinces, and Portugal

- **The mosaic is the real game's density now.** `buildProvEdges` spaced
  its Voronoi cells for ~4,000 over the world's land (S=6), so France was a
  dozen cells. Now ~13,000 (measured 14,688, S=3.11, not rounded -- an
  integer spacing only offers 4k, 9k or 17k), which is HOI4's own count of
  land provinces. Costs ~250ms once at world build (425-517ms -> 650-782ms,
  A/B in the same run); nothing per frame beyond more hairlines in view.
- The 251 gameplay provinces are unchanged -- they now play the part of
  HOI4's states, and the cells the part of its provinces. A selected
  country also strokes its cells (`PROV_CELLS[id]`, built beside
  `PROV_OUTLINE`) in faint gold, so its small provinces show.
- **Portugal was Spain.** lis/opo were SPA in every era but modern, and
  NAME2NAT sent the Portugal shape to SPA, so it was painted Spanish yellow.
  POR now exists 1861-2026 (`eras`), owns lis/opo via NEWPROV_OWN, is light
  green (HOI4COL #8fd07a, at the owner's request) and is no longer a ww2
  releasable. Cold War: the "Portugal & Spain" rename is gone and Angola
  and Mozambique are Portuguese until 1979. In 1861-1945 those colonies
  still belong to whoever the era table gave them (CON/SAF/UK) -- untouched.
- Probe pitfall: headless frame times drop MAPQ below 2, which turns off
  the vector mosaic entirely. `setMapQuality(3)` before screenshotting it.

### South America was independent

Owner: "all of South America was independent". Before the modern era the
Peru and Chile provinces were owned by ARG and Colombia by BRA, and
NAME2NAT painted Chile/Peru/Bolivia Argentine and Colombia/Venezuela/
Ecuador Brazilian. COL, PERU and CHL now exist 1861-2026 (`eras`), own
bog/lim/scl via NEWPROV_OWN, and NAME2NAT routes Venezuela and Ecuador to
COL, Bolivia to PERU, Chile to CHL (as the modern override already did).
Uruguay and Paraguay still paint as ARG and the Guianas as BRA: they have
no province of their own. Napoleonic 1803 stays Spanish colonies, which is
right for that date.

## The world as it was: Raj, Arabia, the Nordics, South America, Alaska

Owner, looking at the 1936 world map: the Raj should be its own state,
Persia did not rule the Middle East, Scandinavia was never a country, the
rest of South America was independent, Alaska is tilted, and IRON FRONT
should be written big across the bottom like the real game's title.

- `ERA_FIX_OWN` (beside NEWPROV_OWN) is applied OVER the era owner tables,
  not just into gaps, because those tables already named these provinces.
  - Raj: del/bom/cal/mad2/lah/hyd/dac/srn -> IND in civilwar, ww1, ww2,
    renamed "British Raj", and started as UK's puppet and ally through a new
    `ERAS[e].puppets` hook in startGame. Verified it enters the war with
    Britain on 3 September 1939. Cold War keeps the existing 1949 handover.
  - Iraq (bag, bas) and Saudi Arabia (ria, jed) in ww2/coldwar; Ottoman in
    1861 and 1914 (Nejd under Ibn Saud's house in 1861). Routing: Iraq,
    Saudi, Afghanistan out of PER; Gulf sheikhdoms to UK; Yemen paints
    Saudi in 1936 (it has no province).
  - SCA is gone (eras:[]): 1861 Sweden-Norway + Denmark (Finland Russian),
    1914 Norway/Sweden/Denmark, 1936 + Finland, Cold War + Iceland. The two
    ww2 history events against "SCA" are now SOV-FIN, GER-NOR and GER-DEN,
    and they fire on 30 Nov 1939 and 9 Apr 1940.
- Seven new provinces: ccs Caracas, uio Quito, lpb La Paz, mvd Montevideo,
  asu Asuncion, geo Georgetown, pbm Paramaribo, with nations VEN ECU BOL URU
  PAR (1861-2026) and GUY SUR (modern; British and Dutch before), drawn
  flags, links and terrain. All 26 probe cities paint correctly in all eras.
- **Alaska's border slanted at world zoom** because `tileRings` thinned
  rings by taking every Nth vertex, and the 141st meridian is two vertices:
  dropping the St Elias corner drew a diagonal into the Yukon. Vertices at
  either end of a segment over 2 world units are now always kept.
- IRON FRONT is drawn at the end of the country-label pass, in NAMEFONT
  over Antarctica at 58 world units, so it scales with the map.
- Not done: Greenland is still its own nation (GRE) before the modern era.

## Yemen, Oman, Finland 1939, the Dominion, and the close-up map

- **Yemen (sna) and Oman (msc)** are provinces and nations now. Yemen is
  Ottoman in 1861 and 1914 and the Kingdom of Yemen from 1936; Oman ("Muscat
  and Oman" in 1861/1914/1936) exists in every era but the Mongol one.
  Routing sends the Yemen and Oman shapes to YEM/OMN everywhere. Aden is not
  a province, so the 1936 British colony there still paints Yemeni.
- **Finland's 1920-1940 border.** Natural Earth gives the post-1944 line, so
  1936 painted Viipuri, Ladoga Karelia, Salla and Petsamo Soviet.
  HIST_REGIONS gained `until` (Karelia and Salla until 1940, Petsamo until
  1944, cands hel/tku), HIST_LINES gained `hideUntil`/`linesUntil`, and the
  bands that hide today's line are built by `_bandOf()` from a
  Douglas-Peucker'd copy of the atlas line. Verified: Vyborg, Terijoki,
  Sortavala, Suojarvi, Salmi, Kuolajarvi and Pechenga Finnish in 1936/39,
  only Pechenga in 1941, none in 1944; Leningrad, Petrozavodsk, Murmansk
  Soviet throughout. The old line is not drawn across Ladoga.
- The dark soft spots along the Karelian Isthmus in 1936 are NOT a border
  artefact -- they are the Vuoksi lakes in the painted ground, visible only
  because that ground is now Finnish pale rather than Soviet maroon. Ruled
  out by switching off the coast, border and mesh passes one at a time.
- Canada is "Dominion of Canada" in ww1 and ww2.
- **The close-up map was a different, worse map.** Two coarse layers
  (political wash and painted ground) were drawn at one texel per world unit
  with land/sea decided on the 2-unit grid, so up close their edge was a
  staircase of blocks running out over the sea while the vector coastline
  sat elsewhere. Now, from zoom 2.2: each layer has a copy grown three
  texels out over coarse "sea" (`_dilateCoast`), drawn clipped to the vector
  land (`_maskedBlit`). The clip is built from `_landTiles` (each land ring
  Sutherland-Hodgman-clipped into 25-unit tiles at load) so a frame clips to
  the few tiles in view.
  - First version composed through a screen-sized offscreen canvas and cost
    150-270ms per camera move (the blit back flushed the deferred work).
    Clipping ctx directly measured the same as no mask in the same run.
- **Lakes were never drawn on the real map** -- only on the fallback one --
  so Ladoga, Onega, the Great Lakes and Victoria were dry land in national
  colour. They are filled with the sea colour after the wash now.

### Carpathian Ruthenia

Owner: the isolated part of Ukraine between Poland, Czechoslovakia and
Romania should be Czechoslovak. Zakarpattia is inside today's Ukraine shape,
so it painted Soviet in every ww2 start. `_RUTH` (HIST_REGIONS) routes it to
`bra` (Czechoslovakia) before 1939 and to `deb`/`bud` (Hungary) from 1939,
and to Hungary/Austria-Hungary in 1861 and 1914. `_RUTH_EDGE`, the
Carpathian crest, is stroked as the frontier (no admin-0 line carries it),
and today's Slovak-Ukrainian line is hidden while both sides are one state.
Verified Uzhhorod, Mukachevo, Khust: CZE 1936, HUN 1939-44, SOV 1945, UKR
2026. Not touched: Galicia (Lviv) still paints Russian in 1914, though it
was Austrian.

## Exact territory, Siam, Central America and the Caribbean

**"Remove the pieces of territory on borders, they're squares."** The
political layer is one texel per world unit (~20 km) read off a 2-unit
ownership grid, so every frontier was a staircase a texel either side of the
drawn border (Belgium yellow round Lille, France blue into the Saarland).
From zoom 2.2 the wash is now drawn EXACT (`_drawWashExact`):
- per 32-unit tile at 4/8/16 px per unit, cached until a border moves
  (`_exTiles`, ~16M px LRU, at most 6 builds a frame, the old masked raster
  fills tiles not built yet);
- in a tile, each modern country is clipped to its own atlas outline
  (`_featPath`, kept from rasterizeCountries) and painted from its own
  owners, texels across the border taking this country's dominant colour --
  so frontiers between countries are the atlas line, exactly;
- where a country is split by a HIST_REGIONS polygon (1936 Poland holds
  German Pomerania/Silesia), each side whose colour holds 80% of its texels
  is filled flat inside the polygon, so that frontier is exact too. Only
  splits with no polygon (a conquest mid-war) are still the grid, smoothed;
- the frontier-strong alpha is applied as a mask (`_exAlpha`).
Measured in the same run, panning at zoom 14 got cheaper (the exact tiles
replace a 2000x1000 upscale); what remains slow is the ground layer's own
upscale, which was there before.

Two routing faults the exact wash exposed:
- Bohemia carries home GER in 1936, so Saxony's edge routed to Prague and
  painted Czech inside Germany. A shape now prefers its home nation's
  provinces STANDING IN IT (`_bch`) before byHome anywhere.
- A country with nothing of its own in the shape (Laos under France) took
  the nearest French-HOME province, which could be on another continent.
  Now byHome plus byOwner, so Laos goes to French Hanoi.
- Islands too small for the coarse grid (Jamaica) took the nearest province
  across the water; their fine texels now ask natOfCountry.
- Routing token "*": share the shape between the provinces standing in it
  by distance (1861 Burma: British Rangoon, royal Mandalay).

**Siam** held Burma, Indochina and Malaya in every era. Now: 1914/1936 Burma
in the Raj, Indochina French, Malaya British; 1861 Lower Burma British,
Upper Burma "Kingdom of Burma" (MMR), Cochinchina and Cambodia French,
Tonkin/Annam "Dai Nam" (NVN), Laos Siamese.

**Central America and the Caribbean** had no provinces, so Mexico painted
over all of it. Nine provinces and nations: GUA HON ELS NIC CRI (1861-),
PAN (1914-, Colombian 1861), CUB (1914-, Spanish 1861), HAI DOM, with drawn
flags. Belize, Jamaica, the Bahamas and Trinidad are British before the
modern era.

## The 1936 audit

Asked to "continue and update a lot": every province's 1936 owner was
listed and checked against history, and everything wrong was fixed:

- Ethiopia (add) independent, Italian from a 1937 start; history event
  ITA-ETH war on 1936-01-02. Ireland (dub) the Irish Free State. Albania
  (tir) independent, Italian from 1939. Greenland Danish (GRE retired).
  Mongolia (uln) MON, Soviet puppet. Manchuria (har) MANCHUKUO (MAN, new),
  Japanese puppet. Tibet (lhs) TBT (new; the releasable TIB stays for the
  other eras). Angola/Mozambique Portuguese, the Congo Belgian (until 1961
  in the Cold War), Bangui French. Indies Dutch, Philippines American,
  Papua Australian, Timor Portuguese. Bessarabia and northern Bukovina
  Romanian until 1940 (`_BESS_UA`, `_TRANSN`, today's RO-UA and MD-UA lines
  hidden, `_ROM40_EDGE` drawn along the Dniester). 1914 Galicia Austrian
  (`_GALICIA`). History event JAP-CHN on 1937-07-07 (fires on the day).
- **The region check was a hard-coded list of five countries** (Poland,
  Russia, Lithuania, Belarus, Ukraine) in renderTerritory, so Manchukuo,
  Tibet, Moldova -- and Finland's Karelia, which happened to be in Russia --
  only worked when their shape was on it. Now `_hrNames` from the era's
  HIST_REGIONS.
- Shape with nothing of its own: byOwner first, then byHome. byHome first
  sent the Dutch Indies to Germany the moment Amsterdam fell.
- Verified 30 probe cities across 1861/1913/1936/1939/1941/1945; Bangui
  and Izmail read the neighbour because they sit on a border river (the
  40-km grid, not data).

### Real borders from admin-1, not ruled lines

Owner: "don't make just line borders, make real borders". The historical
regions added for 1936 were 7-15 point hand polygons. Most of those
frontiers are TODAY'S PROVINCIAL BOUNDARIES, and Natural Earth's
`ne_10m_admin_1_states_provinces` (public domain, fetched from the
nvkelso mirror, 40MB) carries them. Built offline (`a1build.py` in the
scratchpad, no shapely -- admin-1 units share vertices, so a union is the
edges that appear once):
- Ruthenia = Zakarpattia; Bessarabia/Bukovina in Ukraine = Chernivtsi +
  the Budjak (Odessa oblast west of the Dniester liman -- the liman is the
  one hand edge); Transnistria = Moldova's left-bank districts; 1914
  Galicia = Lviv + Ternopil + Ivano-Frankivsk + Chernivtsi; interwar Polish
  Volhynia/Galicia = Volyn + Rivne + Lviv + Ternopil + Ivano-Frankivsk;
  Tibet = Xizang; Manchukuo = Heilongjiang + Jilin + Liaoning + a hand
  polygon for Jehol and the Hinggan (no province line there).
- `_A1.<R>.rings` route pixels (`_inRegion`, bbox-filtered); `.lines` are
  only the INTERNAL edges -- shared with another province of the same
  country -- since the atlas draws the foreign ones. The 1940 hide set
  (`HIDE_ROUA`) is classified the same way: every MD-UA and RO-UA edge
  except Moldova-proper against Vinnytsia, which is the Dniester and was
  the real 1936 Romania-USSR frontier.
- Region rings are wound one way so nonzero is their union; exclusion
  clips one ring at a time (`_clipOut`).
Still hand-placed: the Belarus/Lithuania part of the Kresy, Germany's
1937 line in Poland, Finland's 1920-40 line, Jehol/Hinggan.

### The Mediterranean and Africa, 1936

Owner: "improve it more". A close audit of Europe, North Africa and the
Middle East found, and fixed:
- Cyprus British (the atlas splits it: "Cyprus" and "N. Cyprus" -- both
  routed), Malta British, Qatar/Bahrain British, Liberia a nation with a
  province (mnr, LBR, 1861-), Gambia and Sierra Leone British.
- Regions from admin-1: the Aden Protectorate (Aden, Lahij, Abyan,
  Shabwah, Hadramawt, al-Mahrah, ad-Dali -> British, cands psud/sue);
  Hatay French until 1939; Spanish Morocco (Tanger-Tetouan + a hand Rif
  strip) and Spanish Sahara (the Laayoune and Oued ed-Dahab regions --
  the atlas draws Western Sahara inside Morocco's shape); Ifni and the
  Dodecanese as small hand boxes (islands/enclave, so a box is exact).
- **The pocket pass was eating them.** A connected area with none of its
  owner's provinces, a short coast and one neighbour is handed to that
  neighbour (meant for stale wedges after a conquest) -- and Spanish
  Sahara is exactly that shape. Cells a historical region placed are now
  flagged (`_hrCell`) and never count as a pocket.
- A country thinner than a 40-km cell (the Gambia) never owned a coarse
  cell, so its fine texels now ask natOfCountry whenever no neighbouring
  cell of their own country exists, not only when the cell is sea.
- Yemen recoloured #d08a3c: it sat next to British Aden in nearly the same red.

## Every country carries its name

Owner: "give every country its name". `drawNationNames` skipped any label
under 10px or over 64px, which at the world view dropped 51 of 69 nations
in 1936 (Germany, Italy, Japan, Poland among them) and, zoomed on Europe,
dropped France for being too BIG. Measured with `drawNationNames.last`
(the nations drawn this frame) against every alive nation whose label
centre is on screen:

                         before   after
    world view (z1.2)    18/69    64/69
    Europe, Balkans, Central America, W. Africa, Gulf: none missing

- A big name is held at 64px instead of vanishing; a small country is
  written at 8.5px with a firmer fill, after every name that fits, nudged
  round its own centre (above, below, either side) to find room.
- A full-size name that clashes steps down in size before it is dropped.
- **Labels went on the largest landmass**, which for an empire is a colony:
  FRANCE across the Sahara, ITALY across Libya, nothing on either homeland.
  The landmass holding the capital carries it now (`_lblComps`, picked by
  the capital's coarse cell); another landmass gets a copy only where the
  name fits at full size (French West Africa, Alaska, Syria).
- **Clash on the turned box.** The test used the upright box round a
  rotated label, so SWEDEN on its diagonal reserved most of Norway and
  Norway's name could never land. Separating-axis test on the rotated
  rectangles fixed it.
- Slanting every small name to its country's axis cost four labels at the
  world view (64 -> 60), so it is only the last try before a name is
  dropped.
- What remains missing at the world view (Bulgaria, Hungary, Netherlands,
  Luxembourg, El Salvador) is countries packed tighter than letters; one
  zoom step in, they come through.

## Construction tab, adding divisions to army groups, the PROVINCES button

Owner: "a construction tab for forts, add divisions like add army group, and
make the province button work faster".

- **🏛 PROVINCES took 7,980ms and then ran at 1.0 fps.** `loadStates`
  downloaded Natural Earth's 10m admin-1 LINES (tens of MB) from a CDN and
  stroked every vertex every frame. It now shows the game's own province
  borders: `provBordersPath()` joins the `PROV_OUTLINE` paths
  `buildProvEdges` already traces for the selection highlight, once per
  outline set. Click to visible: 1ms. A/B in the same run, off/on/off/on:
  1.25 / 1.25 / 1.5 / 1.25 fps -- no measurable cost (this container's
  absolute fps is meaningless; only the comparison is). The local Voronoi
  fallback stays for when the outlines are not built. No download remains.
- **CONSTRUCTION side tab** (`curPanel==="con"`): sites working/waiting,
  every site under construction with progress and call-off, and a tab per
  BUILDINGS entry (forts, factories, depots) listing every province you hold
  in bands -- FRONT LINE (faces an enemy at war), BORDERS (faces a foreign
  power that is not an ally/puppet), COAST, INTERIOR -- with level pips
  (built / building / free) and a build button. Forts also get "Fortify the
  front" and "Fortify all borders": one level each, front first, until the
  PP runs out (`conBuildBand`). Uses the same `G.build` queue as the
  province panel's construction rows, so nothing new to save.
- **Army groups made first, filled after.** Groups were fixed at four and
  only appeared once something was in one. Now up to `GROUP_MAX`=8, with
  NEW ARMY GROUP, and per group "Add divisions": infantry, armour (and
  artillery/AA from v9.0) raised straight into the group at the selected
  province or the capital, x3 infantry, and "add the selected division".
  A queue order carries `grp` (saved as queue field a[6], sent with the MP
  "buy" message); `productionTick` puts the new unit in the group if the
  group still exists. Verified: 2 ordered for a group, save, load, 120
  production days -> 2 divisions in that group.
- The province panel itself was measured too in case that was the "button":
  8.8ms to open, goProv 0.2ms. Not the problem.

## Wrong colour inside other countries

Owner: "there is still different country's colors in others". Measured,
not eyeballed: for every era, every atlas country whose coarse cells carry
more than one owner, with how many of those cells a historical region
placed (`window.__terOwner.hr`). A split the regions account for is
history (Manchukuo, the Kresy); one they do not is a fault. 1936 had
Corsica Italian, Sumatra and Borneo British, Celebes and Dutch New Guinea
Australian, Saguia el-Hamra French, a slice of the Philippines Japanese,
Baffin Danish, the Alaska panhandle Canadian.

- **Every one of those was correct after the nearest-province pass and
  wrong after the POCKET pass.** Snapshotting ownership after each pass
  showed it directly (France 542 FRA -> 535 FRA + 7 ITA). The pocket pass
  hands a province-less component to its one neighbour -- meant for stale
  wedges after a conquest -- and an island or colony with no province of
  its own looks exactly like that (Corsica touches Sardinia on the 40-km
  grid; Dutch Borneo touches Sarawak). Now a pocket may only go to a
  nation the first pass already painted inside the same atlas country
  (`_shapeAllow`). A conquest wedge still qualifies (the conqueror is
  painting there); Corsica does not (Italy never is).
- A/B in the same run, eight scripted conquests: stale components added
  +3/+13 cells before and +3/+13 after -- the conquest clean-up is
  unchanged. The baseline count rose by the islands and colonies now kept.
- **Routing to a nation that is not in the game.** The Cold War table
  sends Chad, Mali, Gabon... to WAF/CON, blocs that exist only after
  decolonisation; in a 1945 start they are dead, and the shape fell to the
  nearest province whose HOME was the bloc -- British Kano for Chad and
  Niger, Liberia for Guinea and Mali, Belgium for Gabon. `ERA_FALLBACK`
  gives the power that actually held it, used only when the routed nation
  was not alive when the game began (`_aliveAtStart`, cached on G), so a
  nation conquered later still hands its soil to the conqueror.
  `natOfCountry` wraps the old lookup (`_natOfCountry`).
- Guinea-Bissau was routed to "CON" in 1936 (painted French): now POR.
  1914 Spanish Guinea and Portuguese Guinea routed. 1861 no longer has an
  Egyptian Aden: the ADEN region came from ww1, where its candidates are
  British, but in 1861 they are Egyptian.
- **The Philippines had no province**, so it was painted from whoever was
  nearest: USA through byOwner in 1936, TAIWAN in 2026. New province `mnl`
  Manila and nation PHI (coldwar, modern; flag drawn): Spanish 1803/1861,
  American 1914-41, Japanese 1942, American 1945, independent 1946.
- After: 1936 has no split a historical region does not explain. What is
  left in other eras is history -- 1861 Lower Burma and Cochinchina,
  Ottoman/British Yemen in 1914, the 1943 occupations, East Germany.

## The Cold War, country by country

Owner: "improve it more". Listing every nation alive at each Cold War start
(`natlist.js`) showed the era was built on blocs: "North Africa" (Libya
in 1945, then Morocco, Tunisia, Algeria), "West Africa" (Nigeria + Ghana +
Senegal + Ivory Coast as one state), "East Africa", "Congo" (with Angola),
and Siam RENAMED "South Vietnam", holding Bangkok and taking Burma, Saigon,
Phnom Penh, Malaya and Singapore. Sudan went to Egypt in 1956, Mozambique
to South Africa in 1979, East Pakistan was Bangladesh from 1949, China was
"People's Republic" in 1945, Iran was still "Persia", Tibet was Chinese in
1945, Kashmir and the Guianas stayed British for the whole era.

- The modern-era nations (MAR DZA TUN LBY SDN NGA GHA SEN CIV KEN UGA TZA
  COD CAF AGO MOZ ZWE ZMB MYS SGP CAM SOM GUY SUR, and MMR) now exist in the
  Cold War too, and `ERAS.coldwar.yearOwners` hands each its provinces on
  its real date: 1949 India/Pakistan (Bengal as PAKISTAN)/Burma/Koreas/
  Taiwan/Kashmir; 1953 Libya, Somalia to Italy, Eritrea to Ethiopia, Tibet
  to China; 1956 Morocco, Tunisia, Sudan, South Vietnam (new SVN), Cambodia;
  1961 Ghana, Nigeria, Senegal, Ivory Coast, Somalia, Malaya, the two
  Congos (CAF for Bangui); 1968 Algeria, Kenya, Uganda, Tanzania, Zambia,
  Singapore, Guyana; 1979 Angola, Mozambique, reunified Vietnam,
  Bangladesh, Suriname; 1989 Zimbabwe. 1945 base: Libya, Mogadishu,
  Asmara, Lusaka British; Lhasa Tibetan; Taipei Chinese.
- Renames: SEA "Thailand", PER "Iran", CHN "China", MMR "Burma".
- **New province `jkt` Jakarta and nation INA.** Indonesia had no
  province, so it stayed Dutch for the whole Cold War, and in 2026 it was
  painted MALAYSIAN. Dutch 1861-1941, Japanese 1942, Dutch 1945,
  independent 1949. The Netherlands now holds a real colony in WWII.
- **Routing follows the province.** A shape holding a province is "*" (its
  holder paints it), so independence and later conquest both show without
  a per-year table. New token "@pid" = whoever holds that province, for a
  shape whose province is off its own cell on the 40-km grid (Mogadishu,
  Singapore are ports on sea cells; Bangui stands inside the DR Congo
  outline). Shapes with no province go to the neighbouring state that took
  their place (Mali to Senegal, Chad to Nigeria...), the modern era's
  convention, with `ERA_FALLBACK` giving the colonial power before it
  exists.
- Tried and reverted: letting a coastal province look 1-2 cells inland
  for its country. It found Singapore's country as MALAYSIA (Johor), so
  southern Malaysia went Singaporean. "@sin" instead.
- Regions in the Cold War: Tibet to 1953; Spanish Morocco to 1956, Ifni to
  1969, Spanish Sahara to 1976 (also the W. Sahara shape); Vietnam north
  of the 17th parallel goes with Hanoi (the Hanoi-Da Nang bisector put the
  line at 18.5N). Papua New Guinea routed to Australia (it was "SEA" and
  painted THAILAND), Brunei to Britain, Madagascar to France, Egypt to
  Egypt (as "*" it gave Britain a Suez slice half of Sinai).
- Checked: no unexplained split in any Cold War year (1945-1989) or any
  other era; a simulated year from 1945, 1961 and 1979 runs with no errors.
- Not done: the Sahel, Cameroon, Laos, Ceylon and similar countries with
  no province still take a neighbour's colour after independence.

## Sixteen countries that had no province

Owner said yes to giving the province-less countries their own ground.
Mali, Niger, Chad, Cameroon, Mauritania, Burkina Faso (Upper Volta in the
Cold War), Guinea, Sierra Leone, Gabon, Congo-Brazzaville, Madagascar,
Laos, Sri Lanka (Ceylon), Nepal, Malawi and Botswana each get a province
(bko nim ndj yao nkc oua cky fna lbv bzv tnr vte cmb ktm llw gbe), a
nation (Cold War and modern; Nepal from 1803, Madagascar's Merina kingdom
in 1861) and a drawn flag.

- Base owners were read off the map first (`newloc.js`: who paints each
  new city's location in each era), so eras that were right stay exactly
  as they were; then corrected where history differs: 1913 Kamerun
  German, French West and Equatorial Africa French (1913 painted them the
  "West Africa" bloc), Nyasaland and Bechuanaland British, Nepal
  independent in every era but the Mongol one.
- Independence on the real date: Ceylon 1948, Laos 1953, the French
  colonies and Madagascar 1960, Sierra Leone 1961, Malawi 1964, Botswana
  1966 (Cold War starts 1949/1956/1961/1968).
- Routing: every one of the sixteen shapes is "@pid" in every era's table,
  so the shape is always painted by whoever holds its province --
  independence, colonial rule and conquest all fall out of ownership.
- Placement checked against the grid: all sixteen stand inside their own
  country (Vientiane moved to 102.9,18.6 -- the real city is on the
  Mekong and its cell is Thai). Madagascar and Ceylon have sea links only.
- After: no new split in any era; a simulated year from 1861, 1914, 1936,
  1945, 1961 and 2026 runs with no errors.
- Still without a province: Togo and Benin (Ghana/Nigeria colours),
  Rwanda/Burundi, Bhutan, Eswatini/Lesotho, Djibouti, Eq. Guinea,
  Guinea-Bissau, Gambia.

## The war layer: front line, battle plates, offensive arrows

Owner: "make it more like hoi4". Screenshots of a 1939 war showed what the
real game puts on the map the moment a war starts and this one lacked: at
war with Poland, nothing said where Poland began. Three additions, all
drawn under/over the counters in `draw()`:

- **Front line** (`frontPath`): thick red line with ticks across it,
  wherever the player's side (self, allies, puppets, overlord) touches an
  enemy. Traced off the ownership grid the colours are painted from,
  chained into polylines, Douglas-Peucker at 0.75 cell (a staircase becomes
  the diagonal it stands for) then Chaikin x3. Fragments under five grid
  points are dropped (coast slivers). Rebuilt only when ownership, the
  push state or the war/alliance set changes (~13ms).
- **The old front was in the wrong place.** `buildFronts` drew a thin
  orange line from `provPix`, the nearest-province raster, whose
  boundaries are bisectors between province centres -- a straight line
  through Pomerania at 15.5E, nowhere near the 1937 frontier the map
  paints. It now applies the same push cut (each contested province split
  along its attack axis by p.push) to the PAINTED ownership grid and
  exports `frontEff`; the old hairline draw is gone. Drift checked: a
  province at push 0 / .25 / .5 / .75 hands 0 / 9 / 20 / 29 cells to the
  attacker, so the line still walks across a province as it is taken.
- **Battle plates** (`drawBattleMarkers`): a dark plate under each battle
  province with crossed swords and a bar -- attacker share on the left,
  green for the player's side, red for the enemy, nation colours for
  battles the player is not in. Replaces an emoji that blinked on
  alternate days. Under the marker because counters stack on top of it.
- **Offensive arrows** (`drawOffensiveArrows`): the player's divisions
  marching on enemy ground, grouped by final destination, one curved red
  arrow per target from their mean position, thicker for more divisions.
- Probe note: a battle does not survive a full `tick()` in a probe -- the
  AI marches units off first. Place both sides and call `battles()`.

## Ground taken a small province at a time

Owner: "make them take by province like real hoi4". The 251 gameplay
provinces play HOI4's states; the ~14,000 mosaic cells play its provinces.
An attack's p.push used to move a STRAIGHT CUT across the state and showed
only in the front line -- the territory kept the defender's colour until
the whole state fell.

- `buildProvEdges` now keeps the cell map (`provSub`) and each province's
  grid squares (`provSquares`, CSR).
- `pushRank` orders a contested state's cells by walking distance from
  ground the attacker already holds (multi-source BFS inside the state,
  mean distance per cell), so the front cells go first and the far side
  last; cached per province/owner/attacker until ownership is repainted.
- `buildFronts` hands cells over WHOLE until their area reaches p.push.
  Checked on Poznan: push .25/.5/.75 -> 3/6/8 of 9 cells. The straight cut
  remains only as the fallback with no cell map. `PUSH_TAKEN[id]` gives
  "N of M provinces" for the province panel.
- Taken cells are painted (`frontEff.capPaths`) in the occupier's colour
  at 0.62 with screen-constant diagonal stripes, clipped to land
  (`_visibleLand()` needs "evenodd", as `_maskedBlit` uses -- nonzero
  showed nothing). The front line follows their edge.
- **A closed loop simplified whole vanishes.** Douglas-Peucker on a ring
  measures every point against a zero-length line from its start to its
  (identical) end, finds distance 0 everywhere and keeps two points. Every
  captured region is a ring, so the fill was empty until `_chainPath`
  split rings at the point farthest from the start. Encircled pockets on
  the front line had the same fault.
- Cost: 40 states contested at once, 71ms first build (ranking) and 39ms
  after; rebuilds are limited to four a second.

## The original score, rewritten as a soundtrack

Owner wanted the built-in music to feel like HOI4's without shipping
Paradox's. So everything here is still synthesised at run time and composed
by this file; nothing is sampled, transcribed or imitated note-for-note.

- **Form, not a loop.** Each piece is 32 bars: opening (strings + a solo,
  piano or horn), build (low-string ostinato, timpani, swelling bar by bar,
  a timpani/snare roll and a reversed cymbal into), climax (theme in brass or
  choir, trombones an octave under, horns holding the chord, bass drum),
  fall (horn or trombone chorale, the solo returning, a final held chord).
  Then a NEW piece: new theme, key shifted, tempo +-5% -- a playlist.
- **A theme you can hum.** A two-bar motif from the dice, stated as
  A A' B A-cadence; strong beats snap to the chord, the cadence lands on
  its root. Upper strings are voice-led (nearest inversion).
- **A war starts at the build**, with a timpani/bass-drum/crash stinger.
- Measured offline (OfflineAudioContext, `mus.js` in the scratchpad):
  quiet-to-loud spread 1-2 dB -> 7-8 dB, stereo width 0.04 -> 0.2,
  peaks ~0.4, no clipping.
- **CPU was the real cost**: first cut rendered at 3x the old score. Two
  saws per note instead of three, one for the ostinato, and rolls as ONE
  tremolo voice instead of twenty separate strokes brought it to ~2x;
  still well under real time on this container, and it runs on the audio
  thread, not the game's.
- Do not `pkill -f "until grep"` from a shell whose own command contains
  that text: it kills itself (exit 144).

## "I load in and it's just grey terrain"

Could not be reproduced on a clean profile -- colours paint at every
MAPQ (share of strongly coloured pixels 0.70 at q0/q1/q3, 0.87 at world
zoom). The cause was STORED STATE: `MAPSTYLE` was saved to localStorage
(`if_mapstyle`) for every style, and TERRAIN draws bare ground with no
country colours and no names (0.43 coloured, all of it ground). One click
on it made every later campaign open grey. Worse, the Settings row that
offered TERRAIN had no POLITICAL button, and its text claimed terrain
gives "each country one dark color", which it does not.

- Only political/flags are remembered (`MAPSTYLE_KEEP`); any other stored
  value is ignored and removed at load. startGame resets a lens back to
  `MAPSTYLE_HOME`.
- A map-mode button beside PROVINCES (`#bMode`, `cycleMapMode`) names the
  mode, cycles it, and lights up while the colours are off.
- Lesson: when a user sees something a clean profile cannot, look at what
  localStorage carries before suspecting the renderer.

## Mexico's flag from the owner's picture

The owner sent a picture of Mexico's flag after an audit of all 189 drawn
flags (sorted into exact / simplified emblem / invented). FLAG.MEX is now
`imgBacked` from it (cropped of its 2px frame, 288x180 JPEG, 5KB), with the
old drawn seal as the fallback until it decodes. Same mechanism as the
Spanish arms, Brazil 1889 and Prussia. Other flags with a simplified emblem
can go the same way if the owner sends pictures; keep each under ~10KB.
- Eritrea (FLAG.ERI) likewise from the owner's picture, 13KB. It must be
  assigned AFTER the `Object.assign(FLAG,{...})` block that defines the
  drawn ERI, or that block silently overwrites it -- which the first cut did.
- Bosnia was drawn MIRRORED (hypotenuse from top-right). Redrawn on the
  official 16x8 construction: triangle top edge x 4.24..12.24, upright side
  at 12.24, nine stars at ((2.88+i)/16, i/8), the end ones cut by the edges.
  The owner's Bosnia picture was a cropped screenshot with a Google Lens
  button over it, so it was not used.
- The owner re-sent pictures of royal Romania, royal Serbia, Napoleon's
  France, Brazil 1889 and Prussia. The first four were already image-backed
  but as small palette-reduced PNG traces (Romania's arms came out washed
  pale); they are now 300x200 JPEGs from the new pictures (9-13KB each,
  file +~35KB). Prussia's picture was cropped top, bottom and hoist, so the
  existing full traced flag stays.
- British Raj (FLAG.RAJ, used by IND before the Cold War): the owner's
  picture was a waving-cloth render with a bent Union Jack, so only the Star
  of India badge was cut from it (`_rajStar`, 210x180, 18KB) and laid on the
  flat drawn red ensign, its red surround keyed out (r>120,g<105,b<115 ->
  field). Cold War IND now flies the tricolour too; it used to show the Raj
  ensign until the modern era.

## 1803: Sweden and Denmark-Norway, and the Austrian Empire's flag

Owner: "Scandinavia was never a country; close to it was the Kalmar Union"
(1397-1523, which no era here falls in). The Napoleonic era had one nation,
SWE "Scandinavia", holding Sweden, Norway, Denmark, Iceland and Greenland,
with Finland Russian -- wrong both ways for 1803.

- SWE is now "Sweden" and holds Finland (hel, tku) until 1810 (ceded
  September 1809). DEN exists in the napoleon era, renamed
  "Denmark-Norway" there, holding cph, osl, brg, nar, trd, rey, nuk, thu,
  sco2. Norway goes to Sweden in 1814 (Kiel). ERA_FIX_OWN.napoleon,
  ERA_FIX_YEARS.napoleon; NAPO_OVERRIDE routes Norway "@osl", Finland
  "@hel", Denmark/Iceland/Greenland DEN. The ERA_FIX_OWN loop's Raj
  puppet hook now skips napoleon as well as coldwar (no IND/UK there).
- Probed by city: Oslo/Bergen/Trondheim DEN 1803-1810, SWE 1815; Turku and
  Oulu SWE 1803, RUE 1810. Malmo reads PRU/FEM, but its coarse cell is
  "sea" on the Sound and read the same before the change -- a probe
  artefact. A year from 1803, 1809 and 1813 runs clean.
- AUE (Austrian Empire) flew red-white-red here; it is now the black-gold
  flag with the double eagle from the owner's picture (`_aueFlag`, 300x200,
  16KB).
- The Mongol era's "Christian Kingdoms" (EEU) still covers Scandinavia as
  a bloc; not touched.

## Flags in the interface, not emoji

Owner, after the flag pictures: "WHY ARE THEY ALL EMOJIS". The drawn flags
only reached the map and the country cards; 76 places in the panels, top
bar, start button, message log and newspaper printed `NATIONS[k].f`, the
device's emoji -- two letters on Windows, and today's flag for an 1803
kingdom.

- `FL(k)` returns `<i class="fl fl-KEY"></i>`; the flag is painted once from
  the same FLAG[k] painter as the map into a style rule (#flagRules), so log
  lines and saves stay small. `FLR(id,col)` does the same for a releasable
  that is not a nation yet (REL_FLAG). `refreshFlagRules()` repaints the
  rules when the era/FLAGSTYLE changes or a picture-backed flag decodes
  (hooked into refreshFlagIcons), so tags already on screen follow.
- **The blanket replace hit data, not just display.** `NATIONS[k].f=...`
  in applyEra/renames/load and `f:NATIONS[k].f` in NAT_BASE and the save
  array became FL(k) and had to be put back -- the emoji field is still the
  stored identity and the pre-5.3 Time Machine path still shows it.
- Every remaining FL( call was checked to land in innerHTML (no <option>,
  title= or textContent). In a 1939 game the page shows 32 drawn flags and
  0 emoji flags; smoke, a year in six eras and save/load all clean.
- Portugal (FLAG.POR): the Republic's flag from the owner's picture
  (`_porRep`, 300x200, 7KB) for 1914 onward; it had been a gold ring round a
  red disc, and the same republic flag was flown in 1861. The 1861 kingdom
  now flies the blue-white royal flag with the shield of the quinas under a
  crown (`_porRoyal`, drawn). Assigned beside FLAG.ERI, after the
  Object.assign block that defines the drawn one.
- 1861 Portugal now uses the owner's blue-white picture (`_porBW`, 7KB):
  armillary sphere and shield, no crown. The real 1830-1910 royal flag had
  the crowned shield and no sphere; the owner's choice stands, and the drawn
  `_porRoyal` (with the crown) remains only as the decode fallback.

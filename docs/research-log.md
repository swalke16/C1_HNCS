# Research log: reverse-engineering Phocus with synthetic raws

Working notes behind [phocus-hncs.md](phocus-hncs.md). Dated entries, newest last. Numbers are ΔE00 (CIEDE2000) unless
stated; "greys" means near-neutral patches or blocks. Camera: Hasselblad X2D II 100C; Phocus 4.2.2 (Mac);
Capture One 16.9.

**In plain terms:** Phocus can't be scripted and doesn't document its colour science (HNCS). So the
approach was to feed it raw files whose content is known exactly, and measure what comes out. X2D II
raws store the sensor data uncompressed, so a real raw can be copied and its picture replaced with
patches of known colour. That answered questions real photos couldn't: why two cameras render
differently in Phocus, what Phocus does at the edges of the frame, and how its colour changes with
white balance. It also produced better Capture One profiles than fitting to real photos did.

## Background (before 2026-11-15)

**In plain terms:** Before this, profiles were fitted by rendering real photos in both apps and
matching them. That got within about 0.7 ΔE00 (hard to see), but left puzzles: frames from another
camera rendered slightly differently, and the edges of the frame never quite matched.

- E1–E3: Capture One camera profiles (33³ RGB→Lab LUT) fitted to Phocus renders of real frames, owner's
  XCD 28P and XCD 35-100. E3 held-out ≈ 0.7–0.8.
- E4 (2026-11-15): added 33 sample raws from DPReview's X2D II gallery (portraits, saturated colours).
  Phocus rendered DPReview greys about +1.0 b*, −0.6 a* differently from the owner's frames (relative to
  the same Capture One input). E4 fitted a per-source offset so the owner's frames set the balance.
- Found along the way:
  - Phocus colour depends on white balance: one transform holds ~4800–6500 K; tungsten (~3400 K) is
    1.6–2.1 off.
  - BaselineExposure (IFD0 50730, 0/+1/+2 EV on DPReview raws) is applied by both apps.
  - Phocus 4.2.x Modify dialog starts with all boxes ticked, so batch-applying a preset copies white
    balance (this produced a Daylight-WB export of 25 frames by accident; it was used as a second data set).

## 2026-11-15: synthetic raws

### Raw format facts (X2D II 3FR, firmware 1.1.6.3 and 1.3.16.2)

**In plain terms:** An X2D II raw is a TIFF file holding a small preview and the full sensor readout as
plain, uncompressed 16-bit numbers. So new sensor data can be written into a copy of a real file while
everything else (camera, lens, white balance) stays as it was.

- TIFF container, little-endian. IFD0 holds a 3888×2918 JPEG preview plus DNG-style tags:
  AsShotNeutral, ColorMatrix1, BaselineExposure, BaselineExposureOffset (−0.02), CameraSerialNumber,
  UniqueCameraModel, RawDataUniqueID.
- One SubIFD holds the raw: 11904 × 8842, uint16, **uncompressed** (Compression 1, CFA photometric 32803),
  a single strip at file offset 32768 (210,510,336 bytes).
- BlackLevel 4096, WhiteLevel 64762/64763, DefaultCropOrigin (128, 96), DefaultCropSize 11656 × 8742,
  MaskedAreas present (optical black borders).
- CFA **RGGB** from the raw origin (red at even row, even column). First attempt assumed GBRG from a
  sample window that started on an odd row; Capture One's render showed swapped channels, which caught it.
- ColorMatrix1 is identical across the two bodies examined.

### Method

**In plain terms:** Each test file is a real raw with its picture replaced by a grid of flat colour
squares. Capture One shows exactly what camera colour it reads for each square, and Phocus shows what it
turns that colour into. Phocus has no scripting, but its "hot folder" exports any raw dropped into a
watched folder, and it can be switched on from the command line, so the loop runs unattended.

- `tools/make_synthetic_raw.py HOST.3fr OUT.3fr`: copies a real raw and overwrites the active area with
  flat 32 px patches of chosen white-balanced camera RGB (using the host's AsShotNeutral to set raw
  values), on an L* 50 grey background. Masked borders and all metadata kept; RawDataUniqueID changed.
  Patch values spaced evenly in L* per channel (22 or 33 levels). Patches placed in a central square,
  shuffled, within ~0.32–0.34 of the half-diagonal.
- Capture One renders it through the probe profile (device RGB = ROMM) → exact camera RGB per patch.
  Check: channel correlation 0.999 with intended values, greys exactly neutral, patches flat to ~0.15%.
  Capture One's Linear Response maps raw linearly with a gain ≈ 0.75 above the shadows and lower gain in
  deep shadows; raw white ≈ device 0.89 (γ1.8 encoded).
- Phocus renders it via its **hot folder**, driven without the GUI:
  - preferences domain `dk.hasselblad.phocus`: `AutoExport = 1`, `exportDirectory = <folder>`,
    `lastOutputPreset` = the ProPhoto 16-bit preset; quit Phocus, write prefs, relaunch;
  - Phocus watches the folder its browser last showed;
  - drop `<name>.3fr.phos` (sidecar with the wanted settings) first, then the raw; export appears in
    ~10 s (in a `Previews` subfolder, renamed with a counter; the sidecar's `ExportName` records it);
  - minimal sidecar: one `ImageSettings` entry from a real sidecar (Standard, HDR off, LensCorrection 0,
    ColorTemperature 0 = As Shot), new GUID, `ImageParameterCache` dropped, XML plist.
  - Phocus renders the patches perfectly flat (std ~1/65535): no local processing interferes.
- Scripting Phocus with AppleEvents triggers a macOS Automation prompt; avoided entirely.

### Finding 1: the DPReview offset comes from file metadata, not scene content

**In plain terms:** The same synthetic picture renders slightly differently in Phocus depending on which
camera's file it sits in: a small, even colour shift. So it comes from the file's metadata, not from
what was photographed. Capture One renders both identically.

Same synthetic patches written into a DPReview raw and into the owner's 35-100 raw (white balance nearly
identical, AsShotNeutral within 0.4%):

| Phocus output, same pixels | median ΔE00 | greys dL / da / db |
|---|---|---|
| owner's 35-100 host − DPReview 35-100 host | 0.34 | +0.38 / −0.66 / +0.74 |

Capture One's input was identical (|Δdevice| < 0.0001): Capture One ignores whatever differs.

### Finding 2: it's one factory calibration value in the maker note

**In plain terms:** Swapping metadata between files one piece at a time narrowed the cause to a single
4-byte value: two numbers in a factory calibration record that each camera writes into every raw.
Phocus uses them to make every body render alike; Capture One ignores them. So a Capture One profile is
tied to the camera it was made on, and other X2D IIs may differ slightly.

Fields differing between the hosts: Software (firmware), CameraSerialNumber, MakerNote (29,840 vs 23,188
bytes), trivially AsShotNeutral and WhiteLevel. Transplants into an otherwise identical file:

- firmware string: no effect (byte-identical output); serial: no effect.
- MakerNote: a TIFF IFD (59/62 entries) with absolute offsets. Same-size entries swapped in place:
  - all swappable entries: reproduces the colour shift (greys a*/b* within 0.03), lightness half way;
  - first group (0x0005, 0x000f, 0x0017, 0x0019, 0x0042, 0x0047, 0x004b, 0x0060, 0x0061, 0x0063): same;
  - second group (0x0064–0x007e, large blocks): no effect;
  - ID strings 0x0060/0x0061 (sensor/lens IDs?): no effect; numeric fields: no effect;
  - **0x0019 alone: full colour shift.** 0x0017, 0x0042, 0x004b: no effect.
- **0x0019** (512 bytes) is a factory calibration record: starts with ASCII `GAIN`; contains a date
  (DPReview body 2025-05-05, owner's 2025-11-21), tool string `isctool 4.0.50` / `4.0.51`, a sensor ID string, and a trailing checksum-like block. It is identical across every
  owner frame (two lenses, Sep–Oct 2026), so per body, not per shot.
- Byte-level: only bytes 84–87 matter (copying date, ID and checksum alone: no effect; Phocus doesn't
  validate). Those 4 bytes are **two IEEE half-floats**: owner (2.600, 2.260), DPReview (2.363, 2.518).
  Sweep: 0 → mild shift (likely "no calibration" fallback); arbitrary floats decoding to garbage halves →
  huge green cast. Reading: per-body channel sensitivity gains (probably R/G and B/G at a reference
  illuminant) that Phocus uses to normalise every body to a reference. Capture One ignores them, so a
  Capture One profile is inherently per body.
- Unexplained remainder: ~0.2 L* of lightness between the two hosts, in maker-note entries that can't be
  swapped in place (pre-release-only 0x0072/0x0073; newer focus fields 0x0081–0x0083).

### Finding 3: lens metadata changes Phocus's rendering (lens corrections off)

**In plain terms:** The same picture tagged as shot with the XCD 28P comes out slightly darker in Phocus
than when tagged as the XCD 35-100, even with lens corrections turned off. Phocus applies some
lens-specific processing whatever that setting says.

Same patches, owner's body, XCD 28P host vs XCD 35-100 host; then the 28P host with its AsShotNeutral
set equal to the 35-100 host's:

| Phocus − E4 on greys | dL | da | db |
|---|---|---|---|
| 35-100 host | +0.52 | −0.11 | −0.20 |
| 28P host, own WB | −0.27 | −0.05 | +0.26 |
| 28P host, 35-100's WB | −0.27 | −0.12 | +0.02 |

So at the frame centre the lens changes brightness by about −0.8 L* (28P darker), with little colour.

### Finding 4: E5, a profile fitted on synthetic patches only

**In plain terms:** A profile fitted only on synthetic squares, with no real photos at all, matched real
photos as well as the previous best profile on the 35-100. On the 28P it did worse, which led to the
next finding.

33³ colours per lens, 3 files each, hosted in the owner's raws (35-100: B0000727; 28P: B0000931); ~33,000
usable patches per lens; edge lift removed per patch with each lens's measured lift.

| Owner's real frames (none used in E5) | E5 synthetic only | E4 |
|---|---|---|
| 35-100 held-out (623, 679, 703, 829, 863) | 0.71 | 0.71 |
| 28P held-out (912, 929, 939) | 1.08 | 0.82 |

Synthetic fit residual median 1.0, p90 7.1, dominated by extremely saturated patches beyond what the
legacy Lab encoding and the storable range allow.

### Finding 5: Phocus applies radial colour shading even with lens corrections off

**In plain terms:** Phocus warms the image toward the edges of the frame, strongly with the 28P and
mildly with the 35-100, even with lens corrections off. A camera profile treats every part of the
frame the same, so it can't do this. This explains most of the "colour difference between the two
lenses" seen earlier.

E5 minus Phocus on the owner's real frames, by distance from the centre (fraction of the half-diagonal):

| r | 28P dL / da / db | 35-100 dL / da / db |
|---|---|---|
| 0.0–0.1 | −0.09 / −0.25 / +0.07 | +0.14 / −0.13 / +0.27 |
| 0.2–0.3 | −0.10 / −0.30 / −0.59 | +0.15 / +0.02 / −0.04 |
| 0.4–0.5 | −0.30 / −0.49 / −1.76 | +0.24 / +0.20 / −0.44 |
| 0.5–0.6 | −0.42 / −0.56 / −2.44 | +0.29 / +0.28 / −0.72 |

At the centre E5 matches real photos within ~0.25 per channel; outward, Phocus adds yellow (and a little
red): +2.4 b* by r 0.55 on the 28P, ~+1 on the 35-100. Likely the per-lens "blue-corner shading" data in
the camera's lens files (reported by blog.tonalphoto.com in firmware 1.3.16.1/1.3.18.1), applied even
with lens corrections off. This is most of the "1 b* grey difference between the two lenses" seen in
real-image fits, which average over the frame. Scene-cast options in the sidecar are all off
(ApplySceneCastCorrection, ApplySceneEqualization, ApplySceneCalibration = False), so it isn't
scene-adaptive.

### Next

- Measure Phocus's spatial shading directly with synthetic flat fields per lens / aperture / focus,
  lens corrections off and on.
- Encode it as a Capture One LCC (flat-field) made from a synthetic raw, so Capture One reproduces
  Phocus's spatial behaviour; per-lens E5 profiles then only need to describe the centre.

### Finding 6: Phocus's spatial processing, measured with synthetic flat fields

**In plain terms:** Rendering a perfectly even grey file shows exactly what Phocus does across the frame.
It brightens the edges a little with corrections off and a lot with them on, and either way it reduces
blue toward the edges. The colour part barely changes with aperture; the brightness part (with
corrections on) depends on aperture and focal length.

Uniform L* 45 grey written into six host raws (35-100 at 35 mm f/11, 100 mm f/11, 45 mm f/8; 28P f/8 ∞,
f/4 ∞, f/4 0.3 m), each exported twice: sidecar `LensCorrection` 0 (off) and 4 (vignetting on). Phocus's
output inverted per 64 px block through the centre-accurate E5 LUT gives its per-channel gain map
(`tools/flat_gain.py`). Inversion residual 0.000 except a few clipped corners in the f/4 "on" renders.

At r = 0.6 (fraction of the half-diagonal):

| | 28P f/8 | 28P f/4 | 35-100 @35 | 35-100 @45 | 35-100 @100 |
|---|---|---|---|---|---|
| off: green gain (stops) | +0.245 | +0.191 | +0.056 | +0.050 | +0.007 |
| off: B−G (stops) | −0.104 | −0.102 | −0.034 | −0.028 | −0.022 |
| on: green gain | +0.910 | +1.182 | +0.333 | +0.253 | +0.095 |
| on: B−G | −0.101 | −0.096 | −0.033 | −0.027 | −0.021 |

At r = 0.9: 28P off +0.76 st (B−G −0.19), on +2.02 st at f/8 / +2.29 at f/4; 35-100 @35 on +0.84 st.

- Colour shading (blue reduced toward the edges, red slightly raised) is the same with corrections
  on or off, nearly aperture- and focus-independent on the 28P (B−G at r 0.9: −0.16 f/8, −0.22 f/4),
  and eases with focal length on the 35-100. On L* 45 grey it is +5 b* at r 0.6 on the 28P.
- With corrections off, Phocus still lifts brightness (28P +0.25 st at r 0.6; 35-100 +0.06 at 35 mm):
  matches the lift estimated from real frames (0.26 / 0.07).
- With corrections on, the brightness correction depends on aperture and focal length.
- Mild left/right and top/bottom asymmetries (~0.1 stop at the edges of the 28P map).

### Finding 7: Capture One's Light Falloff vs Phocus, and a synthetic LCC

**In plain terms:** Capture One's Light Falloff changes brightness only. At its maximum it matches Phocus
on the 35-100 and on the 28P at f/4, but not on the 28P at smaller apertures. The colour part can be
reproduced with an LCC (Capture One's flat-field correction): a synthetic LCC reference built from
Phocus's measurements does it, once its strength is corrected.

Capture One renders of the same flat fields through the probe profile (`tools/compare_gain_maps.py`):

- Light Falloff is brightness only (B−G exactly 0).
- vs Phocus corrections on, at LF 120, brightness difference at r 0.6 / 0.8 / 0.9:
  - 35-100 @35: +0.015 / −0.045 / −0.042; @45: +0.003 / −0.037 / −0.026; @100: +0.013 / −0.028 / +0.004.
  - 28P f/8: −0.12 / −0.30 / −0.40; f/4 ∞: +0.03 / −0.05 / −0.07; f/4 0.3 m: −0.01 / −0.12 / −0.14.
  So LF 120 matches the 35-100 and the 28P at f/4; the 28P at f/8 needs ~0.4 stop more at the corners.
  Phocus and Capture One scale vignetting correction with aperture differently.
- Synthetic LCC (`tools/make_lcc_raw.py`): reference raw = host metadata with pixel values set to the
  inverse of the wanted per-channel gain (2-D degree-6 polynomial fit to Phocus's map, rms 0.002 stop).
  Created as an LCC in Capture One by script (`create LCC`, `apply LCC`, `LCC color cast` on,
  `LCC uniform light` off).
- Result: Capture One reproduces Phocus's colour shading in shape at every radius but ~14% too strongly
  (B−G −0.119 vs −0.104 at r 0.6 on the 28P; −0.039 vs −0.034 on the 35-100). Scale the wanted colour map
  by ~0.875.

### Finding 8: Phocus's colour depends on white balance even within daylight

**In plain terms:** Phocus's colour changes with white balance even between ordinary daylight settings. A
profile fitted at one white balance is excellent near it and less good on warmer or cooler shots.

E5b (synthetic grid at one host white balance, Phocus gain maps applied to the patches) scored at the
frame centre (r < 0.2, per-pixel tetrahedral weights) on every owner frame: 0.25–0.65 on most, but
0.8–1.2 on the warmest-balanced frames. Across frames, centre error vs |log(B/R neutral ratio relative
to the host)|: correlation +0.74 (da, db also correlate, +0.63 / +0.59). A ~25% change in the B/R
balance (e.g. B/R 1.32 vs 1.74 on the 28P) costs ~0.5–1 ΔE00. HNCS is "strongly linked to white balance
parameters" (Hasselblad, via blog.tonalphoto.com), and that holds inside 4700–6300 K too.

White balance can be set synthetically: write another frame's AsShotNeutral into IFD0 and the maker-note
copy (0xc628). Capture One reads it (As Shot 4667 K … 6306 K for the five chosen neutrals); Phocus does
too (its grey patches stay at a* −1.3, b* +0.4–0.5 across all five, i.e. consistently white-balanced; HNCS
renders neutral grey slightly green).

### Finding 9: E5c, synthetic only, at six white balances

**In plain terms:** Fitting across six white balances, still from synthetic data only, gives the best
profiles so far. They beat the profile fitted to real photos on photos neither ever saw.

Per lens: the 33³ grid at the host white balance plus 22³ grids at five owner frames' neutrals (926, 939,
701, 746, 901: B/R from 1.32 to 1.86); ~82,000 patches; Phocus gain map (corrections off, same host)
applied per patch; one LUT. No real photos used.

| Frame centre (r < 0.2), median ΔE00 | E5c | E5b | E4 |
|---|---|---|---|
| 35-100, all 19 frames | 0.52 | 0.56 | 0.55 (mostly its training frames) |
| 35-100, E4's held-out 5 | 0.52 | — | 0.72 |
| 28P, all 10 frames | 0.59 | 0.68 | 0.69 |
| 28P, E4's held-out 3 | 0.50 | — | 0.69 |
| warmest frames 917 / 926 / 939 | 0.92 / 0.92 / 0.57 | 1.11 / 1.17 / 0.80 | — |

Synthetic fit residual median rises to ~1.5 because one LUT can't follow the white-balance dependence
exactly; it settles on the average, which is what a single Capture One profile can do.

### Finding 10: how Capture One's LCC colour correction works, and the full chain on real photos

**In plain terms:** Capture One's LCC shifts the colour of the whole image, not just the edges, because
it corrects relative to the reference's average colour. Building the opposite shift into the profile
cancels it. With that, the combination of a per-lens profile, Light Falloff 120 and a colour LCC
matches Phocus better than before across the whole frame. What remains on the 28P is edge brightness at
f/5.6 and narrower.

- Capture One's LCC colour correction, measured on synthetic references:
  correction_c(x) = (mean of reference_c / reference_c(x))^1.14, per channel relative to green.
  Check: 28P reference blue mean/centre 1.0675 → predicted centre gain 1.0773, measured 1.0774;
  red 0.9665 → 0.9619 predicted, 0.9616 measured. So beyond the 14% overshoot, the LCC also shifts the
  whole image by the reference's mean colour (a white-balance-like shift), which no reference pattern can
  avoid (scaling the reference cancels).
- Compensation: profile variants for use with the LCC ("HNCS E5 28P LCC", "HNCS E5 35-100 LCC") with the
  exact inverse per-channel gain in the ICC input curves (`write_profile.py --input-gain`):
  28P R 1.0365, B 0.9380; 35-100 R 1.0028, B 0.9830. The LUT itself is unchanged.
- Final colour LCC references: mean of the landscape-host corrections-off maps (28P: f/8 and f/4;
  35-100: 35 mm and 100 mm), colour part only, scaled by 0.875; written into a host raw of the same lens.
- Full chain vs Phocus with lens corrections ON, real photos, whole frame (r < 0.95), median ΔE00:

| frame | E4 + LF 120 | E5 (LCC variant) + LF 120 + colour LCC |
|---|---|---|
| 28P B0000901 (f/16) | 2.19 | 1.56 |
| 28P B0000917 (f/5.6) | 2.11 | 1.56 |
| 28P B0000931 (f/8) | 2.37 | 1.43 |
| 35-100 B0000623 (35 mm f/11) | 1.08 | 0.76 |
| 35-100 B0000679 (35 mm f/11) | 1.10 | 0.69 |
| 35-100 B0000702 (50 mm f/8) | 0.65 | 0.73 |
| mean 28P / 35-100 | 2.23 / 0.94 | 1.51 / 0.73 |

  By radius, 35-100 corners (r 0.8–1.0) went from 1.2–1.6 to 0.7–0.9; 28P edge colour error (db) from
  −6/−7 to −0.2/−2.8. The 28P's remaining error is brightness at the edges (dL −3 to −7 at r 0.6–1.0)
  at f/5.6–f/16, where Light Falloff 120 under-corrects (Finding 7).
- First attempt without the compensating input gain was worse than E4 (28P 3.40): the LCC's global shift
  made the centre −3.5 b* too blue. The flat-field test had hidden it by normalising to the centre.

### Finding 11: what HNCS is made of

**In plain terms:** For everyday colours, HNCS behaves mostly like a classic camera profile: a colour
matrix followed by a tone curve applied to each channel separately (not a "hue-preserving" curve). That
alone gets within about 1 ΔE00. The rest is genuinely three-dimensional, so colour adjustments depend
on brightness too, and saturated colours are reshaped in ways no simple model follows. Phocus's
film-curve setting changes only the tone curve: Standard and a brighter alternative share the same
colour. (Correction, Finding 19: that alternative isn't the Reproduction response.)

35-100, host white balance, ~33,000 synthetic patches with Phocus's spatial gain removed; simple
models fitted on camera-RGB saturation < 0.6 (s = 1 − min/max of white-balanced camera RGB), scored on
held-out patches (median ΔE00):

| model | s 0–0.3 | 0.3–0.6 | 0.6–0.85 | 0.85–1 |
|---|---|---|---|---|
| A 3×3 matrix → ProPhoto → one tone curve per channel | 1.02 | 1.05 | 2.87 | 6.94 |
| B 3×3 matrix → tone curve on luminance (hue-preserving) | 3.05 | 3.74 | 5.85 | 7.48 |
| C A + 2.5-D hue/saturation table (36 hue × 10 chroma bins) | 0.84 | 0.83 | 2.79 | 6.93 |
| D full 33³ LUT | 0.43 | 0.27 | 0.52 | 1.34 |

- Neutral tone curve (camera green linear, Capture One's scale, where 0.18 ≈ middle grey): L* 24 at
  −3 EV, 38 at −1.8, 52 at −1.0, 73 at +0.15, 90 at +1.2, 97 at +1.7, 99 at +2.0. HNCS greys sit at
  a* −0.6 to −1.8 (slightly green).
- Hue when a colour is brightened 1 and 2 stops (same camera RGB ratios): red, yellow, cyan, blue,
  magenta within ±1.5°; green −3.5°; orange and skin +6–7° as they approach the highlights.
- 6% of all synthetic patches (all extreme) come out with a ProPhoto channel clipped at 0; none at full
  scale. Output chroma reaches C* 175.
- Sidecar `FilmCurve` (Standard = 4): values 1–5 render identically; 0 renders about a stop brighter in
  the midtones with a gentler highlight roll-off (grey L* 57.9 vs 43.6; 88.0 vs 80.2), with chroma
  unchanged (median ratio 1.004). I took this for the blog's Reproduction response; Finding 19 shows
  Reproduction is `ColorModel` 1, and `FilmCurve` 0 is the film-curve menu's "As Shot".

### Finding 12: no focus-distance dependence on the 28P; an exposure tag mimics one

**In plain terms:** Two real 28P frames at f/4, one focused at infinity and one at 0.24 m, got
slightly different vignetting correction in Phocus, which looked like focus mattering. It was
the exposure tag: one file carries BaselineExposure +0.76. Swapping just that tag swaps the
correction maps exactly. Phocus's correction shrinks slightly as the image gets brighter.

Synthetic L* 45 flat fields, lens corrections on (vignetting), hosts B0000939 (28P f/4, 0.24 m,
BaselineExposure +0.764) and B0000926 (28P f/4, ∞, BaselineExposure 0); green gain at r 0.4 / 0.6 / 0.7:

| file | gain (stops) |
|---|---|
| near focus, own BE (+0.764) | +0.562 / +1.126 / +1.484 |
| far focus, own BE (0) | +0.590 / +1.184 / +1.569 |
| near focus, BE set to 0 | +0.591 / +1.184 / +1.569 |
| far focus, BE set to +0.764 | +0.561 / +1.126 / +1.483 |

- No effect at all (byte-identical output) from: maker-note focus fields 0x0081–0x0083 transplanted
  from the far file; EXIF SubjectDistance set to infinity. This confirms blog.tonalphoto.com's finding
  that Phocus ignores these.
- Transplanting every same-size maker-note entry from the far file: no change within r ≤ 0.6 (corners
  of the near file clip at +0.76 EV, so not comparable there).
- The correction's effective gain is ~0.06 stop smaller at r 0.6 when the image is 0.76 EV brighter.
  (Correction, Finding 22: that's Capture One's raw response, which the gain map is measured through;
  as raw gains the two are identical.)
- The blog's focus-distance dependence was measured on the XCD 55V and 90V. This doesn't contradict it,
  but exposure differences between frames (BaselineExposure) can look like a focus effect.

### Finding 13: how Phocus uses the factory gains, and per-body profiles

**In plain terms:** The two calibration numbers each camera stores (Finding 2) act like small
adjustments to how strongly Phocus reads the blue and green channels. A sweep of synthetic files with
different values gave a simple formula. With it, a profile made on one camera can be adjusted for any
other X2D II from a single raw of that camera, and tested on 32 photos from another body it removes
that body's colour offset.

Owner's 35-100 synthetic grid (host white balance) with the two half-floats changed one at a time.
For each, the camera-RGB multipliers (G fixed at 1) that make the E5 table reproduce Phocus's new
output:

| gains (g1, g2) | greys Δa* / Δb* | kR | kB | residual after fit |
|---|---|---|---|---|
| 2.200, 2.260 | +0.91 / −0.63 | 1.0118 | 1.0147 | 0.66 → 0.23 |
| 2.400, 2.260 | +0.45 / −0.31 | 1.0058 | 1.0072 | 0.33 → 0.11 |
| 2.800, 2.260 | −0.45 / +0.32 | 0.9942 | 0.9928 | 0.33 → 0.12 |
| 3.000, 2.260 | −0.90 / +0.63 | 0.9884 | 0.9859 | 0.65 → 0.23 |
| 2.600, 1.960 | −0.25 / +0.64 | 1.0000 | 0.9890 | 0.35 → 0.03 |
| 2.600, 2.460 | +0.15 / −0.38 | 1.0000 | 1.0064 | 0.20 → 0.02 |
| 2.600, 2.660 | +0.30 / −0.74 | 0.9999 | 1.0128 | 0.40 → 0.03 |

- Second value: a pure blue multiplier, ln kB = 0.0775 · ln(g2 / 2.260).
- First value: mostly an inverse green multiplier (red and blue together), ln kR = −0.0705 · ln(g1 / 2.600),
  ln kB = −0.0875 · ln(g1 / 2.600).
- The effect is weak (a 15% change in a stored value moves a channel ~1%), so the stored numbers aren't
  used as raw sensitivity ratios directly.
- Prediction for the DPReview body (2.363, 2.518): kR 1.0068, kB 1.0169. Applied to E5 on 32 DPReview
  photos (frame centre): grey offset a*/b* −0.52/+0.53 → −0.08/−0.03; median 1.23 → 1.02 (held-out 9:
  1.02 → 0.95).
- `tools/body_gains.py RAW` reads a raw's record and prints the profile input gain for that body.

## 2026-11-15 (round 3): closing the gaps

Scripts: `tools/round3/` (they read the saved measurements in the local `data/synth/r3/`),
`tools/hncs_structure.py`, `tools/hncs_wb_structure.py`. Logs: `data/logs/r3_*.txt`,
`data/logs/hncs_structure_*.txt`, `data/logs/hncs_wb_structure_*.txt`.

### Finding 14: HNCS is a colour stage, then a tone curve per channel in "Hasselblad RGB"

**In plain terms:** Phocus first converts the camera's colour, then applies its tone curve to each
channel separately in an RGB space of Hasselblad's own, whose definition ships inside Phocus as
`Hasselblad RGB.icc`. Knowing the space, a colour matrix plus that tone curve explains saturated colours
far better than before; what's left is a 3D colour table that also depends on lightness. Those are the
building blocks of a Lightroom (DNG) profile, except Lightroom applies its tone curve in a different
space and in a different way. White balance changes the colour matrix, like a DNG profile with two
calibration illuminants. The 28P and the 35-100 behave the same.

**Stage order.** The owner's 35-100 grid rendered with sidecar `FilmCurve` 0 (a brighter response) and 4
(Standard). Each model tries to produce the FilmCurve 0 render from the Standard render; median ΔE00 on
moderate | saturated colours (camera-RGB saturation < 0.6 | ≥ 0.6):

| model | host WB | tungsten WB |
|---|---|---|
| no change | 7.01 \| 6.42 | |
| tone first: one curve on camera RGB, then the Standard colour transform | 1.72 \| 2.31 | |
| tone last: one curve on each channel, ProPhoto | 1.02 \| 2.17 | 0.99 \| 2.14 |
| tone last: one curve on each channel, Hasselblad L\* RGB (wider primaries) | 1.87 \| 2.98 | 1.60 \| 2.74 |
| **tone last: one curve on each channel, Hasselblad RGB** | **0.31 \| 0.19** (p90 0.39) | **0.32 \| 0.17** |
| tone last: lightness only (a\*, b\* kept) | 5.00 \| 4.12 | |
| tone last: luminance, RGB ratios kept | 6.23 \| 4.19 | |

- Hasselblad RGB (`HBImageProcessing.framework/…/Profiles/Hasselblad RGB.icc`): D50 primaries R (0.6491,
  0.2989, 0.0046), G (0.1886, 0.6575, 0.0430), B (0.1265, 0.0437, 0.7772); γ 2.2. Phocus also ships
  "Hasselblad L\* RGB v1" (ProPhoto primaries with an L\* curve) and "Hasselblad L\* RGB" (wider
  primaries); neither fits.
- This resolves Finding 11's puzzle (a per-channel curve, yet unchanged chroma across responses): the
  median chroma ratio 1.004 hid opposite shifts; hue moves a median 3.0° (p90 8.8°), as a per-channel
  curve predicts.
- Phocus's library symbols agree: a "gradation manager" keyed on the film-curve type, separate from the
  colour-correction filter (which takes sensor type, IR filter type, film-curve type and colour model).

**Structure with the right space.** Held-out patches, models fitted on saturation < 0.6 unless noted,
median ΔE00 by camera-RGB saturation (35-100 | 28P):

| model | 0–0.3 | 0.3–0.6 | 0.6–0.85 | 0.85–1 |
|---|---|---|---|---|
| A matrix → ProPhoto → per-channel curve (Finding 11) | 1.02 \| 1.25 | 1.05 \| 1.03 | 2.87 \| 2.69 | 6.94 \| 6.78 |
| A3 matrix → Hasselblad RGB → per-channel curve | 0.86 \| 1.02 | 1.01 \| 0.96 | 1.74 \| 1.71 | 2.22 \| 2.52 |
| A2 matrix → ProPhoto → Adobe-style curve (max/min channel, hue kept) | 1.05 \| 1.06 | 1.83 \| 1.82 | 4.54 \| 4.29 | 7.07 \| 6.94 |
| C A3 + 2.5-D hue/saturation table | 0.73 \| 0.66 | 0.68 \| 0.62 | 1.67 \| 1.64 | 2.18 \| 2.48 |
| E A3 + 3-D hue × chroma × lightness table (36 × 9 × 8), fitted on all | 0.63 \| 0.59 | 0.37 \| 0.37 | 0.74 \| 0.73 | 0.87 \| 0.82 |
| D full 33³ table (the profile) | 0.43 \| 0.49 | 0.27 \| 0.25 | 0.52 \| 0.48 | 1.34 \| 1.31 |

- With the curve in Hasselblad RGB, a matrix and a curve fitted only on moderate colours predict the most
  saturated ones at 2.2–2.5 instead of 6.8–6.9. Most of what looked like "3D reshaping of saturated
  colours" is the tone curve acting in a smaller space.
- Moderate colours still need lightness-dependent corrections: a value-independent table (C) leaves
  0.6–0.7; adding 8 lightness levels (E) brings it to 0.37, as good as the full table. So HNCS fits a
  DNG-style matrix + 3D LookTable + tone curve, but a DNG profile's tone curve runs in ProPhoto with
  Adobe's hue-holding method (A2), which fits worst; its LookTable has to absorb the difference.

**White balance.** Grids rendered at other As Shot white balances, against the table fitted at the host
white balance, after a correction fitted on moderate colours (near-whites above L\* 93 left out); median
ΔE00, moderate | all:

| lens, R/B of AsShotNeutral | no correction | WB gains (diagonal) | 3×3 matrix before the table | 3×3 after (XYZ) |
|---|---|---|---|---|
| 35-100, host 0.625 (reference) | 0.22 \| 0.92 | 0.24 \| 0.89 | 0.21 \| 0.88 | 0.20 \| 1.02 |
| 35-100, 0.759 (warmest) | 1.06 \| 2.25 | 1.10 \| 2.18 | **0.37 \| 1.57** | 0.43 \| 2.33 |
| 35-100, 0.691 | 0.66 \| 1.62 | 0.65 \| 1.52 | 0.30 \| 1.21 | 0.32 \| 1.59 |
| 35-100, 0.538 (coolest) | 0.62 \| 1.48 | 0.63 \| 1.46 | 0.34 \| 1.37 | 0.31 \| 1.45 |
| 28P, 0.759 | 1.32 \| 2.56 | 1.40 \| 2.50 | **0.43 \| 1.74** | 0.51 \| 2.69 |

- For moderate colours, the white-balance dependence is a change of colour matrix: off-diagonal terms
  grow steadily with white balance (±0.02–0.05 between R/B 0.54 and 0.76), with opposite signs on either
  side of the host. Plain white-balance gains do nothing.
- Saturated colours change by more than a matrix (1.57 vs the reference 0.88), so the table or the
  matrix's effect on saturated colours also moves.
- Practical consequence: a tungsten profile (open question 2) could start from E5 plus a matrix
  extrapolated along this trend, then be refined on tungsten grids.

### Finding 15: the sidecar keys behind Phocus's response options

**In plain terms:** Two settings in Phocus's sidecar change the rendering of X2D II files. `FilmCurve` 0
gives a brighter tone curve with the same colour stage; `ColorModel` 1 changes colour as well as tone.
The other values of those keys, and the `Mode` key, do nothing. Phocus's Reproduction tool offers four
responses (Standard, Reproduction, Reproduction Low Gain, Negative); Finding 19 maps them to
`ColorModel` values.

Same grid, one key changed from the Standard sidecar (`FilmCurve` 4, `ColorModel` 0, `Mode` 0):

| setting | median ΔE00 vs Standard | grey L\* at camera 0.05 / 0.18 / 0.45 | chroma ratio | median \|Δhue\| |
|---|---|---|---|---|
| Standard | — | 29.4 / 57.5 / 84.9 | | |
| `FilmCurve` 0 | 6.10 | 42.3 / 70.8 / 91.3 | 1.004 | 3.0° |
| `FilmCurve` 5 (1–3 earlier) | 0.00 | same | | |
| `ColorModel` 1 | 3.63 | 36.9 / 59.6 / 87.6 | 1.058 | 1.9° |
| `ColorModel` 2, 3; `Mode` 1, 2 | 0.00 | same | | |

- `FilmCurve` 0 is a pure tone change (Finding 14: one Hasselblad RGB curve explains it to 0.31).
  `ColorModel` 1 isn't: no per-channel curve in any of the three spaces explains it (best 2.5), so it
  selects a different colour transform.
- Phocus's menu strings for `FilmCurve` are As Shot, Standard, Low Contrast, High Contrast, Old Standard,
  Linear (0–5), yet 1–5 render identically for X2D II files and real sidecars carry 4. Those names
  probably belong to older backs.

### Finding 16: the factory gains are a green gain and a blue gain

**In plain terms:** The two calibration numbers each X2D II stores (Finding 2) act as a green gain and a
blue gain on the camera's raw colour, with no effect on red. The effect is the same for both lenses and
nearly the same at tungsten, and changing both numbers combines the two effects. A file with the numbers
zeroed renders as if they were about 2.1. Adding the green gain to the per-body tool made the check
against another camera's photos slightly better, including lightness.

Camera-RGB gains (all three free) that make the E5 table reproduce Phocus's change, and the median ΔE00
left over:

| grid | gains (g1, g2) | kR | kG | kB | left over |
|---|---|---|---|---|---|
| 35-100, host WB | 2.20, 2.26 | 1.0009 | 0.9881 | 1.0013 | 0.05 |
| | 3.00, 2.26 | 0.9991 | 1.0118 | 0.9987 | 0.05 |
| | 2.20, 2.66 (both) | 0.9996 | 0.9871 | 1.0129 | 0.06 |
| | 0, 0 | 1.0015 | 0.9825 | 0.9940 | 0.07 |
| 28P | 2.20, 2.26 | 1.0014 | 0.9888 | 1.0020 | 0.06 |
| | 2.60, 2.66 | 1.0000 | 1.0001 | 1.0129 | 0.03 |
| 35-100, tungsten WB (R/B 1.28) | 2.20, 2.26 | 1.0005 | 0.9911 | 1.0052 | 0.08 |
| | 2.60, 2.66 | 1.0003 | 1.0026 | 1.0145 | 0.07 |

- Rule: **kG = (g1 / 2.600)^0.077, kB = (g2 / 2.260)^0.078**, kR = 1 (owner's body as reference).
  Finding 13's formula (red and blue, green fixed) gave the same colour but missed the brightness change
  (grey L\* −0.28 when g1 drops to 2.2); that was its 0.23 left over.
- Lens: none (28P matches the 35-100 to 0.001). White balance: at tungsten the green effect is about a
  quarter weaker and brings a small blue change; the same rule still leaves only 0.24 vs 0.21.
- Both changed: the effects multiply (kG 0.9871 vs 0.9881 alone; kB 1.0129 vs 1.0128 alone).
- Zero: equivalent to g1 ≈ 2.07, g2 ≈ 2.09 under the rule, so probably a default near 2.1.
- Check on 32 DPReview photos (body gains 2.363, 2.518), frame centre, E5 35-100:

| per-body correction | median ΔE00 | median ΔL\* | greys Δa\* / Δb\* |
|---|---|---|---|
| none | 1.23 | +0.71 | −0.52 / +0.53 |
| Finding 13 formula (1.0068, 1, 1.0169) | 1.02 | +0.70 | −0.08 / −0.03 |
| green/blue rule (1, 0.9927, 1.0085) | **0.96** | **+0.56** | −0.08 / +0.01 |

- `tools/body_gains.py` now prints `--input-gain 1,kG,kB`.

### Finding 17: no focus-distance dependence on the 35-100, or on the 28P at f/5.6 and f/8

*Superseded by Finding 31: the near/far pairs here were chosen by EXIF SubjectDistance, which doesn't
record where these lenses were focused.*

**In plain terms:** Changing where the lens was focused doesn't change Phocus's corrections for either
lens, from the centre to the corners, with lens corrections on or off. The blog's focus effect was on the
XCD 55V and 90V, which remain untested here.

Synthetic L\* 40 flat fields in matched pairs of the owner's raws (same focal length and aperture,
BaselineExposure set to 0 in all); green gain near − far, in stops, at r 0.6 / 0.8 / 0.95, and the largest
colour (B/G, R/G) difference:

| pair | corrections | green difference | colour |
|---|---|---|---|
| 35-100 100 mm f/8, 5.7 m vs ∞ | on | −0.000 / −0.002 / +0.013 | ≤ 0.001 |
| | off | −0.000 / −0.001 / −0.001 | ≤ 0.001 |
| 35-100 40 mm f/8, 1.9 m vs 5.7 m | on, off | ≤ 0.000 everywhere | ≤ 0.001 |
| 28P f/8, 0.75 m vs ∞ | on | +0.002 / +0.003 / −0.001 | ≤ 0.001 |
| 28P f/5.6, 0.75 m vs 7.9 m | on | +0.004 / +0.004 / −0.000 | ≤ 0.005 |

- The closest matched 35-100 frame was 1.9 m, so close focus on the zoom isn't covered.
- Side note: the 28P's correction at r 0.95 is +2.46 stops at f/5.6 and +2.29 at f/8 (+0.84 for the
  35-100 at 40 mm f/8).

### Finding 18: ColorModel 1 is a separate rendering, and the working space changes colour too

**In plain terms:** `ColorModel` 1 is a complete alternative rendering. It starts from the same colour
stage as Standard, then adds a little exposure and saturation and a much lighter shadow curve, and treats
saturated dark colours differently. When it's on, `FilmCurve` is ignored. Phocus's "working space"
setting (`ColorWorkingSpace`) also changes the rendering, by more than moving the tone curve to another
space would. All of the owner's real sidecars use the defaults that the profiles match.

`ColorModel` 1 against Standard (`tools/round3/colormodel.py`), held-out median ΔE00 by camera-RGB
saturation (0–0.3 / 0.3–0.6 / 0.6–0.85 / 0.85–1):

| model of the ColorModel 1 render | | | | |
|---|---|---|---|---|
| Standard as is | 4.57 | 2.92 | 3.01 | 3.61 |
| Standard with a different camera matrix in front | 4.79 | 2.97 | 3.00 | 3.79 |
| the raw's own DNG ColorMatrix1 + a Hasselblad RGB curve | 2.37 | 2.01 | 2.64 | 3.03 |
| Standard with a new Hasselblad RGB curve | 1.78 | 1.79 | 2.19 | 2.80 |
| **Standard's colour stage, then a new 3×3 and a new curve in linear Hasselblad RGB** | **0.57** | **0.49** | **0.77** | **1.09** |

- Greys at camera level 0.011 / 0.028 / 0.165 / 0.50: L\* 14.1 / 25.4 / 59.6 / 93.4, against Standard's
  4.7 / 15.8 / 57.5 / 88.8. Grey tint about the same.
- The 3×3 is about 1.11 on the diagonal and −0.01 to −0.04 off it: roughly +7% exposure and +5%
  saturation. What's left is in saturated dark and mid colours (L\* < 70: median 1.8–2.6; lighter: 0.5).
- Not colorimetric: the raw's DNG matrix fits it worse than it fits Standard (2.0–2.4 vs 1.4–1.7).
- `ColorModel` 1 + `FilmCurve` 0 renders identically to `ColorModel` 1 alone (0.00).

`ColorWorkingSpace` (`tools/round3/working_space.py`, `colormodel_combos.py`):

| value | median ΔE00 vs Standard | grey L\* at camera 0.028 / 0.165 / 0.428 | chroma ratio | median \|Δhue\| |
|---|---|---|---|---|
| 0 (default) | — | 15.8 / 57.5 / 84.9 | 1.000 | — |
| 1 | 6.55 | 14.0 / 61.3 / 86.9 | 0.883 | 5.2° |
| 2 | 4.58 | 14.0 / 61.3 / 86.9 | 0.920 | 3.3° |
| 3 | 2.93 | 14.0 / 61.3 / 86.9 | 1.047 | 1.7° |

- Values 1–3 share one grey curve (more midtone contrast than 0) and differ in saturation and hue.
- Undoing Standard's curve and applying a curve in another space (Hasselblad L\* RGB, ProPhoto, Rec.2020,
  Display P3, Adobe RGB, sRGB) doesn't explain them (best 2.6–3.3 on moderate colours), so the working
  space selects a different colour stage, as the library's colour-correction filter signature suggests.
  The same test on the Standard render returns it exactly (0.00), which checks the method.
- Rec.2020 comes close to Hasselblad RGB in that check (0.18): the two spaces' primaries are similar.
- The owner's real sidecars (33) all use `ColorWorkingSpace` 0, `ColorModel` 0, `FilmCurve` 4.
- Finding 20 names the working spaces.

### Finding 19: the Response menu is the ColorModel key

**In plain terms:** Phocus's Response menu (Standard, Reproduction, Reproduction Low Gain, Negative) is
stored as the sidecar's `ColorModel` value: 0, 1, 4 and 5. This came from reading Phocus's own code and
was confirmed by rendering each value: 5 inverts the picture, as a Negative response should. The
brighter `FilmCurve` 0 is a different, older setting, not Reproduction.

**From the code.** The Reproduction tool's menu is bound to an `imageProperties.responseIx` property in
the Phocus app. Its accessors (found through the Objective-C method lists of the arm64 binary and
disassembled with `otool`) read and write one integer in the image's correction settings:
- `responseIx` getter: stored value v → v − 2 if v > 1, else v;
- `setResponseIx:`: menu index i → stores i + 2 if i > 1, else i;
- `reproductionMode` getter: stored value == 1; `setReproductionMode:` stores 0 or 1.

So menu items 0, 1, 2, 3 store 0, 1, 4, 5, and stored values 2 and 3 are skipped. Of the sidecar keys,
only `ColorModel` behaves like that: 1 changes the rendering, and 2 and 3 render as Standard (Finding 15).

**Confirmed by rendering** (`tools/round3/responses_all.py`):

| `ColorModel` | menu item | median ΔE00 vs Standard | chroma ratio | grey L\* at camera 0.011 / 0.057 / 0.165 / 0.428 / 0.6 |
|---|---|---|---|---|
| 0 | Standard | — | 1.000 | 4.7 / 29.4 / 57.5 / 84.9 / 91.9 |
| 1 | Reproduction | 3.63 | 1.058 | 14.1 / 36.9 / 59.6 / 87.6 / 98.7 |
| 4 | Reproduction Low Gain | 6.86 | 0.964 | 9.2 / 28.4 / 47.4 / 70.8 / 80.1 |
| 5 | Negative | 98.5 | 1.034 | 89.2 / 76.1 / 60.0 / 35.1 / 23.1 (inverted) |
| 2, 3, 6 | not in the menu | 0.00 | 1.000 | as Standard |

- Reproduction (Finding 18): Standard's colour stage, then about +7% exposure, +5% saturation and a much
  lighter shadow curve. Low Gain is darker throughout, with slightly less chroma.
- `FilmCurve`'s menu strings are As Shot, Standard, Low Contrast, High Contrast, Old Standard, Linear
  (0–5); the brighter rendering is "As Shot", which may not be offered for X2D II files.

### Finding 20: Phocus's working spaces

**In plain terms:** Phocus's Working Space menu offers Hasselblad RGB (the default) and Hasselblad L\*
RGB, stored as `ColorWorkingSpace` 0 and 1. The code knows four more values that aren't in the menu. All
but the default use an L\* encoding instead of a gamma curve, which changes the tone curve, and each one
also changes saturation. The default is the space where Finding 14 found the tone curve.

**From the code.** The menu (`imageProperties.workingSpaceIx` in the Reproduction tool) stores its index
unchanged; its items are "Hasselblad RGB" and "Hasselblad L\* RGB". In `HBRawCorrections`:
- `IsLStarWorkingSpace(ws)`: true for 1–5; `DeduceCompanding(ws)`: L\* companding for 1–5, otherwise gamma;
- `GetColorSpaceChangeMatrix(ws)`: identity for 0; for the others, a stored matrix (inverted where it
  holds primaries) times `kMHasselbladRGB`, except value 3, which skips the Hasselblad RGB step. The stored
  matrices are Hasselblad L\* RGB's primaries (1), XYZ → ProPhoto (2), the D50 white as a diagonal (3, 4)
  and ProPhoto's primaries (5).

**Rendered** (`tools/round3/working_spaces_all.py`):

| value | space | in the menu | median ΔE00 vs 0 | chroma ratio | grey L\* at camera 0.028 / 0.165 / 0.428 |
|---|---|---|---|---|---|
| 0 | Hasselblad RGB, gamma | yes (default) | — | 1.000 | 15.8 / 57.5 / 84.9 |
| 1 | Hasselblad L\* RGB, L\* | yes | 5.80 | 0.884 | 14.0 / 61.3 / 86.9 |
| 2 | ProPhoto primaries, L\* | no | 4.15 | 0.921 | 14.0 / 61.3 / 86.9 |
| 3 | XYZ (D50), L\*, no Hasselblad RGB step | no | 2.62 | 1.047 | 14.0 / 61.3 / 86.9 |
| 4 | XYZ (D50), L\*, via Hasselblad RGB | no | 11.16 | 0.747 | 13.9 / 61.2 / 86.8 |
| 5 | ProPhoto primaries, L\* | no | 4.15 | 0.921 | as 2 |

- Values 2 and 5 render identically (0.00), as the code predicts: two routes to the same matrix.
- All L\* spaces share one grey curve, different from the default's (more midtone contrast).
- Moving Standard's curve into these spaces doesn't reproduce them (Finding 18), so the colour stage also
  depends on the working space, as the colour-correction filter's signature (which takes it) suggests.

## 2026-11-15 (round 4): building on the findings

### Finding 21: tungsten profiles

**In plain terms:** Profiles for tungsten light, fitted the same way as E5 but on synthetic grids at
four tungsten white balances (about 3100–4200 K). On three real tungsten-lit photos they halve the
error of the daylight profiles. Use them by hand for tungsten-lit frames; Capture One can't switch
profiles by white balance.

- Grids: the E5 hosts (35-100 and 28P, owner's body), 22-level grids, As Shot neutral set with
  `make_synthetic_raw.py --neutral r,g,b` along the log-neutral line from the host's daylight neutral
  (0.368, 1, 0.589) through a DPReview tungsten frame's (0.569, 1, 0.445) at t = 0.5, 0.75, 1.0, 1.25.
  Capture One reads them as 4175 K, 3747 K, 3395 K and 3105 K (tints −3 to −12).
- Pairs saved with `tools/save_synth_pairs.py`; fitted with `fit_synthetic.py --pairs` (new option: fit
  from saved pairs). Refitting E5c from its saved pairs reproduces the shipped table exactly, which
  checks the path.
- Synthetic grids, 35-100, median ΔE00 moderate | all colours:

| grid (Capture One reading) | E5 (daylight) | E5 tungsten |
|---|---|---|
| warmest daylight grid (R/B 0.76) | 1.00 \| 2.13 | 2.05 \| 3.64 |
| 4175 K | 1.81 \| 3.32 | 1.26 \| 2.51 |
| 3747 K | 2.48 \| 4.42 | 0.53 \| 1.52 |
| 3395 K | 3.05 \| 5.42 | 0.49 \| 1.38 |
| 3105 K | 3.92 \| 6.81 | 1.24 \| 2.69 |

- Real tungsten-lit photos (DPReview, 35-100, As Shot, other body with its factory gains applied),
  median ΔE00 at the frame centre (r < 0.2):

| frame | E5 (daylight) | E5 tungsten (4 WBs) | tungsten fit on 2 WBs (3400–3750 K) |
|---|---|---|---|
| 2462814331 | 1.95 | **1.71** | 1.76 |
| 5727340084 | 2.77 | **1.07** | 1.21 |
| 8671272247 | 2.75 | **1.03** | 1.13 |
| mean | 2.49 | **1.27** | 1.37 |

- The fit's own error is larger than E5's (2.1 vs about 1.5 for all colours) because Phocus changes
  colour across the 1000 K range more than within daylight; a narrower fit did worse on real photos.
- The 28P tungsten profile is validated only on synthetic grids (no real 28P tungsten frames).
- Files: `profiles/… E5 28P LCC Tungsten.icm`, `… E5 35-100 LCC Tungsten.icm` (with the same LCC input
  gains as E5), four styles, `fit/lut_e5_*_tungsten.npz`, pairs in `fit/synthetic/pairs/HNCStung*.npz`.

### Finding 22: pre-processed raws match Phocus across the whole frame

**In plain terms:** Writing Phocus's lens correction and colour shading straight into a copy of the raw,
then rendering it in Capture One with the plain E5 profile and no Light Falloff or LCC, matches Phocus
(lens corrections on) to 0.3–0.8 ΔE00 over the whole frame, corners included: about twice as close as
the LCC route, and the 28P's aperture problem disappears. The catch that made the first try fail:
Capture One's raw response isn't proportional in the shadows, so a gain measured in its colour scale
has to be converted before it's written into the raw. The same effect explains Finding 12's
"correction shrinks for brighter images": Phocus's vignetting correction is a fixed multiplier on the
raw values.

**Method.**
1. For each test frame, a synthetic L\* 30 flat with the frame's own metadata (`make_synthetic_raw.py
   --flat 30`), rendered by Phocus with `LensCorrection` 4; `flat_gain.py` gives the per-channel gain
   map in Capture One's camera-RGB scale.
2. `make_preprocess_map.py` converts it to raw gains through Capture One's response
   (`fit/c1_response.npz`, measured from the E5 grids' nominal vs probe values) at the flat's level and
   BaselineExposure, and saves it by lens state (`fit/preprocess_maps/XCD28P_28mm_f8.0.npz` …).
3. `preprocess_3fr.py IN OUT --maps fit/preprocess_maps` multiplies each raw channel by its gain
   (interpolated per photosite) in a copy.
4. Capture One: style "HNCS E5 … (pre-processed)" = plain E5 profile (no LCC compensation), Linear
   Response, Light Falloff 0. Capture One's default Light Falloff for these lenses is 100, so the style
   has to set 0.

**Capture One's response** (white-balanced raw, as a fraction of 0.92 × white − black, to the camera RGB
a profile receives): ratio 0.17 at 0.01, 0.64 at 0.1, 0.74 at 0.4, 0.75 from 0.6 up (affine above 0.05:
0.761 × raw − 0.0096). Applying the map to raw values directly overshot (corrected flat L\* 49.8 vs
Phocus 45.6 at r 0.6; real frames dL +4 to +6 at the edges).

**Check on flats.** Corrected flat in Capture One vs Phocus's corrections-on flat, dL\* from centre to
r 0.9: +0.02 to +0.07 (28P f/8, BaselineExposure 0) and −0.04 to −0.10 (28P f/5.6, BaselineExposure
+0.458).

**Real photos**, whole frame (r < 0.95) vs Phocus with lens corrections on, median ΔE00:

| frame | E4 + LF 120 | E5 LCC + LF 120 + colour LCC (Finding 10) | **pre-processed + plain E5** |
|---|---|---|---|
| 28P B0000901 (f/16) | 2.19 | 1.56 | **0.72** |
| 28P B0000917 (f/5.6) | 2.11 | 1.56 | **0.80** |
| 28P B0000931 (f/8) | 2.37 | 1.43 | **0.48** |
| 35-100 B0000623 (35 mm f/11) | 1.08 | 0.76 | **0.43** |
| 35-100 B0000679 (35 mm f/11) | 1.10 | 0.69 | **0.31** |
| 35-100 B0000702 (50 mm f/8) | 0.65 | 0.73 | **0.65** |
| mean 28P / 35-100 | 2.23 / 0.94 | 1.51 / 0.73 | **0.67 / 0.47** |

- By radius the error is flat: 28P 0.4–0.9 and dL within ±0.3 from centre to r 0.8–1.0 (was dL −3 to −7).
  B0000702's remaining 0.65 is a uniform +0.5 L\*, the per-frame lightness offset of open question 4.
- Raw gains (green, stops) at r 0.4 / 0.6 / 0.8 / 0.9: 28P f/5.6 +0.37 / +0.84 / +1.64 / +2.08; f/8
  +0.38 / +0.84 / +1.50 / +1.91; f/16 +0.39 / +0.84 / +1.50 / +1.89; 35-100 35 mm f/11 +0.12 / +0.30 /
  +0.61 / +0.76; 50 mm f/8 +0.09 / +0.20 / +0.41 / +0.49.
- Highlight headroom: clipped photosites 0.0% on most frames, 0.1% on B0000702, 2.3% on B0000901 (bright
  sky in the f/16 corners, up to 5.4× gain).
- Finding 12 revisited: converted to raw gains, near/far focus and BaselineExposure 0/+0.764 give the
  same correction to within 0.002 stop at r 0.4–0.6 (0.015 at r 0.7). The 0.06-stop difference reported
  there was Capture One's response, not Phocus compressing its gain.

### Finding 23: highlight handling, Phocus vs Capture One

**In plain terms:** At default settings the two apps treat highlights the same: the HNCS tone curve
reaches white about a quarter stop before the sensor clips, in both. The differences are in what
happens once a sensor channel clips. Phocus keeps some colour where one channel has clipped (Capture One
goes to pure white), and Phocus's Highlight Recovery setting genuinely rebuilds a clipped channel from
the others, keeping tone up to about 1.5 stops past clipping while staying neutral. Capture One's
Highlight slider only compresses the tones below clipping. Neither can come from a profile; it happens
before the profile is applied.

Test raw (`make_synthetic_raw.py --device-npy … --scale 1.0`, values clipped at the white level like a
sensor): greys and 12 hues at saturation 0.3 / 0.6 / 0.9, each at 17 exposures from 2 stops under to 2
stops over the point where its brightest channel reaches white. Phocus via the hot folder (default;
`HighlightRecovery` 50 and 100; `HighlightDesaturation` 0 and 100; `AutoHighlight` 1). Capture One with
HNCS E5 35-100 (default; Highlight −100; White −100). Script: `tools/highlight_test.py`; log
`data/logs/highlight_test.txt`.

Greys, L\* at exposure (× the green channel's clip point):

| exposure | 0.5 | 0.71 | 0.84 | 1.0 | 1.41 | 2.0 | 2.83 |
|---|---|---|---|---|---|---|---|
| raw channels clipped | 0 | 0 | 0 | 1 | 1 | 2 | 3 |
| Phocus default | 90.6 | 96.8 | 99.1 | 100 | 100 | 99.9 | 99.8 |
| Phocus HighlightRecovery 50 | 80.5 | 87.6 | 91.0 | 94.2 | 98.7 | 100 | 99.9 |
| Phocus HighlightRecovery 100 | 66.9 | 72.8 | 76.2 | 80.9 | 87.4 | 95.9 | 100 |
| Capture One + HNCS | 90.6 | 97.0 | 99.3 | 100 | 100 | 100 | 100 |
| Capture One Highlight −100 | 85.4 | 91.1 | 93.6 | 98.0 | 100 | 100 | 100 |
| Capture One White −100 | 89.6 | 95.8 | 97.9 | 100 | 100 | 100 | 100 |

Colours with one raw channel clipped (137 patches), median L\* and chroma: Phocus default 98.8 / 16.7;
HighlightRecovery 50: 94.6 / 32.0; HighlightRecovery 100: 82.4 / 40.6; Capture One + HNCS 100.0 / 0.1;
Highlight −100: 97.1 / 6.2; White −100: 99.5 / 0.4. Below clipping the two apps agree (0.50 ΔE00 on 367
unclipped colour patches; with two or three channels clipped both are white, 0.3–0.5).

- Phocus's `HighlightRecovery` compresses the whole upper range (an unclipped grey at 0.84× goes from L\*
  99 to 76 at 100) and reconstructs the clipped channel: greys stay neutral (a\*, b\* within 0.6) and keep
  gaining tone until all three channels clip. Across the 9 exposures from the clip point to 2 stops over,
  it keeps 8 distinct steps at 100, against 4 for every other setting in either app.
- `HighlightDesaturation` and `AutoHighlight` changed nothing here (they may act only with recovery or HDR
  on, or on real images).
- Limits: uniform patches test per-pixel reconstruction only. Methods that borrow from neighbouring
  pixels would behave differently on real images; a frame with clipped sky would confirm the effect.

### Finding 24: automatic maps, and pre-processing validated on 23 unseen photos

**In plain terms:** Correction maps can now be made without touching Phocus: a script renders flat fields
through Phocus's hot folder and caches one map per lens, focal length and aperture (42 so far). With
them, the pre-processing route was checked on 23 of the owner's photos that no fit had used. It's the
closest route on every 28P frame and on most 35-100 frames, and its corners are as good as its centres.

**Automatic maps.** `tools/phocus_hotfolder.py` renders any raw in Phocus without its GUI. Phocus shows
(and watches) the last folder of its `ThumbnailsRecent` list (macOS bookmarks), so the helper appends a
bookmark to a private folder under `~/Pictures` (macOS didn't let Phocus read a folder under
`~/Documents`), copies the raw there with a sidecar, waits for the export to finish writing (Phocus
creates it empty first), and restores Phocus's preferences afterwards. `tools/auto_maps.py RAW…` makes
the maps for every lens state among the raws that `fit/preprocess_maps/` lacks: 34 new ones for the
owner's 28P and 35-100 frames, about 25 s each.

**Validation.** 7 28P and 16 35-100 frames not used by any fit (`data/val/frames.txt`), across focal
lengths and apertures (28P B0000911, 912, 916, 919, 926, 929, 939; 35-100 B0000633, 639, 646, 656, 658,
664, 671, 673, 689, 711, 722, 737, 773, 777, 822, 846). Reference: Phocus with lens corrections on (vignetting), As Shot. Capture One
renders at 50%, scored per 32 px block after alignment, leaving out clipped blocks and the busiest 30%
(`tools/score_render.py`). Median ΔE00 over the whole frame (r < 0.95):

| | pre-processed + plain E5 | E5 LCC + LF 120 + colour LCC | E4 + LF 120 |
|---|---|---|---|
| 28P (7 frames) | **0.65** (0.38–0.91) | 1.38 (1.09–1.86) | 2.07 (1.72–2.39) |
| 35-100 (16 frames) | **0.49** (0.26–0.61) | 0.64 (0.44–0.87) | 0.65 (0.35–1.20) |

- 28P corners (r 0.8–0.95): pre-processed 0.43–1.09 with dL within ±0.7; LCC route 1.85–4.26 and E4
  3.42–5.75, both several L\* too dark (dL −1.8 to −5.7).
- 35-100 corners: pre-processed 0.26–0.74; LCC route up to 1.55; E4 up to 2.36. On a few long-focal
  frames with little vignetting, E4 is as close or closer at the centre (B0000639, 658, 673, 711, 722).
- Highlight headroom: corner gains clip 1.5–2.0% of photosites on three frames (B0000912, B0000737,
  B0000671 at 1.0%), under 0.3% elsewhere.

### Finding 25: headroom doesn't work cleanly; real clipped sky; batch pre-processing

**In plain terms:** I tried to stop pre-processing from clipping bright corners by darkening the whole raw
and telling Capture One to brighten it back (BaselineExposure). Capture One does render it identically,
but it breaks how Capture One recognises sensor-clipped areas, so recovered skies get a hard band. The
corners that clip sit in HNCS's white shoulder anyway, so the option is dropped. A real photo with a
clipped sky confirms Finding 23: at defaults Phocus and Capture One + HNCS look the same, and Phocus's
Highlight Recovery rebuilds a smooth, plausible sky where Capture One's Highlight slider can't. A batch
tool now turns a folder of raws into corrected copies.

**Headroom test** (B0000737, 35-100 35 mm f/11, sky clipped in camera on 1.5% of photosites; the corner
gain clips another 0.5%):
- Scaling the corrected raw down 0.9 stop and raising BaselineExposure by 0.9: Capture One renders it the
  same as the unscaled copy (blocks away from white: median 0.045 ΔE00, p99 0.37, mean dL +0.01). So
  Capture One applies BaselineExposure before its raw response.
- Both score the same against Phocus (0.59 / 0.62); the newly clipped corner areas are L\* 100 in both.
- With Highlight −100 the scaled copy shows a hard band: photosites clipped in camera stay at white so
  Capture One treats them as clipped, but their scaled neighbours are 0.9 stop darker. Scaling them as
  well would hide the clipping from Capture One (it would tint them instead). Without scaling the sky
  stays smooth. `preprocess_3fr.py` keeps photosites clipped in camera at white and reports the newly
  clipped ones separately.

**Real clipped sky** (same frame), in blocks Phocus renders white (L\* > 97, 3212 blocks):

| rendering | median L\* | tone within blocks (std of Y) |
|---|---|---|
| Phocus default | ~100 | 0.0018 |
| Phocus Highlight Recovery 100 | 87.8 | 0.0048 |
| Capture One + HNCS, Highlight −100 | 99.8 | 0.0037 |

At defaults the two apps' skies look the same; Phocus's recovery gives a smooth, plausible gradient.

**Batch:** `tools/preprocess_folder.py IN_DIR OUT_DIR [--make-maps]` writes a corrected copy of every
raw (same file name) into another folder, skips existing copies, and with `--make-maps` makes missing
maps through Phocus first.

### Finding 26: E5 profiles for any lens, from Phocus alone

**In plain terms:** A new lens can now get its own E5 profile without Capture One in the loop. Capture One
turns each synthetic patch into exactly the same camera RGB whatever the lens or white balance, so its
side of every pair can be looked up from the measurements already made, and only Phocus has to render the
new lens's grids. Rebuilt this way, the 28P profile comes out within 0.16 ΔE00 of the shipped one.

- Capture One's camera RGB for each of the 46,577 patch values of the standard 22- and 33-level grids,
  across the 24 grids already measured (two lenses, 3100–6300 K): spread median 0.0003, max 0.005 in
  profile-input units, which is render noise. Saved as `fit/c1_patch_lookup.npz`.
- Capture One's per-channel response curve alone isn't enough: for about 7% of patches, extremely
  saturated ones with one channel near zero, Capture One mixes a little into the empty channel and
  compresses the strong one (a pure blue at nominal 0.85 comes out 0.47 instead of 0.63). The lookup
  carries that.
- `tools/auto_lens_profile.py RAW…`: three 33-level parts at the raw's white balance, five 22-level grids
  at the E5 daylight white balances, and an L\* 45 flat, all written into copies of one of the lens's raws
  (landscape, BaselineExposure set to 0); nine Phocus renders through the hot folder; pairs from the
  lookup; `fit_synthetic.py`; profile, plain style and "(pre-processed)" style. About five minutes.
- BaselineExposure trap: the first run used a host carrying +0.067 and came out 1.2 L\* brighter in the
  midtones than the shipped profile (+0.067 EV on HNCS's curve, about 18 L\* per stop there). Phocus
  applies it to the grids, but the lookup was measured at 0. The copies now get 0.
- Check, 28P rebuilt from scratch vs the shipped E5 28P, on the shipped 28P synthetic data (moderate
  colours, median ΔE00 vs Phocus): host white balance 0.28 vs 0.40; other daylight white balances
  0.57–1.22 vs 0.67–1.11; the two tables 0.16 apart.
- `preprocess_folder.py --make-profiles` runs it for any lens in a folder that has no E5 profile, and
  installs the result.

**Contact sheet.** `docs/images/contact-sheet.jpg` now shows six of the owner's frames (28P at f/4, f/5.6
and f/16; 35-100 at 35 mm and 100 mm): Phocus with lens corrections on, Capture One with pre-processed
copies and HNCS E5 (whole-frame median 0.25–0.75 ΔE00), and Capture One's defaults (2.95–6.27).
`tools/contact_sheet.py` now builds it from finished renders.

### Finding 27: Phocus normalises brightness per lens setting

**In plain terms:** Phocus brightens or darkens each photo slightly depending on the lens, focal length
and aperture, separately from its vignetting correction and whether or not lens corrections are on.
Capture One doesn't, and the pre-processing maps missed it because they're measured relative to the
frame centre. That explained most of the leftover per-frame lightness differences. The maps now carry
it, and the frames that were off by 0.3–0.5 L\* come within about 0.1.

- Flats at the same raw level (L\* 30, BaselineExposure 0) through Phocus, centre L\* by lens setting:
  28P 30.45 at f/5.6–f/16, 30.78 at f/4; 35-100 31.43–31.63 at 35 mm, falling to 30.90–30.99 at 50–65
  mm and back to 31.1–31.3 at 80–100 mm (31.69 at 100 mm f/4). Corrections on or off: the same.
- As raw gains relative to each profile's reference setting (28P f/8, 35-100 35 mm f/11, the lens states
  the E5 grids were made at): +0.020 stop for the 28P at f/4, −0.025 to −0.033 for the 35-100 at 45–75 mm.
  Small in stops, about 0.5 L\* in the midtones.
- Before the fix, the 29 validation frames' lightness offsets correlated +0.67 with this.
- `make_preprocess_map.py` now stores the flat's centre Lab; `preprocess_3fr.py` multiplies the map by
  the gain between that and the reference state's (`fit/lens_references.json`); flats get
  BaselineExposure 0; all maps remade.
- 14 validation frames re-rendered (lightness offset dL\*, median ΔE00, before → after): B0000702
  +0.49 → −0.17, 0.65 → 0.50; B0000646 +0.36 → −0.06; B0000671 +0.33 → −0.11; B0000658 +0.35 → +0.03;
  B0000711 +0.48 → +0.13; B0000822 +0.38 → +0.01, 0.37 → 0.25; B0000926 −0.26 → +0.10; B0000929
  −0.54 → −0.18, 0.75 → 0.51; B0000939 −0.39 → 0.00, 0.54 → 0.42. Mean |dL\*| 0.33 → 0.12 (11 frames
  scored first), mean median ΔE00 about 0.55 → 0.47 (14). B0000846, at the same setting as B0000702,
  went from −0.24 to −0.45: its remaining offset is something in that frame.

### Finding 28: corrected copies can open with the right settings

**In plain terms:** Capture One reads a small settings file next to each image the first time it sees it.
The batch tool now writes one for every corrected copy, so the copies open with the lens's E5 profile,
Linear Response and Light Falloff 0, and nothing needs applying.

- The sidecar is `CaptureOne/Settings<version>/<name>.cos` (Settings1690 for Capture One 16.9.0), XML.
  A sidecar with only the three settings was ignored and overwritten. With an image UUID, `Version`
  1530, a `VAR` holding `ProductID` and its own UUID, and the three settings, Capture One applies them and
  fills in the rest itself, including the image's own lens data. A full sidecar copied from another
  frame also works but carries that frame's aperture and distortion curves, so the minimal one is used.
- End to end: `preprocess_folder.py` into the session's Capture folder; all 14 copies opened with the
  right profile, Linear Response and Light Falloff 0.
- Capture One still applies its own distortion-correction default per image (100 on some frames, 0 on
  others); that's geometry, not colour, and left alone.

### Finding 29: where Phocus's lens behaviour comes from

**In plain terms:** Phocus doesn't read lens corrections from the raw. The camera records a 34-byte lens
identity in two maker-note entries, and Phocus looks the lens up in data built into the app. Copy those
two entries from a 28P raw into a 35-100 file and Phocus treats it as a 28P in every way: colour shading,
edge lift, vignetting correction and brightness.

35-100 flat field (35 mm f/11) with maker-note entries copied from a 28P raw (f/8); green gain and B−G in
stops at r 0.6 / 0.9, corrections off, and centre L\*:

| file | green gain | B−G | centre L\* |
|---|---|---|---|
| 35-100 flat | +0.07 / +0.24 | −0.04 / −0.07 | 31.43 |
| 28P flat | +0.29 / +0.89 | −0.13 / −0.21 | 30.42 |
| 35-100 + all 28P maker-note entries | +0.29 / +0.89 | −0.13 / −0.21 | 30.42 |
| 35-100 + 28P EXIF lens name only | as the 35-100 | | |
| 35-100 + 28P 0x0017 and 0x0018 | +0.29 / +0.89 | −0.13 / −0.21 | 30.42 |
| 35-100 + 28P 0x0018 only | 0 / 0 | 0 / 0 | 29.97 |

- Bisected from all 26 differing entries; the large blocks (0x0042, 0x004b, 0x0064–0x0074) do nothing.
  0x0018 (17 bytes) with 0x0017 (17 bytes) carries the lens identity; 0x0018 alone, mismatched with the
  rest, switches lens processing off entirely, which shows Phocus's "no lens" baseline (L\* 29.97 at the
  centre against 30.42 for the 28P and 31.43 for the 35-100 at 35 mm f/11).
- Corrections on behave the same way (all entries: +1.05 / +2.27 stops like the 28P).
- The raw can't hold the tables (the whole maker note is 22.5 KB). Phocus's HBUtil framework is 119 MB
  and names every XCD lens (21, 28P, 30, 35-75, 35-100E, 45, 45P, 65, 80, 90, 120, 135 …), which is
  where the lens data must live. blog.tonalphoto.com found the same per-lens files in the camera's
  firmware; Phocus carries its own copy.

### Finding 30: the lens identity decoded, and the 55V and 38V on an X2D II file

**In plain terms:** The two maker-note entries that identify the lens also carry the shot's aperture and
focal length, which Phocus reads from there rather than from EXIF. Lens identities from original-X2D raws
work on X2D II files, so Phocus's processing for lenses I don't own can be measured: the 55V and 38V each
get their own colour shading and vignetting correction. (The focus conclusion below is wrong: Phocus
reads focus from byte 6 of 0x0018, which these single raws couldn't show. See Finding 31.)

- 0x0018 (17 bytes) describes the lens: byte 1 a type code (0x43 for the primes, 0x53 for the 35-100),
  bytes 2–3 its shortest and longest focal-length codes (28P 0x27/0x27, 35-100 0x2f/0x53, 55V 0x3e/0x3e,
  38V 0x31/0x31).
- 0x0017 (17 bytes) is the shot: byte 8 is the aperture as 12 × APEX (0x30 f/4, 0x3c f/5.6, 0x48 f/8,
  0x54 f/11, 0x60 f/16); byte 14 the focal length, about 24 × log2(mm) − 76 (28 mm 0x27, 35 mm 0x2f,
  50 mm 0x3b, 55 mm 0x3e, 100 mm 0x53). Both decode correctly on all files checked.
- Raws: one 55V (f/4) and one 38V (f/11) from DPReview's original-X2D gallery (firmware 1.0.0, same
  uncompressed sensor layout, 33 maker-note entries including 0x0017/0x0018).
- On an X2D II 35-100 flat field (35 mm f/11, BaselineExposure 0), with each lens's 0x0017/0x0018:

| identity | centre L\* | corrections off: lift / B−G at r 0.6, 0.9 (stops) | corrections on: green gain at r 0.6, 0.9 |
|---|---|---|---|
| 35-100, 35 mm f/11 (host) | 31.43 | +0.07, +0.24 / −0.04, −0.07 | +0.39, +0.96 |
| 55V, f/4 | 30.16 | +0.21, +0.56 / −0.12, −0.17 | +0.68, +1.69 |
| 55V with byte 8 set to f/8 | 30.16 | unchanged | +0.62, +1.35 |
| 38V, f/11 | 30.43 | +0.15, +0.50 / −0.08, −0.16 | +0.70, +1.52 |

- Focus: changing any other byte of the 55V's 0x0017 (bytes 4–7, 9–13, 15, 16; the V-lens files carry
  non-zero bytes 11–13 where the 28P and 35-100 have zeros) changes nothing. Setting byte 14 outside the
  lens's range removes the vignetting correction and keeps the shading. The sidecar's `VLensDistance` and
  `VLensType` (0.5–100, with type −1 or 1) change nothing either; they're probably for V-system film lenses.
- Correction to Finding 12: maker-note 0x0081–0x0083 aren't focus fields. Across 28P frames from 0.24 m
  to infinity they read 25.8–28.8, 0–0.13 and 43.9–44.7 with no relation to distance (0x0081 and 0x0083
  look like temperatures).
- What this leaves of blog.tonalphoto.com's 55V/90V focus effect: it isn't driven by anything in these
  files. Two raws of the same lens at different focus distances, rendered in Phocus, would show whether it
  appears at all and where it comes from.

### Finding 31: Phocus reads focus from byte 6 of 0x0018

**In plain terms:** Phocus does change its lens correction with focus, and it reads where the lens was
focused from one byte of the lens descriptor that changes from shot to shot. On the 55V at f/2.5, close
focus makes the centre about a third of an L\* darker and moves the correction outward. On the 28P,
close focus lowers the corner correction by up to 0.07 stop. On the 35-100 it barely matters. Findings 17
and 30 missed it: Finding 17 picked its near and far frames by EXIF SubjectDistance, which on these files
doesn't record where the lens was focused, and Finding 30 had one raw per lens. Pre-processing maps are
now made per focus code.

- Four original-X2D 55V raws at f/2.5 from one body, each as a synthetic flat (L\* 30, landscape),
  gave three different maps. The only maker-note difference that groups them that way is byte 6 of
  0x0018: 1 for the two distant scenes, 5 for a person and dog at about 1.5 m, 7 for a head-and-shoulders
  portrait at about 1 m. Writing 7 into a code-1 flat reproduces the code-7 file's map exactly (centre
  L\* 30.05; gains equal to 0.001 stop from r 0.4 to 0.9).
- My own raws agree on what the byte means. 28P close-ups of clover and grass are codes 10 and 11, a
  flower spray at about a metre is 8, distant trees and a canopy are 0 and 1. EXIF SubjectDistance
  disagrees with the pictures (∞ for the clover close-up, 0.30 m for a distant tree), so Finding 17's
  "0.75 m vs ∞" 28P pair at f/8 was two far-focus frames (codes 0 and 1), and its f/5.6 pair was codes 4
  and 8.
- Byte 6 swept on one flat per lens setting, lens corrections on (BaselineExposure 0); green gain in stops:

| lens setting | focus code | centre L\* | r 0.2 | r 0.4 | r 0.6 | r 0.8 | r 0.9 | r 0.97 |
|---|---|---|---|---|---|---|---|---|
| 55V f/2.5 | 0 | 30.37 | +0.22 | +0.62 | +1.13 | +1.78 | +2.20 | +2.49 |
| | 7 | 30.05 | +0.17 | +0.55 | +1.10 | +1.81 | +2.25 | +2.55 |
| 28P f/4 | 0 | 30.74 | +0.21 | +0.69 | +1.36 | +2.19 | +2.59 | +2.83 |
| | 11 | 30.78 | +0.22 | +0.68 | +1.34 | +2.15 | +2.56 | +2.80 |
| | 15, 30 | 30.69 | +0.01 | +0.07 | +0.23 | +0.56 | +0.75 | +0.88 |
| 28P f/8 | 0 | 30.45 | +0.11 | +0.50 | +1.05 | +1.81 | +2.26 | +2.55 |
| | 11 | 30.45 | +0.11 | +0.47 | +1.00 | +1.74 | +2.22 | +2.53 |
| 35-100 35 mm f/11 | 0 | 31.44 | +0.04 | +0.16 | +0.39 | +0.78 | +0.96 | +1.10 |
| | 12 | 31.44 | +0.04 | +0.17 | +0.40 | +0.79 | +0.98 | +1.14 |
| 35-100 100 mm f/8 | 0 | 31.11 | +0.01 | +0.05 | +0.11 | +0.22 | +0.26 | +0.38 |
| | 12 | 31.11 | +0.01 | +0.05 | +0.12 | +0.24 | +0.28 | +0.30 |

- Intermediate codes fall in between (28P f/4 at 2, 4, 8; 35-100 at 2, 5, 8). Colour shading (B−G) doesn't
  change with focus on any lens.
- Past the end of a lens's table Phocus drops to its no-lens processing: the 28P at codes 15 and 30
  matches what an out-of-range focal length gives (Finding 30), with slightly more colour shading (B−G at
  r 0.9 −0.21 instead of −0.18). My 28P raws go up to 11 and my 35-100 raws up to 5.
- With corrections off the 55V changes too, less: code 1 vs 7 gives centre L\* 30.17 vs 30.04 and lift at
  r 0.4 of +0.04 vs +0.07 stop.
- Pre-processing: the map key is now lens, focal length, aperture and focus code
  (`XCD28P_28mm_f4.0_d10`). A missing code falls back to the nearest code for the same lens setting.
  `fit/preprocess_maps/` was rebuilt: 46 maps for the 39 lens settings in my raws.
- Real frames, Capture One renders of copies made with the old and the focus-keyed maps, against Phocus
  (whole frame, median ΔE00, lightness offset). Each old map had been made from whichever frame of that
  lens setting came first, so only three of the 23 validation frames get a different map:

| frame | focus code | old map's code | old | focus-keyed |
|---|---|---|---|---|
| B0000929, 28P f/4 | 0 | 10 | 0.51, −0.18 L\* (centre +0.1, r 0.6–0.8 −0.4) | **0.41, +0.05 L\*** (±0.1 throughout) |
| B0000916, 28P f/16 | 4 | 0 | 0.66, +0.16 L\* | 0.67, +0.02 L\* |
| B0000822, 35-100 100 mm f/11 | 0 | 5 | 0.25, +0.01 L\* | 0.26, −0.01 L\* |

  The other 20 frames' maps differ from the old ones by at most 0.004 stop.
- Correction to Findings 17 and 30: focus distance does change Phocus's lens processing on the 28P and
  the 55V, through 0x0018 byte 6.

### Finding 32: an E5 profile for the XCD 38V, from the owner's raws

**In plain terms:** `auto_lens_profile.py` built an HNCS E5 profile for my 38V from nine of my raws, used
only as templates for synthetic patches; Phocus did the rest. With pre-processing, those nine frames land
0.35–0.53 ΔE00 from Phocus over the whole frame, flat from centre to corner, about 0.2 L\* darker
throughout.

- Raws: nine X2D II frames, ISO 50: an aperture series (f/2.5–f/8, focus code 4) of a leaf on timber at
  about 1.5–2 m, and a house and dog at f/2.5 (code 1). Host B0001059 (f/2.5, code 4), which is also the
  profile's reference lens state.
- Synthetic fit residual median 1.46, p90 6.74 (the 28P's on the same test: 1.62 / 7.30, Finding 26).
- Maps for the six lens states: green raw gain at r 0.9 +2.12 stops at f/2.5, +1.93 at f/2.8, +1.54 at
  f/4, +1.34 at f/5.6, +1.25 at f/8. Codes 1 and 4 at f/2.5 differ by at most 0.01 stop.
- Capture One renders of the pre-processed copies with "HNCS E5 38V (pre-processed)", against Phocus with
  corrections on (whole frame, median ΔE00 / p90 / lightness offset):

| frame | setting | median | p90 | dL\* |
|---|---|---|---|---|
| B0001059 | f/2.5 | 0.35 | 0.56 | −0.21 |
| B0001060 | f/2.5 | 0.35 | 0.57 | −0.19 |
| B0001066 | f/2.5 | 0.41 | 0.55 | −0.22 |
| B0001065 | f/2.8 | 0.40 | 0.54 | −0.19 |
| B0001064 | f/4 | 0.42 | 0.59 | −0.20 |
| B0001061 | f/5.6 | 0.37 | 0.57 | −0.24 |
| B0001063 | f/5.6 | 0.42 | 0.62 | −0.20 |
| B0001062 | f/8 | 0.43 | 0.63 | −0.23 |
| B0001067 | f/2.5, house | 0.53 | 0.93 | −0.20 |

- The −0.2 L\* is the same on both scenes and at every aperture, so it belongs to the profile, not the
  maps. It's within the ±0.2 L\* the other lenses leave per frame, but there it varies in sign; here it
  doesn't. Eight of the nine frames are one scene, so this is a narrow test of colour.
- Contact sheet regenerated with the current maps and a 38V row: frame 939 0.54 → 0.43 (lens-setting
  brightness); the other five unchanged; 38V frame 1059 0.35 against Capture One's default 4.44.

### Finding 33: bright colours, exposure pushes and white balance above 6500 K

**In plain terms:** These are three checks prompted by blog.tonalphoto.com's Lightroom results. Pushing
exposure in either app works exactly like giving the sensor more light, so a push adds no difference of
its own. What a push (or a bright scene) does expose is a real gap. Once a saturated yellow, orange or
yellow-green reaches about L\* 95 in Phocus, the profiles fade it toward grey while Phocus keeps it
vivid. That fade almost certainly comes from a rule in `write_profile.py`, not from Capture One, so it
can be changed. The daylight profiles match Phocus best near 5500 K. Colourful patches drift to about
1.3 ΔE00 at 6500 K and 2.2 at 8000 K, but greys stay put, and three real shade frames scored 0.58–0.78.

- Raws (`tools/round11/`): copies of the Finding 23 host (35-100 at 35 mm f/11; Capture One reads its
  As Shot as 5451 K).
  - Ladder: grey plus 12 hues (HSV in white-balanced camera RGB: 0 red, 60 yellow, 120 green, 180
    cyan, 240 blue, 300 magenta) at saturation 0.3, 0.6 and 0.9. Each is placed at 23 levels of its
    brightest raw channel: 1/16 to 1.0 of the white level in quarter stops, plus 0.76–0.99.
  - White-balance grids: `--levels 9` at As Shot neutrals for 4200–8000 K, computed from ColorMatrix1
    on the daylight locus. Capture One reads them as 4169–8227 K.
- Phocus rendered through the hot folder at its defaults, and with the sidecar's `EV` at 2.0 and 4.0.
  `EV` is a linear gain applied before the tone curve. EV 2.0 renders a grey exactly as twice the raw
  level does, and 4.0 as four times (+1 and +2 stops).
- Capture One rendered with HNCS E5 35-100, Linear Response, Light Falloff 0, and exposure 0, +1 and +2.
- Script: `tools/round11/analyze.py`; log `data/logs/r11_synthetic.txt`.

**Bright colours** (EV 0; median ΔE00 over saturation 0.6 and 0.9, by the brightest raw channel's level):

| hue | ≤ 0.50 | 0.59–0.84 | 0.88–0.99 | what happens |
|---|---|---|---|---|
| 0 red | 0.58 | 7.92 | 6.45 | Phocus fades to white sooner |
| 30 orange | 0.33 | 16.42 | 22.07 | Capture One fades to grey |
| 60 yellow | 0.29 | 21.77 | 25.47 | Capture One fades to grey |
| 90 yellow-green | 0.31 | 6.54 | 18.68 | Capture One fades to grey |
| 120 green | 0.47 | 4.17 | 4.69 | Capture One −24 C\* |
| 150 | 0.32 | 1.80 | 1.83 | |
| 180 cyan | 0.43 | 0.36 | 0.70 | matches |
| 210 | 0.31 | 2.42 | 4.03 | |
| 240 blue | 5.11 | 13.71 | 16.58 | Phocus renders saturation 0.9 black (below) |
| 270 | 0.50 | 2.40 | 2.82 | |
| 300 magenta | 0.45 | 4.02 | 6.71 | Capture One about −10 C\* at saturation 0.9 |
| 330 pink | 0.49 | 7.20 | 5.68 | Phocus fades to white sooner |

- **Yellows, oranges and yellow-greens.** Green is their brightest raw channel. Phocus keeps them
  saturated right up to white. Yellow at saturation 0.9 sits at L\* 98, C\* 120–121 from level 0.71 to 1.0.
  Capture One with HNCS gives C\* 17 at 0.71 and 4 at 1.0. The loss starts where Phocus's L\* passes 95.
  At saturation 0.6, Phocus gives L\* 95 / C\* 60 at level 0.59, where Capture One gives C\* 44; at L\*
  98 the two are 55 against 8. That's where `write_profile.finalize` starts fading chroma to zero (table
  L\* 95 to 105, "beyond Phocus white"). The rule assumes L\* above 95 is near white, but Phocus's
  saturated yellows live there. It's very likely the cause; a rebuilt profile would confirm it.
- **Reds and pinks.** Their red channel passes white after white balance well before the sensor clips,
  because the red neutral is 0.37. Phocus fades them to white by about level 0.6–0.7. Pink at saturation
  0.6 gives C\* 41, 12 and 0 at levels 0.50, 0.59 and 0.71; Capture One gives 58, 47 and 4. This is the
  apps' own handling, before the profile.
- **Very saturated magenta and blue** (saturation 0.9, LED-like). From level 0.25 up, magenta has
  10–22 C\* less in Capture One: Phocus C\* 114–128 against 104–112. That fits the profiles' soft chroma
  compression above C\* 90. Phocus renders the saturation-0.9 blue (camera RGB about 0.1, 0.1, 1) as
  black at every level below 1.0 (L\* 0); Capture One renders it dark blue (L\* 1–7). It's outside the
  range HNCS handles, so it isn't a useful comparison. Blue at saturation 0.6 matches up to level 0.92.
- Greys stay within 1.0 ΔE00 at every level, with L\* within 0.3. Cyan matches at every level (0.36–0.70).
  blog.tonalphoto.com's worst Lightroom colours near clipping were cyan (12.6) and magenta (10.7).

**Exposure pushes.** These compare patches whose pushed level stays at or below 0.84:

| | Phocus vs Capture One, pushed | same colours unpushed at 2× / 4× the level | push vs more light, within each app (Phocus / Capture One) | dC\* (Capture One − Phocus) |
|---|---|---|---|---|
| +1 EV | 0.49 (p90 7.12) | 0.44 | 0.10 / 0.02 | −0.35 |
| +2 EV | 0.88 (p90 9.81) | 0.57 | 0.10 / 0.02 | −1.02 |

- In both apps, a push renders a patch the same as more light would. In Capture One, exposure acts before
  the profile, as BaselineExposure does.
- The difference between the apps is the bright-colour gap above, reached sooner. At saturation 0.9, a +2
  EV push costs 11.9 C\*. At saturation 0.3 and 0.6 it's 0.30 and 0.52 ΔE00.
- blog.tonalphoto.com's 1 C\* Lightroom gap on pushed frames is the same size as the +2 EV median here.
  Whether it has the same cause in Lightroom is unknown.

**White balance** (median ΔE00 on patches whose brightest raw channel is below 0.5, by Phocus chroma):

| As Shot | C\* 20–40 | C\* 40–60 | all | greys Δb\* |
|---|---|---|---|---|
| 4200 K | 1.67 | 2.47 | 3.04 | +0.23 |
| 4500 K | 1.28 | 1.98 | 2.47 | +0.24 |
| 5000 K | 0.67 | 0.90 | 1.45 | +0.28 |
| 5500 K | 0.52 | 0.48 | 0.95 | +0.30 |
| 6000 K | 0.90 | 0.93 | 1.33 | +0.32 |
| 6500 K | 1.26 | 1.29 | 1.69 | +0.34 |
| 7000 K | 1.52 | 1.54 | 2.21 | +0.39 |
| 7500 K | 1.87 | 1.85 | 2.85 | +0.40 |
| 8000 K | 2.12 | 2.35 | 3.48 | +0.41 |

("all" is dominated by the grid's very saturated corners.)

- Greys hold at every white balance: within 0.5 ΔE00, b\* +0.2 to +0.4. Colours drift either side of
  5500 K. Phocus's colour changes with white balance beyond what Capture One's white balance does
  (Finding 14), and one profile can follow it only near the middle of its fitting range. The daylight
  profiles don't hold down to 4500 K, so the 4200–4700 K gap in question 2 stays.
- Real frames, scored like Finding 24 (pre-processed + E5 vs Phocus with corrections on, whole frame):
  - B0000840 (Capture One As Shot 6500 K): 0.58
  - B0000688 (6802 K, ISO 400): 0.63
  - B0000686 (7606 K, ISO 800): 0.78, dL\* +0.35

  Finding 24's 16 35-100 frames, at about 4500–6600 K (estimated from ColorMatrix1), scored 0.26–0.61
  with no trend by temperature (correlation 0.05). Real scenes are mostly low in chroma, so the drift
  shows much less there. Three frames is a small sample. Log: `data/logs/r11_real.txt`.

### Finding 34: bright yellows kept saturated; E5 profiles rebuilt

**In plain terms:** Finding 33's faded yellows came from two things in how the profiles were written. One
rule faded colour to grey above L\* 95. Then the profile writer ran that rule a second time. The rebuilt E5
profiles fade colour only above L\* 100. They keep a fade near white only for nearly neutral colours, so
whites stay clean. In Capture One, bright yellows and oranges went from about 19 ΔE00 to 2.4. Everything
else is unchanged or slightly better. All seven E5 profiles were rebuilt and installed under the same
names.

- **Causes.**
  - `finalize` faded chroma between table L\* 95 and 105, but Phocus's saturated yellows sit at L\* 98.
  - `fit_synthetic.py` saved an already finalized table, and `write_profile.py` finalized it again. That
    faded nodes at L\* 95–100 a second time and compressed high chroma twice. 19,654 of 35,937 nodes
    changed.
- **The new `finalize`.** Colour fades over the 10 L\* above 100. Near-neutral nodes also fade from L\* 95:
  fully at C\* 2.4 or less, not at all from C\* 6. Phocus's white is neutral, and the fit has no unclipped
  data there. Without that fade the brightest greys took a green tint (a\* −2 at L\* 100).
- **Fitted tables** now carry a `finalized` flag and their settings, so `write_profile.py` doesn't finalize
  them twice. Older tables without the flag still rebuild byte-identical; this was checked on all seven
  shipped profiles.
- **Defaults.** `fit_synthetic.py` now defaults to `--fade-start 100 --grey-fade-start 95 --grey-chroma 6`.
  `--fade-start 95 --grey-fade-start -1 --grey-chroma 20` reproduces the old tables exactly.
- **Offline scoring.** `tools/round12/eval_tables.py` puts the round-11 ladder through Capture One's own
  camera RGB (a probe render) and through each table. Against a real Capture One render it agrees to
  a median of 0.035 ΔE00. It skips patches whose camera RGB the probe clips at 1: the brightest blues,
  reds and magentas.

35-100 variants (ladder: median ΔE00 at saturation 0.6 and 0.9; real: 19 held-out frames, frame centre):

| table | yellow-orange 0.59–0.84 | yellow-orange 0.88–0.99 | worst grey | all colours p90 | real (mean of medians) | real blocks at L\* > 90 |
|---|---|---|---|---|---|---|
| shipped before | 15.91 | 20.94 | 0.99 | 10.21 | 0.630 | 0.61 |
| finalized once, fade from 95 | 15.07 | 20.37 | 0.93 | 9.66 | 0.630 | 0.61 |
| fade from 100 | 1.89 | 1.57 | 2.90 | 3.21 | 0.630 | 0.49 |
| fade from 100, greys (C\* < 20) from 95 | 1.89 | 1.58 | 0.93 | 5.30 | 0.631 | 0.59 |
| **fade from 100, greys (C\* < 6) from 95** | **1.89** | **1.57** | **0.94** | **2.92** | **0.630** | **0.55** |

- **Capture One renders of the ladder**, old 35-100 profile against the new one:
  - Bright yellow-orange (saturation 0.6 and up, level 0.59–0.99): 19.39 → 2.39.
  - Other hues: 2.85 → 2.78.
  - Greys: 0.40 → 0.37, neutral up to white (a\* and b\* within 0.3 from level 0.71).
  - Yellow at saturation 0.9 now holds C\* 105–106 up to white. Phocus gives 116–121; the old profile
    gave 4–17. The rest of that gap is the soft chroma compression above C\* 90.
  - The installed profile renders identically to the tested one.
- **Rebuilt tables** (synthetic fit residual, median / p90):
  - 35-100: 1.48 / 7.16 → 1.39 / 5.71.
  - 28P: 1.51 / 7.02 → 1.42 / 5.73. Real held-out frames unchanged (1.401). Bright yellow-orange
    14.6 / 20.6 → 2.2 / 1.9.
  - 38V, rebuilt by `auto_lens_profile.py` from B0001059: 1.46 / 6.74 → 1.38 / 5.48. Bright yellow-orange
    14.1 / 20.7 → 2.3 / 2.0.
  - Tungsten 35-100: 2.08 / 6.10 → 2.00 / 5.38.
  - Tungsten 28P: 2.05 / 5.95 → 1.98 / 5.35.
- **LCC profiles** use the same tables with the gains in `fit/lcc/input_gains.txt`.
- **E4 wasn't rebuilt.** It comes from `fit_anchored.py`, which still uses the old rule. The E4 profile and its
  styles were then removed from the repo; they're in git history. `fit/lut_e4.npz` stays, because
  `auto_lens_profile.py` and `auto_maps.py` use it to read Phocus's grey flats for lenses without an E5 table.
- **Not checked:** whole real photos rendered in Capture One. The held-out real pairs, scored offline at
  the frame centre, didn't change.
- Logs: `data/logs/r12_fade_35100.txt`, `r12_fade_28p.txt`, `r12_fade_38v.txt`, `r12_c1_ladder.txt`.

### Finding 35: five more lenses and a third body, from blog.tonalphoto.com's raws

**In plain terms:** blog.tonalphoto.com's author shared one X2D II raw for each of five lenses I don't
own: the 25V, 55V, 90V, 120 Macro and 135, all from one body. Each became an HNCS E5 profile, built for
my body. Their camera's factory calibration is further from mine than any seen before, and Phocus
treats it exactly as the formula predicts. On their own photos the result is 0.16–0.51 ΔE00 from Phocus
over the whole frame once the body correction is applied, against 1.33–1.98 without it. That's the
third body, and the first test of the body correction on real photos from many lenses.

- Raws: B0041052 (25V f/9), B0039850 (55V f/6.3), B0038945 (90V f/6.3), B_23969 (XCD 120 Macro f/8,
  focus code 5), B0040287 (135 f/6.3). ISO 50–200, all at As Shot 5463 K by Capture One. Kept locally in
  `data/tonal/`; not redistributed.
- **Their body's GAIN record:** 2.049, 2.250. Mine is 2.600, 2.260 and DPReview's 2.363, 2.518.
  `body_gains.py` predicts camera RGB × (1, 0.9818, 0.9997), a green change almost three times
  DPReview's.
- **Synthetic check (`tools/round13/body_test.py`).** Phocus rendered the round-11 ladder with my record
  and with theirs (0x0019 swapped in, nothing else changed). The two renders differ by a median 0.75 ΔE00:
  greys move a\* +1.37, b\* −0.92. The formula, put through the E5 35-100 table, predicts +1.41, −0.95.
  Their-record render against my render shifted by the prediction: median 0.07 ΔE00 (p90 0.18). The
  formula holds well outside the range it was fitted on.
- **Profiles for my body.** Each raw got a copy with my GAIN record swapped in (`makernote_transplant.py
  … 0x0019`). That copy was the host for `auto_lens_profile.py --install`, which uses only metadata.
  The results are HNCS E5 XCD25V, XCD55V, XCD90V, XCD120 and XCD135, plus maps for each lens setting
  (made the same way).
- **Real frames.** Each original went through `preprocess_3fr.py`, with and without `--body-gains`, and
  was rendered in Capture One with its lens's profile. Scored against Phocus with corrections on, whole
  frame, median ΔE00 and lightness offset:

| frame | lens | without body gains | with body gains |
|---|---|---|---|
| B0041052 | 25V f/9 | 1.98, +0.51 L\* | **0.47**, +0.07 L\* |
| B0039850 | 55V f/6.3 | 1.79, +0.64 L\* | **0.51**, +0.23 L\* |
| B0038945 | 90V f/6.3 | 1.78, +0.45 L\* | **0.26**, −0.01 L\* |
| B_23969 | 120 Macro f/8 | 1.33, +0.45 L\* | **0.16**, −0.04 L\* |
| B0040287 | 135 f/6.3 | 1.80, +0.37 L\* | **0.18**, −0.02 L\* |

- Corners match centres: with body gains, the band from r 0.8 to 0.95 scores 0.13–0.66.
- Without the correction there's also a +0.4 to +0.6 L\* lightness error. The green gain moves lightness
  as well as colour.
- **What this doesn't cover.** Each profile's host was the frame being scored, so each lens was tested at
  its reference setting only; only the metadata was used, never the pixels. Other apertures, focal
  lengths and focus distances need their own maps, and `preprocess_folder.py --make-maps` makes them from
  any raw.
- `body_gains.py` and `preprocess_3fr.py --body-gains` had been broken since Round 10 (`16a001b`). They
  borrowed code from `makernote_transplant.py` by splitting its source text, and that file was
  restructured in Round 10. Both now import it normally.
- Logs: `data/logs/r13_body_test.txt`, `r13_tonal_scores.txt`, `tonal_phocus.txt`; scores in
  `data/val/r13/`.

### Finding 36: the 90V's correction rises steeply with close focus

**In plain terms:** blog.tonalphoto.com's 90V raw made Finding 31's focus sweep possible for that lens.
Of the lenses measured so far, the 90V moves most with focus, but only at the edges and only with lens
corrections on. Focused at infinity, Phocus lifts the outer frame by about a quarter stop; at the closest
focus code, by two thirds of a stop. The centre and the colour shading don't change. With corrections
off, focus makes no difference.

- Host B0038945 (90V f/6.3, focus code 0), synthetic flat at L\* 30, focus byte set to 0–20;
  `tools/round14/focus_sweep.py`, read through the E5 XCD90V table. Green gain in stops, relative to the
  centre:

| focus code | corrections | r 0.4 | r 0.6 | r 0.8 | r 0.9 |
|---|---|---|---|---|---|
| 0 | on | +0.057 | +0.118 | +0.188 | +0.240 |
| 4 | on | +0.071 | +0.149 | +0.246 | +0.275 |
| 8 | on | +0.083 | +0.178 | +0.303 | +0.381 |
| 12 | on | +0.095 | +0.207 | +0.381 | +0.534 |
| 14 | on | +0.101 | +0.222 | +0.426 | +0.654 |
| 16, 20 | on | +0.034 | +0.076 | +0.139 | +0.147 |
| 0, 20 | off | +0.034 | +0.076 | +0.139 | +0.147 |

- Codes in between fall in between. Centre L\* is 31.04 at every code. B−G at r 0.9 stays at −0.09.
- From code 16 up, corrections on renders exactly like corrections off: past the end of the 90V's table,
  Phocus drops its vignetting correction. The 28P does the same from code 15 (Finding 31).
- With corrections off, the 90V's edge lift is +0.15 stop at r 0.9 relative to the centre. That fits
  blog.tonalphoto.com's measurement of +0.09 EV at the centre to +0.29 at the edge against the raw
  (corrections off, minimum focus, f/8), an edge-over-centre of +0.20. Their edge and aperture
  differ from mine.
- Pre-processing maps already key on the focus code. The 90V has only code 0 so far, and
  `--make-maps` makes the rest. The nearest-code fallback would be off by up to 0.4 stop in the corners
  at close focus, so for 90V frames, make the maps.
- Log: `data/logs/r14_focus_90v.txt`.

### Finding 37: the yellow fix on real photos

**In plain terms:** Finding 34's rebuilt profiles were checked on whole real photos in Capture One. On my
23 validation photos nothing got worse; two 28P frames got better, and the rest are within 0.01. Bright
saturated yellows are rare in real photos: no region of any of my 120 X2D II raws qualifies, and three of
DPReview's do, at under 2% of the frame. In those regions the error drops from 4.9 to 1.6 and from 3.2 to
0.5 ΔE00, with the colour back to Phocus's level.

- Every frame was pre-processed (`preprocess_3fr.py`; `--body-gains` for DPReview's body) and rendered
  twice in Capture One: once with the old E5 tables (rebuilt from `8532a3b^` as "HNCS E5o"), once with the
  new. Both were scored against Phocus with corrections on.
- **23 validation photos, whole frame:** mean of medians 0.506 → 0.495, mean p90 0.997 → 0.987.
  - B0000919 (28P): 0.38 → 0.23.
  - B0000916 (28P): 0.67 → 0.59.
  - No other frame changed by more than 0.01.
- **Searching for bright yellows** (`tools/round14/find_bright_yellow.py`): blocks whose green raw level
  is at least 0.55 of white, with blue under half of green and red over 0.6 of green. That's where Phocus's
  yellows pass L\* 95.
  - My 120 raws: none.
  - DPReview's 33: four, the largest at 0.6% of the frame.
  - blog.tonalphoto.com's five: none.
- **DPReview frames, scored over blocks where Phocus has L\* above 85, C\* above 40 and hue 50–110°:**

| frame | blocks | old | new | C\* Phocus / old / new |
|---|---|---|---|---|
| 0880020828 (100 mm f/11) | 0.45% of frame | 4.94 | **1.58** | 44 / 33 / 48 |
| 1097831660 (100 mm f/8) | 0.39% | 3.22 | **0.46** | 54 / 41 / 54 |
| 0041848181 (45 mm f/3.2) | 1.69% | 0.81 | **0.50** | 58 / 58 / 59 |

  The whole frame scores the same either way (0.38, 0.47, 0.36–0.37), because the regions are small.
- The three maps `auto_maps.py` made for the DPReview frames' lens settings came from another body's
  raws, so they weren't kept.
- Logs: `data/logs/r14_real_summary.txt`, `r14_c1_v.txt`, `r14_c1_dp.txt`, `r14_bright_yellow_scan*.txt`;
  scores in `data/val/r14/`.

### Finding 38: the profiles keep Phocus's most saturated colours

**In plain terms:** The profiles used to squeeze every colour above C\* 90 toward a ceiling of about 117.
The Lab encoding inside a Capture One profile doesn't limit chroma, though. It limits a\* and b\*
separately, each to about ±128, which allows chroma up to about 180 along diagonal hues. Phocus's
synthetic patches reach C\* 175. The rebuilt profiles compress only where a\* or b\* nears that limit,
starting at 110, and keep the hue. Very saturated colours went from 2.2 to 0.4 ΔE00 in Capture One, and the
fit's own error halved. Greys, moderate colours and real photos are unchanged.

- `finalize` has a new `limit="box"` mode: it soft-compresses max(|a\*|, |b\*|) from a knee toward the cap,
  scaling a\* and b\* together. `fit_synthetic.py` now defaults to `--limit box --knee 110 --cmax 127.5`.
  The old behaviour is `--limit chroma --knee 90 --cmax 127`.
- **35-100 variants, offline** (`tools/round12/eval_tables.py`; ladder patches the probe can score):

| limit | synthetic fit residual (median / p90) | very saturated (Phocus C\* > 90) | bright yellow-orange, 0.59–0.84 | all colours p90 | real held-out |
|---|---|---|---|---|---|
| chroma, knee 90 (before) | 1.39 / 5.71 | 1.33 | 1.89 | 2.92 | 0.630 |
| chroma, knee 110 | 0.85 / 4.37 | 0.51 | 0.62 | 1.83 | 0.630 |
| box, knee 100 | 0.78 / 2.97 | 0.47 | 0.80 | 1.76 | 0.630 |
| **box, knee 110** | **0.68 / 2.59** | **0.33** | **0.44** | **1.39** | **0.630** |

- **Capture One renders against Phocus**, old 35-100 profile against the new one (median ΔE00, by
  Phocus chroma):

| raw | C\* 0–40 | C\* 40–90 | C\* 90–130 | C\* 130+ |
|---|---|---|---|---|
| ladder (Finding 33) | 0.31 → 0.31 | 0.34 → 0.34 | 2.20 → **0.43** | 7.24 → **1.34** |
| full-gamut grid, 5500 K | 0.44 → 0.44 | 0.50 → 0.49 | 1.63 → **0.37** | 7.17 → **2.75** |

  No patch got worse by more than 1 ΔE00; 103 and 229 improved by more than 1. The grid's median went
  from 1.29 to 0.47. Above C\* 130 the new profile still runs 9–13 C\* short.
- **Rebuilt, synthetic residual median / p90:**
  - 35-100: 1.39 / 5.71 → 0.68 / 2.59.
  - 28P: 1.42 / 5.73 → 0.76 / 2.62.
  - Tungsten 35-100 and 28P: about 2.0 / 5.4 → 1.24 / 3.5.
  - 38V, 25V, 55V, 90V, 120 Macro and 135, rebuilt through Phocus by `auto_lens_profile.py`: 0.68 /
    2.5–2.6 each (the 38V was 1.38 / 5.48 after Finding 34).
  - All seven pair-built profiles and the six lens profiles are installed under the same names.
- **Real photos**, pre-processed and rendered with the Finding 34 profiles and the new ones:
  - 23 validation photos: mean of medians 0.495 → 0.496. No frame moved by 0.02.
  - DPReview frames (body gains applied), blocks where Phocus has C\* above 90: 0.78 → 0.57 (0.24% of
    the frame), 0.80 → 0.61 (4.4%) and 0.50 → 0.42 (0.04%). Chroma now matches Phocus.
  - Colours that saturated are uncommon in real scenes, and real ones rarely go far past C\* 100.
- Logs: `data/logs/r15_chroma_35100.txt`, `r15_c1_check.txt`, `r15_real_summary.txt`, `r15_phocus.txt`;
  scores in `data/val/r15/`.

### Finding 39: Capture One's own settings can do most of pre-processing, but only radially

**In plain terms:** Instead of writing a corrected copy of every raw, the batch tool can write a Capture
One settings file for the original. The file carries Phocus's correction as a light-falloff curve, the
lens's colour shading as an LCC, and the lens-setting brightness as exposure. Capture One honours all of
it, and on flat fields it reproduces Phocus's correction to within 0.01 stop. On real photos it lands
between pre-processing and the old LCC route: 0.58 ΔE00 on the 35-100 (pre-processing 0.48) and 0.93 on
the 28P (0.54). The gap is geometry. A falloff curve is a function of radius only, and Phocus's
correction isn't radially symmetric. On the 28P it's up to 0.19 stop stronger on the left than at the top
at the same radius.

- **Test A: a curve in the settings file is used** (`tools/round16/falloff_sidecar_test.py`).
  - `LensLightFallOff` holds `radius µm,EV;…|amount|enabled`, up to 16 points.
  - A hand-written curve in a new raw's `.cos` was kept and applied.
  - In raw terms the correction matched the curve within 0.01 stop (+0.24/+0.49/+0.74/+0.92 stop at
    7/14/21/26 mm, for a curve of 0.25/0.50/0.75/0.93).
- **Test B: it's applied in linear raw space.**
  - Flats at L\* 10, 30 and 60 got the same raw-domain correction to 0.003 stop.
  - In Capture One's own output the gain varies widely with level, so the curve acts before the
    Linear Response, where pre-processing applies its gain.
- **Test C: the settings file can carry an LCC** (`tools/round16/lcc_sidecar_test.py`).
  - `create LCC` / `apply LCC` (AppleScript) store an applied LCC as `<image>.<LccID>.lcc` beside the
    `.cos`. The copy is byte-identical to the reference's, and the `.cos` `<AL>` block gets
    `ColorCastCorrection`, `LccID`, `LensLccLightFalloff` (uniform light) and its amount.
  - A hand-written `.cos` with those keys and a copied `.lcc` applied the LCC (colour cast on, uniform
    light off) together with a custom falloff curve.
  - Measured blue-minus-green followed the LCC's colour map within 0.002 stop.
  - The `.lcc` records the reference raw's camera serial number, so the LCC files stay in `data/lcc_c1/`.
- **`tools/c1_sidecars.py`** writes such a file for each raw from its map:
  - the green gain as a 14-point radial curve;
  - the lens's LCC with its "E5 … LCC" profile;
  - the lens-setting brightness as `Exposure`.
  On a 28P f/4 flat it reproduced the map's radial profile within 0.005 stop for green and 0.01 for
  blue and red relative to green, unchanged from L\* 10 to 60. Only corners brightened past white clip,
  as with pre-processing.
- **23 validation photos**, whole frame against Phocus with corrections on (`data/logs/r16_native_summary.txt`):

| lens | pre-processed copy | original + settings file | old LCC route (Finding 24) |
|---|---|---|---|
| 28P (7) | **0.54** | 0.93 | 1.38 |
| 35-100 (16) | **0.48** | 0.58 | 0.64 |

  The extra error grows outward from r 0.2. On the 28P it shows as +0.2 to +0.5 L\* in the mid-frame.
- **Why.** At the same radius the maps differ by direction (`data/logs/r16_map_asymmetry.txt`):

| map | rms / max departure from radial (stops) | r 0.6, left / right / top / bottom minus mean |
|---|---|---|
| 28P f/4 | 0.059 / 0.187 | +0.11 / +0.04 / −0.07 / −0.08 |
| 28P f/8 | 0.060 / 0.190 | +0.11 / +0.03 / −0.06 / −0.07 |
| 35-100 35 mm f/8 | 0.041 / 0.143 | +0.07 / +0.06 / −0.04 / −0.09 |
| 35-100 100 mm f/8 | 0.021 / 0.066 | +0.03 / +0.01 / 0.00 / −0.04 |

  Stronger along the long side and a little stronger on the left: an elliptical, slightly off-centre
  correction. A radial curve can only follow the average around each ring.
- **What would close it:** a second LCC with uniform light on, carrying the non-radial part, per lens
  setting. That's one Capture One-made LCC per map (57 so far). Uniform light rescales to the frame's
  mean (the exposure key can undo that), and its response to a reference hasn't been measured; the
  colour LCC's was (mean / reference)^1.14.

### Finding 40: one LCC per lens setting closes the gap, once its colour offset is cancelled

**In plain terms:** Finding 39's settings files fell short because a falloff curve can only follow
distance from the centre. An LCC can hold a full 2-D pattern, so I tried carrying Phocus's whole
correction for one lens setting (28P at f/4, focused at infinity) in an LCC. Capture One then reproduces
Phocus's map in 2-D to about 0.015 stop. The only catch is that an LCC shifts the overall colour
slightly, and that has to be cancelled with white balance. On the real photo for that setting
(B0000929) the result is 0.43 ΔE00, level with pre-processing (0.40) and better in the corners. The last
bit of that offset was fitted on this same photo, so it still needs a check on a photo it wasn't
fitted to.

- **Uniform light switches Light Falloff off.** An LCC with uniform light on, plus a falloff curve,
  renders as if the curve weren't there (−1.1 stop short at r 0.6). Capture One does the same when it
  applies an LCC itself, setting falloff to 0. So the LCC has to carry the whole correction, not only
  the non-radial part.
- **The full-correction LCC** (`tools/round17/make_state_lcc.py MAP HOST OUT --full`).
  - The reference is the inverse of the map, per channel, smoothed with a degree-8 polynomial
    (rms 0.004–0.005 stop). It's applied with colour cast and uniform light on, and Light Falloff 0.
  - On flat fields (`tools/round17/flat_2d.py`), render minus map is 0.013 / 0.017 / 0.019 stop rms for
    R / G / B, the same at L\* 30 and 60. At r 0.6 left, right, top and bottom agree within 0.003 stop;
    the radial curve was 0.11–0.19 stop out.
  - The LCC's response is a plain inverse of the reference: exponent 1.
- **Global offset.**
  - Against an identical flat with no LCC, the LCC leaves brightness within 0.004 stop. It shifts
    colour by R/G −2.8% and B/G +4.7% at Capture One's output, about −2.0% and +3.7% in raw terms.
  - An apparent +0.12 stop of brightness was a scale error in `fit/c1_response.npz`'s conversion; it
    shows on flats without an LCC too.
  - Cancel the colour with the settings file's `WhiteBalance` multipliers: the raw's As Shot multipliers
    times (kR, 1, kB). `c1_sidecars.py --state-lcc ID --state-comp EV,kR,kB` writes this.
- **B0000929** (28P f/4, code 0), whole frame against Phocus with corrections on, median ΔE00:

| route | median | p90 | dL\* | corners (r 0.8–0.95) |
|---|---|---|---|---|
| pre-processed copy | 0.40 | 1.16 | +0.05 | 0.65 |
| settings file, radial curve + per-lens colour LCC (Finding 39) | 1.01 | 1.58 | +0.26 | 1.07 |
| settings file, full-correction LCC, no offset cancelled | 1.95 | 2.21 | −0.69 | 1.73 |
| … white balance from the flat (kR 1.0196, kB 0.9635) | 0.54 | 1.35 | −0.28 | 0.75 |
| … tuned on this photo (kB 0.9684, +0.025 EV) | **0.43** | 1.31 | +0.17 | **0.54** |

  The neutral offset went from b\* −2.36 (none cancelled), to +0.37 (flat-based), to +0.02 (tuned). The
  flat-based values removed 87% of the cast. The rest probably comes from converting Capture One's
  output back to raw through `c1_response.npz`.
- **Still open:**
  - a calibration that doesn't touch the photo. One option: have Capture One find the white balance
    that makes the LCC flat match the no-LCC flat;
  - a test on photos the calibration never saw;
  - one LCC per lens setting (57 so far), made and calibrated by script.
- Logs: `data/logs/r17_flat_state_lcc.txt`, `r17_929.txt`. The LCC files are in `data/lcc_c1/` (not
  committed: they record the camera's serial number).

### Finding 41: calibrating each LCC from flats alone works on unseen photos

**In plain terms:** Finding 40's last step was tuned on the photo being scored. Calibrating each
lens-setting LCC from flat fields alone, in Capture One's own output, gets within 0.01–0.11 ΔE00 of
pre-processing on four photos the calibration never saw, two lenses and three lens settings. So the
settings-file route can nearly match pre-processing without copying a single raw. The cost is one LCC
per lens setting, made and calibrated by script in Capture One.

- **`tools/state_lcc.py`:**
  - `make HOST MAP ID` builds the full-correction reference and runs Create LCC in the open session.
  - `calibrate HOST ID` renders three L\* 60 flats through the probe: no LCC; the LCC; and the LCC with
    a known step (white balance R and B ×1.05, +0.05 EV).
  - Per channel, Capture One's output is locally out ∝ gain^s (s ≈ 1.06 for every channel here), so the
    two LCC flats give s and the no-LCC flat gives the gain that cancels the offset.
  - Saved to `data/lcc_c1/ID.json`, which `c1_sidecars.py --state-lcc ID` picks up.
- **Calibrations** (exposure, kR, kB):
  - 28P f/4 code 0: +0.003 EV, 1.019, 0.963. Finding 40's photo-tuned values were +0.025 EV and kB 0.968.
  - 28P f/8 code 0: +0.001 EV, 1.020, 0.960.
  - 35-100 35 mm f/8 code 0: −0.003 EV, 1.003, 0.983.
- **Four photos**, whole frame against Phocus with corrections on, median ΔE00 (r 0.8–0.95 in brackets):

| photo | pre-processed copy | settings, radial curve (Finding 39) | settings, calibrated LCC |
|---|---|---|---|
| B0000929, 28P f/4 | **0.40** (0.65) | 1.01 (1.07) | 0.51 (0.74) |
| B0000919, 28P f/8 | **0.23** (0.15) | 0.65 (0.68) | 0.30 (0.30) |
| B0000633, 35-100 35 mm f/8 | **0.63** (0.66) | 0.66 (0.59) | 0.64 (0.67) |
| B0000656, 35-100 35 mm f/8 | **0.24** (0.25) | 0.66 (0.69) | 0.29 (0.32) |

  The remaining near-neutral offsets are small: b\* +0.06 to +0.37, L\* −0.35 to +0.28. They're largest on
  B0000929, as in Finding 40.
- **To make it the default route:**
  - make and calibrate an LCC with every map, as part of `--make-maps`;
  - have `c1_sidecars.py` find the LCC by lens setting, with the radial curve as the fallback;
  - give each body its profile.
- The calibration needs the probe profile installed and Capture One open; the LCC files stay local.
- Logs: `data/logs/r18_calibration.txt`, `r18_scores.txt`; scores in `data/val/r18/`.

### Finding 42: the settings-file route replaces pre-processing

**In plain terms:** Every lens setting in the validation set now has its own calibrated LCC, and one
command writes a Capture One settings file for each original raw. On 28 photos no fit had seen, from two
bodies and seven lenses, the whole-frame median is 0.488 ΔE00 against pre-processing's 0.464, and no photo
is worse by more than 0.12. That's close enough to drop pre-processing: no corrected copies, no lost
highlight headroom, and the raws are never touched. Some photos come out slightly brighter or darker
overall than they did with pre-processing, for a reason not yet found.

- **One command.** `tools/prepare_folder.py FOLDER [--make-maps] [--make-profiles] [--make-lccs]` runs
  `auto_maps.py`, `auto_lens_profile.py --install`, `state_lcc.py ensure` and `c1_sidecars.py` in turn.
- **An LCC per lens setting, named by its map.** `c1_sidecars.lcc_id` turns a map's file stem into the LCC
  id (`XCD28P_28mm_f4.0_d0.npz` → `XCD28P_28mm_f4p0_d0`), so `c1_sidecars.py` finds a lens setting's LCC
  and calibration in `data/lcc_c1/` with no lookup table. Without one it falls back to the radial Light
  Falloff curve (Finding 39) and says so.
- **28 LCCs made and calibrated** unattended in the lab session, about 1.5 minutes each. Calibrations:
  exposure −0.008 to +0.007 EV, kR 1.001 to 1.023, kB 0.956 to 0.990, local slope 1.053 to 1.069 on every
  channel.
- **Body correction through white balance.** The body's gains (1, kG, kB) relative to the reference record
  (2.600, 2.260) multiply the LCC calibration's (kR, 1, kB) and 2^EV. The settings file gets
  Exposure = lens-setting brightness + log2(green gain), and WhiteBalance = As Shot multipliers × gains /
  green gain. So one profile per lens serves every body, with no per-body profile.
- **Reference-body maps and profiles.** `auto_maps.py` and `auto_lens_profile.py` now write the reference
  body's GAIN record into their synthetic copies (`phocus_hotfolder.set_reference_body`), so maps and
  profiles made from another body's raws still describe the reference body.
- **Tungsten profiles picked automatically.** Below an As Shot of 4400 K (McCamy, from ColorMatrix1 and
  AsShotNeutral) the lens's Tungsten profile is used where one exists. DPReview's tungsten frames
  (3207–3306 K) get the 35-100 Tungsten profile.

| 28 photos not used in fitting, against Phocus with corrections on | pre-processed copy | settings file |
|---|---|---|
| 28P (7): median / p90 / corners (r > 0.8) | 0.541 / 1.058 / 0.601 | 0.586 / 1.083 / 0.687 |
| 35-100 (16) | 0.476 / 0.956 / 0.487 | 0.494 / 0.967 / 0.503 |
| Third body (5): 25V, 55V, 90V, 120 Macro, 135 | 0.318 / 0.825 / 0.319 | 0.332 / 0.833 / 0.343 |
| All 28, median | 0.464 | 0.488 |

- **Worst frame differences** (settings file minus pre-processed): +0.11 on the 28P, +0.12 on the 35-100,
  +0.03 on the third body.
- **Overall brightness moved on some frames.** The near-neutral dL\* against Phocus changed by up to
  ±0.4 on the 28P (B0000939: +0.01 → −0.41; B0000919: +0.06 → +0.27), and on the 38V contact-sheet frame
  B0001059 from −0.21 to −0.85, which takes it from 0.35 to 0.82 ΔE00. The offset is uniform across the
  frame. It doesn't follow orientation, BaselineExposure or focus code. Open.
- **Contact sheet** rebuilt with the settings-file route: Phocus (Standard, corrections on), Capture One
  with the settings file, Capture One's default. Whole-frame medians 0.28–0.82 against 2.95–6.29.
- **Removed:** `preprocess_folder.py`, `preprocess_3fr.py`, `make_lcc_raw.py`, `fit/lcc/`, the
  "(pre-processed)" styles, and the per-lens colour LCC profiles and styles. The plain styles stay, for
  photos Capture One has already seen, and the tungsten profiles gained styles.
- Logs: `data/logs/r19_prepare.txt`, `r19_scores.txt`, `r19_summary.txt`, `sheet_prepare.txt`,
  `sheet.txt`; scores in `data/val/r19/`.

### Finding 43: calibrating each LCC over the frame removes the brightness offsets

**In plain terms:** Finding 42's leftover brightness offsets came from where each LCC was calibrated, not
from the LCC itself. Capture One's LCC follows Phocus's correction across the frame, except at a small spot
in the very centre. The calibration measured exactly that spot, so every photo came out offset by however
much the spot differed from the rest: darker at wide apertures, brighter on the 28P stopped down.
Calibrating against a flat with Phocus's correction multiplied into the raw, averaged over the frame, puts
the settings-file route within 0.01 ΔE00 of pre-processing, and the offsets are gone.

- **Isolating it** (`tools/round20/lcc_brightness.py`, renders through the probe). On real photos:
  - A WhiteBalance key equal to As Shot changes nothing (0.0000 stop on all five photos), so the key isn't
    the cause.
  - The LCC's effect at the frame centre (r < 0.15) is −0.019 stop on B0000939 (28P f/4), −0.016 on
    B0000926, +0.008 and +0.007 on B0000919 and B0000916 (f/8, f/16), and −0.031 on B0001059 (38V f/2.5).
    The calibration predicted −0.005, −0.005, −0.001, −0.001 and −0.008.
  - On flats in B0000939's raw it's −0.005 to −0.009 at the exact centre at every level from L\* 10 to
    80, matching the calibration. So the level isn't the cause either.
- **The cause.** A flat with the map multiplied into the raw (what pre-processing did) against the same
  flat through the LCC route, both in Capture One:

| photo | exact centre (r < 0.02) | whole frame | r 0.1 to 0.9 | real photo vs pre-processing (Finding 42) |
|---|---|---|---|---|
| B0000939, 28P f/4 | +0.008 | −0.024 | −0.021 to −0.031 | about −0.028 |
| B0000926, 28P f/4 | +0.008 | −0.024 | −0.020 to −0.029 | about −0.025 |
| B0000919, 28P f/8 | +0.002 | +0.017 | +0.015 to +0.024 | about +0.014 |
| B0000916, 28P f/16 | +0.002 | +0.017 | +0.012 to +0.025 | about +0.014 |
| B0001059, 38V f/2.5 | +0.016 | −0.042 | −0.040 to −0.051 | about −0.043 |

  (Green, stops. The last column converts Finding 42's dL\* difference at about 15 L\* per stop.) The
  offset is flat from r 0.1 to the corners; only the centre spot differs.
- **New calibration** (`state_lcc.py calibrate`): the reference flat is the map multiplied into the raw
  (`make_preprocess_map.apply_map`), with no LCC and no Light Falloff, instead of a flat with no correction.
  All three flats are at L\* 40 so the reference's corners don't clip at f/2.5, and they're compared over
  r 0.05 to 0.9, not at the centre. Old calibrations are kept in `data/lcc_c1/calibration_f41/`.
  `state_lcc.py --recalibrate ensure RAW…` redoes old ones.
- **All 28 recalibrated.** Exposure moved +0.016 to +0.022 stop on the 28P at f/4, −0.010 to −0.015 on the
  28P at f/5.6 to f/16, +0.037 on the 38V at f/2.5, +0.006 and +0.012 on the 35-100 at f/4, and within
  ±0.004 elsewhere. kR and kB moved by at most 0.003.
- **Validation**, the same 28 photos against fresh Phocus renders (mean of per-photo medians, ΔE00):

| | pre-processed | Finding 42 | now |
|---|---|---|---|
| 28P (7): median / p90 / corners | 0.540 / 1.057 / 0.601 | 0.584 / 1.084 / 0.686 | 0.554 / 1.084 / 0.632 |
| 28P mean \|dL\*\| | 0.06 | 0.23 | 0.06 |
| 35-100 (16): median / p90 / corners | 0.477 / 0.956 / 0.488 | 0.495 / 0.967 / 0.504 | 0.483 / 0.962 / 0.501 |
| Third body (5): median / p90 / corners | 0.316 / 0.824 / 0.318 | 0.334 / 0.834 / 0.344 | 0.328 / 0.830 / 0.346 |
| All 28, median | 0.464 | 0.489 | 0.473 |

  No photo is worse than pre-processing by more than 0.04. B0001059 (38V) is 0.37 with dL\* −0.21, against
  0.35 / −0.21 pre-processed and 0.82 / −0.85 in Finding 42.
- **Contact sheet** rebuilt: 0.27–0.74 against Capture One's default 2.95–6.29 (unchanged).
- Still unexplained: the small spot at the centre where the LCC differs (+0.002 to +0.016 stop, inside
  r 0.02), and about ±0.2 L\* per photo that pre-processing had too.
- Logs: `data/logs/r20_lcc_brightness.txt`, `r20_refflats.txt`, `r20_recalibrate.txt`, `r20_validate.txt`,
  `r20_summary.txt`, `sheet_r20.txt`; scores in `data/val/r20/val/`.

### Finding 44: shade profiles for the 28P and 35-100

**In plain terms:** In shade and overcast light, Phocus's colour moves away from what one daylight profile
can follow. Like the tungsten profiles, a separate profile fitted on synthetic grids at shade white
balances fixes most of it. On six real 35-100 photos at 6221 to 7369 K it brings Phocus-matching from
0.46–0.81 ΔE00 down to 0.44–0.54, and on the one 28P photo in that range from 0.65 to 0.42. Settings files
now pick the shade profile automatically above 6200 K.

- **Where the grids go.** Real X2D II As Shot neutrals don't sit on the ColorMatrix1 daylight locus. Across
  116 raws (the owner's and blog.tonalphoto.com's) they're offset by a median −0.040 in log red and −0.057
  in log blue, and by −0.063 and −0.067 between 6000 and 6600 K. Shade grids use the locus at each
  temperature with an offset of (−0.055, −0.06). At 6300 K that's (0.339, 1, 0.632), next to B0000850's
  (0.338, 1, 0.625). `c1_sidecars.daylight_neutral` computes it.
- **The fit** (`auto_lens_profile.py --light shade`): a 33-level grid at 7300 K in three parts plus 22-level
  grids at 6300, 6800, 7900 and 8600 K, in copies of the daylight profiles' hosts (B0000931 for the 28P,
  B0000727 for the 35-100) with the reference body's record, with Phocus's spatial gain from a flat removed.
  Synthetic fit residual median 0.58, p90 2.5, for both lenses. Pairs saved as
  `fit/synthetic/pairs/HNCSshade<lens>_*.npz`.
- **Offline scoring** (`tools/round21/shade_eval.py`, Capture One's camera RGB from the patch lookup). On
  Finding 33's grids (35-100 host, on the locus, not used in either fit) it reproduces Finding 33's Capture
  One numbers to within 0.03 for the daylight profile. Median ΔE00, C\* 20–40 | C\* 40–60:

| grid | 35-100 daylight | 35-100 shade |
|---|---|---|
| 5500 K | 0.37 \| 0.47 | 1.28 \| 1.55 |
| 6000 K | 0.86 \| 0.92 | 0.99 \| 1.21 |
| 6500 K | 1.28 \| 1.26 | 0.72 \| 0.85 |
| 7000 K | 1.54 \| 1.58 | 0.37 \| 0.38 |
| 7500 K | 1.85 \| 1.88 | 0.77 \| 0.49 |
| 8000 K | 2.13 \| 2.36 | 1.12 \| 0.94 |

  On its own grids (training data) the 28P shade table goes from 1.13 | 1.59 to 0.64 | 0.62 at 7300 K and
  from 1.71 | 2.62 to 0.85 | 1.11 at 8600 K; at 6300 K the two are about equal (0.64 | 0.94 against
  0.81 | 1.03).
- **Real photos** against Phocus with corrections on, whole frame, settings file with the lens setting's
  LCC (two LCCs made for this: 35-100 35 mm f/5.6 and 40 mm f/16). Median, corners (r > 0.8), dL\*:

| photo | As Shot | daylight profile | shade profile |
|---|---|---|---|
| B0000850, 35-100 | 6221 K | 0.46, 0.41, −0.32 | 0.44, 0.44, −0.34 |
| B0000841, 35-100 | 6241 K | 0.56, 0.61, −0.09 | 0.45, 0.40, −0.12 |
| B0000901, 28P | 6255 K | 0.65, 0.77, −0.06 | 0.42, 0.33, +0.03 |
| B0000840, 35-100 | 6420 K | 0.58, 0.62, −0.07 | 0.44, 0.45, −0.22 |
| B0000633, 35-100 | 6574 K | 0.62, 0.66, +0.27 | 0.54, 0.44, −0.16 |
| B0000688, 35-100 | 6658 K | 0.69, 0.55, +0.12 | 0.50, 0.28, +0.04 |
| B0000686, 35-100 | 7369 K | 0.81, 1.01, +0.32 | 0.50, 0.72, +0.05 |

- **Switch point.** The synthetic grids tie at about 6300 K, and every real photo from 6221 K up is better
  with the shade profile, so `c1_sidecars.py` uses it above 6200 K (`SHADE_ABOVE_K`). Photos below that are
  unchanged. Of the 28 validation photos only B0000633 crosses it, which takes the 35-100 mean from 0.483 to
  about 0.478.
- **Contact sheet:** frame 901 (6255 K) now gets the 28P shade profile: 0.65 → 0.42. The other rows are
  unchanged.
- Still open: shade profiles for the other lenses, and more real 28P shade photos.
- Logs: `data/logs/r21_shade_build.txt`, `r21_eval_35-100.txt`, `r21_eval_28p.txt`, `r21_real.txt`,
  `r21_summary.txt`, `sheet_r21.txt`; scores in `data/val/r21/`.

### Finding 45: shade profiles for every lens

**In plain terms:** The other six lenses now have shade profiles too, built the same way from the same raws
as their daylight profiles. None of those lenses has a real photo above 6200 K to test on, so they were
checked on synthetic grids at 6500 and 7500 K that no fit used, placed on the theoretical daylight line
rather than the line the fitting grids follow. Every lens behaves like the 28P and 35-100: the shade profile
is better at both, and roughly halves the error at 7500 K.

- Built with `auto_lens_profile.py RAW --light shade --install` from B0001059 (38V) and blog.tonalphoto.com's
  B0041052 (25V), B0039850 (55V), B0038945 (90V), B_23969 (120 Macro) and B0040287 (135), each with the
  reference body's record. Synthetic fit residual median 0.58 for every lens, p90 2.45–2.52.
- Held-out grids (`tools/round21/heldout_grids.py`): one 22-level grid per lens at 6500 K and at 7500 K on
  the ColorMatrix1 daylight locus, with no off-locus offset, in copies of the same hosts. Scored offline
  through Capture One's camera RGB from the patch lookup. Median ΔE00, C\* 20–40 | C\* 40–60:

| lens | 6500 K daylight | 6500 K shade | 7500 K daylight | 7500 K shade |
|---|---|---|---|---|
| 28P | 0.78 \| 1.13 | 0.75 \| 0.87 | 1.31 \| 2.00 | 0.56 \| 0.61 |
| 35-100 | 0.83 \| 1.26 | 0.63 \| 0.79 | 1.35 \| 2.10 | 0.42 \| 0.48 |
| 38V | 1.07 \| 1.66 | 0.65 \| 0.79 | 1.64 \| 2.54 | 0.46 \| 0.48 |
| 25V | 0.82 \| 1.27 | 0.66 \| 0.81 | 1.38 \| 2.12 | 0.47 \| 0.54 |
| 55V | 0.83 \| 1.20 | 0.73 \| 0.82 | 1.38 \| 2.10 | 0.54 \| 0.56 |
| 90V | 0.80 \| 1.23 | 0.64 \| 0.81 | 1.38 \| 2.11 | 0.44 \| 0.48 |
| 120 Macro | 0.78 \| 1.21 | 0.56 \| 0.74 | 1.33 \| 2.10 | 0.35 \| 0.44 |
| 135 | 0.81 \| 1.20 | 0.56 \| 0.74 | 1.33 \| 2.11 | 0.35 \| 0.43 |

- These scores include Phocus's spatial gain at each patch (not removed here), the same for both profiles.
- Still open: real shade photos for these six lenses.
- Logs: `data/logs/r22_shade_build.txt`, `r22_heldout.txt`, `r22_heldout_scores.txt`; held-out pairs in
  `data/synth/r22/`.

### Finding 46: tungsten profiles for every lens

**In plain terms:** `auto_lens_profile.py` can now build a lens's tungsten profile by itself, the same way
the 28P and 35-100 ones were built by hand in Finding 21. Rebuilt this way, the 35-100's comes out the same
as the shipped one, so the other six lenses got theirs. On synthetic grids at white balances between the
fitting ones, the tungsten profiles cut the error under tungsten light two- to fourfold for every lens.

- **`auto_lens_profile.py RAW --light tungsten`**: 22-level grids at Finding 21's four tungsten neutrals
  (Capture One reads them as about 4175, 3747, 3395 and 3105 K), in a copy of one of the lens's raws with the
  reference body's record, rendered by Phocus with lens corrections off; Phocus's spatial gain removed with
  a flat; fitted like every other profile.
- **Check:** the 35-100 tungsten profile rebuilt from B0000727 matches the shipped table to a median of
  0.01 ΔE00 (p90 0.02) over the patch lookup's moderate colours, and scores the same on the shipped tungsten
  pairs. Synthetic fit residual median 1.24, p90 3.4–3.5 for every lens (higher than daylight, as in
  Finding 21, because Phocus's colour moves more across 1000 K of tungsten than across daylight).
- **Built** for the 38V (B0001059) and the 25V, 55V, 90V, 120 Macro and 135 (blog.tonalphoto.com's raws).
- **Held-out grids** (`tools/round21/heldout_grids.py --tungsten`): one 22-level grid per lens halfway, in
  log space, between the first two and between the last two tungsten grids (t = 0.625 and 1.125). Median
  ΔE00, C\* 20–40 | C\* 40–60:

| lens | warmer pair, daylight | warmer pair, tungsten | cooler pair, daylight | cooler pair, tungsten |
|---|---|---|---|---|
| 28P | 3.28 \| 6.12 | 0.94 \| 1.44 | 1.99 \| 3.39 | 0.87 \| 1.26 |
| 35-100 | 3.17 \| 5.69 | 0.87 \| 1.36 | 1.85 \| 3.16 | 0.79 \| 1.25 |
| 38V | 2.90 \| 5.42 | 0.91 \| 1.39 | 1.66 \| 2.73 | 0.81 \| 1.22 |
| 25V | 3.22 \| 5.70 | 0.90 \| 1.38 | 1.87 \| 3.22 | 0.82 \| 1.27 |
| 55V | 3.17 \| 5.99 | 0.96 \| 1.44 | 1.86 \| 3.22 | 0.81 \| 1.21 |
| 90V | 3.16 \| 5.74 | 0.89 \| 1.36 | 1.88 \| 3.15 | 0.81 \| 1.21 |
| 120 Macro | 3.12 \| 5.83 | 0.84 \| 1.34 | 1.87 \| 3.19 | 0.78 \| 1.24 |
| 135 | 3.12 \| 5.77 | 0.83 \| 1.33 | 1.89 \| 3.19 | 0.79 \| 1.25 |

  ("Warmer pair" is between the two warmest grids, about 3100–3400 K; "cooler pair" between the two coolest,
  about 3750–4175 K. Phocus's spatial gain at each patch is left in, the same for both profiles.)
- `c1_sidecars.py` already picks a lens's Tungsten profile below 4400 K when there is one.
- Logs: `data/logs/r23_tungsten_build.txt`, `r23_tungsten_check.txt`, `r23_heldout_tungsten.txt`,
  `r23_heldout_tungsten_scores.txt`; held-out pairs in `data/synth/r23/heldout/`.

### Finding 47: the LCC's colour is built into the profiles, so white balance can be changed

**In plain terms:** Each LCC shifts a photo's overall colour slightly, and the settings file used to cancel
that through white balance. Choosing a white-balance preset, a Kelvin value or As Shot in Capture One threw
the correction away, which on the 28P and 55V left greys about 2.5–2.9 ΔE00 off. The shift is nearly the
same for every setting of a lens, so each lens's average is now built into its profiles, and only a tiny
per-setting remainder stays in the white balance. On the owner's camera, choosing As Shot now scores the
same as the prepared photo (0.26–0.64 ΔE00, against 0.61–1.67 before). The styles were removed, because
a profile with this correction is only right with the LCC.

- **It can't be cancelled inside the LCC.** Capture One normalises each channel of an LCC by its own
  average. Painting every photosite outside the default crop (3.2% of the sensor) a strong colour changed
  nothing, to five decimals (`tools/round23/lcc_margin_test.py`), so the margin isn't in that average.
- **Size of the shift** if the white balance loses it (offline, through each profile; ΔE00 on greys,
  median | max, and on moderate colours, median): 28P 2.51 | 2.93, 0.88; 55V 2.88 | 3.34, 1.02; 90V 1.85,
  0.68; 38V 1.58, 0.57; 25V 1.14, 0.37; 35-100 0.92, 0.27; 120 Macro 0.90, 0.33; 135 0.67, 0.24.
- **The factor per lens** is the mean of its calibrated LCCs (`fit/lcc_colour.json`): 28P red ×1.0189,
  blue ×0.9623 (9 LCCs, each within 0.003); 35-100 ×1.0018, ×0.9872 (18); 38V ×1.0146, ×0.9763; 25V ×1.0060,
  ×0.9828; 55V ×1.0219, ×0.9568; 90V ×1.0173, ×0.9720; 120 Macro ×1.0084, ×0.9867; 135 ×1.0057, ×0.9900.
  A lens without LCCs gets an estimate from its maps: each channel's mean of 1/map, which is what the LCC is
  normalised by, is within rms 0.003 (red) and 0.005 (blue) of calibration over the 30 calibrated LCCs.
- **A plain input gain wasn't equivalent.** Built into the profile as a gain at its input, the factor
  changed real photos even with the white balance divided by the same amount: B0000939 0.44 → 0.59,
  B0001059 0.37 → 0.50, B0039850 0.54 → 0.44. Capture One does apply the gain (doubling it took B0000939
  to 0.91, squaring it to 2.02), but white balance multiplies the raw values before Capture One's raw
  response, whose toe has a local slope of 1.4 to 1.9 in the shadows and about 1.0 from mid-grey up. So a raw
  multiplier counts for more in dark tones than the same gain after the response.
- **The fix:** `write_profile.py --raw-gain` builds the factor into the input curves through that response
  (`fit/c1_response.npz`), so it acts exactly like a raw multiplier. Prepared photos then score as before
  the change: B0000939 0.44, B0001059 0.37, B0039850 0.53 (0.54), B0000919 0.26 (0.25). Without a gain the
  profiles still rebuild byte for byte.
- **Validation:** each photo as prepared, with the white balance set to plain As Shot (what choosing As
  Shot in Capture One does), and with As Shot as it behaved before (the profile's factor divided out
  again). Median ΔE00 against Phocus with corrections on:

| photo | lens, As Shot | prepared | As Shot now | As Shot before |
|---|---|---|---|---|
| B0000919 | 28P, 6092 K | 0.26 | 0.26 | 1.27 |
| B0000939 | 28P, 5095 K | 0.44 | 0.38 | 1.67 |
| B0000633 | 35-100, 6574 K (Shade) | 0.55 | 0.54 | 0.81 |
| B0000773 | 35-100, 4768 K | 0.60 | 0.64 | 0.61 |
| B0001059 | 38V, 4924 K | 0.37 | 0.37 | 1.28 |
| B0039850 | 55V, 5497 K, third body | 0.53 | 1.81 | 1.30 |
| 2462814331 | 35-100, 3306 K (Tungsten), DPReview body | 1.68 | 1.57 | 1.70 |
| 5727340084 | 35-100, 3207 K (Tungsten), DPReview body | 0.78 | 1.39 | 1.19 |
| 8671272247 | 35-100, 3208 K (Tungsten), DPReview body | 1.12 | 1.36 | 1.29 |

- **Other bodies:** their factory-gain correction still rides in the white balance (it differs per camera,
  and the profiles are shared), so As Shot or a preset drops it there; for the third body that's the
  1.3–2.0 ΔE00 of Finding 35. The white-balance picker on a grey isn't affected.
- **Tools:** `bake_lcc_colour.py` writes the factor into every profile of a lens and records it;
  `c1_sidecars.py` divides it out of each photo's white balance (and, on the fallback without an LCC,
  cancels it there); `auto_lens_profile.py` bakes new profiles; `prepare_folder.py --make-lccs` replaces a
  map estimate with the calibrated factor once, and never changes a calibrated one under photos already
  prepared. The styles are gone, and `write_profile.py` no longer writes them.
- The tungsten profiles of Finding 46 picked up their lens's factor too. Three new LCCs were made for the
  DPReview tungsten frames' settings (35-100 at 85, 90 and 100 mm, f/4.5).
- Logs: `data/logs/r23_margin_test.txt`, `r23_wb_shift_size.txt`, `r23_validate.txt`, `r23_validate2.txt`,
  `r23_summary.txt`; scores in `data/val/r23/`.

### Finding 48: profiles installed for your own camera keep its correction under any white balance

**In plain terms:** After Finding 47, a camera other than the one the profiles were made on still had its
factory calibration corrected through white balance, so choosing As Shot or a preset in Capture One dropped
it. `bake_lcc_colour.py --body` now builds one camera's correction into the profiles it installs. Tested on
blog.tonalphoto.com's camera: As Shot scores the same as the prepared photo (0.56 against 1.81 before on the
55V raw), and the prepared photos score within 0.03 ΔE00 of before.

- **`bake_lcc_colour.py --body RAW`** reads RAW's GAIN record, computes the camera RGB gains (1, kG, kB) as
  `c1_sidecars.body_gain` does (Findings 13, 35), and writes every installed profile with the lens's LCC
  colour factor times those gains, through Capture One's raw response (`--raw-gain`). It records them in
  `data/installed_body.json`, which isn't committed. `--body reference` removes that file and installs the
  repo's profiles again. The repo's profiles are always made for the reference body.
- **Settings files** (`c1_sidecars.baked_colour`) divide the installed gains out of each photo's white
  balance and exposure as well as the LCC colour factor. For the installed camera the body correction then
  cancels completely; for any other camera the ratio stays in the white balance, as before.
- `bake(…, install=True)`, used by `auto_lens_profile.py --install` and `prepare_folder.py`, writes installed
  profiles with the recorded camera's gains too, so a new lens's profile matches the rest.
- **Test:** profiles installed for the third body (record 2.049, 2.250: green ×0.9818, blue ×0.9997), its
  five raws scored as prepared and with the white balance set to plain As Shot:

| raw | lens | prepared, reference profiles | prepared, its own profiles | As Shot, its own profiles |
|---|---|---|---|---|
| B0038945 | 90V | 0.27 | 0.28 | 0.28 |
| B0039850 | 55V | 0.53 | 0.56 | 0.56 (1.81 with reference profiles) |
| B0040287 | 135 | 0.19 | 0.22 | 0.22 |
| B0041052 | 25V | 0.48 | 0.49 | 0.49 |
| B_23969 | 120 Macro | 0.16 | 0.15 | 0.15 |

  As Shot equals prepared here because each of these lenses has one LCC, so the per-setting remainder is
  exactly 1. The prepared photos come out up to 0.03 worse with the camera's green gain in the profile than
  in Capture One's Exposure; the cause isn't known.
- The owner's camera is the reference body, so its installed profiles are the repo's.
- Logs: `data/logs/r24_run.txt`, `r24_summary.txt`; scores in `data/val/r24/`.

### Finding 49: the 38V's profiles are corrected for a constant 0.2 L\* darkness

**In plain terms:** Finding 32 saw the 38V render about 0.2 L\* darker than Phocus. With the settings-file
route and an LCC for each of its settings, all nine of the owner's 38V frames show it: 0.19 to 0.24 L\*
darker, at every aperture from f/2.5 to f/8 and at two focus distances. A brightness gain of +0.012 stop,
built into the 38V's profiles, removes it, and every frame scores better.

- Five new LCCs for the frames' settings (38V f/2.8, f/4, f/5.6 and f/8 at focus code 4, and f/2.5 at code 1).
  The 38V's LCC colour factor is now the mean of six (red ×1.0164, blue ×0.9735, from ×1.0146 and ×0.9763).
- `fit/lcc_colour.json` can carry a lens's `ev`; `bake_lcc_colour.py` builds 2^ev into all its profiles with
  the colour factor (and any installed camera gains), and settings files don't divide it out. The 38V's is
  +0.012 stop: 0.21 L\* at about 18 L\* per stop on HNCS's curve in the midtones.
- Median ΔE00 against Phocus with corrections on, and dL\*:

| frame | setting | before | corrected | corrected, As Shot |
|---|---|---|---|---|
| B0001059 | f/2.5, code 4 | 0.37, −0.21 | 0.31, −0.00 | 0.31, +0.02 |
| B0001060 | f/2.5, code 4 | 0.37, −0.19 | 0.30, +0.01 | 0.30, +0.03 |
| B0001061 | f/5.6, code 4 | 0.38, −0.24 | 0.32, −0.02 | 0.36, −0.04 |
| B0001062 | f/8, code 4 | 0.44, −0.23 | 0.40, −0.00 | 0.44, −0.03 |
| B0001063 | f/5.6, code 4 | 0.42, −0.20 | 0.40, +0.03 | 0.42, +0.01 |
| B0001064 | f/4, code 4 | 0.43, −0.19 | 0.40, +0.04 | 0.41, +0.02 |
| B0001065 | f/2.8, code 4 | 0.40, −0.20 | 0.36, +0.03 | 0.36, +0.04 |
| B0001066 | f/2.5, code 4 | 0.41, −0.23 | 0.36, −0.01 | 0.37, +0.02 |
| B0001067 | f/2.5, code 1 | 0.50, −0.19 | 0.45, +0.03 | 0.59, +0.07 |
| mean | | 0.415, −0.207 | 0.367, +0.013 | 0.395, +0.016 |

- **Contact sheet** rebuilt: 0.26–0.74 against Capture One's default 2.95–6.29. Frame 1059 goes from 0.37 to
  0.31, and 901 from 0.42 to 0.38 (the colour factor of Finding 47 now built into its shade profile).
- Why the 38V is darker isn't known. No other lens shows a consistent offset (the 28P's and 35-100's
  dL\* vary in sign from photo to photo; the other lenses have one photo each).
- Logs: `data/logs/r24_run.txt`, `r24b_run.txt`, `r24_summary.txt`; scores in `data/val/r24/`.

### Finding 50: settings files work in catalogs

**In plain terms:** Everything so far was tested in Capture One sessions. Imported into a catalog, prepared
raws get every setting from their settings files too: profile, Linear Response, exposure, white balance and
the lens setting's LCC. They render identically to the same photos in a session, whether the files stay
where they are or are copied into the catalog. The one condition is the import option **Include existing
adjustments**, which was already on in a new catalog.

- A new catalog (`make new document with properties {kind:catalog, …}` through AppleScript), three raws
  (B0000919 28P, B0001059 38V, B0000633 35-100 in shade) prepared with `prepare_folder.py` in three folders
  and imported three ways: referenced with Include existing adjustments, copied into the catalog with it,
  and referenced without it.
- Read back through AppleScript, the first two got exactly the settings files' values (B0000633 the 35-100
  Shade profile, exposure, white balance 6736 K / 6.4, LCC with colour cast and uniform light on, Light
  Falloff 0). Without the option, they got Capture One's defaults: ProStandard, the Auto curve, As Shot,
  no LCC, Light Falloff 100.
- Rendered at 50% (sharpening, noise reduction and distortion off) and scored against Phocus, both imports
  match the session results exactly: B0000919 0.26, B0001059 0.31, B0000633 0.55, with the same dL\* and
  per-radius scores.
- In AppleScript a catalog's import settings must be set inside a `tell import settings of document` block;
  setting them through a variable fails with "Invalid key form" (−10002), as does addressing a document by
  name.
- Logs: `data/logs/r25_catalog_settings.txt`, `r25_catalog_scores.txt`.

### Finding 51: warm profiles fill the gap between tungsten and daylight

**In plain terms:** Between about 4000 and 4850 K (early morning, late afternoon, warm indoor light) neither
the tungsten nor the daylight profile fitted Phocus well. A fourth profile per lens, fitted on grids at
4200–5000 K, roughly halves the error there on synthetic grids for every lens, and beats the daylight
profile on all ten real photos up to 4829 K (one ties) (for example 0.74 → 0.31 and 0.77 → 0.28 on two 28P photos).
Settings files pick it automatically. With all four profiles picked this way, the 28 validation photos
average 0.444 ΔE00, better than pre-processing's 0.464, which only had daylight profiles.

- **`auto_lens_profile.py --light warm`**: a 33-level grid at 4600 K and 22-level grids at 4200, 4400, 4800
  and 5000 K, on the ColorMatrix1 daylight locus offset by (−0.02, −0.04) in log red and blue, where the
  owner's photos in that range sit (they average about −0.025 and −0.047 from 4200 to 5000 K, closer to
  the locus than in shade). Built for all eight lenses from their daylight hosts; synthetic fit residual
  median 0.48, p90 2.1.
- **Synthetic grids not used in any fit**, 35-100, median ΔE00 C\* 20–40 | C\* 40–60:

| grid | tungsten | warm | daylight |
|---|---|---|---|
| tungsten held-out, about 3950 K | 0.79 \| 1.25 | 1.03 \| 1.68 | 1.85 \| 3.16 |
| locus 4200 K (Finding 33) | 1.61 \| 1.59 | 0.67 \| 1.10 | 1.47 \| 2.52 |
| held-out 4300 K | 1.41 \| 2.06 | 0.60 \| 0.89 | 1.38 \| 2.21 |
| locus 4500 K | 2.01 \| 2.20 | 0.29 \| 0.43 | 1.17 \| 1.97 |
| held-out 4700 K | 1.88 \| 2.84 | 0.35 \| 0.43 | 0.96 \| 1.50 |
| locus 5000 K | 2.61 \| 3.12 | 0.72 \| 0.92 | 0.63 \| 0.88 |
| locus 5500 K | 3.29 \| 4.21 | 1.47 \| 1.89 | 0.37 \| 0.47 |

  For every lens, held-out grids at 4300 and 4700 K go from 1.18–1.50 to 0.57–0.75, and from 0.78–1.10 to
  0.29–0.48 (C\* 20–40) with the warm profile.
- **Real photos** (settings file with the lens setting's LCC; four new LCCs for 35-100 settings), median
  ΔE00 and dL\*:

| photo | As Shot | daylight | warm |
|---|---|---|---|
| B0000862, 35-100 | 4367 K | 0.60, −0.21 | 0.47, −0.33 |
| B0000672, 35-100 | 4490 K | 0.49, +0.11 | 0.49, −0.01 |
| B0000673, 35-100 | 4548 K | 0.48, +0.18 | 0.45, +0.02 |
| B0000678, 35-100 | 4622 K | 0.77, +0.20 | 0.61, +0.10 |
| B0000863, 35-100 | 4668 K | 0.63, −0.00 | 0.47, −0.22 |
| B0000917, 28P | 4737 K | 0.74, +0.08 | 0.31, −0.05 |
| B0000926, 28P | 4751 K | 0.77, +0.10 | 0.28, −0.09 |
| B0000707, 35-100 | 4764 K | 0.54, +0.08 | 0.51, −0.11 |
| B0000773, 35-100 | 4768 K | 0.60, +0.18 | 0.40, +0.00 |
| B0000807, 35-100 | 4829 K | 0.46, +0.11 | 0.39, −0.04 |
| B0000864, 35-100 | 4872 K | 0.48, −0.03 | 0.58, −0.15 |
| B0000868, 35-100 | 4924 K | 0.44, −0.01 | 0.55, −0.13 |
| B0000654, 35-100 | 4936 K | 0.49, −0.04 | 0.51, −0.10 |
| B0000711, 35-100 | 4951 K | 0.46, +0.13 | 0.50, −0.23 |

- **Switch points** (`c1_sidecars.py`): tungsten below 4000 K, where it and the warm profile tie on the
  synthetic grids; warm up to 4850 K, between the last real photo it wins (4829 K) and the first it loses
  (4872 K), a little below where the synthetic grids tie (about 4950 K); daylight to 6200 K; shade above.
- **The validation set** (28 photos), with each photo's profile picked this way (B0000926, B0000673 and
  B0000773 now get the warm profile, B0000633 the shade one): 28P 0.554 → 0.483 (corners 0.632 → 0.603),
  35-100 0.483 → 0.464 (0.501 → 0.480), third body unchanged at 0.328; all 28, 0.473 → 0.444.
- **Contact sheet** rebuilt: 917 0.74 → 0.31 and 773 0.60 → 0.40 (both warm); 0.26–0.44 overall against
  Capture One's default 2.95–6.29.
- Logs: `data/logs/r26_warm_build.txt`, `r26_build6.txt`, `r26_eval.txt`, `r26_heldout_scores.txt`,
  `r26_real.txt`, `r26_real_summary.txt`, `r26_validation_now.txt`, `sheet_r26.txt`; held-out pairs in
  `data/synth/r26/`; scores in `data/val/r26/`.

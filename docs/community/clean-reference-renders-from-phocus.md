# Making clean Phocus reference renders

**What this adds:** edge-lift figures for the 55V and 90V with the lens tool off (`docs/results.md` has the 28P
and 35-100), and evidence that Phocus chooses its own `VignetteAmount` per file when it first opens it. Two
verification steps for hot-folder renders are at the end. The repo used Phocus 4.2.2 and I used 4.2.3; I have
not tested 4.2.2.

In this note "stops" means stops of scene light, measured with the tone curve taken out by an exposure bracket.
"Output EV" means measured on the rendered picture, after the tone curve.

## Phocus lifts the edge with the lens tool off

Five wall frames, XCD 55V at f/8, ISO 50, 1/6 s to 3 s, focused at about 0.6 m. Gain in stops of scene light
relative to Phocus's own centre (radius 1 is the corner; median over the three or four exposures that keep
the ring):

| ring | 0.2-0.3 | 0.4-0.5 | 0.6-0.7 | 0.8-0.9 | 0.9-1.0 |
|---|---|---|---|---|---|
| Phocus as it opens the raw | +0.021 | +0.095 | +0.231 | +0.409 | +0.498 |
| Phocus lens tool off | +0.019 | +0.091 | +0.231 | +0.408 | +0.495 |
| Phocus lens on, vignetting 12 | +0.027 | +0.117 | +0.278 | +0.490 | +0.608 |

These figures come from a pass in which the leave-one-exposure-out check missed its bar at 1.4 s.

Lens on at amount 0 against lens off is within 0.004 stops in every ring (55V). On a 90V woodland set (f/5.6
and smaller) lens on at amount 0 differs from lens off by under 0.001 output EV in every ring and channel.
With the tool off, ten frames rendered at `VignetteAmount` 0 and 12 were identical (+0.000 output EV in every
radius bin).

## Phocus sets its own vignetting amount

On the ten f/8 wall frames (five 55V, five 90V), opened with no sidecar and nothing touched, Phocus came up
with Lens Corrections on, Chromatic Aberration Standard, Distortion ticked and Vignetting ticked with the
slider at 0 (read on screen for four frames, from the written sidecars for all ten). Exports made that way
matched my amount-0 renders to 0.0001 stops in every channel. On a 90V set at f/2.5, Phocus's first-load
amount was 10 to 20 (7 frames). My earlier template sidecars carried 12 on the ten wall frames, where
Phocus's own value was 0.

The repo's template sidecar (`tools/phocus_standard.phos`) carries `VignetteAmount` 100. For scale, on a 55V
f/2.5 frame at infinity, amount 100 against lens off gives log2 +2.277 of the output ratio at r 0.9 (after the
tone curve, on a wall that is not flat).

## Two checks for hot-folder renders

The repo's `phocus_hotfolder.py` returns the first new TIFF that appears after a drop and waits for its size to settle.
Two checks I add on top, not in that tool as far as I can tell: compare the TIFF's EXIF (capture time, ISO,
f-number, shutter) with the raw's to catch a mispaired neighbour, and read back the sidecar Phocus rewrote
(its last settings entry) to confirm what it actually rendered.

## What this does not show

- 55V numbers: f/8, close focus, one wall, one session. Phocus 4.2.3, not 4.2.2.
- Two checks in the 55V pass missed their bars and are not passes: leave-one-exposure-out on the tone
  response at 1.4 s (+0.055 to +0.085 against 0.03), and a check against a second converter's lens
  correction in the two outermost rings (unresolved).
- Absolute lift depends on how brightness is measured, by about 0.03 stops (see the pitfalls note).

## Status

Not independently replicated.

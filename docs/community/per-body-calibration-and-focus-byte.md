# Per-body calibration and the focus byte: a transplant test on Phocus 4.2.3

**What this adds:** (1) One data point on the question left open in `docs/writeup.md` ("If Lightroom doesn't
read it either..."): Lightroom Classic 16.0 responded to bytes 84-87 on the one pair tested. (2) The effect of
the focus byte depends strongly on `VignetteAmount`. The rest repeats Findings 2, 13, 16 and 31 with controls.
The repo used Phocus 4.2.2 and I used 4.2.3; I have not tested 4.2.2.

## Lightroom Classic 16.0 responded to bytes 84-87 on one pair

`docs/writeup.md` says "Capture One ignores it; I haven't tested Lightroom." I tested one pair: one host body
and one donor record. Lightroom Classic 16.0 (process version 15.4), a camera profile, As Shot, Exposure 0,
profile corrections and CA removal off, TIFF 16-bit Adobe RGB, no resize; exported XMP settings identical on
all three files. Two unmodified copies export bit-identical (0 of 305,690,256 samples). Bytes 84-87 replaced
with a second body's values: ΔE00 P50 / P90 / P99 1.01 / 1.18 / 1.26, dL\* / da\* / db\* (P50) +0.17 / -0.34 /
-0.99. The same change in Phocus gives 1.11 / 1.22 / 1.25 and +0.17 / -0.35 / -1.10. So Lightroom Classic's
output changed by nearly the same size and direction as Phocus's. One pair, one direction, one profile; I did
not test Capture One, other bodies or other Lightroom versions.

## Focus byte depends on the vignetting amount

55V at f/2.5, byte 6 code 0 against 14 on flat-wall frames at infinity and minimum focus. Lens on at
`VignetteAmount` 100: ΔE00 P50 / P90 / P99 0.89 / 1.63 / 1.74 and 0.84 / 1.62 / 1.93. At `VignetteAmount` 0:
0.19 / 0.34 / 0.47. Lens tool off: 0.06 / 0.10 / 0.13. Swapping the byte swaps the correction curves (a real
and a transplanted file at the same code differ by at most 0.007 in log2 green ratio inside r 0.5, 0.02-0.03
at the edge). The research log does not say which amount Finding 31's table used.

## Repeats of Findings 2, 13, 16 and 31

Hot folder, TIFF 16-bit, Adobe RGB (1998), Standard, EV 1.0, as-shot WB. Hosts are my own raws
(XCD 55V); the record donor is a file from a second body. Each test file is a host copy with one named
change; comparison on 64 px block means, channels between 0.02 and 0.90 linear in both renders; ΔE00 is
CIEDE2000, checked against a published test pair.

- My body's half-floats are 2.0488 / 2.2500. Replacing the whole donor record except bytes 84-87 (25 bytes
  differ) is bit-identical to the control; bytes 84-87 alone equal the whole record. Bytes 82-83 were never
  varied.
- Both values from the second body, lens on: ΔE00 P50 1.11, db\* -1.10. Raising only the first value by a
  factor of about 1.27 gives 2.04 and db\* +0.91.
- The repo's rule as raw gains comes close to each single change (P50 0.16 and 0.09). The exponents I read are
  about 0.072 (green) and 0.082 (blue) against its 0.077 and 0.078, estimates from medians, one step each, not
  a fit.
- Transplanting 0x0081-0x0083 (lens on, `VignetteAmount` 0) was bit-identical to the control.

## What this does not show

- One host body, one step per value, one lens, daylight-ish white balance; Lightroom one pair.
- Byte 6: codes 0 and 14 only. Output ratios are after the tone curve. The wall frames are not flat fields
  (L\* 53 at the centre to 23 in the corner, lens tool off).
- Phocus 4.2.3 here; the repo used 4.2.2 and I have not tested 4.2.2.

## Status

Not independently replicated.

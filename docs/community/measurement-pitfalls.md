# Measurement pitfalls when comparing renders by radius

**What this adds:** a unit warning (my blog's 90V figure that Finding 36 compares with is output EV, not
stops) and four method points from my radial-brightness work on the XCD 55V and 90V, Phocus 4.2.3 (the repo
used 4.2.2; I have not tested it).

"Stops" here means stops of scene light, measured with the tone curve taken out by an exposure bracket (point
4). "Output EV" means measured on the rendered picture, after the tone curve.

**1. Select and bin blocks on raw level, not on rendered output.** The radial difference itself moves the
rendered level, so an output floor or ceiling trims the edge unevenly. For the woodland set I keep blocks with
the raw unclipped and raw green at least 2^-9 of full scale, binned in 1-stop bands of raw green.

**2. The raw's colour changes with radius on some lenses.** On the 55V, red over green falls about 2% and blue
over green rises about 9% by r 0.8-0.9, so level measured as raw green differs from level measured as a
luminance formed from all three channels. A control render from another converter with its lens corrections
off drifted by -0.029 stops at the corner with raw green, over my 0.02 bar, and was flat within 0.003 in every
ring with the luminance. I changed the measure after seeing that. It moves Phocus's absolute corner lift on
the 55V from +0.47 to +0.50 stops; differences between renders move by under 0.01.

**3. Output EV is larger than stops of exposure.** On the dim wall frames the output moves about 1.4 output EV
per stop of scene light. My earlier figures were +0.67 output EV at the 55V edge and +0.09 output EV at the
90V centre to +0.29 at its edge, the range Finding 36 compares with its +0.15 stop. Recomputed from the same
arrays, the edge bins are +0.68 output EV (55V) and +0.19 output EV (90V); divided by 1.4 that is +0.49 and
+0.14 stops (estimates). The direct measurement in stops gives +0.41 to +0.50 in the two outer 55V rings and
+0.14 for the 90V (90V: reported, not read). Another table's unit needs checking before comparison.

**4. An exposure bracket takes the tone curve out.** Five frames of one wall, 1/6 s to 3 s. Each
render's tone response comes from its own centre, and a block's gain is how far its output sits from what
that response gives for its raw level. Each ring's gain then agrees across the exposures that keep it to
within 0.013 stops (bar 0.05), so the lift is applied before the tone curve and one figure per ring makes
sense.

**5. Walls are not flat fields, and lens-on renders move blocks.** Wall lighting cannot be separated from
lens falloff, so I compare renders. About a fifth of the wall blocks were objects. Unmasked,
their edges, shifted by the lens-on warp, corrupted the tone response, so I mask them from the raw. Phocus's
lens-on render enlarges the 90V image by 2.75% and the 55V by 0.25%.

## What this does not show

- Close focus only: about 0.6 m on the 55V at f/8, minimum focus on the 90V. EXIF SubjectDistance is the
  4294967295 sentinel on the wall frames of both lenses.
- One wall, one session, five frames.
- Failed or unresolved: leave-one-exposure-out on the tone response missed its bar at 1.4 s (+0.055 to
  +0.085 against 0.03), and a check against a second converter's lens correction missed its bar in the two
  outermost rings, unresolved.

## Status

Not independently replicated.

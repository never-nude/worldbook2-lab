# Worldbook lab — session rules

This repo is the staging lab for worldbook.earth (production: `never-nude/worldbook2`, GitHub Pages). The whole app is one `index.html`.

## Border containment canon (LOCKED 2026-07-02)

**A country's edge color must never paint outside that country's own borders — on any layer, at any zoom.** This was signed off by Mike and is not open for re-litigation by future sessions.

The implementation lives in `index.html` at the `world-lines` source and the `countries-edge-soft` / `countries-edge-tight` layers, under the banner comment `CANON — COUNTRY BORDER-CONTAINMENT RULES`. Its four invariants (winding normalization; inset offset = width/2 at every stop; per-ring + low-zoom size clamps; round joins/caps) are load-bearing. Do **not** change, relax, or "simplify" them — restyles of the Countries view must preserve all four. If a task appears to require breaking one, stop and ask Mike first.

Verification protocol when touching anything near the border/glow rendering: check coastlines at z1.8 (North Atlantic — Ireland/Canada/Norway), z2.6, z3.6-3.8 (W. Europe), z4.5 (Aegean islands), z5.5 (Adriatic), and the ZAF/LSO enclave. Bump `WB_BUILD` when shipping rendering changes.

## Syncing to production

NEVER wholesale-copy `index.html` between this repo and `never-nude/worldbook2` without first fetching and diffing against the destination's HEAD. Multiple sessions push to both repos in parallel. Fetch, diff, merge to the union of features, verify feature grep markers, then push.

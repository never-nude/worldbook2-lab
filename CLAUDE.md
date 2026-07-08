# Worldbook lab — session rules

**⚠️ THIS REPO IS DEMOTED (2026-07-06). `never-nude/worldbook2` (production) is CANONICAL.** Do all development THERE, on branches — merging to its `main` deploys to worldbook.earth. This lab repo is kept only as a mirror; its `index.html` is refreshed FROM production after prod merges (prod→lab copies are safe; lab→prod copies are FORBIDDEN — they clobbered GA4/og:image/migration/geo-ping features in early July 2026). The canon and testing rules below also live in production's own CLAUDE.md, which is the maintained copy.

## Border containment canon (LOCKED 2026-07-02)

**A country's edge color must never paint outside that country's own borders — on any layer, at any zoom.** This was signed off by Mike and is not open for re-litigation by future sessions.

The implementation lives in `index.html` at the `world-lines` source and the `countries-edge-soft` / `countries-edge-tight` layers, under the banner comment `CANON — COUNTRY BORDER-CONTAINMENT RULES`. Its four invariants (winding normalization; inset offset = width/2 at every stop; per-ring + low-zoom size clamps; round joins/caps) are load-bearing. Do **not** change, relax, or "simplify" them — restyles of the Countries view must preserve all four. If a task appears to require breaking one, stop and ask Mike first.

Verification protocol when touching anything near the border/glow rendering: check coastlines at z1.8 (North Atlantic — Ireland/Canada/Norway), z2.6, z3.6-3.8 (W. Europe), z4.5 (Aegean islands), z5.5 (Adriatic), and the ZAF/LSO enclave. Bump `WB_BUILD` when shipping rendering changes.

## Syncing (one direction only)

Production (`worldbook2`) → lab, never the reverse. After a prod merge, copy prod's `index.html` here and commit ("Sync from production <sha>"). Do not develop in this repo; do not push this repo's `index.html` toward production under any circumstances.

# Worldbook beta-hardening — handoff

_Prepared 2026-07-05. Pick this up with Claude Code on any machine that has the `worldbook2-lab` repo. Everything below is versioned in git, so it travels with the repo. Line numbers are against index.html at commit `d913a91` (`WB_BUILD="2026-07-05.1 subnat-fallback"`)._

## TL;DR

An 8-dimension multi-agent audit of `index.html` for public-beta reliability completed its **find** phase and most of its **verify** phase before the session ended. Findings are recovered and captured below — **no need to re-run the audit.** One **P0 cluster is confirmed** (the app dies behind a permanent "Building your world…" screen whenever the map engine can't start). The rest are small, one-line **P1** touch/robustness fixes. Nothing has been applied yet; the working tree is clean.

## Already done & LIVE on worldbook.earth

- **Safari/Ecosia regional drill-down fix** — deployed, verified live (lab `d913a91` / prod `0ac1001`). Root cause: content blockers block `cdn.jsdelivr.net`, so subnational boundary fetches silently failed. Fix added a `raw.githubusercontent.com` fallback + 10s `AbortController` timeout + an inline error message (`_wbSubnatUrls` / `_wbSubnatFetch` near the top of `drillIn`). This is the pattern to imitate for the CDN-resilience fixes below.

## ⚠️ READ FIRST — repo topology & a divergence landmine (verified 2026-07-05)

There are **two GitHub repos with the same app but divergent histories**:

| | repo | laptop path | role |
|---|---|---|---|
| **lab** | `never-nude/worldbook2-lab` | (dev-session checkout) | dev repo, ~only `index.html` |
| **prod** | `never-nude/worldbook2` | `~/Documents/Claude/Projects/worldbook2` | GitHub Pages deploy → worldbook.earth; also has methodology.html, docs/, favicon, .py scripts |

**Prod is currently AHEAD of lab.** Prod's `index.html` has commits never back-ported to lab: **GA4 analytics (`gtag.js`, `G-R62Z7Y55WH`), og:image social-share tags, a replaced methodology page, and a favicon.** Because of the GA4 block in `<head>`, **every line number in this doc (lab-based) is ~+18 in prod** (e.g. the map constructor is lab:513 / prod:531). → **Use the quoted `code strings` in each fix as the real anchor; they are byte-identical in both repos. Ignore the exact line numbers.**

**DEPLOY SAFETY — do NOT `cp lab/index.html → prod/index.html`.** That would clobber prod's GA4 + og:image + methodology and regress analytics. Safe path for the hardening work:
- **Make the edits directly in the prod repo** (`worldbook2`) — it's the current canonical code and where the laptop already is. Anchor by grep string. Commit to a branch, merge to `main` to deploy (Pages Action ~20s).
- **Then re-apply the same edits to `worldbook2-lab`** so the repos don't drift further. (Cleanup for later: back-port prod's GA4/og:image/methodology/favicon into lab so lab is a true superset again.)

## Ground rules (from the repo's memory + CLAUDE.md — do not violate)

- **Border canon is LOCKED.** Do not touch the `world-lines` source or `countries-edge-soft/tight` layers, and don't restyle the Countries view. None of the fixes below go near it.
- **Bump `WB_BUILD`** (index.html:535) on any shipping change.
- **Two-repo deploy — see the divergence warning above.** Because prod is currently ahead of lab (GA4/og:image/methodology), the old "cp lab/index.html → prod" shortcut is unsafe right now. Make hardening edits directly in prod, deploy by merging to prod `main` (Pages Action ~20s), then re-apply to lab. Verify live: `curl -s https://worldbook.earth/index.html?cb=$(date +%s) | grep WB_BUILD`.
- Multiple Claude sessions push to both repos in parallel — **`git fetch` and diff before pushing.**

## How to test (this app has traps — see memory `worldbook-lab-testing-gotchas`)

- Serve: `.claude/launch.json` has a `worldbook` config (`python3 -m http.server 8642`). Use the Claude preview or plain localhost.
- The globe **auto-spins**; `window.paused` is a no-op (it's a closure `let`). Pause via `document.getElementById('tbPlay').click()` before visual checks.
- Boot can take 15–60s; `_atlasBooted === true` is the reliable "layers exist" signal — don't await `map.loaded()` (stays false while spinning).
- Country ownership of a pixel: `map.queryRenderedFeatures(map.project([lng,lat]),{layers:['fills']})[0].properties.iso3` (fills has `promoteId:"iso3"`).
- **Simulate a content blocker** (how the P0s were reproduced): monkeypatch `fetch`/block a `<script>`, or in the preview override `window.fetch` to reject `unpkg.com`/`cdn.jsdelivr.net` URLs. For the map-engine P0, test by loading with `maplibregl` forced undefined or WebGL disabled (`chrome://flags` → disable graphics acceleration, or `about:config` `webgl.disabled=true`).

---

## FIX LIST — apply in this order

### P0-1 — CONFIRMED — Unguarded map engine start = permanent dead "Building your world…" screen
**Four audit agents across three dimensions independently flagged this; verify phase CONFIRMED it as P0.** This is the single most important fix.

- **Root cause:** `index.html:513` `const map = new maplibregl.Map({…})` runs at top level of the one big inline script with no guard. If `maplibregl` is undefined (unpkg.com blocked by a corporate proxy / ad-block list / CDN outage) it throws `ReferenceError`; if the browser can't get a WebGL context (hardware accel off, GPU blocklisted, VM/remote desktop, enterprise policy, `webgl.disabled`) the constructor throws `Failed to initialize WebGL`. Either way the uncaught throw aborts the entire remaining script — all UI wiring, boot, watchdogs — so the `#loading` overlay (index.html:253, "Building your world…") never clears and every control is dead.
- **User impact:** A real cohort at 100k scale (~1% WebGL-less, plus anyone on a network that filters unpkg) sees a solid dark page reading "Building your world…" forever, with zero explanation. Looks permanently broken.
- **Minimal fix (surgical, success path unchanged):**
  1. Immediately **before** line 513 add a CDN guard:
     ```js
     if(typeof maplibregl==="undefined"){
       var _l=document.getElementById("loading");
       if(_l) _l.textContent="Couldn't load the map engine — a content blocker or network issue may be stopping unpkg.com. Please allow it and reload.";
       throw new Error("worldbook: maplibre-gl failed to load");
     }
     ```
  2. Change `const map = new maplibregl.Map({…});` (513–519) to `let map;` + `try{ map = new maplibregl.Map({…}); }catch(err){ var _l=document.getElementById("loading"); if(_l) _l.textContent="Worldbook needs WebGL to draw the globe, and this browser has it disabled or unavailable. Try enabling hardware acceleration or another browser."; console.error("[worldbook] map init failed:",err); throw err; }`
  - Keep the re-throw: the rest of the script references `map` and would break anyway; we're only swapping a silent freeze for a clear message. `const`→`let` is the only structural change.
- **Why safe:** additive guard that executes only when the app is already dead; normal boot untouched. Verify a clean load still works AND that forcing `maplibregl=undefined` / disabling WebGL now shows the message.

### P0-2 → downgraded P1 — "Style is not done loading" boot catch gives up permanently
- **Root cause:** `index.html:1178` — the only `_atlasBoot` catch does `console.error(...) ; loading.style.display="none"` and stops. If `map.addSource` throws the known transient "Style is not done loading", the user gets chrome (topbar/timebar) but no globe, no layers, permanently. Verify agent **downgraded to P1/P2**: in production this path currently only triggers in sandbox/iframe contexts and the 40×250ms retry poller mostly covers it — but it's cheap insurance.
- **Minimal fix:** in the catch at 1178, before hiding the overlay: `if(/not done loading/i.test(String(_e&&_e.message)) && (window._wbBootRetries=(window._wbBootRetries||0)+1)<=5){ _atlasBooted=false; setTimeout(_atlasBoot,500); return; }` then set a human "Something went wrong building the globe — please reload." message instead of a silent dark page.
- **Why safe:** bounded to 5 retries; `_atlasBoot` is idempotent-guarded.

### P1-3 — subnational hover popup never cleaned up → permanent undismissable popup on touch
- **Root cause:** `subPopup` (created index.html:3448, `closeButton:false, closeOnClick:false`) is only removed in `subLeave` (needs a mouse-leave). `cleanupSub` (3491) and `drillOut` don't remove it. On touch, tap a region (browser synthesizes mousemove → popup shows) then tap "Back" → popup floats over the globe forever, no close button, unrecoverable without reload.
- **Minimal fix:** in `cleanupSub` (3491) add `if(subPopup){ try{ subPopup.remove(); }catch(e){} subPopup=null; }`. One line covers all exits (drillOut, re-entrant drillIn, openCountry all call cleanupSub).
- **Why safe:** removing a popup that shouldn't outlive the sub layers; desktop hover recreates it on next drillIn.

### P1-4 — regional drill-down is hover-only → dead on mobile, and the note literally says "Hover"
- **Root cause:** the only `sub-fills` listeners are `mousemove`/`mouseleave` (3449–3450). `subMove` is the sole path that shows region name / religion bars / 2024 vote. Mobile users get colored provinces they can't interrogate, and a panel note telling them to hover a mouse.
- **Minimal fix:** after line 3450 add `map.on("click","sub-fills",subMove);` (MapLibre synthesizes layer-click from tap; subMove already sets popup + hover filter). Optionally make the note pointer-aware using the existing coarse-pointer check in `initHint` (~line 1272): say "Tap regions…" on touch.
- **Why safe:** reuses subMove unchanged; a desktop click during hover just re-shows the same popup. Pairs with P1-3 (the cleanup) so the tap popup is dismissed on Back.

### P1-5 — flow-route tooltips hover-only, and tapping an arc silently destroys country focus
- **Root cause:** flow readouts exist only in `map.on("mousemove","flow-lines",…)` (3169). On mobile, flow layers (trafficking, trade, refugees, debt, minerals…) show anonymous arcs with no names/quantities ever. Worse, the map-level click at ~3180 clears flow focus when a tap lands off a country — tapping an arc over ocean silently un-isolates the web the user just built.
- **Minimal fix:** after 3176 add `map.on("click","flow-lines",e=>{ if(subOn||!flowActive())return; pop.setLngLat(e.lngLat).setHTML(flowTip(flowKey,e.features[0].properties)).addTo(map); });` and widen the clear-focus guard (~3180) to also treat `flow-lines` as a hit: `!map.queryRenderedFeatures(e.point,{layers:["fills","flow-lines"]}).length`.
- **Why safe:** reuses the existing `pop` popup and `flowTip()`; the guard change only makes clearFlowFocus stricter (fewer accidental clears).

### P1-6 — geo-IP lookup can teleport the camera (jumpTo) after the user has already navigated
- **Root cause:** `index.html:1201` `if(!_wbGeoCentered){ map.jumpTo({center:…, zoom:1.6, …}); }`. Gates at 1186 don't check `subOn`/panel/interaction. If the IP lookup resolves late (primary `ipwho.is` black-holed → ~60s to the geojs fallback), a user who has zoomed in or drilled into a region gets snapped to zoom-1.6 over their IP, or dumped onto the dimmed drill-out world.
- **Minimal fix:** gate the late recenter: `if(!_wbGeoCentered && lastInteract===0 && !subOn){ map.jumpTo(…); }` (still drop the ripple marker regardless). Confirm the exact interaction flag name in-file (`lastInteract` is referenced elsewhere in the spin logic).
- **Why safe:** only suppresses an *automatic* recenter once the user has taken control; normal boot (lookup resolves before reveal) is unchanged.

### P1 — no global safety net (cheap, high-value insurance)
- **Root cause:** zero `window.onerror` / `unhandledrejection` handlers anywhere. Any uncaught throw in a click/hover/rAF handler surfaces only in devtools.
- **Minimal fix (optional but recommended):** add near the very top of the main script:
  ```js
  window.addEventListener("unhandledrejection",e=>{ console.warn("[worldbook] unhandled rejection:", e&&e.reason); });
  window.addEventListener("error",e=>{ console.warn("[worldbook] error:", e&&(e.error||e.message)); });
  ```
  Purely diagnostic (don't `preventDefault`); makes production failures observable without changing behavior. Also add a `<noscript>` in `<head>` with a one-line "Worldbook needs JavaScript" message.

## Also worth a look (P1s not fully specced — read the audit JSON for detail)
- **Layer buttons stay live while drilled in** — picking a world layer paints the whole region no-data gray under the drill UI (stale state). Drill round-trip also silently overwrites `activeLayer` with `'religion'`. (race-conditions dimension)
- **Duplicate `iso3:"AUS"`** on three MAPDATA features makes flow tooltips mislabel Australia as "Ashmore and Cartier Is." (data-integrity — needs care, it's data not code).
- **iOS Safari auto-zoom** on the date-popover number inputs (13px font < 16px triggers zoom-on-focus in the `overflow:hidden` fullscreen). Bump those inputs to ≥16px font.
- **A11y one-liners:** icon-only buttons (`?`, pause, sun, crosshair, panel `×`, zoom) need `aria-label`/`title`; `aria-live`/`role=status` on the loading overlay; `prefers-reduced-motion` respected for the geo-ping flash at least.

## The full recovered audit data
`/private/tmp/.../scratchpad/audit_findings.json` holds all 8 dimensions (9 P0-tagged, 22 P1, 17 P2 raw) + 15 verify verdicts — **but that's in a session-local scratchpad and won't travel to another machine.** If you need the raw detail on the laptop, re-run the audit workflow (script at `.../workflows/scripts/worldbook-beta-hardening-audit-wf_58b0ac35-05a.js`) or just work from this doc — the P0 + P1 fixes above are the actionable output and are complete.

## Recommended next-session plan
1. `git fetch`, confirm clean tree, serve locally.
2. Apply **P0-1** first; test both failure modes (WebGL off, maplibregl blocked) + a clean load. This alone removes the worst trust-breaker.
3. Batch the touch/robustness one-liners **P1-3, P1-4, P1-5** (all in the drill/flow area) + the global handlers + `<noscript>`. Test drill-down and a flow layer with DevTools touch emulation.
4. Add **P0-2** and **P1-6** guards.
5. Bump `WB_BUILD` (e.g. `2026-07-05.2 beta-hardening`), commit + merge to prod `main` to deploy, verify live, then re-apply to lab to reconverge.
6. Leave the deeper P1s (layer-buttons-live-while-drilled, AUS dup, a11y) for a follow-up unless time allows.

## Confidence
After P0-1 alone: the catastrophic "site is just broken" class is closed. After the touch fixes: mobile goes from "flagship features are dead" to usable. That's the bar for a credible public beta. The remaining P1s are polish, not trust-breakers.

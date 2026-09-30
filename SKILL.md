---
name: seek-film
description: Build short explainer motion graphics (algorithm walkthroughs, data-flow, report/number reveals) as a single HTML page whose every frame is a pure function seek(t), then render to 60 fps MP4 with sub-frame motion blur via Playwright + ffmpeg, with an on-page layout audit and a single-frame pop scan. Use when asked for a technical explainer video, animated diagram, "motion graphic of how X works", or when HyperFrames/Remotion is not wanted and a zero-framework, deterministic pipeline is.
---

# seek-film

A film is one `film.html` (1920x1080) that exposes three globals:

- `seek(t)` sets **every** element for time `t` (seconds). No timelines, no CSS transitions, no `requestAnimationFrame`, no state carried between calls. Seeking to 40.9 then 3.2 then 40.9 must give identical pixels.
- `audit()` returns a list of layout problems at the current `t` (text under 27.5 px, text clipped by the frame edge, two visible texts overlapping).
- `window.READY = true` once fonts are loaded and the DOM is built.

Everything else (stills, render, per-scene cuts, pop scan) is Node driving that page.

## Start a film

```bash
cp -r ~/.claude/skills/seek-film/template my_film && cd my_film
npm i && npx playwright install chromium
node still.js 0.5 3 5.5        # PNGs in stills/ + audit per time
```

Edit `film.json` (`duration`, `scenes: [[name, start, end], ...]`) and the data/layout/build/scene sections of `film.html`. Leave the engine sections (palette & math, DOM helpers, camera, checks, boot) alone unless you need a new primitive.

## Workflow (this order, every time)

1. **Data + self-check first.** Put the numbers the film shows in a `data` section and make `check()` throw if they stop adding up (table totals, percentages, which items should hit/miss). A film that lies is worse than one that stutters. Put a `values illustrative` tag in the HUD when numbers are not real.
2. **Layout as named rects in world px**, origin at frame centre: `R(x,y,w,h)`, `X(x0,y0,x1,y1)`. Timing as named functions (`TC(i)`, `LT(i)`), never magic numbers inside scene code.
3. **Build every element once** in `build()` before the first seek; scenes only show/hide/move them.
4. **One `gScene(t)` per concern**, called from `seek(t)`. Each is idempotent: compute from `t`, write with `st()`.
5. **Stills at key beats** (`node still.js ...`) until each prints `ok`. Look at the PNGs yourself; send approval stills to the user before a full render — a full 60 s render is ~15-20 min.
6. **Render** `node render.js [start] [end] [out]` — 60 fps, 4 sub-frames per frame (180° shutter) blended by ffmpeg `tmix`, H.264 crf 16 yuv420p.
7. **Post** `node post.js` — per-scene MP4s + last-frame PNG per scene + pop scan. `node post.js scan [file.mp4]` re-scans only.
8. Fix every `POP` line (see below), re-render just that range to check, then full render.

## Engine primitives (all in film.html)

| helper | use |
|---|---|
| `P(t,a,b)` | 0..1 progress of `t` through `[a,b]`, clamped |
| `io`, `oc`, `ic`, `sine` | cubic in-out / out / in, sine in-out |
| `spr(dt,f,z)` | closed-form damped spring step (use for anything that "lands"; `f` Hz, `z` damping ~0.8) |
| `sv(t,[[t0,v0],[t1,v1],...])` / `rsp` | value / rect driven by a sum of springs, one per target change |
| `kAt(t,[t1,t2,...],d)` | step counter 0,1,2... easing over `d` s at each time; drives rolls |
| `rl(a,b,p)`, hop arcs via `r.y -= h*sin(pi*p)` | rect lerp |
| `mix(c1,c2,p)` | hex colour blend |
| `st(el,prop,val)` | cached style write (only touches the DOM on change) |
| `N`, `box`, `pos`, `show`, `wipe` | create div, place rect (+scale), place point, display, clip-path wipe |
| `mkRoll(par,['',a,b,''],'i6 52',z)` + `setRoll(r,k)` + `placeRoll` | **every text appear/swap/exit** is a masked vertical roll through strings; first and last `''` are the empty in/out states |
| `mkDigits(par,['     0','12,480'],'m7 64',z)` + `setDigits(d,t,[t1],dur,stag)` | odometer digits, right-to-left stagger |
| `mkSvg`, `mkPath`, `setA`, `polyPart(pts,p)`, `qpts` | SVG lines drawn on by length |
| `camera(t)` with `CAM=[[t,zoom,x,y],...]` | single world transform, log-space zoom, eased |
| `mulberry32(seed)` | deterministic randomness, precompute once at load |

Fonts: `i4..i7` Inter 400-700 (tabular nums on), `m4`/`m7` JetBrains Mono. Font string format is `'<class> <px>'`. `→ ≥` in strings render as drawn SVG glyphs via `rich()`; add more the same way.

## Rules learned the hard way

- **Minimum 28 px text** at final zoom. The audit flags below 27.5.
- **No text overlaps** at any still. When two labels must share space, roll one out before the other rolls in.
- **Visual language stays consistent**: one palette (`COL`), ink borders 3 px, grey = pending, green = hit/ok, orange = miss/problem, blue = the focus. Reuse the same object across scenes (a card becomes a chunk becomes a bin becomes a table row) instead of cutting.
- **Pops**: the scan flags a frame whose luma diff is >3x both neighbours. Causes seen, and fixes:
  - element shown/hidden at non-zero size: scale/wipe it to 0 first;
  - roll `k` jumping (two `kAt` times too close): space them at least the roll duration apart;
  - integer-rounded camera during slow drift: 1 px steps pop the whole frame, so the camera translate is sub-pixel (`toFixed(2)`) and slow drifts are avoided;
  - slow zoom on text re-hints glyphs (diff ~0.3-0.4): prefer holds at zoom 1, or zooms that finish within ~1 s.
  Diffs under 0.25 are sub-pixel glyph snaps at ease tails and are invisible.
- **Deterministic or nothing**: no `Date`, no `Math.random` at seek time, no async inside `seek`.
- Keep the page self-contained (local woff2 fonts, no CDN) so render machines match.

## Files in template/

`film.html` engine + 6 s demo · `film.json` duration & scenes · `still.js` local server + Playwright `open()` + stills/audit CLI · `render.js` sub-frame render · `post.js` scene cuts + pop scan · `fonts/`.

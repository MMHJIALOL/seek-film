# seek-film

A Claude Code skill for deterministic explainer motion graphics: one HTML page, every frame a pure function `seek(t)`, rendered to 60 fps MP4 with sub-frame motion blur (Playwright + ffmpeg), plus a layout audit and a single-frame pop scan.

## Install

```bash
git clone https://github.com/MMHJIALOL/seek-film ~/.claude/skills/seek-film
```

Then ask Claude Code for "a motion graphic explaining X". See [SKILL.md](SKILL.md) for the workflow and engine reference.

## Try the demo

```bash
cp -r ~/.claude/skills/seek-film/template demo && cd demo
npm i && npx playwright install chromium
node still.js 0.5 3 5.5   # stills + audit
node render.js            # out/film.mp4
node post.js              # per-scene cuts + pop scan
```

Fonts: Inter and JetBrains Mono, SIL Open Font License.

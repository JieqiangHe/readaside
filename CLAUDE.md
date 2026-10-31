# miniread

A single-file PDF reader with side-by-side AI translation (GLM / DeepSeek, your own key). Live at https://jieqianghe.github.io/miniread/ (GitHub Pages serves `main`). Forked from an older Chinese 12-provider app and rewritten all-English, minimal.

## Repo rules

- The whole app is one file, `index.html`. The repo mirrors minisvg's layout: `index.html`, `README.md`, `LICENSE` (GPL-3.0, same as minisvg), `.gitignore`, `CLAUDE.md`. Do not add test files or other folders.
- Plain JavaScript, no framework, no build step. Only external dependency: pdf.js 3.11.174 from cdnjs (viewer + worker); it must keep working as a local/static page.
- Commit to `main` and push to GitHub (remote is SSH: `git@github.com:JieqiangHe/miniread.git`) after each change — standing OK from the user (2026-10-07: everything in the folder goes up, only .DS_Store stays ignored).
- UI text, tooltips, messages and README are English only. Never leave Chinese in the app.
- Code style: dense and short, section comments `// ---------- name ----------`, match the surrounding one-liner style. Don't refactor unasked.
- README is one sentence and the `**Live: ...**` line. Keep it that short.
- UI philosophy: minimal chrome. The user keeps trimming the interface (Help cut to a 5-row shortcut table, prompt textarea hidden, Provider/Model/Think/Key/URL all inside the collapsible "AI settings", language only Simplified Chinese/English, toolbar buttons are plain text). Don't add visible controls or filler text without asking; tuck settings into collapsed `<details>` instead.

## Architecture (names in index.html)

- Helpers: `$(id)`, `V` = `#mView`, `ls(k,v)` = localStorage get/set, `el/btn/msg/nl`.
- AI config: `AI` map (provider → `[default URL, ...models]`); per-provider storage `mrd<Glm|Ds><Key|Model|Ep>`, global `mrdAi`, `mrdThink`, `mrdLang`, `mrdPrompt`, `mrdAuto`, `mrdThumbs`, `mrdTheme`, `mrdGlmRelay`. `aiLoad()` refills Model/Key/URL on provider switch.
- GLM relay (mirrors svguest): BigModel rejects browser CORS from GitHub Pages, so GLM can go through svguest's Cloudflare Worker `RLY` (`/glm/standard`, `/glm/coding`, mapped from the official URLs in `RLYR`). Opt-in via the Relay checkbox in AI settings, shown only when `rlyOk()` (origin `https://jieqianghe.github.io` and provider GLM); fetch with `credentials:'omit'`, `redirect:'error'`; any other URL errors out. The Worker's `ORIGINS` must keep listing that origin.
- `translate(go)`: no argument (button click) acts as Stop when a request is running; a truthy argument (Ctrl/⌘+Enter, auto-selection) aborts the previous request and starts a new one. Ownership token `myAc`/`ok()` keeps a superseded run from clobbering `ac`, the button label or `msg`. A reply cut off at the token limit (`finish_reason:'length'`) continues automatically, up to 3 rounds (partial + "Continue…" appended to the messages).
- `ask(model,msgs,myAc,ui)` is the single request path (svguest's `call()`): same body for both providers (`thinking` + `reasoning_effort`); SSE lines parsed by `on()` (`j.error` → throw, `delta.reasoning_content` → `ui.think`, `delta.content` → `ui.text`, `finish_reason:'length'` → `cut`). GLM falls back to `stream:false` when fetch throws TypeError or the reply is not `text/event-stream` — `glmJ` remembers such URLs, and the JSON reply is read from `choices[0].message`; relay errors get their own messages; `apiMsg()` decodes JSON error bodies with `[code]`.
- Chat UI (mirrors svguest's `#mLog`, same class names): `#mHist` is a bottom-appended flex column (gap 12) with stick-to-bottom (`near()`); each `.item` = source bubble `.u` (right, max-width 88%) + reply `.a` (thinking `<details class=tk>` with `.th` body and ✓ `.ok` when done; waiting row `.wt` = bobbing `MARK` icon + shimmer `.sh` + model `.wm` + seconds `.tm`; error `er`, stopped `dim`) and a meta row `.mt` (model · time · s · chars) with `.rtb` Copy/Remove. Composer card `#mComp` (ring `--btnb`, `--onb` on focus) holds `#mSrc` (borderless) + `#mOps` (Auto, Clear, char count, filled `#mGo`).
- PDF: `open/openPdf/layout/draw/zoom/fit/goto/cur`. `gen` cancels stale layouts; `layP` is the last layout promise (`goto` retries through it when the page div isn't built yet); pages lazy-render via IntersectionObserver (800px margin), thumbnails (96px wide) via `tgen/tobs/tdraw`. The `--scale-factor` CSS var must stay on `.page` (pdf.js text layer needs it). Scale is clamped to 0.1–8; Ctrl-wheel zoom is debounced 60 ms (`wzT`).
- Keys: `?` help, `+`/`-`/`0` zoom, `←`/`→` page step, `↑`/`↓`/`Space` scroll, `Home`/`End` first/last. The handler skips events targeted at inputs/buttons.
- Selection: `mouseup` on the viewer → dehyphenate `-\n` before lowercase → collapse whitespace → `lastSel` dedupe → auto-translate if `#mAuto`. Drag&drop is handled at document level and only for files, so text drops into the textarea still work.
- Actions: `data-a` buttons mapped in the object `A`; theme via `setTheme` + `mrdTheme`, with an inline `<script>` applying it before paint.

## Pitfalls

- `#mRight details label{display:grid}` beats the UA `[hidden]` rule; hidden panel rows rely on `#mRight [hidden],#mRight details label.up[hidden]{display:none}`. Don't remove that.
- The word-count line contains literal `\uXXXX` ranges; those can't be matched through the Edit tool (escapes get decoded). Edit that line via python or split edits around it.
- pdf.js is pinned to 3.11.174: v4 replaced `pdfjsLib.renderTextLayer` with a `TextLayer` class — don't upgrade casually.
- DeepSeek receives both `thinking:{type:'enabled'}` and `reasoning_effort`; fine with the current models, revisit if the API rejects them.
- The chat classes (`u/a/er/tk/th/wt/sh/wm/tm/mt/rtb`), the `MARK` waiting icon and the request fallbacks are copied from svguest's `#mLog` and `call()`; keep them in sync when svguest changes.

## Testing

- No test suite. Syntax-check every inline `<script>` block with node (`~/.local/bin/node`, `new Function(src)` per block). `/usr/bin/python3` and `osascript` are broken Xcode CLT stubs on this machine — don't use them.
- Static checks in the same node run: id uniqueness, every `$()` reference has an element, every `data-a` button has an `A` entry.
- Visual check: `open index.html`, then ⌘R to reload (the user reviews screenshots; `open` alone does not reload an existing tab).

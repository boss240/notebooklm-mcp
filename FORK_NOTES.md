# Fork notes: localised NotebookLM / "Gemini Notebook" support

This fork exists because **the upstream project is archived** (`PleasePrompto/notebooklm-mcp`
was marked unmaintained on 2026-09-10, so GitHub no longer accepts pull requests there),
while the published package no longer matches the NotebookLM UI.

Two things changed on Google's side:

1. **Rebranding** — NotebookLM became *Gemini Notebook*: `notebooklm.google.com/notebook/<id>`
   now returns `301` to `notebook.google.com/notebook/<id>`.
2. **Localisation + markup changes** — the UI is fully localised and Material components
   were renamed, so the shipped selector sets (eight locales, English/older markup as the
   base) stopped matching. On a Ukrainian account three flows failed outright.

## Fixes in this fork (branch `fix/localised-ui-selectors`)

`src/notebooklm/chat.ts`

* `isPlaceholder()` matched loading phrases by `endsWith("...")` (three ASCII dots), but the
  UI emits the ellipsis character **U+2026 (…)**, so a localised loading phrase was accepted
  as the final answer and `ask_question` returned 16–20 characters instead of the full text.
  Both forms are handled now and the length threshold is 60 characters.
* The `[AI-GENERATED …]` banner is stripped before that check — it made short phrases look long.
* Ukrainian and Russian loading phrases added (the list covered EN/DE/FR/ES/PT/IT/NL/JA only).
* `sanitizeAnswer()` drops the leaked "thinking panel" heading (`Thoughts` / `Думки` / `Роздуми` / `Мысли`).

`src/notebooklm/selectors.ts`

| Flow | What broke | Anchor used instead |
|---|---|---|
| Add source button | class `.add-source-button` removed; label localised | `button[aria-label="Додати джерело"]` |
| Add-source dialog detection | `[role="dialog"]` + `.first()` matched a *different, hidden* dialog, so the 8 s visibility wait always timed out | `mat-dialog-container.mdc-dialog--open` |
| Source type options | class renamed `drop-zone-icon-button` → `source-action-button` | `button.source-action-button:has-text("Вебсайти")` / `…"Скопійований текст"` |
| Submit button | labels missing for uk | `…:has-text("Вставити")` / `…:has-text("Додати")` |
| Audio download item | menu item uses the **`save_alt`** glyph, not `download` | `[role="menuitem"]:has(mat-icon:text-is("save_alt"))` |
| Which artifact is downloaded | the "first tile" was taken, so a PNG report was saved instead of the audio | tile with `button.artifact-action-button` / `mat-icon: audio_spark` |

Where possible the anchor is **language-neutral** (a class or a Material icon name) rather
than a per-locale label, so the same class of failure cannot reappear for the next locale.

## Verified against a live notebook (145 sources, UI in Ukrainian)

| Flow | Before | After |
|---|---|---|
| `ask_question` | 16–20 characters («Пошук ключових слів…») | full answer with citations |
| `add_source(url)` | `Could not open the "Add source" dialog` | source count 145 → 146 |
| `download_audio` | menu item not found, then the wrong artifact (PNG) | `.m4a`, 50 MB saved |

Build checks: `npx tsc -p tsconfig.json` clean, `npx eslint src` 0 errors,
`npx prettier --check` clean.

## Reproducing the working build

```bash
git clone https://github.com/boss240/notebooklm-mcp
cd notebooklm-mcp
git checkout fix/localised-ui-selectors
npm ci
npx tsc -p tsconfig.json          # compiles to dist/
node dist/index.js                # stdio MCP server
```

Registering it with a client (e.g. Hermes MCP config) needs the node entry point and,
on Windows, a browser for the persistent profile:

```yaml
mcp_servers:
  notebooklm:
    command: node
    args: ["<repo>/dist/index.js"]
    env:
      PLAYWRIGHT_BROWSERS_PATH: D:/Cache/playwright
      NOTEBOOK_PROFILE_STRATEGY: isolated   # clone the logged-in profile
      NOTEBOOK_CLONE_PROFILE: "true"
```

The Google session lives **inside the browser profile** that `setup_auth` writes; it is never
exported as a cookie string, which is why Device Bound Session Credentials (DBSC) does not
reject it. Cookies can only be copied between runs that use the same browser channel
(`channel: "chrome"`), the same profile directory and the same OS user.

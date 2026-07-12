# Copilot Instructions

## Project shape

- The whole application lives in `index.html`. Template markup, CSS overrides, Vue state, persistence, import/export, audio processing, and video generation are all defined there.
- This is a browser-first tool for AI audiobook production. The README positions it around LLM-driven script analysis plus IndexTTS / Qwen3TTS workflows, with users supplying OpenAI-compatible LLM endpoints and TTS service URLs.
- Frontend dependencies are loaded in the page, not through a package manager: Vue 3 and Tailwind via CDN, `mp4-muxer` via CDN, and ffmpeg helpers from `vendor\ffmpeg` plus ffmpeg core wasm loaded at runtime.

## Commands and validation

- There is no repository build, test, or lint manifest (`package.json`, `requirements.txt`, etc. are absent).
- There is no built-in single-test command because the repository does not ship a test suite.
- Typical local run path is to open `index.html` in a browser or serve the repository as static files.
- For quick syntax validation after editing the inline Vue app, extract the main `<script>` block from `index.html` and run `node --check` on the extracted file.

## High-level architecture

### Single-page workspace

- The top tab bar switches between several work areas (`config`, `timbres`, `sfx`, `script`, `prompt`), but they all share one `Vue.createApp({ setup() { ... } })` state container.
- Most changes require touching both the template section and the values returned from `setup()`. If a new helper is not returned, the template cannot use it.

### State and persistence

- Project persistence is browser-side. IndexedDB stores:
  - `project`: the serialized project state
  - `assets`: binary blobs such as imported audio and generated resources
- `saveProjectToDB()` strips runtime-only fields (for example `audioUrl`, `imageUrl`, `isGenerating`, `abortController`) before saving. Keep that cleanup logic aligned with any new transient fields you add.
- Lightweight UI state such as theme and active tab is stored in `localStorage`, not IndexedDB.

### Script authoring model

- The script workspace is multi-document. `scriptList` holds all script tabs, while `currentScriptId`, `rawScript`, `scriptLines`, `rawAnalysisResult`, and `characters` reflect the active script.
- `scriptLines` is a mixed timeline, not just dialogue. It contains dialogue lines plus non-dialogue blocks such as BGM and background-image events. Changes to ordering, timing, export, or playback must preserve that mixed-timeline model.
- Character data, resource libraries, and prompt templates are all edited in-browser and are expected to survive page reloads through autosave/import-export flows.

### Export pipeline

- Audio export (`exportAudio`) builds a full timeline in the browser, loads referenced dialogue/SFX/BGM assets, and mixes them with `OfflineAudioContext`.
- Subtitle export (`exportSRT`) and video export (`generateVideo`) rely on the same dialogue timing rules as audio export. If you change offsets, trim behavior, speed handling, or break timing, keep these paths consistent.
- Video export is browser-native: it uses WebCodecs plus `mp4-muxer`, while ffmpeg.wasm is used for dialogue/audio processing tasks.

## Repository-specific conventions

- Prefer surgical edits inside `index.html` instead of inventing a new build structure. This codebase is intentionally organized as a single-file browser app.
- When adding UI, wire all three layers together:
  1. template markup
  2. reactive state / actions inside `setup()`
  3. the returned bindings at the bottom of `setup()`
- Treat IndexedDB project JSON as clean persisted state and keep blobs / large binary assets out of that JSON unless the existing import/export flow already packages them explicitly.
- Be careful with timing constants and scheduling logic. Real-time playback, WAV export, SRT export, and MP4 export are expected to stay in sync.
- Theme behavior is driven through `document.body.dataset.theme` and `localStorage` (`unitale_theme`). UI polish changes often require CSS overrides near the top of `index.html`, not just template class changes.

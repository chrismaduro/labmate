# LabMate — Architecture & Design Notes

This file captures key architectural decisions, the reasoning behind them, and
context that lives nowhere else. It lets any new Claude Code session get up to
speed without needing the full conversation history.

---

## Tech stack

| Layer | Choice | Why |
|---|---|---|
| Runtime | Node.js 20+ ESM (`"type":"module"`) | No native-build headaches; great async/stream support |
| LLM providers | Google Gemini + Anthropic Claude | Free-tier Gemini for most users; Claude as fallback |
| UI transport | Local HTTP server + SSE | Electron was tried first but has no headless display support in the dev environment — switch to HTTP+browser was the fix that unblocked all debugging |
| Test framework | `node:test` + `node:assert` | Zero extra dependencies |
| Executable | Node SEA + esbuild + postject | Produces a standalone `dist/LabMate.exe` with no Node install required |

---

## Why not Electron?

Electron was the original choice. It was abandoned because:
- Claude Code has no display server, so `electron .` hangs silently — all fixes were unverified guesswork
- The real bug (a TDZ crash on `MINS_PER_ROUND`) was invisible in Electron but immediately surfaced in the browser via preview tools
- The HTTP + browser approach is fully testable in CI and works identically on any OS

The Electron files (`electron/main.cjs`, `electron/preload.cjs`) are still present but are not the recommended path. Use `node server.js` / `LabMate.cmd` instead.

---

## Transport: HTTP + SSE

`server.js` is the single entry point. It:
- Boots an HTTP server (default port 3000, auto-increments if busy)
- Serves the renderer files from `electron/renderer/`
- Exposes `/api/` routes for all backend actions
- Streams live events to the browser via Server-Sent Events on `/api/events`
- Intercepts `console.log/warn/error` so everything appears in the in-app log

`electron/renderer/web-bridge.js` defines `window.cs` via `fetch`/`EventSource`
when not running inside Electron. The renderer never knows or cares which
transport is underneath.

---

## Agent pipeline (faithful to Gottweis et al. 2025, arXiv:2502.18864v1)

```
Supervisor
  └─ each round:
       1. Generation   → new hypotheses
       2. Proximity    → cluster by semantic similarity
       3. Reflection   → gate active/rejected on 5 criteria
       4. Ranking      → Elo tournament (Tier 1 pairwise + Tier 2 debates)
       5. Evolution    → refine/combine top-ranked hypotheses
       6. Meta-review  → synthesise feedback for next round
```

**Shared state** (`state/state.json`) is the single source of truth. Each agent
owns specific fields and reads others as read-only. State is written to disk
after every agent completes.

---

## Elo tournament details

- Starting rating: 1200 for all hypotheses
- K = 32 for Tier 1 pairwise matches
- K = 64 for Tier 2 structured debates
- **Ranking is always derived from Elo** — the LLM's free-text `ranking` field
  in its JSON output is intentionally ignored. This was a deliberate faithfulness
  fix: letting the LLM override computed Elo ratings defeated the tournament.
- Convergence: Spearman ρ ≥ 0.95 on top-10 AND top-5 churn ≤ 1 across 3
  consecutive rounds → run stops early

---

## Convergence / run completion

A run ends when **any** of these is true:
1. All N rounds complete (hard limit)
2. Timed mode: wall-clock time expires
3. Convergence: rankings stabilise (Spearman ρ ≥ 0.95, top-5 churn ≤ 1 for 3 rounds)

`supervisor.js` exports `computeConvergence`, `isConverged`, `pruneActivePool`,
`runRound`, `runLoop`.

---

## Key files

```
server.js                   Main entry point (HTTP + SSE server)
utils.js                    createClient, callAgent, extractJSON, freshState, saveState/loadState
agents/
  generation.js
  proximity.js
  reflection.js
  ranking.js                also exports expectedScore, applyEloUpdate
  evolution.js
  meta-review.js
  supervisor.js             orchestrates rounds; convergence logic
electron/
  renderer/
    index.html              loads web-bridge.js then app.js
    web-bridge.js           defines window.cs via fetch/SSE (non-Electron path)
    app.js                  all UI logic; IIFE pattern; TDZ-safe initialisation
    style.css               Mono dark theme (#0d0d0d bg, #e4e4e4 accent)
  main.cjs                  Electron main (not the recommended path)
  preload.cjs               Electron preload (CommonJS)
prompts/                    Markdown system prompts for each agent
test/
  run.mjs                   Test runner; isolates I/O under a throwaway COSCI_BASE
  helpers.mjs               makeMockClient, makeHyp, jsonBlock
  elo.test.mjs
  convergence.test.mjs
  utils.test.mjs
  agents.test.mjs
  server.test.mjs
  renderer-smoke.test.mjs   Loads UI scripts in a vm; catches TDZ / load-time crashes
  static.test.mjs           node --check syntax pass on all source files
build-exe.mjs               Node SEA build pipeline → dist/LabMate.exe
LabMate.cmd                 Windows launcher (double-click to run)
```

---

## Environment / config

- API keys in `.env` (gitignored). Set `GOOGLE_API_KEY` or `ANTHROPIC_API_KEY`.
- Provider is auto-detected from whichever key is present.
- Default models: `gemini-2.0-flash` (Google), `claude-opus-4-5` (Anthropic).
- `COSCI_BASE` env var overrides the base path for state/output/prompts (used
  by SEA build and test isolation).
- `COSCI_NO_AUTOSTART=1` prevents server.js from auto-starting (used in tests).

---

## Known quirks

- **TDZ bug history**: `MINS_PER_ROUND` was declared after it was used in
  `app.js`, crashing all event listeners silently. Fixed by moving the `const`
  before first use. `renderer-smoke.test.mjs` now catches this class of bug.
- **dotenv override**: `.env` loading uses `{ override: true }` so stale shell
  env vars don't shadow the file.
- **`import.meta.url` in SEA**: guarded with try/catch in `utils.js` and
  `server.js` because `import.meta.url` is empty in bundled CJS.
- **Ranking agent**: the LLM returns a `ranking` array in its JSON but this is
  intentionally discarded. Elo is the only source of truth for ordering.

---

## GitHub

- Repo: https://github.com/chrismaduro/labmate
- CI: Node 20 & 22, tests + Windows exe smoke test on every push/PR
- Badge in README links to Actions

---

## Paper attribution

Independent, unofficial implementation of:
> Gottweis, J., et al. (2025). *Towards an AI co-scientist.* arXiv:2502.18864v1.

No source code from the original system was used. Not affiliated with Google or
Google DeepMind.

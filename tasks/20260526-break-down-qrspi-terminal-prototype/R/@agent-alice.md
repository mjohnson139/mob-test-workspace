# R: @agent-alice — QRSPI Terminal Visualizer

**Task:** 20260526-break-down-qrspi-terminal-prototype

---

## Q1: What are the distinct UI components in the prototype — enumerate each named region (header, phase chain, panels, event log) and describe its visual structure and the data it displays?

The prototype is divided into five named regions rendered inside `#app`, which uses `display: grid` with `grid-template-rows: auto auto 1fr auto`.

**1. Header (`#header`)** — A single flex row with two child groups. Left side contains the title string "AGENT MOB · QRSPI · SMOKE TEST MONITOR" (`.title.glow2`). Right side is `.status-row`, a flex row of five inline `<span>` elements displaying: repo name (`REPO mjohnson139/mob-test-workspace`), branch name and protection status (`BRANCH main ✓protected`), participant count (`PARTICIPANTS 3`), current phase (`PHASE R-ACTIVE`), and a blinking red `◉ LIVE` indicator (class `.live.glow`, animated with `@keyframes blink` at 1.2s).

**2. Phase Chain (`#phases`)** — A horizontal flex row containing five `.phase-box` elements (one per QRSPI phase) separated by `.ph-arrow` dividers. Each `.phase-box` shows a phase label (`.ph-name`), a subtitle (`.ph-sub`), and a status dot (`.ph-dot`). Color is controlled per-phase via classes: `.phase-q` (amber), `.phase-r` (green), `.phase-s` (cyan, opacity 0.5), `.phase-p` (magenta, opacity 0.4), `.phase-i` (dark red, opacity 0.35). The active phase (`.phase-r`) carries `.phase-active` which sets `opacity: 1 !important` and a box-shadow glow. Completed phases carry `.phase-done`. The arrow between Q and R has class `.active-arrow` and contains a `.flow-char` `<span>` animated with `@keyframes flow-anim` (translateX slide). To the right, separated by a flex spacer and a vertical border, is an inline Q Questions sidebar showing three question labels (`Q1`, `Q2`, `Q3`) in amber text.

**3. Main Content (`#content`)** — A `display: grid` with `grid-template-columns: 1fr 1fr 1fr 1fr`, containing four equal `.panel` components. Each panel has a `.panel-header` (title + badge) and a `.panel-body` (scrollable, `overflow-y: auto`).

- **Participants panel** — Lists three named participants (`mjohnson139`, `agent-alice`, `agent-bob`), each in a bordered sub-block. Fields shown per participant: indicator glyph (`.p-indicator`, `◉` for lead, `◈` for simulated), name, status badge (`.p-active` blinking amber for writing, `.p-done` lime for done). The lead participant block also shows ENV, CMD, and a live typing preview element (`id="p-typing"`) in amber.

- **R Artifacts panel** — Shows a progress bar (`#art-bar` / `#art-pct`) and three `.artifact-item` entries, one per participant. Each item shows a filename (`.art-file` in orange), byte count + answer count, and a status badge (`.art-status`: `.art-done` lime / `.art-writing` blinking amber). A "NEXT STEP" ASCII box at the bottom shows instructional text. The panel header badge (`#art-count`) shows `3/3`.

- **D-Phase Gate panel** — Divided into three visual sections. First: artifact count (`#gate-num`) as a large `28px` number with a secondary progress bar (`#gate-bar`). Second: a `.gate-pumpkin` box displaying the literal string `"pumpkin"` in amber with `@keyframes pumpkin-pulse` scale animation when the `.waiting` class is applied. Third: a bordered "D-PHASE READY / PENDING" status block with descriptive text.

- **Infra + Commands panel** — Two stacked sections. INFRASTRUCTURE is a list of `.infra-item` rows (github-repo, main branch, plugin, codespace, PROJECT.yml) each with a `.infra-name` label and a `.infra-val` or `.infra-alert` badge floated right. COMMANDS is a list of `.cmd-item` rows showing slash command names (`.cmd-name`) and short descriptions (`.cmd-desc`); `/mob fork` is highlighted green/glow and `/mob push` is amber.

**4. Event Log (`#log`)** — A fixed-height (`110px`) `overflow: hidden` region at the bottom. Contains `#log-title` ("SYSTEM EVENT LOG", top-right absolute) and `#log-scroll` (full-height scroll container, `overflow: hidden`). Individual entries are `.log-line` elements, each a flex row with a `.log-time` timestamp (`[HH:MM:SS]`) and a colored message span. Message color classes: `.log-event-q` (amber), `.log-event-r` (green), `.log-event-s` (cyan), `.log-event-ok` (lime), `.log-event-gate` (red), `.log-event-info` (dim green), `.log-event-cmd` (lime). New lines fade in via `@keyframes fade-in` (`opacity: 0 → 1` over 0.3s). The log is capped at `MAX_LOG_LINES = 7`; oldest entries are removed with `.shift().remove()`.

**5. Global visual effects** — `body::before` applies a CRT scanline overlay (`repeating-linear-gradient` at 4px period, `z-index: 9000`). `body::after` applies a vignette (`radial-gradient`, `z-index: 9001`). Font is `Share Tech Mono` loaded from Google Fonts. Helper classes `.glow` and `.glow2` apply `text-shadow` at 6px and 10px/20px radii respectively.

---

## Q2: What JavaScript and browser APIs does the current implementation use — identify all DOM manipulation patterns, CSS animation techniques, timer usage (setTimeout/setInterval), and event-driven patterns present in the HTML?

**DOM Manipulation**

- `document.getElementById` is used throughout to retrieve specific elements by id: `log-scroll`, `p-typing`, `mjohnson-bytes`, `mjohnson-status`, `mjohnson-artifact`, `art-bar`, `art-pct`, `art-count`, `gate-num`, `gate-bar`, `pumpkin-box`.
- `document.createElement('div')` is used in `addLogLine` to create new `.log-line` nodes dynamically.
- `.innerHTML` assignment sets the inner HTML of each new log line (interpolating timestamp and message text).
- `.textContent` assignment updates `.p-typing` text in the typing simulation and byte/progress counters.
- `.className` assignment replaces the full class string on `#mjohnson-status`.
- `.style.width`, `.style.color`, `.style.background`, `.style.boxShadow`, `.style.filter` are set directly as inline style mutations on DOM elements.
- `logScroll.appendChild(el)` appends new log lines.
- `logLines.shift().remove()` removes oldest log entries when `MAX_LOG_LINES` (7) is exceeded.
- `document.querySelectorAll('.phase-box')` returns a NodeList iterated with `.forEach` in `selectPhase`.

**CSS Animation Techniques**

All animations are declared in `<style>` using `@keyframes`:
- `blink` (50% opacity 0) — used on `.live`, `.p-active`, `.art-writing`, and `.ph-dot` in the active phase box.
- `pulse-arrow` (opacity 0.4 → 1 → 0.4) — used on `.ph-arrow.active-arrow`.
- `flow-anim` (translateX -8px → +8px with opacity 0→1→0) — used on `.flow-char` with per-element CSS custom properties `--dur` and `--delay`.
- `fade-in` (opacity 0 → 1, 0.3s forwards) — used on `.log-line`.
- `cursor-blink` (50% opacity 0, 0.9s) — used on `.cursor`.
- `pumpkin-pulse` (scale 1 → 1.08 → 1, 2s) — used on `.gate-pumpkin.waiting`.
- `@keyframes` are applied via `animation` shorthand properties in the CSS rules.

**Timer Usage**

All timers use the browser's `setTimeout` and `setInterval` global functions (no `requestAnimationFrame`, no Web Workers).

- `LOG_EVENTS.forEach(...)` schedules 14 one-shot `setTimeout` calls with delays from 0ms to 14000ms to replay the initial event sequence.
- A single `setInterval` at 3200ms cycles through `CYCLE_EVENTS`, started via a `setTimeout` set to fire after the last `LOG_EVENTS` entry delay plus 2000ms (line ~570).
- The artifact byte-count animation uses a `setInterval` at 280ms, started via a `setTimeout` at 14500ms (line ~577). It calls `clearInterval` when `bytesWritten >= targetBytes`.
- `completeArtifact` contains a nested `setTimeout` at 1200ms (line ~609) for triggering the pumpkin gate log entry.
- The typing simulation (`typeChar`) uses recursive `setTimeout` calls: 45ms + random(30)ms per character, 1200ms pause before advancing to the next phrase, 800ms pause before restarting typing. Started via `setTimeout` at 14200ms (line ~640).
- `selectPhase` uses a `setTimeout` at 600ms to clear a `brightness(1.5)` filter after phase click.

**Event-Driven Patterns**

- `onclick="selectPhase('q')"` etc. are inline HTML event attributes on each `.phase-box` element (lines 285, 291, 297, 303, 309).
- `selectPhase(id)` references the global `event` object (`event.currentTarget`) to read the clicked element — this relies on the implicit global `event` available in inline handlers.
- No `addEventListener` calls are present anywhere in the script.
- No `fetch`, `WebSocket`, `EventSource`, or `XMLHttpRequest` calls are present; all data is hardcoded.

---

## Q3: What real-time data sources would a production version need to poll or subscribe to — identify each piece of live state the UI displays (participant status, artifact presence, phase, log events) and what backend API or git operation would supply it?

The following live state values are currently hardcoded in the prototype's HTML or JS:

**Current phase** — `PHASE R-ACTIVE` in `#header .status-row` and the `.phase-active` class on `.phase-r`. In production this would come from reading `tasks/{task-id}/` directory structure or a `PROJECT.yml` field indicating current phase. A git-based approach: parse the presence/absence of subdirectory `R/` vs `S/` vs `P/` to infer phase. An API approach: a backend endpoint that reads `PROJECT.yml` and returns `{ phase: "R" }`.

**Participant list and per-participant status** — Hardcoded as three named participant blocks (`mjohnson139`, `agent-alice`, `agent-bob`) with static ROLE, ENV, CMD, and status fields. In production: participant roster comes from `PROJECT.yml` (`members` list). Per-participant status (WRITING vs DONE) would require polling git for the presence of each `R/@{participant}.md` file and checking whether it has been committed. Active "typing" state has no git-native equivalent — it would require either a heartbeat API or polling a shared state document.

**Artifact presence and byte count** — `R/@agent-alice.md`, `R/@agent-bob.md`, `R/@mjohnson139.md` with byte sizes (256, 312, 428). In production: `git ls-tree` or GitHub API `GET /repos/{owner}/{repo}/contents/tasks/{task-id}/R/` returns a directory listing with file sizes. The progress bar and `#art-count` badge are derived from `files_present / expected_participants`.

**Artifact commit status** (✓ committed vs WRITING) — Polling git log or GitHub API `GET /repos/{owner}/{repo}/commits?path=tasks/{task-id}/R/@{agent}.md` to determine whether the file has been committed. Alternatively, `git log --oneline -- path/to/file` locally.

**Gate state** (`gate-num`, `gate-bar`, D-PHASE READY/PENDING) — Derived from artifact count vs expected participant count. No separate data source; it is computed from artifact presence data.

**Pumpkin signal** — The `.waiting` state on `#pumpkin-box`. In production: the "pumpkin" signal is a human utterance (chat message or commit message keyword). A production system could poll PR comments, a chat webhook, or a dedicated `signals/pumpkin.txt` file in the repository.

**Event log entries** — All 14 initial `LOG_EVENTS` and 5 `CYCLE_EVENTS` are hardcoded strings. In production: a structured event stream would be needed. Possible sources: git commit log for the task branch (parsed with `git log --pretty=format:...`), a GitHub Actions webhook stream, or a server-sent events (SSE) endpoint backed by a small service watching the repo.

**Infrastructure status** (github-repo visibility, branch lock, plugin installed, codespace running) — Hardcoded as static labels. In production: GitHub API `GET /repos/{owner}/{repo}` for visibility and branch protection; `GET /repos/{owner}/{repo}/contents/.claude-plugin/plugin.json` for plugin presence; GitHub Codespaces API for codespace state.

**Typing preview (`#p-typing`)** — Purely simulated. No production equivalent exists in the current Agent Mob protocol; this field represents in-progress, uncommitted text and has no observable git state.

---

## Q4: What state management model is implied by the prototype — how does the current code represent application state, what would need to change to make that state reactive, and what framework or library approach would fit this use case?

**Current state representation**

There is no explicit application state object. State is represented entirely as DOM node content: text nodes, `className` strings, and inline `style` properties. The only named mutable variables are:

- `logLines` (array, lines 542, 551–554) — tracks references to log DOM nodes for eviction.
- `bytesWritten` (number, line 573) — tracks simulated byte progress; mutated inside `setInterval`.
- `cycleIdx` (number, line 563) — cycles through `CYCLE_EVENTS`; mutated inside `setInterval`.
- `tIdx`, `tCharIdx` (numbers, lines 624–625) — typing simulation cursor; mutated in recursive `setTimeout`.

All other "state" (participant names, statuses, artifact names, phase, infrastructure values) is baked into the initial HTML markup as static strings.

**What would need to change for reactivity**

A reactive version would require extracting state out of the DOM into a data model — an object or set of objects holding phase, participants (with per-participant status and artifact info), gate conditions, and log entries. The UI rendering functions would need to read from that model and update the DOM (or re-render components) when the model changes. Currently there is no separation between data and view: `completeArtifact()` directly mutates multiple DOM nodes by id rather than updating a data structure and letting a render function respond.

**Framework or library fit**

The prototype's structure maps naturally to a component-per-panel model. Any reactive UI library could be used. Given the existing tech stack (TypeScript/React noted in the user profile), React with `useState`/`useEffect` hooks or a lightweight state store (Zustand, Jotai) would match. For a terminal-aesthetic tool with frequent small DOM updates (log lines, counters, blinking states), a virtual-DOM library introduces minimal overhead. If no build toolchain is desired, `Alpine.js` or plain `MutationObserver`-driven reactive bindings could work since all panels are relatively simple. The polling interval pattern already present (`setInterval` at 280ms and 3200ms) maps directly to `useEffect` with `setInterval` cleanup.

---

## Q5: What are the production engineering concerns for deploying this as a live tool — consider hosting, authentication, multi-session support, and any constraints imposed by Claude Code or the Agent Mob plugin architecture?

**Hosting**

The prototype is a single static HTML file with no server-side logic. A production version that polls real git state requires either: (a) a backend service with access to the git repo (a Node.js or Python server deployed on Railway, Vercel serverless functions, or GitHub Actions-triggered endpoints), or (b) direct use of the GitHub REST API from the browser with a token. The current file loads `Share Tech Mono` from Google Fonts via `@import url(...)`, which requires network access; a fully offline/local deployment would need to bundle the font.

**Authentication**

The GitHub API requires authentication for private repos and to avoid rate limiting (unauthenticated requests are limited to 60/hour). A production deployment must manage a GitHub personal access token or OAuth app token. Browser-side token storage (localStorage, sessionStorage) exposes the token to JavaScript; a backend proxy is the standard mitigation. If the mob-test-workspace repo is public, read-only polling can be done unauthenticated, but write operations (detecting commits, branch creation) still benefit from auth for rate limits.

**Multi-session support**

The prototype renders a single hardcoded session (`mjohnson139/mob-test-workspace`, task `QRSPI Smoke Test`). A production tool must support: selecting a repo and task from a list, viewing multiple concurrent tasks, and potentially multiple browser clients viewing the same task simultaneously. This requires parameterized URLs (`?repo=...&task=...`) or a routing layer. Concurrent viewers share read-only state derived from git, so no write-coordination is needed between sessions — the git repo is the single source of truth.

**Polling frequency and rate limits**

The prototype's `setInterval` at 280ms (artifact bytes) and 3200ms (log cycling) are purely cosmetic. A production polling interval against the GitHub API must stay within rate limits: 5000 authenticated requests/hour. Polling every 5–10 seconds for all data sources on a single task is feasible, but watching many tasks simultaneously would require longer intervals or webhook-based push delivery (GitHub webhooks + a receiver endpoint).

**Agent Mob plugin architecture constraints**

The plugin is installed via `claude plugin install` and operates inside Claude Code's process. The current prototype is a standalone HTML file in `docs/brainstorms/` — it is not integrated into any plugin command or skill. A production deployment would need a defined launch mechanism: either a `/mob dashboard` command that opens the HTML (using `open` or a local HTTP server), or a web-based deployment decoupled from the CLI entirely. The plugin's commit format (`[mob] action: description`) and directory conventions (`tasks/{id}/R/@{agent}.md`) are the schema that a production scraper would parse — any changes to those conventions would break the live tool's data extraction logic.

**Security**

The prototype renders message strings from `LOG_EVENTS` and `CYCLE_EVENTS` directly via `innerHTML` (line 549): `el.innerHTML = \`...<span class="${cls}">${msg}</span>\``. In production, if `msg` or `cls` values are derived from git commit messages or file contents written by participants, this is an XSS vector. Sanitization (e.g., `textContent` assignment or a sanitizer library) would be required before rendering externally-sourced strings.

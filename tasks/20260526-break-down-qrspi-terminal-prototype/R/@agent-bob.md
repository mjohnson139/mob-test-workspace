# R: @agent-bob — QRSPI Terminal Visualizer

**Task:** 20260526-break-down-qrspi-terminal-prototype

---

## Q1: What are the distinct UI components in the prototype — enumerate each named region (header, phase chain, panels, event log) and describe its visual structure and the data it displays?

The prototype is a single-page layout defined by `#app`, which uses a CSS grid with `grid-template-rows: auto auto 1fr auto` — four rows: header, phase chain, main content panels, event log.

**Header (`#header`):** A flex row spanning full width. Left side holds the `.title` string "AGENT MOB · QRSPI · SMOKE TEST MONITOR". Right side holds `.status-row`, a flex row of `<span>` elements displaying: REPO name (`mjohnson139/mob-test-workspace`), BRANCH with a protected indicator, PARTICIPANTS count (`3`), current PHASE (`R-ACTIVE`), and a `.live` indicator (◉ LIVE) with a CSS blink animation.

**Phase Chain (`#phases`):** A horizontal flex row of five `.phase-box` elements — Q, R, S, P, I — separated by `.ph-arrow` elements. Each phase box has three child elements: `.ph-dot` (completion/active indicator), `.ph-name`, and `.ph-sub` (subtitle). Phase colors are set by classes `phase-q` (amber), `phase-r` (green), `phase-s` (cyan/dim), `phase-p` (magenta/dim), `phase-i` (red/very dim). Active phase uses `.phase-active` (full opacity, glow box-shadow); completed phase uses `.phase-done` (0.8 opacity). The rightmost section of `#phases` displays a static "Q QUESTIONS ACTIVE" sidebar showing Q1, Q2, Q3 question labels.

**Main Content (`#content`):** A four-column CSS grid (`grid-template-columns: 1fr 1fr 1fr 1fr`) containing four `.panel` elements, each with a `.panel-header` and `.panel-body`.

- **Participants panel:** Displays three participant cards. Each card contains a `.participant-row` with `.p-indicator`, `.p-name`, `.p-status`, and an expandable detail block showing ROLE, ENV, CMD, and a `#p-typing` element for the active participant. Status badges use classes `.p-done` (lime), `.p-active` (amber, blinking), `.p-wait` (dim green).

- **R Artifacts panel:** Displays a `.progress-bar`/`.progress-fill` bar with `#art-bar` and `#art-pct` text, followed by three `.artifact-item` rows. Each artifact row shows `.art-file` (filename in orange), `.art-status` badge (`#mjohnson-status`), and a byte count (`#mjohnson-bytes`). Artifact statuses use `.art-done` (lime), `.art-wait` (dim), `.art-writing` (amber, blinking). A "NEXT STEP" text box below the artifacts shows ASCII-art border text.

- **D-Phase Gate panel:** Contains two `.gate-section` blocks. First shows `ARTIFACTS PRESENT` with `#gate-num` as a large `.gate-count` number and a `#gate-bar` progress bar. Second shows `PUMPKIN SIGNAL` with `#pumpkin-box` (a `.gate-pumpkin` div with `.waiting` state) and explanatory text. Below is a pending status block showing "D-PHASE READY — PENDING."

- **Infra + Commands panel:** Two subsections. INFRASTRUCTURE renders five `.infra-item` rows with `.infra-name` (left) and `.infra-val`/`.infra-alert` (right-floated). COMMANDS renders five `.cmd-item` rows with `.cmd-name` and `.cmd-desc`.

**Event Log (`#log`):** A fixed-height (`110px`) strip at the bottom. `#log-title` is absolutely positioned top-right. `#log-scroll` contains `.log-line` elements, each composed of a `.log-time` timestamp span and a color-coded event span. Log line classes: `log-event-q` (amber), `log-event-r` (green), `log-event-s` (cyan), `log-event-ok` (lime), `log-event-gate` (red), `log-event-info` (dim green), `log-event-cmd` (lime). Maximum 7 lines displayed (`MAX_LOG_LINES = 7`); oldest lines are removed when the buffer overflows.

**Visual overlay effects:** Two CSS pseudo-elements on `body` — `body::before` applies a CRT scanline effect via `repeating-linear-gradient`; `body::after` applies a vignette via `radial-gradient`. Both are `pointer-events: none` and use high z-index (9000, 9001). The font is 'Share Tech Mono' from Google Fonts with 'Courier New' fallback.

---

## Q2: What JavaScript and browser APIs does the current implementation use — identify all DOM manipulation patterns, CSS animation techniques, timer usage (setTimeout/setInterval), and event-driven patterns present in the HTML?

**DOM manipulation patterns:**

- `document.getElementById()` is used throughout to target specific elements by ID: `log-scroll`, `p-typing`, `mjohnson-bytes`, `art-bar`, `art-pct`, `art-count`, `mjohnson-status`, `mjohnson-artifact`, `gate-num`, `gate-bar`, `pumpkin-box`.
- `document.createElement('div')` is used in `addLogLine()` to create new `.log-line` elements programmatically.
- `el.innerHTML` is used to inject HTML strings (timestamp + event spans) into new log line elements.
- `document.querySelectorAll('.phase-box')` is used in `selectPhase()` to iterate all phase boxes and reset their `style.filter`.
- Direct property mutation: `element.textContent`, `element.className`, `element.style.width`, `element.style.color`, `element.style.background`, `element.style.boxShadow`, `element.style.filter`.
- `logScroll.appendChild(el)` appends new log lines; `logLines.shift().remove()` removes oldest lines when `logLines.length > MAX_LOG_LINES`.
- `element.classList.add('waiting')` is called on `#pumpkin-box` inside `completeArtifact()`.

**CSS animation techniques (declarative, no JS Web Animations API):**

- `@keyframes blink` — `50% { opacity: 0 }` — used on `.live` (1.2s), `.p-active` (0.8s), `.art-writing` (0.6s), and the R-phase active dot (0.7s).
- `@keyframes pulse-arrow` — opacity 0.4 → 1 → 0.4 at 1.5s — applied to `.ph-arrow.active-arrow`.
- `@keyframes flow-anim` — combined opacity fade and `translateX(-8px → 8px)` — applied to `.flow-char` elements with CSS custom properties `--dur` and `--delay` controlling per-element timing.
- `@keyframes pumpkin-pulse` — `scale(1) → scale(1.08) → scale(1)` at 2s — applied to `.gate-pumpkin.waiting`.
- `@keyframes fade-in` — `opacity: 0 → 1` at 0.3s — applied to `.log-line` elements when appended.
- `@keyframes cursor-blink` — `50% { opacity: 0 }` at 0.9s — applied to `.cursor` class (defined but the cursor element itself is not present in the HTML body at load; it is referenced as an inline `▌` character in text content).
- CSS `transition: width 1s ease` on `.progress-fill` for smooth bar updates when `style.width` is changed via JS.
- CSS `transition: all 0.2s` on `.phase-box` for hover state.

**Timer usage:**

- `LOG_EVENTS.forEach(({ delay, cls, msg }) => setTimeout(...))` — 14 individual `setTimeout` calls schedule the initial log event sequence, with delays ranging from 0ms to 14000ms.
- `setTimeout(() => { setInterval(..., 3200); }, LOG_EVENTS[...].delay + 2000)` — a `setTimeout` fires ~16000ms after load, then launches a `setInterval` that cycles through `CYCLE_EVENTS` every 3200ms indefinitely.
- `setTimeout(() => { const interval = setInterval(..., 280); }, 14500)` — artifact byte counter animation starts at 14500ms, ticks every 280ms, and calls `clearInterval(interval)` when `bytesWritten >= targetBytes`.
- Inside `completeArtifact()`: `setTimeout(() => { addLogLine(...); ... }, 1200)` — a 1200ms delay before the pumpkin gate log line fires.
- `setTimeout(typeChar, 45 + Math.random() * 30)` — recursive `setTimeout` chain drives character-by-character typing simulation with jitter. Pauses: `setTimeout(..., 1200)` between phrases, `setTimeout(typeChar, 800)` before next phrase starts.
- `setTimeout(typeChar, 14200)` — typing animation starts at 14200ms.

**Event-driven patterns:**

- `onclick="selectPhase('q')"` (and 'r', 's', 'p', 'i') inline HTML event handlers on each `.phase-box`.
- `selectPhase(id)` references the implicit `event` global (the browser's `window.event`) to access `event.currentTarget` — this is not passed as a parameter, relying on the legacy global event object.
- No `addEventListener` calls are present; all event binding is via inline HTML `onclick` attributes.
- No `fetch`, `WebSocket`, `EventSource`, or `XMLHttpRequest` calls are present — all data is hardcoded or simulated.

---

## Q3: What real-time data sources would a production version need to poll or subscribe to — identify each piece of live state the UI displays (participant status, artifact presence, phase, log events) and what backend API or git operation would supply it?

The prototype hardcodes all state. The following pieces of live state are displayed and would require external data sources in production:

**Current QRSPI phase:** Displayed in `#header` as "R-ACTIVE" and reflected in `.phase-active`/`.phase-done` CSS classes on phase boxes. A production source would need to read `tasks/<task-id>/phase` from the task manifest (e.g., `PROJECT.yml` or a task-specific YAML file), obtained via a git read operation or a server-side file watcher on the workspace repository.

**Participant list and per-participant status:** The prototype shows three hardcoded participants (mjohnson139, agent-alice, agent-bob) with statuses WRITING, DONE, DONE. In production, participant membership comes from `PROJECT.yml` (the `participants` list), and per-participant status must be derived by checking whether that participant's R artifact file exists and has been committed — i.e., `git ls-files tasks/<task-id>/R/@<participant>.md` or equivalent file-existence check.

**Artifact presence and byte count:** The "R ARTIFACTS" panel lists `R/@agent-alice.md`, `R/@agent-bob.md`, `R/@mjohnson139.md` with byte counts and commit status. Production state comes from: (1) `git ls-tree` or `git log` on the task's R directory to enumerate committed files; (2) `git show` or filesystem `stat()` for byte counts; (3) `git status` to distinguish "staged/committed" vs "in progress."

**Artifact progress percentage:** `#art-bar` and `#art-pct` reflect how many of N expected artifacts are committed. This is a derived count: (committed artifact files in R/) / (total participants) × 100.

**Gate state (D-phase readiness):** `#gate-num` and `#gate-bar` show 3/3 artifacts present. This is the same artifact-count check plus a secondary condition: all participants have pushed (i.e., all artifact files are committed on the branch). No separate gate file is shown in the prototype.

**Pumpkin signal:** `#pumpkin-box` shows a "waiting" state for the human-typed `"pumpkin"` synchronization word. In a real system this would come from monitoring chat/commit messages or a dedicated signal file (e.g., `tasks/<task-id>/signals/pumpkin`) that a participant creates when ready to advance.

**Event log stream:** The `#log` strip replays simulated events. A production log would consume a server-side event stream derived from: git commit webhooks (GitHub/GitLab webhook events), file-system watchers (inotify/FSEvents on the workspace directory), or a lightweight event-bus writing to a log file that the UI polls. Each log entry maps to a specific git or filesystem event: branch creation, file commit, push, phase transition.

**Infrastructure status:** The INFRA panel shows static labels (public, locked, installed, running, pre-listed). Production values would come from GitHub API (repo visibility, branch protection rules), Claude Code plugin manifest presence check, and Codespace/container status API.

**Typing simulation (`#p-typing`):** This purely simulated element has no real data analog in the prototype. A production equivalent would require access to real-time agent activity — e.g., a Claude Code hook that emits "agent is writing" events, or polling for uncommitted file modifications.

---

## Q4: What state management model is implied by the prototype — how does the current code represent application state, what would need to change to make that state reactive, and what framework or library approach would fit this use case?

**Current state representation:**

The prototype has no centralized state object. Application state is entirely implicit and spread across:
1. The DOM itself — element text content, CSS classes, and inline styles are the only record of state (e.g., `#gate-num.textContent`, `#art-bar.style.width`, `#mjohnson-status.className`).
2. Local JS variables — `bytesWritten` (number), `logLines` (array of DOM element references), `cycleIdx` (number), `tIdx` / `tCharIdx` (typing animation cursors).
3. The `LOG_EVENTS` and `CYCLE_EVENTS` arrays — these are static data tables baked into the script, not mutable state.

There is no state object, no store, no model layer, and no data-binding layer. All updates are imperative direct DOM mutations triggered by timer callbacks.

**What would need to change for reactivity:**

A production version needs a single source-of-truth state object (e.g., a plain JS object or a reactive store) containing: `currentPhase` (string), `participants` (array of `{id, status, artifactPath, bytesCommitted}`), `artifacts` (array of `{path, status, bytes}`), `gateState` (`{artifactsPresent, pumpkinSignaled}`), `logEvents` (array of `{timestamp, cls, msg}`), and `infraStatus` (object of key-value pairs).

Each field in this state object needs to be updated when the backend source changes (poll result or push event), and UI elements need to re-render when the corresponding field changes. Currently every UI update is a one-off imperative call scattered across `completeArtifact()`, the `setInterval` callback, and `typeChar()`.

**Framework / library fit:**

The prototype's structure maps straightforwardly to a reactive component model:
- Vanilla JS with a polling loop and `requestAnimationFrame` or `setInterval` re-render cycle is the minimal path — the prototype already uses this pattern implicitly.
- A lightweight reactive library (Preact, Solid.js, or Vue 3 Composition API) would allow the four panels and event log to be declared as components bound to a shared reactive state object, with automatic DOM updates when state changes.
- React with `useReducer` or Zustand would also fit but adds more overhead than the UI's complexity requires.
- No server-side rendering is implied — the prototype is a pure client-side SPA with no initial data fetch.

---

## Q5: What are the production engineering concerns for deploying this as a live tool — consider hosting, authentication, multi-session support, and any constraints imposed by Claude Code or the Agent Mob plugin architecture?

**Hosting:**

The prototype is a single static HTML file with no server component. A production version that fetches live git/workspace state requires a backend process with access to the workspace repository. Options observed in the prototype's own content: the INFRA panel lists "codespace: running", implying the intended runtime is a GitHub Codespace or similar container. A minimal backend could be a small HTTP server (Node.js/Python) running inside the Codespace or on the same host as the Claude Code process, serving a REST or SSE endpoint that reads workspace files and git state. Static file hosting (Vercel, Netlify CDN) alone is insufficient because live git reads require server-side execution.

**Authentication:**

The prototype shows `REPO: mjohnson139/mob-test-workspace` as a public repo with `✓protected` on main branch. The INFRA panel shows `github-repo: public`. For a public repo, unauthenticated read access to the GitHub API (rate-limited at 60 req/hr) suffices for artifact presence checks. If the workspace repo is private, authentication via GitHub token (stored as a Codespace secret or environment variable) is required for API calls. The INFRA panel shows no auth state, suggesting the current design assumes public repo access. Multi-user browser access to the monitoring UI has no auth layer in the prototype.

**Multi-session support:**

The prototype is a single-viewer static page. In production, multiple concurrent browser sessions viewing the same workspace must all see consistent state. Because all state is derived from git (commits, file presence) rather than in-memory server state, multiple viewers polling the same git endpoint are inherently consistent — git is the source of truth. No session affinity or shared WebSocket server state is required if the backend is stateless. However, the pumpkin signal (a chat/commit word) requires a consistent detection mechanism across all sessions.

**Polling vs. push:**

The prototype simulates events via `setInterval`. A production tool must choose between: (a) client-side polling of a `/status` REST endpoint at a fixed interval (simplest, works with any HTTP server, introduces latency equal to poll interval); (b) Server-Sent Events (SSE) via `EventSource` (one-way push from server, no WebSocket handshake, fits read-only monitoring well); (c) WebSocket (bidirectional, needed only if the browser must send signals back to the server, e.g., triggering `/mob push`). The event log's `fade-in` animation and the log line cap of 7 lines are already designed for a streaming-append model compatible with SSE.

**Claude Code / Agent Mob plugin architecture constraints:**

The Agent Mob plugin runs inside the Claude Code CLI process on the user's machine or Codespace. The plugin has no persistent HTTP server — it executes commands and writes files. A monitoring UI therefore cannot be served by the plugin process itself without adding an HTTP server subprocess. The plugin's commit format (`[mob] artifact: ...`) and directory structure (`tasks/<task-id>/R/@<agent>.md`) are the observable state boundaries — any backend serving the UI must read these paths. The plugin's `PROJECT.yml` manifest is the authoritative participant list and phase record; a production backend must parse this file. Because the plugin operates on a branch (`active/<project-name>`), the monitoring UI must track the correct branch, not `main`.

**Dependency on external network resources:**

The prototype loads 'Share Tech Mono' from `fonts.googleapis.com`. A production deployment in an air-gapped Codespace or restricted network must either bundle the font locally or fall back to 'Courier New' (already specified as the fallback stack). No other external dependencies are present in the HTML.

# R/@mjohnson139

**Task:** 20260526-break-down-qrspi-terminal-prototype
**Role:** shared
**Date:** 2026-05-26

## Q1

The prototype contains five distinct named UI regions, laid out via a CSS grid (`#app`: `grid-template-rows: auto auto 1fr auto`).

**Header (`#header`)** — `/home/codespace/.claude/plugins/marketplaces/agent-mob/docs/brainstorms/qrspi-terminal.html:270–281`
A full-width flex bar with two zones: a left title ("AGENT MOB · QRSPI · SMOKE TEST MONITOR") and a right status row. The status row displays five fixed data fields: REPO name, BRANCH + protection status, PARTICIPANTS count, PHASE name, and a blinking "◉ LIVE" badge. All values are hardcoded strings.

**Phase Chain (`#phases`)** — `line:284–324`
A horizontal flex row containing five `.phase-box` elements (Q, R, S, P, I) separated by `.ph-arrow` dividers. Each box shows: a phase letter + name, a sub-label (e.g. "Research"), and a `.ph-dot` status glyph (✓, ●, ○). The active phase (R) is at full opacity with a glow box-shadow; future phases are dimmed with reduced opacity (0.5, 0.4, 0.35). A right-side sidebar within `#phases` lists the active Q questions as static text.

**Main Panels (`#content`)** — `line:327–501`
A 4-column CSS grid containing four equal-width panels:

1. **PARTICIPANTS panel** (`line:329–375`): Shows three participant blocks. Each has a `.participant-row` with an indicator glyph, name, and status badge (WRITING / ✓ DONE). Below that: role, environment, last-run command, and a live "typing" line (`#p-typing`). Participant data is fully hardcoded HTML.

2. **R ARTIFACTS panel** (`line:377–412`): Shows a progress bar (`#art-bar`, `#art-pct`), then three `.artifact-item` entries — one per participant — each with filename, byte count, and commit status. The `mjohnson139` entry (`#mjohnson-artifact`, `#mjohnson-bytes`, `#mjohnson-status`) is animated. Also shows a "NEXT STEP" ASCII-box instruction footer.

3. **D-PHASE GATE panel** (`line:414–447`): Shows an artifact count (`#gate-num`/3) with a progress bar, a pulsing "pumpkin" signal box (`#pumpkin-box`), and a D-PHASE READY status section. This panel represents the gate condition before the next phase.

4. **INFRA + COMMANDS panel** (`line:449–500`): Two sub-sections. INFRASTRUCTURE lists five rows (github-repo, main branch, plugin, codespace, PROJECT.yml) with status values. COMMANDS lists six `/mob` commands with descriptions, with `/mob fork` and `/mob push` highlighted as contextually active.

**Event Log (`#log`)** — `line:504–507`
A fixed-height (110px) strip at the bottom of the grid. Contains `#log-scroll` which holds `.log-line` elements, each with a `[HH:MM:SS]` timestamp and a color-coded event message. Maximum 7 lines are shown at once; the log is pre-seeded with static HTML lines and then driven by JS animation.

---

## Q2

**DOM manipulation patterns** — `line:540–614`

- `document.getElementById` is the exclusive selector pattern used throughout; no `querySelector` on non-event-handler code except `document.querySelectorAll('.phase-box')` at `line:645`.
- `document.createElement('div')` at `line:547` creates log-line elements dynamically.
- `el.innerHTML` at `line:549` injects timestamped log HTML into each new element.
- `logScroll.appendChild(el)` at `line:554` appends new log lines to the DOM.
- `logLines.shift().remove()` at `line:552` enforces the 7-line maximum by removing the oldest element.
- `element.textContent` is used for non-HTML updates: byte count (`line:579`), percentage (`line:581`, `line:585`), artifact count (`line:599`), artifact status label (`line:595`).
- `element.style.*` direct inline style writes are used for progress bar width (`line:583`, `line:599`, `line:603`), color (`line:604`), boxShadow (`line:605`), and brightness filter (`line:646`, `line:647`).
- `element.className` is reassigned directly at `line:596`.
- `element.classList.add` is used at `line:611` to add the `waiting` class to `#pumpkin-box`.

**CSS animation techniques** — `line:79–263`

All animations are pure CSS `@keyframes` with no JS-driven animation frames:
- `blink` (`line:79`): `step-end` opacity toggle, used on the LIVE badge (`line:78`), active participant status (`line:166`), artifact-writing status (`line:179`), and the active phase dot (`line:293`).
- `pulse-arrow` (`line:116`): opacity fade on the active phase arrow.
- `flow-anim` (`line:125`): translateX + opacity cycle on `.flow-char` span elements; duration and delay are set via CSS custom properties (`--dur`, `--delay`) on each element (`line:290`, `line:296`).
- `pumpkin-pulse` (`line:208`): `scale` transform pulse on `.gate-pumpkin.waiting`.
- `fade-in` (`line:239`): single-shot opacity ramp on `.log-line` elements.
- `cursor-blink` (`line:256`): step-end opacity on `.cursor`.

**Timer usage** — `line:558–648`

- `setTimeout` (one-shot) is used in 7 places:
  - `line:558–560`: iterates over `LOG_EVENTS` array, scheduling each entry with its own `setTimeout` using the `delay` field.
  - `line:564`: fires after the last log event's delay + 2000ms to start the cyclic log interval.
  - `line:576`: starts the byte-count animation after 14,500ms.
  - `line:609`: schedules `completeArtifact` follow-up (pumpkin gate update) 1,200ms after artifact completion.
  - `line:632–636`: resets the typing text index and restarts the typeChar loop after a 1,200ms pause + 800ms gap.
  - `line:640`: starts the `typeChar` loop after 14,200ms.
  - `line:647`: resets the phase-box brightness filter after 600ms.
- `setInterval` (repeating) is used in 2 places:
  - `line:565–570`: cycles through `CYCLE_EVENTS` every 3,200ms after the initial sequence ends.
  - `line:577–591`: fires every 280ms to increment `bytesWritten` by a random amount (5–22 bytes) until `targetBytes` (428) is reached, then calls `clearInterval`.

**Event-driven patterns** — `line:643–648`

- `onclick` attributes on each `.phase-box` element call `selectPhase(id)` inline: `line:285`, `line:291`, `line:297`, `line:303`, `line:309`.
- `selectPhase` uses the implicit `event` global (not a passed parameter) to access `event.currentTarget` at `line:645` and `line:647`.
- No `addEventListener` calls are present anywhere in the file; all event binding is via HTML attributes.
- No `fetch`, `WebSocket`, `EventSource`, or any network API calls are present.

---

## Q3

The prototype displays the following live-state data items, each requiring a real backend data source:

**1. Current phase and phase completion status**
Displayed in: the phase chain (Q done, R active, S/P/I pending) and the header PHASE field.
Backend source: read `PROJECT.yml` (field: `phase:`) and the task directory structure to determine which phase is active. A git-based API would read the committed state of the task branch.

**2. Participant list and per-participant status (WRITING / DONE / WAITING)**
Displayed in: the PARTICIPANTS panel, participant rows.
Backend source: `PROJECT.yml` (field: `participants:`) for the roster; presence of `R/@{id}.md` files in the task directory (via `git ls-tree` or filesystem stat) to determine DONE status; active status (WRITING) has no durable signal in the current architecture and would require a polling heartbeat or a presence file written by the active agent.

**3. Artifact presence, byte count, and commit status**
Displayed in: the R ARTIFACTS panel — per-file filename, byte count, "✓ committed" status, and aggregate progress bar.
Backend source: `git log --diff-filter=A -- tasks/{task-id}/R/` to detect new artifact commits; `git show HEAD:{path} | wc -c` or `git cat-file -s {blob}` for byte size; `git status` or `git log` for commit status.

**4. D-phase gate artifact count**
Displayed in: the D-PHASE GATE panel (`#gate-num`, gate progress bar).
Backend source: count of `R/@*.md` files present in the task's R/ directory via `git ls-tree tasks/{task-id}/R/`, compared against the participant count from `PROJECT.yml`.

**5. Pumpkin signal state**
Displayed in: the pumpkin box in the GATE panel.
Backend source: no current mechanism exists in the codebase. A production version would need either a dedicated signal file committed to the branch (e.g. `R/.signals/pumpkin-{id}`) or a GitHub PR comment/label that the tool could poll via the GitHub REST API.

**6. System event log entries**
Displayed in: the event log strip at the bottom.
Backend source: `git log --pretty=format:"%ai %s" tasks/{task-id}/` for commit-based events; a structured event log file (e.g. `tasks/{task-id}/events.jsonl`) written by mob hooks (`post-commit`, `post-push`) for fine-grained events; or GitHub webhook events forwarded to the tool.

**7. Infrastructure status (repo visibility, branch protection, plugin installed, codespace running)**
Displayed in: the INFRA + COMMANDS panel.
Backend sources: GitHub REST API (`GET /repos/{owner}/{repo}`) for visibility and branch protection; filesystem check for plugin presence (`~/.claude/plugins/marketplaces/agent-mob/`); GitHub Codespaces API or environment variable detection for codespace state.

**8. Active Q questions**
Displayed in: the right sidebar of the phase chain.
Backend source: read `tasks/{task-id}/Q/questions.md` and parse question lines.

**9. Typing / in-progress text for the active participant**
Displayed in: the `#p-typing` element inside the lead participant block.
Backend source: no durable source. A production version would require a heartbeat file written by the active agent at regular intervals (e.g. `tasks/{task-id}/R/.heartbeat-{id}.txt`), polled by the UI.

---

## Q4

**How the prototype currently represents state** — `line:511–648`

Application state exists in three forms:

1. **Static HTML** (lines 267–507): All initial display values (participant names, phase positions, artifact filenames, infra rows, command list, initial log lines) are hardcoded as HTML strings. There is no data object backing them.

2. **Script-local variables** (`line:540–640`): Three mutable variables hold runtime state: `logLines` (array of DOM elements, `line:542`), `bytesWritten` (integer counter, `line:573`), and `cycleIdx` (integer counter, `line:563`). Typing state uses two more: `tIdx` and `tCharIdx` (`line:624`). These are plain `let`/`const` declarations in script scope — no encapsulation, no pub/sub, no reactivity.

3. **Direct DOM as state store**: The current "truth" of the UI is the DOM itself. For example, the byte count is stored as `textContent` of `#mjohnson-bytes`, not in a JS object. `completeArtifact()` at `line:593` reads no state object — it simply calls a series of direct DOM mutations.

**What would need to change for reactivity**

A reactive model would require:
- A single source-of-truth state object (e.g. `{ phase, participants: [{id, status, bytes}], artifacts: [...], gateCount, logEvents: [...] }`) that the UI renders from.
- A mechanism to re-render or patch the DOM when the state object changes (reactive binding or virtual DOM diffing).
- The polling/subscription layer (see Q3) writing into this state object rather than directly into the DOM.

**Framework or library fit**

The prototype's needs map to a lightweight reactive approach: the state shape is simple (flat object with a few arrays), updates are infrequent (git poll interval, e.g. every 5–10 seconds), and the rendering is largely list-driven. The file itself uses no build tooling and no npm dependencies — only a Google Fonts import (`line:8`). A production version consistent with this single-file, no-build constraint would fit `Alpine.js` (CDN-loaded, `x-data` directives, no build step) or `Preact` with HTM (also CDN-loadable). A heavier framework (React, Vue CLI, Svelte) would require a build pipeline that the current static-file delivery model does not support.

---

## Q5

**Hosting**

The current file is a self-contained static HTML page (no server-side rendering, no backend process). A production version that polls git state would require a thin server process to execute `git` commands and serve JSON to the browser — a static file host alone is insufficient. The comment at `line:2` shows the file was opened directly from the local filesystem (`file:///...`), confirming there is no existing hosting infrastructure.

Options observed in the prototype's own INFRA panel (`line:456–470`): GitHub Codespaces ("running") is listed as the runtime environment. A Codespace could serve a local Node/Python process that exposes a `/api/status` endpoint, keeping the tool within the existing development environment without external hosting.

**Authentication**

The prototype hardcodes a specific repo (`mjohnson139/mob-test-workspace`, `line:275`) and participant list. A production version reading live git data via the GitHub API would need a GitHub token. If served from a Codespace, the `GITHUB_TOKEN` environment variable is available automatically. If served externally, OAuth or a pre-issued PAT would be required. The prototype contains no authentication UI or token-handling code.

**Multi-session support**

The prototype renders a single fixed project and task. A production tool would need to handle multiple concurrent mob projects (multiple `active/*` branches), multiple tasks per project, and potentially multiple users viewing simultaneously. The prototype's DOM-mutation model (direct `getElementById` calls against hardcoded IDs) does not generalize to multiple rendered instances on one page. A component-based approach (one component instance per project/task) would be necessary.

**Claude Code plugin architecture constraints**

The file is located at `/home/codespace/.claude/plugins/marketplaces/agent-mob/docs/brainstorms/qrspi-terminal.html` (the path found during research). Claude Code plugins operate as slash-command handlers that produce text output in the terminal — they do not have a native mechanism to open or serve browser UI. Launching this dashboard would require either:
- A plugin command that writes the HTML to a temp file and instructs the user to open it in a browser (current brainstorm model), or
- A plugin command that spawns a local HTTP server process (e.g. `npx serve` or a Python `http.server`) and prints the URL, relying on Codespace port forwarding to make it accessible.

There is no existing `/mob` command for launching the dashboard in the current plugin; the COMMANDS panel in the prototype (`line:477–498`) lists only workflow commands (`/mob init`, `/mob new-project`, `/mob add-member`, `/mob fork`, `/mob push`) with no dashboard command present.

**Git polling constraints**

Any real-time polling of git state from the browser must go through a server intermediary, since browsers cannot execute `git` CLI commands. The polling interval needs to be chosen carefully to avoid GitHub API rate limits (60 unauthenticated / 5,000 authenticated requests per hour). At a 10-second poll interval with 4–5 endpoints, a single user session would consume ~1,800–2,700 requests/hour — within authenticated limits but exceeding unauthenticated limits.

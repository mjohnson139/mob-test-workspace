# Design: Break Down QRSPI Terminal Prototype

**Date:** 2026-05-26
**Lead:** @mjohnson139
**Task:** 20260526-break-down-qrspi-terminal-prototype

---

## Current State

The prototype is a single self-contained HTML file located at
`docs/brainstorms/qrspi-terminal.html`. It was opened directly from the
filesystem (`file:///...`) — there is no server, no build step, and no
external dependencies beyond a Google Fonts CDN import for Share Tech Mono.

The file renders a terminal-aesthetic dashboard for monitoring a QRSPI mob
session. It is divided into a four-row CSS grid (`grid-template-rows: auto auto
1fr auto`): a header bar, a phase chain, a four-panel main content area, and a
fixed-height event log strip.

**Layout regions (five total):**

- **Header** — flex row with a left title string and a right status row showing
  REPO, BRANCH, PARTICIPANTS count, PHASE, and a blinking LIVE badge. All
  values are hardcoded strings.
- **Phase Chain** — five `.phase-box` elements (Q, R, S, P, I) separated by
  arrow dividers. Active phase carries `.phase-active` (full opacity, glow).
  Completed phases carry `.phase-done`. A right sidebar displays the active Q
  questions as static text.
- **Main Panels** — a four-column grid containing: (1) PARTICIPANTS, listing
  three hardcoded participant blocks with status badges and a simulated typing
  preview; (2) R ARTIFACTS, showing a progress bar and per-participant artifact
  rows with filename, byte count, and commit status; (3) D-PHASE GATE, showing
  artifact count vs. expected with a progress bar and a gate-ready status block;
  (4) INFRA + COMMANDS, listing five infrastructure status rows and six `/mob`
  command descriptions.
- **Event Log** — a 110px fixed-height strip with a 7-line rolling buffer.
  Entries have a `[HH:MM:SS]` timestamp and a color-coded event span. New lines
  fade in via `@keyframes fade-in`. Oldest entries are removed with
  `logLines.shift().remove()`.
- **Global visual effects** — CRT scanline overlay (`body::before`) and
  vignette (`body::after`) using CSS pseudo-elements at z-index 9000/9001.
  Share Tech Mono with Courier New fallback.

**JavaScript implementation (all in a single `<script>` block):**

All DOM manipulation uses `document.getElementById` and
`document.querySelectorAll`. There is no centralized state object — state lives
entirely in the DOM (text content, class strings, inline styles) plus four
script-scoped mutable variables: `logLines`, `bytesWritten`, `cycleIdx`,
`tIdx`/`tCharIdx`. All updates are imperative direct DOM mutations.

The animation sequencer uses 14 one-shot `setTimeout` calls to replay an event
sequence (0–14 000ms), followed by a `setInterval` at 3 200ms for cyclic log
events, a separate `setInterval` at 280ms for the byte-count animation, and a
recursive `setTimeout` chain for the typing simulation. No `requestAnimationFrame`,
no Web Animations API, no `addEventListener` calls — all event binding is via
inline `onclick` HTML attributes. No network calls of any kind are present.

**What all three researchers agree on:**

- Every piece of live state displayed (phase, participants, artifact presence,
  byte counts, infrastructure status, Q questions) is currently hardcoded.
- Moving to a live tool requires a server-side intermediary: browsers cannot
  execute `git` commands, and the GitHub API's unauthenticated rate limit
  (60 req/hr) is too low for a polling tool.
- The `el.innerHTML` pattern used in `addLogLine` is an XSS vector if event
  strings are ever sourced from external data (git commit messages, file
  content written by participants).
- No `/mob` command currently exists to launch the dashboard.

---

## Goal

Evolve the QRSPI terminal visualizer from a static HTML demo into a live,
data-driven monitoring tool. The production version will poll real workspace
state (git artifacts, PROJECT.yml, GitHub API) and render it in the existing
terminal aesthetic. It will be built as a React + TypeScript application with a
thin Node.js backend, served from inside a GitHub Codespace via port forwarding,
with no external hosting required.

---

## Decisions Made

- **Framework choice: should the design remain no-build or accept a build
  pipeline?** React + Zustand with a TypeScript build step. Aligns with the
  lead's existing stack. The static single-file model is not carried forward
  into the production design.

- **Hosting target: Codespace-local server or externally deployed service?**
  Codespace-local HTTP server as the primary and only deployment target. The
  backend process runs inside the same Codespace as the Claude Code session;
  Codespace port forwarding exposes the UI to the browser. No Railway, Vercel,
  or other external hosting is in scope for this design.

- **Pumpkin signal:** Excluded from the production design. It was a smoke-test
  artifact, not a real production concept in the visualizer.

---

## Component Breakdown

The existing layout regions map directly to React components. Each panel
becomes a component that reads from a shared Zustand store.

**`<Header>`** — reads `repo`, `branch`, `participantCount`, `phase` from
store. The LIVE badge pulses whenever the last poll timestamp is within the
poll interval threshold.

**`<PhaseChain>`** — reads `currentPhase` and `completedPhases` from store.
Renders five `<PhaseBox>` components. The Q-questions sidebar reads
`activeQuestions` (sourced from `Q/questions.md`).

**`<ParticipantsPanel>`** — reads `participants[]` from store. Each participant
has `id`, `status` (`WRITING | DONE | WAITING`), `role`, and `artifactPath`.
The typing preview element (`#p-typing` in the prototype) is removed from the
production design — it has no durable data source.

**`<ArtifactsPanel>`** — reads `artifacts[]` from store. Each artifact has
`filename`, `bytes`, `status` (`committed | pending`). Derives the progress
bar percentage from `committed / total`. Panel header badge shows `N/N`.

**`<GatePanel>`** — reads `gateState` (`{ artifactsPresent, total,
gateReady }`) from store. Derived entirely from artifact data; no separate
backend query.

**`<InfraPanel>`** — reads `infraStatus` from store. Infra rows (repo
visibility, branch protection, plugin presence, codespace state) are polled
separately from the GitHub API and filesystem checks.

**`<EventLog>`** — reads `logEvents[]` from store. Maintains the 7-line
rolling buffer. XSS risk from the prototype is resolved by using
`textContent` assignment for event message text rather than `innerHTML`.

---

## State Model

The Zustand store shape, derived from the prototype's implicit state and Q3
research:

```
{
  repo: string,
  branch: string,
  phase: "Q" | "R" | "S" | "P" | "I",
  completedPhases: Phase[],
  participants: {
    id: string,
    status: "WRITING" | "DONE" | "WAITING",
    role: string,
    artifactPath: string | null
  }[],
  artifacts: {
    participantId: string,
    filename: string,
    bytes: number,
    status: "committed" | "pending"
  }[],
  gateState: {
    artifactsPresent: number,
    total: number,
    gateReady: boolean
  },
  activeQuestions: string[],
  infraStatus: Record<string, string>,
  logEvents: {
    timestamp: string,
    cls: string,
    msg: string
  }[],
  lastPolledAt: number
}
```

All state transitions are driven by the polling layer writing into the store.
No component mutates state directly.

---

## Data Sources and Polling

Each piece of live state maps to a specific backend read, consistent across all
three R artifacts.

| UI field | Backend source |
|---|---|
| Current phase | `PROJECT.yml` `task:` field + directory presence of `R/`, `S/`, etc. |
| Participant roster | `PROJECT.yml` `participants:` list |
| Participant status (DONE) | `git ls-files tasks/{id}/R/@{participant}.md` — file present = DONE |
| Participant status (WRITING) | No durable signal exists; display as WAITING unless DONE |
| Artifact filename + bytes | `git ls-tree tasks/{id}/R/` — returns blob sizes |
| Artifact commit status | `git log --diff-filter=A -- tasks/{id}/R/@{p}.md` — non-empty = committed |
| Active Q questions | Parse `tasks/{id}/Q/questions.md` question lines |
| Repo visibility | GitHub API `GET /repos/{owner}/{repo}` |
| Branch protection | GitHub API `GET /repos/{owner}/{repo}/branches/{branch}` |
| Plugin presence | Filesystem check: `.claude-plugin/plugin.json` |
| Codespace state | `CODESPACE_NAME` env var present = running inside Codespace |
| Event log | `git log --pretty=format:"%ai %s" tasks/{id}/` filtered to `[mob]` commits |

The backend exposes a single `/api/status` JSON endpoint. The React client
polls it every 10 seconds via `useEffect` + `setInterval`. This is well within
the authenticated GitHub API limit (5 000 req/hr); at 10s intervals with 5
backend sub-requests, a single session uses ~1 800 req/hr.

The `GITHUB_TOKEN` environment variable is available automatically inside a
Codespace and is used by the backend for all GitHub API calls. No token is
passed to the browser.

---

## Cross-Platform Notes

**@agent-alice's context:** The prototype's `selectPhase` handler references
the implicit `window.event` global (not a parameter) at `event.currentTarget`.
This is a legacy IE-era pattern. The React version uses standard `onClick`
props with the event passed as a parameter.

**@agent-bob's context:** The INFRA panel in the prototype lists "codespace:
running" as a static label, confirming the intended runtime environment is a
Codespace. This validates the Codespace-local-server hosting decision. The
prototype also does not include a `/mob dashboard` command — that command will
need to be added to the plugin as part of the S-phase work.

**@mjohnson139's context:** The prototype file was loaded from
`/home/codespace/.claude/plugins/marketplaces/agent-mob/docs/brainstorms/qrspi-terminal.html`.
The production build output should be placed at a path accessible from within
the plugin directory or a sibling `dashboard/` directory, so the `/mob
dashboard` command can locate and serve it consistently.

---

## Open Questions Deferred

- [ ] Should the `/mob dashboard` command be a new slash command in
  `agents/mob-agent.md`, a new standalone agent, or a new skill in `skills/mob/`?
  The plugin architecture supports all three; the choice affects how the server
  process lifecycle is managed.

- [ ] The `WRITING` participant status has no durable git signal. A heartbeat
  file approach (`tasks/{id}/R/.heartbeat-{participant}`) was proposed by
  @mjohnson139 but not adopted or rejected in this design. If live "in-progress"
  status matters for the UX, this needs a protocol decision.

- [ ] The event log in the prototype uses color classes (`log-event-r`,
  `log-event-ok`, etc.) keyed to QRSPI phases. The mapping from git commit
  message prefixes (`[mob] artifact:`, `[mob] advance:`) to those color classes
  needs to be defined as a parsing rule before the backend can be implemented.

- [ ] Font bundling: the prototype loads Share Tech Mono from Google Fonts CDN.
  If the Codespace has no outbound network access (air-gapped or restricted
  environment), the font must be bundled. This is a build-time concern for the
  S phase.

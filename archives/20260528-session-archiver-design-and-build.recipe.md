---
title: "Designing and Building the Session Archiver Skill for Agent Mob"
date: 2026-05-28
tags: [agent-mob, session-archiving, plugin-development, architecture, knowledge-management, claude-code]
tools-used: [Read, Bash, Edit, Write]
stack: [Claude Code plugins, Markdown, YAML, GitHub]
outcome: working-pattern
session-type: architecture
---

# Session Recipe: Designing and Building the Session Archiver Skill for Agent Mob

## Starting Prompt

"An agent that will read a session's turns and describe the arc of the session, paying particular attention to 1) the starting prompt, 2) the tools used, 3) the interaction with the AI, 4) the outcome achieved. And then this session summary can be published in the workspace where others can use it as a basis for what they are doing."

## Arc

The session opened with an idea for a session arc analyzer and immediately required a fit assessment against the existing Agent Mob system. The first major design pivot was storage: a git-backed `recipes/` directory was proposed, then correctly rejected in favor of an external knowledge store. The second pivot involved a specific Notion + open-brain stack, which was then generalized after the user identified open-brain's immutability as a fragility. The final design landed on a storage-agnostic artifact format with context-aware output routing: in a mob workspace the artifact is written to `archives/` for `/mob contribute`; anywhere else it prints to stdout. A late refinement removed the required name argument entirely — the agent derives title, filename, tags, and stack from the session content.

## Key Turns

- **Turn 1:** User described the session arc analyzer idea → assessed fit with existing system; identified the topics track as the closest analog but noted recipes are a different artifact type (reusable pattern vs. collaborative observation)
- **Turn 2:** Proposed git-backed `recipes/` directory inside project branches → user challenged it ("isn't a git repo, I made a bad choice") — pivoted to external knowledge store
- **Turn 3:** Proposed Notion + open-brain architecture → user confirmed MCP access but surfaced open-brain immutability as a deletion/update risk
- **Turn 4:** Resolved immutability: open-brain stores pointer only (title + summary + Notion URL), Notion is source of truth, stale pointers are detectable and acceptable for append-mostly recipes
- **Turn 5:** User asked to make the design generic — cut out Notion and open-brain, produce a portable artifact format and let users compose their own storage/search layer
- **Turn 6:** Format agreed (6-section markdown + YAML frontmatter) → implemented `capture-session` skill, version bump 0.1.3 → 0.1.4, changelog entry U10
- **Turn 7:** User simplified invocation — remove required name argument; agent derives everything from session
- **Turn 8:** User raised output destination concern ("messy to write to project root") → brainstormed 5 options → resolved: mob workspace writes to `archives/`, non-mob prints to stdout; `archives/` added as first-class workspace directory; version bump 0.1.4 → 0.1.5

## Tools Used

Read (×6 — existing agents, SKILL.md format, plugin.json, changelog, templates/AGENTS.md, branch-creation and pr-description references) → Bash (git log, find, mkdir, git branch/commit/push, gh pr create/view, plugin cache copy) → Write (new SKILL.md, test recipe) → Edit (×6 — SKILL.md invocation simplification, plugin.json ×2, SYSTEM_CHANGELOG.md ×2, templates/AGENTS.md ×3)

## Outcome

`/capture-session` skill shipped on branch `feat/capture-session-skill` (PR #16). Skill at `plugins/agent-mob/skills/capture-session/SKILL.md`. Plugin version 0.1.5. `archives/` added as a first-class artifact directory in `templates/AGENTS.md`. Output routes by workspace context: mob → `archives/` file, anywhere else → stdout. Zero-argument invocation — agent derives everything.

## Reuse Guidance

When designing a new Agent Mob skill that should work outside a mob workspace, use `AGENTS.md` presence as a routing signal rather than a hard guard. The pattern: `ls AGENTS.md 2>/dev/null` → present means write to the workspace; absent means fall back to stdout. This keeps the skill universally usable while integrating naturally into mob projects.

When a storage design debate arises in agent tooling: git is correct for concurrent collaborative artifacts (phase files, topic contributions). Git is wrong for searchable knowledge bases. The signal you've crossed the line: if the primary access pattern is "search for something relevant" rather than "commit and pull," it belongs outside git — or in a dedicated directory that doesn't require git knowledge to consume.

When building a skill that extracts structured information from a conversation: the Reuse Guidance section is the most valuable output. Prioritize writing it last, with the most care. A recipe without Reuse Guidance is just a log; with it, it becomes an instruction set for the next agent.

Start implementation by reading `AGENTS.md`, the existing `skills/mob/SKILL.md`, and `plugins/agent-mob/.claude-plugin/plugin.json` to understand conventions. New skills go in `plugins/agent-mob/skills/{name}/SKILL.md`. Every `update-system` commit requires a `docs/SYSTEM_CHANGELOG.md` entry and a `plugin.json` version bump in the same commit.

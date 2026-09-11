---
name: add-issues
description: Use when the user wants to create, file, or publish one or more Linear issues from a plan — says "create a Linear issue for X", "file a ticket for X", "add issues to Linear", "break this into tickets", "write up a plan and put it in Linear", or wants work handed off to a teammate/agent via Linear rather than implemented right now. Requires the Linear MCP server to be connected.
argument-hint: "issue description(s) to draft and publish - name a team/project/assignee/priority inline if it isn't the workspace default"
---

# Create Linear issues with phased plans

Draft one or more Linear issues — each a complete, phased execution plan good enough that a teammate's own agent can pick it up cold and execute it, no further back-and-forth — then publish them straight to Linear. This skill only **creates issues**; it never implements anything itself, and no plan it writes ever includes a commit step — whoever executes it decides that on their own.

## Step 0 — Require the Linear MCP, fall back if it isn't there

This skill needs the Linear MCP server (`mcp__plugin_linear_linear__*` tools). Check for it before drafting anything — call `mcp__plugin_linear_linear__list_teams`.

**If the call fails or those tools aren't available:** stop — do not draft-then-discard. Instead:

1. Tell the user plainly that the Linear MCP server isn't connected and they need to set it up themselves to publish directly.
2. Still do the useful work: draft the full phased plan(s) per Steps 1-3 below, then deliver them without publishing —
   - **One issue** → print the full plan in chat.
   - **More than one issue** → write one Markdown file per issue (title as the H1, phased plan as the body) to the working directory or a path the user names, then list the file paths in chat. Don't bundle multiple issues into one file — each is meant to become its own Linear issue later.
3. Skip Step 4 (field resolution needs live team/project data) and Step 5 (nothing to publish to).

Only proceed past this step once `list_teams` actually returns.

## Step 1 — Gather context and identify the issue set

Work from whatever is already in the conversation — don't ask the user to re-explain something already established. The arguments passed to this skill are the explanation of the issue(s) to create; parse them for the substance (what to build/fix), whether they describe **one issue or several**, plus any explicitly named project, assignee, or priority (see Step 4) — per issue, if they differ.

If the request is one broad piece of work, judge whether it's naturally one issue or should be split into several independent, separately-executable issues (e.g. "add auth, then add billing, then add the admin panel" is three issues, not phases of one). When in doubt and the pieces are independently shippable, prefer separate issues — each Linear issue this skill creates should be a complete, standalone unit of work.

If an argument names a reference — a spec file path, an existing issue ID/URL, a PR — fetch it and read its full body before drafting anything.

Explore the codebase before drafting any plan:

- Find the files the plan will actually touch. Read them well enough to name exact paths, exact function/component names, and current line ranges — the plan must never say "similar to X" or "the usual pattern," it must show the pattern.
- Use the project's own domain vocabulary for naming — if the repo has a glossary or domain-model doc, read it; an issue that calls something by the wrong noun reads as written by someone who doesn't know the codebase.
- Respect any architecture-decision doc relevant to the area you're touching (an ADR directory, a `docs/decisions/` folder, whatever the repo calls it) — don't draft a plan that silently contradicts one.
- Look for a prefactoring opportunity that would make the real change easier ("make the change easy, then make the easy change"). If one exists, it becomes that issue's first phase.

## Step 2 — Draft each phased plan

For each issue identified in Step 1, structure its description as **phases**, each a complete, checkable unit of work — not a layer-by-layer breakdown (schema-only, then API-only, then UI-only), a **vertical** slice through whatever layers this specific change touches. A phase should be small enough to fit in one fresh context window and leave something genuinely verifiable when it's done.

For each phase, write:

- **A short, specific title.**
- **Files** — exact paths, marked `Create:` or `Modify:` (with a line range for `Modify:` where you know it). Never a vague "the relevant component."
- **Steps** — ordered, concrete actions. Where a step is code, show the actual code (real function names, real types, matching what Step 1 found) — never pseudocode, never "add appropriate validation," never "similar to Task N, repeat the pattern" (the executing agent may read phases out of order and won't have Task N in front of it). Number them or use checkboxes so progress is trackable.
- **A verification step** — the concrete command or check that proves this phase actually works (a specific test file/name to run, a typecheck, a lint pass, a manual repro to confirm) — not "test the change," the actual command.

**Never a commit step.** Every phase ends at verification. Whether and how to commit is left to whoever executes the plan — that's the consuming repo's own convention to apply, not this issue's job to re-decide.

**No placeholders, anywhere.** Never write "TBD," "implement later," "add appropriate error handling," "write tests for the above" without the actual test code, or reference a type/function this issue never defines. A step that describes what to do without showing how is not done — fill it in or cut it.

**Completion criteria must be checkable, not vibes.** "Understanding reached" or "properly handled" tells the executing agent nothing about when to stop; "the 3 acceptance criteria below all pass" does. Every phase and the issue as a whole needs a bound the agent can test itself against, not just narrate satisfaction with.

**State the positive, not the prohibition.** Prefer "use `useTransition` for pending state" over "don't use `useState(false)` for pending state" — a banned pattern named in the plan is a pattern now sitting in the executing agent's context, and negation is a weak filter against it. Reserve an explicit "never do X" for a real guardrail (a destructive action, a security boundary), and even then pair it with what to do instead.

## Step 3 — Self-review before publishing

Re-read every drafted plan once, fresh:

- **Coverage** — does every part of what the user asked for map to a phase in some issue? List any gap and fix it before moving on.
- **Placeholder scan** — re-check for the red flags from Step 2. Fix inline, don't just note them.
- **Consistency** — within each issue, do names, types, and file paths agree across phases? A function called `resolveActiveOrg` in one phase and `getActiveOrganization` in another is a bug the executing agent will trip on. Across issues, watch for the same collision if two issues touch overlapping code.
- **No embedded commit step** — confirm one didn't slip into any issue.

## Step 4 — Resolve each issue's fields

For each issue:

- **Team** — call `mcp__plugin_linear_linear__list_teams`. If there's exactly one team in the workspace, use it without asking. If there's more than one and the user didn't name one, ask which.
- **Priority** — `0=None, 1=Urgent, 2=High, 3=Medium, 4=Low`. Default to **3 (Medium)**. Only use a different value if the user explicitly named one (a word like "urgent," "low priority," "high priority" — map it to the matching number).
- **Project** — only set one if the user named it. Resolve the name via `mcp__plugin_linear_linear__list_projects` (search by name); if more than one project plausibly matches, ask which rather than guessing. If the user named nothing, leave the issue **with no project**.
- **Assignee** — only set one if the user named a person. Leave it **unassigned** otherwise. Never guess an assignee from conversation context alone (e.g. "I'm working on this" isn't the same as "assign it to me" — only act on an explicit ask).

## Step 5 — Publish directly, no confirmation

Once Steps 2-4 are done, create every issue immediately with `mcp__plugin_linear_linear__save_issue` (`team`, `title`, `description` = the full phased plan as Markdown, plus `priority`/`project`/`assignee` per Step 4 — omit `project`/`assignee` entirely when neither was set, never pass a guessed value). Do not show the drafted plan for approval first and do not ask "should I publish this?" — publish, then report. These calls are mandatory — drafting the plan and stopping short of actually creating the issue(s) does not satisfy this skill.

Report back, for every issue created, its identifier as a **clickable Markdown link to its URL** (e.g. `[ENG-123](https://linear.app/...)`) — take the URL verbatim from the `save_issue` response, never construct or guess one. That's the whole report, nothing more elaborate than that list; the link is what lets the user open the issue directly instead of hunting for the identifier in the Linear app. Tell the user to check the issue(s) in Linear themselves if they want to review or adjust the content.

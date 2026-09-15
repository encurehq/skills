---
name: add-issues
description: Use when the user wants to create, file, or publish one or more Linear issues from a spec — says "create a Linear issue for X", "file a ticket for X", "add issues to Linear", "spec this out and put it in Linear", "write up a spec and put it in Linear", or wants work handed off to a teammate/agent via Linear rather than implemented right now. Requires the Linear MCP server to be connected.
argument-hint: "issue description(s) to draft and publish - name a team/project/assignee/priority inline if it isn't the workspace default"
---

# Create Linear issues with specs

Draft one or more Linear issues — each a complete spec synthesized from what's already been discussed, good enough that a teammate's own agent can pick it up cold and turn it into a plan with no further back-and-forth — then publish them straight to Linear. This skill only **creates issues**; it never implements anything itself, and no spec it writes ever includes a commit step — whoever plans and executes it decides that on their own.

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

Do NOT interview the user for any of this — synthesize the spec entirely from what's already in the conversation and what you find exploring the codebase below.

Explore the codebase before drafting any spec:

- Find the files the spec's implementation decisions will reference. Read them well enough to name exact modules, exact function/component names, and existing interfaces — the spec must never say "similar to X" or "the usual pattern," it must name the pattern.
- Use the project's own domain vocabulary for naming — if the repo has a glossary or domain-model doc, read it; an issue that calls something by the wrong noun reads as written by someone who doesn't know the codebase.
- Respect any architecture-decision doc relevant to the area you're touching (an ADR directory, a `docs/decisions/` folder, whatever the repo calls it) — don't draft a spec that silently contradicts one.
- Look for a prefactoring opportunity that would make the real change easier ("make the change easy, then make the easy change"). If one exists, name it as an implementation decision.

## Step 2 — Draft each spec

For each issue identified in Step 1, sketch out the seams at which the feature will be tested. Existing seams should be preferred to new ones. Use the highest seam possible; if a new seam is needed, propose it at the highest point you can — the fewer seams across the codebase, the better, ideally one. If a seam choice is genuinely ambiguous or would touch a boundary the user hasn't weighed in on, ask via `AskUserQuestion` before writing the spec around it — otherwise proceed without asking.

Then write the issue's description as a spec, using this template:

<spec-template>

## Problem Statement

The problem being faced, from the user's perspective.

## Solution

The solution to the problem, from the user's perspective.

## User Stories

A LONG, numbered list of user stories, each in the form:

1. As an <actor>, I want a <feature>, so that <benefit>

This list should be extremely extensive and cover all aspects of the feature.

## Implementation Decisions

A list of implementation decisions made, which can include:

- The modules that will be built/modified
- The interfaces of those modules that will be modified
- Technical clarifications
- Architectural decisions
- Schema changes
- API contracts
- Specific interactions

Do NOT include specific file paths or code snippets — they go stale fast. Exception: if a prototype produced a snippet that encodes a decision more precisely than prose can (state machine, reducer, schema, type shape), inline it within the relevant decision and note briefly that it came from a prototype. Trim to the decision-rich parts, not a working demo, just the important bits.

## Testing Decisions

A list of testing decisions, including:

- A description of what makes a good test for this feature (only test external behavior, not implementation details)
- Which modules/seams will be tested
- Prior art for the tests (similar tests already in the codebase)

## Out of Scope

What's explicitly out of scope for this spec.

## Further Notes

Any further notes about the feature.

</spec-template>

**No placeholders, anywhere.** Never write "TBD," "implement later," or a User Story/Implementation Decision that describes what to decide without actually deciding it. A section that gestures at content without stating it is not done — fill it in or cut it.

**Completion criteria must be checkable, not vibes.** The spec should give whoever plans from it enough to define a testable "done," not just narrate a vague goal.

**State the positive, not the prohibition.** Prefer "use `useTransition` for pending state" over "don't use `useState(false)` for pending state" — a banned pattern named in the spec is a pattern now sitting in the executing agent's context, and negation is a weak filter against it. Reserve an explicit "never do X" for a real guardrail (a destructive action, a security boundary), and even then pair it with what to do instead.

**Never a commit step, and never a phased task breakdown.** A spec describes the problem, the decisions, and the seams — it leaves *how* to sequence and implement the work, and whether/how to commit, entirely to whoever plans from it later.

## Step 3 — Self-review before publishing

Re-read every drafted spec once, fresh:

- **Coverage** — does every part of what the user asked for map to a user story or implementation decision in some issue? List any gap and fix it before moving on.
- **Placeholder scan** — re-check for the red flags from Step 2. Fix inline, don't just note them.
- **Consistency** — within each issue, do names, types, and module references agree across sections? A module called `resolveActiveOrg` in one section and `getActiveOrganization` in another is a bug whoever plans from this will trip on. Across issues, watch for the same collision if two issues touch overlapping code.
- **No file paths/code snippets** other than the prototype-snippet exception, and **no embedded commit step or task breakdown** — confirm neither slipped into any issue.

## Step 4 — Resolve each issue's fields

For each issue:

- **Team** — call `mcp__plugin_linear_linear__list_teams`. If there's exactly one team in the workspace, use it without asking. If there's more than one and the user didn't name one, ask which.
- **Priority** — `0=None, 1=Urgent, 2=High, 3=Medium, 4=Low`. Default to **3 (Medium)**. Only use a different value if the user explicitly named one (a word like "urgent," "low priority," "high priority" — map it to the matching number).
- **Project** — only set one if the user named it. Resolve the name via `mcp__plugin_linear_linear__list_projects` (search by name); if more than one project plausibly matches, ask which rather than guessing. If the user named nothing, leave the issue **with no project**.
- **Assignee** — only set one if the user named a person. Leave it **unassigned** otherwise. Never guess an assignee from conversation context alone (e.g. "I'm working on this" isn't the same as "assign it to me" — only act on an explicit ask).

## Step 5 — Publish directly, no confirmation

Once Steps 2-4 are done, create every issue immediately with `mcp__plugin_linear_linear__save_issue` (`team`, `title`, `description` = the full spec as Markdown, plus `priority`/`project`/`assignee` per Step 4 — omit `project`/`assignee` entirely when neither was set, never pass a guessed value). Apply the `ready-for-agent` triage label if the workspace has one — no need for additional triage. Do not show the drafted spec for approval first and do not ask "should I publish this?" — publish, then report. These calls are mandatory — drafting the spec and stopping short of actually creating the issue(s) does not satisfy this skill.

Report back, for every issue created, its identifier as a **clickable Markdown link to its URL** (e.g. `[ENG-123](https://linear.app/...)`) — take the URL verbatim from the `save_issue` response, never construct or guess one. That's the whole report, nothing more elaborate than that list; the link is what lets the user open the issue directly instead of hunting for the identifier in the Linear app. Tell the user to check the issue(s) in Linear themselves if they want to review or adjust the content.

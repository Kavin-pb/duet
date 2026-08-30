# Issue tracker: Linear

Issues, features, and specs for this repo live in Linear, managed through the Linear MCP tools (`mcp__linear__*`).

- Workspace: **chat chat** (https://linear.app/chat-chat)
- Team: **Chat chat** (id `07b3bb85-2760-4e44-bcef-0946dd9a88e7`)

## Conventions

- One Linear issue per ticket; larger efforts use a parent issue with sub-issues (`parentId`)
- Triage state is recorded with Linear issue labels — the five role strings in `triage-labels.md` (create them with `mcp__linear__create_issue_label` on first use)
- Comments and conversation history go on the issue via `mcp__linear__save_comment`

## When a skill says "publish to the issue tracker"

Create the issue with `mcp__linear__save_issue` against the `Chat chat` team.

## When a skill says "fetch the relevant ticket"

Use `mcp__linear__get_issue` with the identifier the user passed (e.g. `CHA-1`), or search with `mcp__linear__list_issues`.

## Working a story

Build stories live in a Linear project under a milestone and carry a `ready-for-agent` or `ready-for-human` triage label. A session asked to "work on CHA-N" (or to pick up the next story) should:

1. **Pick**: if no issue was named, take the lowest-numbered open `ready-for-agent` story in the earliest unfinished milestone of the active project (`mcp__linear__list_issues` filtered by project + label).
2. **Claim**: set the issue to `In Progress` and assign it to the dev driving the session before any work.
3. **Branch**: use the issue's `gitBranchName` from Linear.
4. **Build**: the story body is the contract — `## What`, `## Acceptance criteria`, `## Tests` naming the seam that proves it (per the spec's Testing Decisions). Work test-first; tick the acceptance checkboxes as they pass.
5. **Close**: set the issue to `Done` only when every acceptance criterion is verified and the named tests pass.

## Wayfinding operations

Used by `/wayfinder`. The **map** is a parent issue whose description holds the Notes / Decisions-so-far / Fog body; **child tickets** are sub-issues of it.

- **Child ticket**: sub-issue created with `parentId` set to the map issue, question in the body. A `Type:` line in the description records the ticket type (`research`/`prototype`/`grilling`/`task`); issue state records progress (`In Progress` = claimed, `Done` = resolved).
- **Blocking**: Linear issue relations (`blockedBy`). A ticket is unblocked when every blocker is `Done`.
- **Frontier**: list sub-issues of the map issue that are open, unblocked, and unstarted; lowest issue number wins.
- **Claim**: set state to `In Progress` before any work.
- **Resolve**: append the answer to the issue description under an `## Answer` heading, set state to `Done`, then append a context pointer (gist + link) to the map issue's Decisions-so-far section.

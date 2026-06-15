---
name: update-linear-status
description: >-
  Update the Linear issue status for the ticket the agent is working on via MCP.
  Use when starting work on a Linear task, finishing implementation, or when the
  user asks to update ticket status (e.g. PRJ-83, mark done, move to in progress).
---

# Update Linear Status

Update the **active Linear issue** for the current session using the Linear MCP server (`plugin-linear-linear`). Do not ask for confirmation unless the target issue or target status is ambiguous.

## Identify the active issue

Resolve the issue ID in this order:

1. **Explicit ID** in the user message or plan (e.g. `PRJ-83`, `PRJ-81`)
2. **Conversation context** - the ticket being implemented, planned, or reviewed in this session
3. **Git branch** - e.g. `project/PRJ-83-...` -> `PRJ-83`
4. **Ask the user** only if none of the above apply

Update **one issue**: the leaf task the agent is doing, not the parent epic, unless the user explicitly names the epic.

## Status transitions

| When | Set state to |
| ------ | ---------------- |
| Agent starts implementing a ticket | `In Progress` |
| User asks to update Linear after completed work | `Done` |
| User asks to send for review | `In Review` |
| User cancels scope | `Canceled` |

When the user says "update the linear task" without a status, default to `Done` if the implementation is complete; otherwise use `In Progress`. Skip the update if the issue is already in the target state.

## MCP workflow

**Always read tool schemas first** from the Linear MCP descriptors before calling.

1. Fetch the current issue:

```yaml
CallMcpTool:
  server: plugin-linear-linear
  toolName: get_issue
  arguments: { "id": "PRJ-83" }
```

1. Update only the `state`:

```yaml
CallMcpTool:
  server: plugin-linear-linear
  toolName: save_issue
  arguments: { "id": "PRJ-83", "state": "Done" }
```

Use the state **name** string (e.g. `"Done"`), not the internal ID. Common PRJnella states: `Backlog`, `Todo`, `In Progress`, `In Review`, `Done`, `Canceled`, `Duplicate`. If unsure or a state fails, call `list_issue_statuses` for team `PRJnella`.

1. Verify with `get_issue` again. Confirm `status` / `statusType` matches the intended transition and report the issue URL to the user.

## Rules

- Do not update parent epics (e.g. PRJ-81) when completing a sub-task (e.g. PRJ-83).
- Do not change assignee, priority, labels, or description unless the user asks.
- Do not push git branches or link commits unless the user asks.
- If Linear MCP is unavailable, tell the user which issue and status would have been set.

---
name: check-in
description: "When requested: execute periodic follow-up checks and update tracking dates."
---

# /check-in — Execute Periodic Follow-ups

Run the actual checks for items in the `periodic-followups` memory (scope: global), report
findings, and update `Last checked` dates.

Use memory-mcp's `read` tool to load the `required-environment-variables` memory (scope: global)
if you haven't already this session, and use those identities for all gh operations throughout.

## Argument handling

`$ARGUMENTS` is optional. Behavior depends on what's provided:

| Argument | Behavior |
|----------|----------|
| *(empty)* | Check all items that are **due or overdue** based on frequency and last-checked date |
| Number (e.g. `1`) | Check item with that number **regardless of due date** |
| Text (e.g. `serena`) | Check items whose title contains the text (case-insensitive) **regardless of due date** |
| `all` | Check every active item regardless of due date |

## Phase 1: Identify items to check

1. Use memory-mcp's `read` tool to load the `periodic-followups` memory (scope: global)
2. Parse each active item's `Last checked` date and `Frequency`
3. Calculate due dates:
   - weekly = last checked + 7 days
   - biweekly = last checked + 14 days
   - monthly = last checked + 30 days
4. Apply argument filtering (see table above)
5. If no items match, report "No follow-ups due" and stop

List the items that will be checked before proceeding.

## Phase 2: Execute checks

For each selected item, run the steps described in its **How** field. These are typically
`gh` commands — run them with the environment from the memory.

Collect command output and interpret results:
- Has the status changed since last check?
- Is there new activity (comments, commits, state changes)?
- Is the "Done when" condition now met?

## Phase 3: Report and update

For each checked item:

1. **Report findings** — summarize what changed (or didn't) since last check
2. **Update `Last checked`** — use memory-mcp's `edit` tool on the `periodic-followups`
   memory (scope: global) to update the date for the checked item.
3. **If "Done when" is met** — ask the user before moving the item to the Completed Items
   section. Don't move automatically.

## Output format

```
## Follow-up Check Results

### 1. <Item title>
**Status**: <changed / no change / done>
**Last checked**: <old date> -> <today>
**Findings**: <what the commands revealed>
<if done-condition met: "This item's completion condition appears met. Move to completed?">

### 2. ...
```

After all items are reported, confirm that memory updates were applied.

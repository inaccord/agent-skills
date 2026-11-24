---
name: gong-update-plan-after-call
description: After a Gong call, add the committed next steps to an existing Accord deal room as seller-only steps with owners and due dates, and update seller-only steps whose dates changed. Buyer-visible changes are left for the rep.
---

# Gong: Update the plan after a call

Example prompt (input: `[CRM_ID]`):

> Find the Accord for [CRM_ID]. Add the next steps we committed to on our latest call, with owners and due dates, and update any steps whose dates changed. Keep new steps internal for my review.

## Working with Accord

**Tools.** Accord's MCP server exposes four tools:

| Tool | Use |
|---|---|
| `whoami` | Who the connection is, its role, and which permissions it has |
| `query` | Read-only GraphQL queries |
| `mutate` | GraphQL mutations (only available when write access is on for the workspace) |
| `introspect` | Look up types, fields, and arguments in the schema |

The steps below name the exact queries and mutations to call. If an argument shape is unclear, call `introspect` with the type name (for example `CreateStepInput`) before writing.

**Call content comes from Gong.** Gong's agent reads the call and passes in the facts listed under "Inputs from Gong" below. Accord's MCP does not return call transcripts.

**Finding the deal room.** The rep may give an Accord link, an Accord ID, or a CRM opportunity ID. Don't ask for a CRM ID if you already have one of the others.

- Accord link: the Accord ID is the part right after `/accord/` in the URL, for example `https://<workspace>.inaccord.com/accord/<accordId>/summary/...`.
- Accord ID:
  ```graphql
  query { accord(id: "<accordId>") { id opportunityName crmId isDraft } }
  ```
- CRM opportunity ID (deal rooms are linked to the opportunity by `crmId`):
  ```graphql
  query { accords(filter: { crmId: { equalTo: "<opportunityId>" }, isTemplate: { equalTo: false } }) { id opportunityName isDraft } }
  ```
- None given: use the CRM opportunity linked to the Gong call. Ask the rep only if the call has no linked opportunity.

**Rep review happens in Accord.** Steps the agent adds are seller-only (`internal: true`) until the rep makes them visible to the buyer.

## When to use

After a follow-up call on an opportunity that already has a deal room.

## Inputs from Gong

An Accord link, Accord ID, or CRM opportunity ID. From the call: each next step someone committed to, with who owns it and the date they agreed. Also any existing step whose date changed on the call, with the new date.

## Steps

1. **`whoami`**: confirm write access. If it's missing, stop and tell the rep.
2. **Find the deal room** (see Finding the deal room). If there's no match, stop and tell the rep to run the deal room skill first. This skill never creates a room.
3. **Read the current plan:**
   ```graphql
   query { accord(id: "<id>") {
     opportunityName isDraft
     firstIncompleteStage { id title }
     accordMembers { workspaceAccount { accountId role profile { firstName lastName } }
       workspacePlaceholderRole { id customerVisibleName } }
     stages { id title steps { id effectiveTitle dueDate status internal ownerId ownerWorkspacePlaceholderRoleId } } } }
   ```
4. **Match each commitment to the plan.** Compare each next step from the call against the existing steps that aren't completed. If one already covers it, treat it as a date change (step 5), not a new step. If you're unsure, add it as new rather than overwrite.
5. **Update changed dates.** For a matching step whose date changed, call `updateStep` with `id` and `patch: { dueDate }`. Only do this on seller-only steps (`internal: true`). For a step the buyer can see, don't change it. List the old and new date in the summary so the rep can update it.
6. **Add new steps.** For each new commitment, call `createStep` with `accordId` and `step: { stageId, title, dueDate, internal: true }`, plus an owner:
   - A seller-side owner goes in `ownerId` (their `accountId`).
   - A buyer-side owner goes in `ownerWorkspacePlaceholderRoleId` (the role id, for example Champion), so the step follows whoever fills that role.
   - Use the first incomplete stage's id for `stageId`, unless the step clearly fits a later stage by title.
7. **Return to Gong.**

## Back to Gong

Return the deal room link, the steps added (title, owner, date), the date changes made, and the buyer-visible date changes left for the rep.

## Rep sees

A Slack message from Gong listing what changed, with a link to review and share the new steps with the buyer.

## Guardrails

- New steps are always seller-only (`internal: true`). The rep decides what the buyer sees.
- Only add steps someone committed to on the call. Only change dates that were discussed on the call.
- Never delete, complete, or retitle a step.
- Never call `createStep` twice for the same commitment. If a call fails, check whether the step exists before you retry.

---
name: gong-deal-room-check-in
description: Read-only summary of an Accord deal room for a rep working in Gong. Shows which buyers have engaged recently, which next steps are overdue, and up to three next actions grounded in that data.
---

# Gong: Deal room check-in

Example prompt (input: `[CRM_ID]`):

> Find the Accord for [CRM_ID]. Summarize which buyers have engaged recently, which next steps are overdue, and what I should do next.

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

**What the buyer can see.** A room with `isDraft: true` hasn't been shared with the buyer yet, and steps with `internal: true` are seller-only.

## When to use

Before a call, or any time a rep asks where a deal stands.

## Inputs from Gong

An Accord link, Accord ID, or CRM opportunity ID. Optionally, the date of the last call and any next steps committed on it.

## Steps

1. **`whoami`**: confirm the connection is a seller with read access. This skill never calls `mutate`. If read access is missing, stop and tell the rep.
2. **Find the deal room** (see Finding the deal room). If there's no match, stop and tell the rep there's no Accord for this opportunity. If more than one matches, use the one with `isDraft: false`, then the most recently updated, and say so in the summary.
3. **Pull engagement and the plan** in one query:
   ```graphql
   query { accord(id: "<id>") {
     opportunityName closeDate isDraft customerLastViewedAt
     firstIncompleteStageTitle firstIncompleteStepTitle
     accordMembers { invitationStatus acceptedAt lastActivity
       workspaceAccount { role profile { firstName lastName } account { email } }
       workspacePlaceholderRole { customerVisibleName } }
     stages(orderBy: ORDER_ASC) { title steps {
       effectiveTitle dueDate status completedAt internal
       ownerProfile { firstName lastName } } } } }
   ```
4. **Buyers engaged.** Buyers are members whose `workspaceAccount.role` is `accord_customer_role`. Sort them by `lastActivity`. Put each buyer in one of three groups: active in the last 7 days, quiet for 7+ days, or invited but never opened (no `acceptedAt` and no `lastActivity`). Give each buyer's role from `workspacePlaceholderRole.customerVisibleName`.
5. **Next steps overdue.** These are steps with a `dueDate` in the past and a `status` other than `COMPLETED`. List each one with its owner and how many days late it is. Mark the seller-only (`internal: true`) ones so the rep knows the buyer can't see them.
6. **What to do next.** Give at most three actions, each tied to something found in steps 4 or 5. Examples: the Economic Buyer hasn't opened the room in 14 days, so send the business case to them directly; or two buyer-owned steps are overdue, so confirm new dates with the Champion. If the room is still an unshared draft (`isDraft: true`), the first action is to share it.
7. **Return to Gong.**

## Back to Gong

Return the deal room link, then the three sections (buyers engaged, overdue steps, next actions), each in a few lines.

## Rep sees

A Slack message from Gong with the summary and a link to the deal room.

## Guardrails

- Read only. Never create, update, or share anything.
- Recommend only from what the data shows. Never guess a buyer's intent, and never give a health score.
- A single view right after an email goes out can be an email link scanner, not a person. Don't call a buyer engaged on that view alone.

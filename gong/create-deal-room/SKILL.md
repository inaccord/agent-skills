---
name: gong-create-deal-room
description: Build an Accord deal room from a Gong discovery call. Picks the playbook, links the CRM opportunity, and adds committed next steps, attendees with their roles, the buyer's business case, and requested resources. The room stays unshared for the rep to review.
---

# Gong: Create a deal room from a discovery call

Example prompt (input: `[CRM_ID]`):

> Build the Accord for [CRM_ID] from our latest discovery call. Add the next steps we committed to, the attendees and their roles, the buyer's business case, and the resources they asked for. Keep it unshared for my review.

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

**Rep review happens in Accord.** The new deal room is created unshared (`isDraft: true`), and the steps the agent adds are seller-only (`internal: true`) until the rep makes them visible to the buyer.

## When to use

After a discovery call on a CRM opportunity.

## Inputs from Gong

- CRM opportunity ID and its fields (stage, amount, owner, owner role, custom fields), if the call has one
- Attendees with name, email, and role
- Business case in the buyer's own words
- Stated timeline
- Committed next steps with dates
- Competitors mentioned
- Resources the buyer asked for

## Steps

1. **`whoami`**: confirm write access and permissions. If one is missing, stop and tell the rep which one.
2. **Check for an existing deal room** by `crmId`, if the call has a linked CRM opportunity. If one exists (for example, Accord created it automatically when the opportunity changed stage), do not create another. Fill that room using steps 6-9, then go to step 10. If there is no linked opportunity, ask the rep to confirm before going on: "This call has no linked CRM opportunity. Create the deal room without one?" If the rep gives a CRM ID, use it and run this check. If the rep confirms, continue without one. If the rep says no, stop.
3. **Choose the playbook** (see below).
4. **Create the deal room**: mutate `duplicateAccord` with `style: "from-template"`, the playbook's `accordId`, and `conditionSelections`.
   - 4b. **Get the new room's id.** If `duplicateAccord` does not return the new room's id, find it: query `accords(filter: { isTemplate: { equalTo: false }, createdFromTemplateId: { equalTo: "<playbook accordId>" }, createdAt: { greaterThanOrEqualTo: "<the current time minus 5 minutes, ISO 8601 UTC>" } })` with `id`, `opportunityName`, `createdAt`, and `isDraft`. Use the most recent result. If none comes back, stop and tell the rep the room may not have been created. Never call `duplicateAccord` a second time. Use this id for steps 5-9.
5. **Link and keep unshared**: mutate `updateAccord` with `isDraft: true`, and with `crmId` set to the opportunity ID when there is one. If the rep confirmed in step 2 to go ahead without a CRM opportunity, leave `crmId` unset and say in the summary that the room isn't linked to the CRM yet.
6. **Seed the plan**: for each committed next step, mutate `createStep` with a title, a due date, and an owner (the rep, or a buyer-side role such as Champion). If the playbook already has a matching step, update its due date with `updateStep` instead of adding a duplicate.
7. **Seed the stakeholders**: for each attendee, mutate `createAccordMember`, then `assignWorkspacePlaceholder` with `shouldSendInvite: false` to give them their role (Champion, Economic Buyer, and so on).
8. **Draft the business case**: mutate `upsertSectionById` with rich text built from the buyer's own words, the stated timeline, and the competitors mentioned.
9. **Attach requested resources**: for each request, query `semanticLibraryResourceSearch(query: "<request>")`, then mutate `linkLibraryResources` with the best matches. If library search isn't available for the workspace, skip this step and list the requests in the summary.
10. **Return to Gong**: the deal room link, the playbook used and why, and what was added.

## Choosing the playbook

Work down this list and stop at the first clear answer.

1. **The workspace's playbook rules.** Query `rules(filter: { type: { equalTo: "accord_creation" }, enabled: { equalTo: true } })`. Each rule's `value` is JSON with CRM opportunity conditions (stage, owner role, amount, custom fields) and the `playbookId` to use when they match. If the opportunity matches a rule, use that playbook. This is the same mapping Accord uses to create deal rooms automatically, so the agent picks what the customer's admin chose.
2. **Best fit among approved playbooks.** Query `accords(filter: { isTemplate: { equalTo: true }, approvedAt: { isNull: false } })` with `id`, `templateName`, `context`, and `lastUsed`. `context` is the background text the admin wrote to describe the playbook. Match the call (segment, use case, products discussed) against each playbook's name and context. Break ties by most recently used.
3. **No clear fit.** Use the closest approved playbook anyway, since the room is an unshared draft, and say in the summary that the match was a best guess.
4. **Conditions.** If the chosen playbook has conditions (for example, industry), pass the choices the call supports as `conditionSelections`, so the room gets the matching stages and resources.

## Back to Gong

Deal room link, status (unshared draft), the playbook used and why, and counts of steps, stakeholders, and resources added.

## Rep sees

A Slack message from Gong that the deal room is ready, linking to the draft in Accord to review and share with the buyer.

## Guardrails

- Never share the deal room with the buyer (never set `isDraft: false`). The rep does that.
- Never send buyer invites (always pass `shouldSendInvite: false`). The rep invites buyers when they share the room.
- Never create a deal room without a CRM opportunity unless the rep confirmed it.
- Only add people who were on the call.
- Only add steps someone committed to on the call.

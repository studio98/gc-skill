---
name: grandcentral
description: Interact with GrandCentral CRM. Use when asked to look up clients, contacts, organizations, deals, projects, or support tickets in GrandCentral; when creating or updating CRM records (orgs, contacts, deals, projects, tasks, tickets); when logging notes or activities; when checking ticket queues, deal pipelines, or project boards; when managing checklists, knowledge base, or tasks. Triggers on any request involving GrandCentral, GC, client records, deals, projects, or support tickets.
---

# GrandCentral CRM Skill

GrandCentral is a CRM platform. This skill provides instructions for interacting with the GrandCentral API.

## Configuration (required before use)

Before using this skill, the following must be configured in your local `TOOLS.md` or environment:

| Setting | Description |
|---------|-------------|
| `GRANDCENTRAL_API_URL` | Base API URL, e.g. `https://api.yourdomain.com/v1/tools` |
| `GRANDCENTRAL_API_KEY` | Your API key |

**How to call the API:**
```
POST {GRANDCENTRAL_API_URL}
Authorization: {GRANDCENTRAL_API_KEY}
Content-Type: application/json

{ "action": "<actionName>", ...params }
```

Every request body must include `"action"`. See [references/api.md](references/api.md) for all available actions and parameters.

---

## Module Reference Files

Load the relevant reference file when working in that area. Do not load all files at once.

| Module | Reference File | When to load |
|--------|---------------|--------------|
| Organizations & Contacts | [references/organizations.md](references/organizations.md) | Looking up, creating, or updating orgs/contacts |
| Deals | [references/deals.md](references/deals.md) | Deal pipeline, stages, creating/updating deals |
| Projects | [references/projects.md](references/projects.md) | Project boards, stages, creating/updating projects |
| Support Tickets | [references/tickets.md](references/tickets.md) | Ticket queues, channels, replies, triage |
| Tasks & Activities | [references/tasks.md](references/tasks.md) | Tasks, checklists, notes, activity log |
| Full API Reference | [references/api.md](references/api.md) | When you need params/response shapes for any action |
| Instance Config | [references/context.md](references/context.md) | Team structure, routing rules, instance-specific IDs |

---

## Common Workflows (quick nav)

### Look up a client
→ Load [references/organizations.md](references/organizations.md)

### Create org + contact
→ Load [references/organizations.md](references/organizations.md)

### Create / update a deal
→ Load [references/deals.md](references/deals.md)

### Create / update a project
→ Load [references/projects.md](references/projects.md)

### Support ticket triage
→ Load [references/tickets.md](references/tickets.md)

### Log a note or task
→ Load [references/tasks.md](references/tasks.md)

### Unknown action / need full param list
→ Load [references/api.md](references/api.md)

---

## Key Rules (always apply)
- Orgs must have at least one contact before creating deals or projects
- Always run lookup sequences to get IDs before creating records that need them (boards, channels, stages)
- When marking a deal `lost`, a `lostExplain` reason is required
- For natural language KB queries, prefer `getAiKbSearch` over `searchKnowledgeBase`

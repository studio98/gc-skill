---
name: grandcentral
description: Interact with GrandCentral CRM. Use when asked to look up clients, contacts, organizations, deals, projects, support tickets, tasks, subscription boards, or the document library in GrandCentral; when creating or updating CRM records (orgs, contacts, deals, projects, tasks, tickets); when logging notes, calls, or activities; when checking ticket queues, deal pipelines, project boards, or task lists; when reading SOPs, policies, or custom documents from the library; when managing subscription board checklists or delivery items. Triggers on any request involving GrandCentral, GC, client records, deals, projects, support tickets, tasks, or internal documents.
---

# GrandCentral CRM Skill

GrandCentral is a full-stack CRM platform. This skill provides instructions for interacting with the GrandCentral REST API.

## Configuration (required before use)

| Setting                  | Description |
|--------------------------|-------------|
| `GRANDCENTRAL_API_KEY`   | API key (prefixed `gc_`) — get from GrandCentral → Settings → API Keys |

**Base URL:** `https://api-v2.grandcentr.al/api`

**How to call the API:**
```
GET/POST/PATCH/DELETE https://api-v2.grandcentr.al/api/{endpoint}
Authorization: Bearer {GRANDCENTRAL_API_KEY}
Content-Type: application/json
```

Example: `GET https://api-v2.grandcentr.al/api/library/items?search=managing+deals`

This is a standard REST API. Use the correct HTTP method for each endpoint (GET to read, POST to create, PATCH to update). Do NOT wrap requests in an `action` field.

---

## Module Reference Files

Load only the reference file you need. Do not load all files at once.

| Module                   | Base Path             | Reference File                                       | When to load |
|--------------------------|-----------------------|------------------------------------------------------|--------------|
| Users (read-only)        | /users                | [references/tickets.md](references/tickets.md)       | Finding user IDs for assignment |
| Support Tickets          | /tickets              | [references/tickets.md](references/tickets.md)       | Ticket queues, replies, triage, ticket tasks |
| Organizations & Contacts | /organizations        | [references/organizations.md](references/organizations.md) | Looking up clients, contacts, orgs |
| Deals                    | /deals                | [references/deals.md](references/deals.md)           | Deal pipeline, stages, proposals, activities |
| Projects                 | /projects, /boards    | [references/projects.md](references/projects.md)     | Project boards, steps, time tracking |
| Tasks (to-dos)           | /tasks                | [references/tasks.md](references/tasks.md)           | Company-wide task list, subtasks, assignment |
| Subscription Board       | /subscription-board   | [references/subscription-board.md](references/subscription-board.md) | Delivery boards, setup/recurring checklists |
| Library                  | /library              | [references/library.md](references/library.md)       | SOPs, policies, how-tos, custom documents |
| Calendar Events          | /calendar-events      | [references/calendar.md](references/calendar.md)     | Scheduling events, checking availability, RSVP |
| Full API index           | —                     | [references/api.md](references/api.md)               | Module list, base URL, response format, activity feed format |
| Instance config          | —                     | [references/context.md](references/context.md)       | Team structure, routing rules, instance-specific IDs |

---

## Common Workflows

### Look up a client / org
→ Load [references/organizations.md](references/organizations.md)
```
GET /organizations?search=acme
GET /organizations/{id}
```

### Find a contact
→ Load [references/organizations.md](references/organizations.md)
```
GET /contacts?search=john
GET /organizations/{orgId}/contacts
```

### Create or update a deal
→ Load [references/deals.md](references/deals.md)
```
GET /deals/boards          → get board & stage IDs
POST /deals                → create
PATCH /deals/{id}/stage    → move pipeline stage
PATCH /deals/{id}/status   → mark won / lost
```

### Log a note, call, or activity
→ Load the relevant module file (deals, projects, or organizations)
```
POST /deals/{id}/activities
POST /projects/{id}/activities
POST /organizations/{id}/activities
```

### Support ticket triage
→ Load [references/tickets.md](references/tickets.md)
```
GET /tickets                         → open tickets
POST /tickets/{id}/reply             → send reply to customer
POST /tickets/{id}/notes             → add internal note
PATCH /tickets/{id}/status           → close or change status
PATCH /tickets/{id}/assign           → assign to team member
```

### Check or manage tasks
→ Load [references/tasks.md](references/tasks.md)
```
GET /tasks                           → my open tasks (default)
GET /tasks?assignedTo={userId}       → specific person's tasks
POST /tasks                          → create a task
PATCH /tasks/{id}/complete           → toggle completion
POST /tasks/{id}/subtasks            → add a subtask
POST /tasks/{id}/comments            → leave a comment
```

### Work with a project
→ Load [references/projects.md](references/projects.md)
```
GET /boards                                        → find board & stage IDs
GET /projects?orgId={id}                           → projects for a client
GET /projects/{id}                                 → full project with steps
POST /projects/{id}/sections/{sectionId}/steps     → add a step
PATCH /projects/{id}/steps/{stepId}/complete       → complete a step
```

### Check subscription board / delivery checklists
→ Load [references/subscription-board.md](references/subscription-board.md)
```
GET /subscription-board/boards             → list boards
GET /subscription-board/items?boardId={id} → items on a board
GET /subscription-board/items/{id}         → item detail with checklists
```

### Read a document or SOP from the library
→ Load [references/library.md](references/library.md)
```
GET /library/items?search=managing+deals   → find by name
GET /library/items/{id}/content            → get full Markdown text
GET /library/folders                       → browse folder structure
GET /library/folders/{id}                  → folder contents
```

---

### Schedule or check calendar availability
→ Load [references/calendar.md](references/calendar.md)
```
GET /calendar-events/free-busy?startTime=...&endTime=...&userIds[]=5   → check if someone is free
GET /calendar-events?startTime=...&endTime=...&userIds[]=5              → list someone's events for a period
POST /calendar-events                                                    → book a new event
PATCH /calendar-events/{id}                                              → update an event
DELETE /calendar-events/{id}                                             → delete an event
```

## Key Rules (always apply)

- **REST only** — use GET, POST, PATCH, DELETE. Never use an `action` field.
- **Look up IDs before creating** — always resolve board IDs, stage IDs, channel IDs, category IDs before using them in create/update calls.
- **Tasks default to your own** — `GET /tasks` returns tasks assigned to the authenticated user. Pass `assignedTo={userId}` for someone else, or `all=true` for everyone.
- **Deals default to in_progress** — `GET /deals` returns active deals only. Pass `status=won` or `status=lost` for closed deals.
- **Projects default to active** — `GET /projects` returns active projects. Pass `status=completed` or `status=archived` for others.
- **Tickets default to open** — `GET /tickets` returns open tickets. Pass `status=closed` for closed ones.
- **Boolean query params** — pass as `true` / `false` strings (e.g. `?completed=true`), not as `1`/`0`.
- **Lost deals require a reason** — when setting `status: "lost"`, always include `lostExplain`.
- **No DELETE for deals or projects** — mark deals as `won`/`lost`; mark projects as `archived`/`completed`.
- **Section delete needs confirmation** — if the API returns 422 (incomplete steps), stop and ask the user before passing `force: true`.
- **Reply body is HTML** — ticket replies and notes use HTML. Use `<p>` tags only, no headers or heavy inline styles.
- **Library content for file/chart types is null** — `file`, `flowChart`, and `orgChart` items return `content: null` from `/content`. Check `type` first.
- **subtaskCount = open subtasks only** — `subtaskCount` reflects pending (non-completed) subtasks, not total.
- **Calendar datetimes are ISO 8601 only** — always include a timezone offset (e.g. `-04:00` or `Z`). Unix timestamps are rejected. Read the `timezone` field on event responses to know the company's timezone.
- **Check free-busy before booking** — call `GET /calendar-events/free-busy` before creating an event. If the API returns 409 (conflict), show the user the conflicting events and ask whether to proceed with `force: true`.
- **Calendar `participants` replaces on update** — if you include `participants` in a PATCH body, it replaces the full list. Omit the field entirely to leave participants unchanged.

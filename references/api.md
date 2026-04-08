# GrandCentral Agent API Reference

Base URL: `https://api-v2.grandcentr.al/api`

All requests require:
```
Authorization: Bearer {GRANDCENTRAL_API_KEY}
Content-Type: application/json
```

## Modules

| Module                     | Base Path          | Reference File                               |
|----------------------------|--------------------|----------------------------------------------|
| Users (read-only)          | /users             | references/tickets.md                        |
| Support Tickets            | /tickets           | references/tickets.md                        |
| Organizations & Contacts   | /organizations     | references/organizations.md                  |
| Projects                   | /projects          | references/projects.md                       |
| Deals                      | /deals             | references/deals.md                          |
| Subscription Boards        | /subscription-board| references/subscription-board.md                |
| Tasks (to-dos)             | /tasks             | references/tasks.md                          |
| Library                    | /library           | references/library.md                        |
| Billing Subscriptions      | /billing           | _(not exposed via agent API)_                |

> **Naming note:** `/subscription-board` refers to the operational marketing delivery boards (checklists, setup, recurring tasks). It is distinct from billing subscriptions (`OrganizationSubscription`), which are not exposed via this API.

## Authentication

API keys are created in GrandCentral → Settings → API Keys.
Keys are prefixed `gc_` and scoped to a specific company within a tenant.

## Response Format

All list endpoints return paginated objects:
```json
{
  "data": [...],
  "pagination": { "total": 100, "perPage": 20, "currentPage": 1, "lastPage": 5 }
}
```

Error responses:
```json
{ "success": false, "message": "Descriptive error message" }
```

## Timestamps

All timestamps are ISO 8601 format: `"2025-04-01T10:00:00+00:00"`

## Activity Feed — Unified Format

Activity feeds appear on deals, projects, and organizations (`GET /{resource}/{id}/activities`).

All items share the same shape regardless of type:

```json
{
  "id": 123,
  "activityType": "note",
  "subject": null,
  "notesResult": "Phone Call",
  "direction": "incoming",
  "body": "Called customer to discuss renewal.",
  "forDate": "2025-04-01T10:00:00+00:00",
  "author": { "id": 10, "name": "Jane Smith" },
  "contact": null,
  "isPinned": false,
  "createdAt": "2025-04-01T10:00:00+00:00"
}
```

### activityType values
| Value    | Source              | Notes |
|----------|---------------------|-------|
| `note`   | Manual log          | Staff note |
| `call`   | Manual log          | Phone call |
| `sms`    | Manual log          | SMS/text message |
| `email`  | Gmail sync          | Read-only — synced automatically. `body` is markdown, `subject` is populated |
| `ticket` | Support ticket      | Org activities only. `subject` = ticket subject |

### direction
`incoming` = customer initiated · `outgoing` = staff initiated · relevant for `call` and `sms`

### Email activities
- `body` is **markdown** (converted from HTML via `league/html-to-markdown`, tracking pixels and layout tables removed)
- `subject` contains the email subject line
- `author` is `null` (emails are not attributed to a specific CRM user)
- Email activities are **read-only** — they cannot be created via POST

### Pagination for activity feeds
- Default `perPage`: 20 (max 100)
- Pass `page` param for subsequent pages
- Activity feeds merge multiple sources (ContactAction, Gmail, SupportTickets) and are sorted by `createdAt` descending

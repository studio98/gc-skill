# Support Tickets Reference

All endpoints require:
```
Authorization: Bearer {GRANDCENTRAL_API_KEY}
Content-Type: application/json
Base URL: https://api-v2.grandcentr.al/api

> All endpoints below are relative to this base URL (e.g. `GET /tickets` → `GET https://api-v2.grandcentr.al/api/tickets`)
```

---

## Content Format for Bodies and Replies

When creating tickets, sending replies, or adding notes, the `body` field must be **HTML**.
Use plain paragraph tags — no `<h1>`–`<h6>` headers, no heavy inline styling, no layout tables.

Good:
```html
<p>Hi John, we've investigated and the issue is resolved on our end. Please try logging in again and let us know if the problem persists.</p>
<p>If you need further help, reply to this email or contact us directly.</p>
```

Avoid:
- `<h1>`, `<h2>`, `<h3>` headers
- Complex inline CSS (`style="font-family:Arial; color:#333; padding:20px"`)
- HTML email layout tables

---

## Enum Reference

### Ticket Status (`status`)
| Value          | Meaning                                      |
|----------------|----------------------------------------------|
| `open`         | Ticket is active (covers "open" and "replied") |
| `closed`       | Ticket is resolved/closed                    |
| `notification` | Flagged as a notification                    |
| `spam`         | Marked as spam (read-only, not settable)     |

**Default when listing:** If no `status` filter is passed, GET /tickets returns `open` tickets only.

**When setting status via PATCH:** Only `open`, `closed`, and `notification` are accepted.
`replied` is managed automatically when a reply is sent. `spam` cannot be set via the API.

### Priority (`priority`)
| Value      | Meaning  |
|------------|----------|
| `normal`   | Normal   |
| `major`    | Major    |
| `critical` | Critical |

### Task Priority (`priority`)
| Value    | Meaning     |
|----------|-------------|
| `none`   | No priority |
| `urgent` | Urgent      |
| `high`   | High        |
| `normal` | Normal      |
| `low`    | Low         |

### Message Type (`messageType`)
| Value           | Meaning                                 |
|-----------------|-----------------------------------------|
| `customer`      | Inbound message from customer           |
| `agent_reply`   | Outbound reply from a support agent     |
| `internal_note` | Internal note — not visible to customer |

---

## Reference Data

### GET /users
List staff users (admin, staff, no-login). Clients and affiliates are excluded.
Use `id` when assigning tickets or tasks.

**Query params:** `search` (optional) — filter by name or email.

**Response:**
```json
[
  { "id": 10, "firstName": "Jane", "lastName": "Smith", "fullName": "Jane Smith", "email": "jane@company.com", "role": 1 }
]
```

---

### GET /tickets/channels
List all support channels (inbox sources).

**Response:**
```json
[
  { "id": 1, "label": "General Support" },
  { "id": 2, "label": "Billing" }
]
```

---

### GET /tickets/categories
List all ticket categories.

**Response:**
```json
[
  { "id": 1, "title": "Bug Report", "channelId": 1 },
  { "id": 2, "title": "Feature Request", "channelId": null }
]
```

---

### POST /tickets/categories
Create a new category.

**Body:**
```json
{ "title": "Onboarding Questions", "channelId": 1 }
```
`channelId` is optional.

**Response:** `201 Created` — CategoryResource

---

### GET /tickets/statuses
List custom (company-defined) ticket status labels.

**Response:**
```json
[
  { "id": 5, "title": "Waiting on Client" },
  { "id": 6, "title": "Escalated" }
]
```

---

## Tickets

### GET /tickets
List tickets. **Defaults to open tickets when no status is passed.**

**Query params:**
| Param        | Type   | Description |
|--------------|--------|-------------|
| `status`     | string | `open` (default) \| `closed` |
| `channelId`  | int    | Filter by channel |
| `categoryId` | int    | Filter by category |
| `assignedTo` | int    | Filter by assigned user ID |
| `search`     | string | Search subject |
| `perPage`    | int    | Results per page (default: 20, max: 100) |

**Response:**
```json
{
  "data": [
    {
      "id": 123,
      "subject": "My login is broken",
      "fromName": "John Doe",
      "fromEmail": "john@example.com",
      "status": "open",
      "customStatus": { "id": 5, "title": "Waiting on Client" },
      "priority": "normal",
      "channel": { "id": 1, "label": "General Support" },
      "category": { "id": 1, "title": "Bug Report" },
      "assignedTo": { "id": 10, "fullName": "Jane Smith" },
      "organization": { "id": 50, "name": "Acme Corp" },
      "createdAt": "2025-04-01T06:00:00-04:00"
    }
  ],
  "pagination": {
    "total": 150,
    "perPage": 20,
    "currentPage": 1,
    "lastPage": 8
  }
}
```

---

### POST /tickets
Create a new ticket.

**Body:**
```json
{
  "subject": "My login is broken",
  "channelId": 1,
  "fromName": "John Doe",
  "fromEmail": "john@example.com",
  "orgId": 50,
  "categoryId": 1,
  "assignedTo": 10,
  "priority": 0,
  "body": "<p>I cannot log in since yesterday. Error: 403 Forbidden.</p>"
}
```
Required: `subject`, `channelId`, `fromName`, `fromEmail`.
`body` is HTML — use `<p>` tags, no headers or heavy styling.

**Response:** `201 Created` — full TicketResource (see GET /tickets/{id})

---

### GET /tickets/{id}
Get full ticket detail including all messages. All timestamps are in the company's timezone.

**Response:**
```json
{
  "id": 123,
  "subject": "My login is broken",
  "fromName": "John Doe",
  "fromEmail": "john@example.com",
  "status": "open",
  "systemStatusId": 1,
  "customStatus": null,
  "priority": "normal",
  "channel": { "id": 1, "label": "General Support" },
  "category": { "id": 1, "title": "Bug Report" },
  "assignedTo": { "id": 10, "fullName": "Jane Smith", "email": "jane@company.com" },
  "organization": { "id": 50, "name": "Acme Corp" },
  "openedAt": "2025-04-01T06:00:00-04:00",
  "closedAt": null,
  "createdAt": "2025-04-01T06:00:00-04:00",
  "messages": [
    {
      "id": 1,
      "messageType": "customer",
      "authorName": "John Doe",
      "authorEmail": "john@example.com",
      "body": "I cannot log in since yesterday.",
      "createdAt": "2025-04-01T06:00:00-04:00"
    },
    {
      "id": 2,
      "messageType": "internal_note",
      "authorName": "Jane Smith",
      "authorEmail": "jane@company.com",
      "body": "Looks like a session expiry issue.",
      "createdAt": "2025-04-01T07:00:00-04:00"
    },
    {
      "id": 3,
      "messageType": "agent_reply",
      "authorName": "Jane Smith",
      "authorEmail": "jane@company.com",
      "body": "Hi John, please try clearing your cache.",
      "createdAt": "2025-04-01T07:05:00-04:00"
    }
  ]
}
```

---

### PATCH /tickets/{id}/status
Change the system or custom status.

Valid system values: `open`, `closed`, `notification`.
Do NOT use `replied` (set automatically on reply) or `spam`.

**Body (system status):**
```json
{ "type": "system", "value": "closed" }
```

**Body (custom status):**
```json
{ "type": "custom", "value": 5 }
```
`value` is the custom status ID from GET /tickets/statuses.

**Response:** Updated TicketResource

---

### PATCH /tickets/{id}/priority
**Body:** `{ "priority": 1 }` — 0=normal | 1=major | 2=critical

---

### PATCH /tickets/{id}/assign
**Body:** `{ "userId": 10 }` — set `userId` to `null` to unassign.

---

### PATCH /tickets/{id}/category
**Body:** `{ "categoryId": 2 }` — set to `null` to clear.

---

## Ticket Messages

### POST /tickets/{id}/reply
Send an agent reply to the customer. Triggers an outbound email to the ticket's `fromEmail`.

The ticket status is automatically set to "replied" after sending.

**Body:**
```json
{
  "body": "<p>Hi John, we've investigated and the issue is now resolved. Please try logging in again.</p>",
  "cc": ["manager@company.com"],
  "bcc": []
}
```
- `body` is required — HTML, use `<p>` tags only, no `<h1>`–`<h6>`, no heavy inline styles
- `cc`, `bcc` are optional arrays of email addresses
- The `to` address is always the ticket's customer email (`fromEmail`)

**Response:** `201 Created` — TicketMessageResource with `messageType: "agent_reply"`

---

### POST /tickets/{id}/notes
Add an internal note to the ticket. Visible to staff only — **no email is sent**.

Use for investigation notes, coordination between staff, or draft replies before sending.

**Body:**
```json
{
  "body": "<p>Checked server logs — session token expired due to IP change. Will fix the auth config.</p>"
}
```
- `body` is required — HTML, same content rules as replies

**Response:** `201 Created` — TicketMessageResource with `messageType: "internal_note"`

---

## Ticket Tasks

### GET /tickets/{id}/tasks
List top-level tasks on a ticket.

**Response:**
```json
[
  {
    "id": 77,
    "ticketId": 123,
    "name": "Investigate the 403 error",
    "description": "Check server logs for IP blocking rules.",
    "priority": "normal",
    "isCompleted": false,
    "completedAt": null,
    "completedBy": null,
    "createdBy": "Jane Smith",
    "dueDate": "2025-04-05",
    "clickupId": "abc123xyz",
    "createdAt": "2025-04-01T07:00:00-04:00"
  }
]
```
`clickupId` is non-null when the task is synced with a ClickUp task.

---

### POST /tickets/{id}/tasks
Create a new task on a ticket.

**Body:**
```json
{
  "name": "Investigate the 403 error",
  "description": "Check server logs for IP blocking rules.",
  "priority": 3,
  "dueDate": "2025-04-05"
}
```
Required: `name`. Priority: 0=none 1=urgent 2=high 3=normal 4=low.
`description` is plain text (not HTML).

**Response:** `201 Created` — TicketTaskResource

---

### PATCH /tickets/{id}/tasks/{taskId}
Update a task. All fields optional.

**Body:**
```json
{
  "name": "Updated task name",
  "description": "Updated instructions",
  "priority": 1,
  "dueDate": "2025-04-10"
}
```

---

### DELETE /tickets/{id}/tasks/{taskId}
Delete a task permanently.

**Response:** `{ "success": true, "message": "Task deleted" }`

---

### PATCH /tickets/{id}/tasks/{taskId}/complete
Mark complete or reopen.

**Body:** `{ "completed": true }` — set `false` to reopen.

---

## Task Comments

### GET /tickets/{id}/tasks/{taskId}/comments
List comments oldest-first.

**Response:**
```json
[
  {
    "id": 5,
    "taskId": 77,
    "comment": "I found the issue in the nginx config.",
    "authorId": 10,
    "authorName": "Jane Smith",
    "createdAt": "2025-04-01T08:00:00-04:00"
  }
]
```

---

### POST /tickets/{id}/tasks/{taskId}/comments
Add a comment.

**Body:** `{ "comment": "I found the issue in the nginx config." }`

**Response:** `201 Created` — TaskCommentResource

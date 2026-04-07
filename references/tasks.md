# Tasks Reference

All endpoints require:
```
Authorization: Bearer {GRANDCENTRAL_API_KEY}
Content-Type: application/json
Base URL: {GRANDCENTRAL_API_URL}/api/gc/v1
```

Tasks are global to-dos assigned to team members. They support priorities, due dates, subtasks, assignees, custom statuses, and comments.

---

## Enum Reference

### Priority
| Integer | Label    |
|---------|----------|
| `0`     | none     |
| `1`     | low      |
| `2`     | medium   |
| `3`     | high     |
| `4`     | urgent   |

---

## Reference Data

### GET /tasks/statuses
List all company-defined custom task status labels. Use the returned `id` when creating or updating tasks.

**Response:**
```json
[
  { "id": 1, "name": "In Review", "color": "#f59e0b" },
  { "id": 2, "name": "Blocked",   "color": "#ef4444" }
]
```

---

## Tasks

### GET /tasks
List top-level tasks (those with no parent).

**Default behaviour:** returns **open** tasks assigned to the **authenticated API user**. You almost never want all company tasks — use the filters below to scope appropriately.

**Query params:**
| Param        | Type    | Description |
|--------------|---------|-------------|
| `completed`  | boolean | Show completed tasks instead of open (default: `false`) |
| `assignedTo` | int     | Show tasks assigned to a specific user ID |
| `all`        | boolean | Return tasks for all company members, not just the caller (default: `false`) |
| `priority`   | int     | Filter by priority (0–4) |
| `orgId`      | int     | Filter by organization ID |
| `search`     | string  | Partial, case-insensitive title search |
| `perPage`    | int     | Results per page (default: 20, max: 100) |

**Response:**
```json
{
  "data": [
    {
      "id": 55,
      "title": "Review Q2 proposals",
      "content": "Check all three proposals before Friday.",
      "priority": 2,
      "isCompleted": false,
      "completedAt": null,
      "completedBy": null,
      "dueDate": "2025-06-30",
      "organization": { "id": 12, "name": "Acme Corp" },
      "author": { "id": 5, "fullName": "Jane Smith" },
      "assignees": [
        { "id": 5, "fullName": "Jane Smith", "email": "jane@company.com" }
      ],
      "taskStatus": { "id": 1, "name": "In Review", "color": "#f59e0b" },
      "parentId": null,
      "subtaskCount": 2,
      "commentCount": 3,
      "createdAt": "2025-04-01T10:00:00+00:00",
      "subtasks": null
    }
  ],
  "pagination": {
    "total": 40,
    "perPage": 20,
    "currentPage": 1,
    "lastPage": 2
  }
}
```

> `subtasks` is `null` on the list endpoint. Use GET /tasks/{id} to get subtasks.

---

### POST /tasks
Create a new top-level task.

**Body:**
```json
{
  "title": "Review Q2 proposals",
  "content": "Check all three proposals before Friday.",
  "priority": 2,
  "dueDate": "2025-06-30",
  "orgId": 12,
  "statusId": 1,
  "assignees": [5, 8]
}
```

Required: `title`.
`assignees` defaults to the authenticated API user if omitted.
`priority`: 0=none 1=low 2=medium 3=high 4=urgent.

**Response:** `201 Created` — full TaskResource (see GET /tasks/{id})

---

### GET /tasks/{id}
Get full task detail including all subtasks.

**Response:**
```json
{
  "id": 55,
  "title": "Review Q2 proposals",
  "content": "Check all three proposals before Friday.",
  "priority": 2,
  "isCompleted": false,
  "completedAt": null,
  "completedBy": null,
  "dueDate": "2025-06-30",
  "organization": { "id": 12, "name": "Acme Corp" },
  "author": { "id": 5, "fullName": "Jane Smith" },
  "assignees": [
    { "id": 5, "fullName": "Jane Smith", "email": "jane@company.com" }
  ],
  "taskStatus": { "id": 1, "name": "In Review", "color": "#f59e0b" },
  "parentId": null,
  "subtaskCount": 2,
  "commentCount": 3,
  "createdAt": "2025-04-01T10:00:00+00:00",
  "subtasks": [
    {
      "id": 56,
      "title": "Read Acme proposal",
      "priority": 1,
      "isCompleted": false,
      "completedAt": null,
      "completedBy": null,
      "dueDate": null,
      "assignees": [{ "id": 5, "fullName": "Jane Smith", "email": "jane@company.com" }],
      "subtaskCount": 0,
      "commentCount": 0,
      "createdAt": "2025-04-01T11:00:00+00:00"
    }
  ]
}
```

Use `GET /tasks/{id}/comments` for the task's comments (not included here).

---

### PATCH /tasks/{id}
Update a task's title, content, priority, due date, or status. Only fields present in the request body are updated.

**Body (all fields optional):**
```json
{
  "title": "Updated task title",
  "content": "Updated description.",
  "priority": 3,
  "dueDate": "2025-07-15",
  "statusId": 2
}
```

Pass `dueDate: null` to clear the due date.
Pass `statusId: 0` or `statusId: null` to clear the custom status.

**Response:** Updated TaskResource

---

### PATCH /tasks/{id}/complete
Toggle a task's completion status. No request body required.

- If currently **open** → marks it complete (records `completedAt` and `completedBy`).
- If currently **complete** → reopens it (clears `completedAt` and `completedBy`).

**Response:** Updated TaskResource

---

### PATCH /tasks/{id}/assign
Replace the full assignee list for a task. Clears all existing assignees and sets the new list.

**Body:**
```json
{ "assignees": [5, 8] }
```

Pass an empty array to remove all assignees:
```json
{ "assignees": [] }
```

**Response:** Updated TaskResource

---

### POST /tasks/{id}/subtasks
Create a subtask under an existing task. The new task is automatically linked via `parentId`.

**Body:**
```json
{
  "title": "Draft the intro section",
  "content": "Focus on the executive summary.",
  "priority": 1,
  "dueDate": "2025-06-25",
  "assignees": [5]
}
```

Required: `title`. `assignees` defaults to the authenticated API user if omitted.

**Response:** `201 Created` — TaskResource for the new subtask

---

## Task Comments

### GET /tasks/{id}/comments
List all comments on a task, oldest first.

**Response:**
```json
{
  "data": [
    {
      "id": 11,
      "taskId": 55,
      "comment": "Reached out to the client for updated numbers.",
      "authorId": 5,
      "authorName": "Jane Smith",
      "createdAt": "2025-04-02T09:30:00+00:00"
    }
  ]
}
```

---

### POST /tasks/{id}/comments
Add a comment to a task.

**Body:**
```json
{ "comment": "Reached out to the client for updated numbers." }
```

Required: `comment` (max 5000 chars).

**Response:** `201 Created` — TaskCommentResource

---

## Agent Guidance

### Finding your own open tasks
- `GET /tasks` — returns your open tasks by default (no params needed)
- `GET /tasks?completed=true` — your completed tasks

### Finding open tasks for someone else
1. Call `GET /users?search=Jane` to find their user ID
2. Call `GET /tasks?assignedTo={userId}` — returns their open tasks
3. `GET /tasks?assignedTo={userId}&completed=true` — their completed tasks

### Listing all company tasks (use sparingly)
- `GET /tasks?all=true` — all open tasks across all team members
- Only use this when you genuinely need the full picture, not as the default

### Completing a task
- Call `PATCH /tasks/{id}/complete` — no body needed, it toggles the current state
- Check `isCompleted` in the response to confirm the new state

### Assigning a task to someone new
- Call `PATCH /tasks/{id}/assign` with `{ "assignees": [userId] }`
- To add to existing assignees, include ALL desired user IDs (not just the new one)

### Setting priority and due date
- Both can be set at creation (`POST /tasks`) or updated later (`PATCH /tasks/{id}`)
- Priority integers: 0=none 1=low 2=medium 3=high 4=urgent

### Working with subtasks
- `GET /tasks/{id}` returns all subtasks inline under `subtasks`
- Create new subtasks with `POST /tasks/{id}/subtasks`
- Subtasks can be updated/completed using their own `id` with the same `PATCH /tasks/{id}` and `PATCH /tasks/{id}/complete` endpoints

### Custom statuses
- Get the company's status labels first: `GET /tasks/statuses`
- Use the returned `id` in `statusId` when creating or updating tasks

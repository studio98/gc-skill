# Projects Reference

All endpoints require:
```
Authorization: Bearer {GRANDCENTRAL_API_KEY}
Content-Type: application/json
Base URL: {GRANDCENTRAL_API_URL}  (e.g. https://api-v2.yourdomain.com/api)
```

---

## Agent Guidance

### Data Model Overview

```
Board
 └── Stage (ordered by menuOrder 0, 1, 2...)
      └── Project (linked to org, has manager/contact)
           └── Section (ordered by menuOrder)
                └── Step / Todo (top-level, parentId = null)
                     └── Sub-step (parentId = step.id, ONE level only)
```

### Working with Sub-steps

- Sub-steps are steps with a non-null `parentId` pointing to the parent step's `id`
- **Only one level of nesting exists.** Sub-steps never have their own sub-steps.
- In GET /projects/{id} and GET /projects/{id}/sections, the `steps` array contains only **top-level steps** (parentId = null). Sub-steps appear inside each step's `subSteps` array.
- To **create a sub-step**, POST to `/projects/{id}/sections/{sectionId}/steps` with `parentId` set to the parent step's `id`. Use the same sectionId as the parent step.
- To **complete a sub-step**, PATCH `/projects/{id}/steps/{subStepId}/complete` — same endpoint as a regular step.
- To **check overall step progress**, count `subSteps` where `isCompleted: true` vs total `subSteps.length`.
- Sub-steps appear in the `subSteps` array on their parent in list responses, but they can also be individually fetched/updated via `/projects/{id}/steps/{subStepId}`.
- The `timeTrackedSeconds` field on a step reflects time logged directly against that step only, not its sub-steps.

### Deleting a Section — Confirmation Required

**Never delete a section without confirming with the user if it has active steps.**

Workflow:
1. Attempt DELETE `/projects/{id}/sections/{sectionId}` (no `force` param)
2. If the API returns `422` with a message about incomplete steps — **stop and tell the user** how many active steps exist and ask if they want to proceed
3. Only if the user explicitly confirms: repeat the request with `{ "force": true }` in the body

Example 422 response:
```json
{ "success": false, "message": "This section has 3 incomplete step(s). Pass force=true to delete anyway. Confirm with the user first." }
```

### Project Deletion

There is no DELETE endpoint for projects. To remove a project from active view:
- Mark it **archived**: `PATCH /projects/{id}/status` with `{ "status": "archived" }`
- Mark it **completed**: `PATCH /projects/{id}/status` with `{ "status": "completed" }`

### What Projects Are Visible

The project list only shows **board-eligible** projects — projects that:
1. Have an associated organization
2. AND meet one of: paid something (`amount != balance`), zero-cost (`amount == 0`), or explicitly placed on board (`on_board = true`)

Not every project in the system is on the board. Projects created outside the board workflow may not appear.

### Reading a Full Project

`GET /projects/{id}` returns everything in one call:
- Sections → Steps → Sub-steps
- Assignees on each step and sub-step
- ClickUp IDs on steps/sub-steps
- Section due dates

Use this for a full picture. For just the step list, use `GET /projects/{id}/sections`.

### ClickUp Integration

Steps and sub-steps may have a `clickupId` field. When non-null, the step is linked to a ClickUp task. Comments posted to a step with a ClickUp link may sync to ClickUp automatically depending on company settings. Always include `clickupId` in any responses you return to the user about steps — it helps them cross-reference with ClickUp.

### Assignees

Each step has an `assignees` array:
```json
{ "id": 10, "name": "Jane Smith", "type": "user" }
```
- `type: "user"` — internal staff member (use IDs from GET /users)
- `type: "projectUser"` — external project team member (client-side contact added to the project)

The assignees list is **read-only** via this API. To assign someone, the action must be done in the GrandCentral UI or via the core API.

### Time Tracking

- `timeTrackedSeconds` on a step (in `GET /projects/{id}/steps/{stepId}`) is the sum of all completed time log entries on that step.
- It is `null` in list responses (sections/projects) — only populated in the individual step detail endpoint.
- Time entries have a `status` field: `"started"` means the timer is still running, `"completed"` means it was stopped.
- To get all time for a project (across all steps): `GET /projects/{id}/time`

---

## Enum Reference

### Project Status (`status`)
| Value       | Meaning                                      |
|-------------|----------------------------------------------|
| `active`    | In progress — not completed, archived, or cancelled |
| `completed` | Marked done (`completedAt` is set)           |
| `archived`  | Hidden from active views (`isArchived: true`) |
| `cancelled` | Cancelled (`cancelledAt` is set)             |

### Step Priority (`priority` in requests = int, in responses = string)
| Request value | Response label | Meaning     |
|---------------|----------------|-------------|
| `0`           | `none`         | No priority |
| `1`           | `urgent`       | Urgent      |
| `2`           | `high`         | High        |
| `3`           | `normal`       | Normal      |
| `4`           | `low`          | Low         |

### Comment Visibility (`visibility`)
| Value     | Meaning                                |
|-----------|----------------------------------------|
| `public`  | Visible to client / project team       |
| `private` | Internal note — staff only             |

### Time Entry Type (`type`)
| Value          | Meaning                         |
|----------------|---------------------------------|
| `project`      | Logged directly on the project  |
| `projectTask`  | Logged on a specific step/todo  |

---

## Boards

### GET /boards
List all project boards with their stages (ordered by menuOrder).

**Response:**
```json
[
  {
    "id": 1,
    "name": "Client Projects",
    "stages": [
      { "id": 10, "title": "Backlog",     "menuOrder": 0, "boardId": 1 },
      { "id": 11, "title": "In Progress", "menuOrder": 1, "boardId": 1 },
      { "id": 12, "title": "Review",      "menuOrder": 2, "boardId": 1 },
      { "id": 13, "title": "Done",        "menuOrder": 3, "boardId": 1 }
    ]
  }
]
```

---

### GET /boards/{id}
Single board detail with stages. Response: same structure, single board object.

---

## Projects

### GET /projects
List board-eligible projects. **Default: active projects only.** Pass `status` to filter by other states.

**Query params:**
| Param     | Type   | Description |
|-----------|--------|-------------|
| `status`  | string | active (default) \| completed \| archived \| cancelled |
| `boardId` | int    | Filter by board |
| `stageId` | int    | Filter by stage |
| `orgId`   | int    | Filter by organization |
| `search`  | string | Search by project title |
| `perPage` | int    | Results per page (default: 20, max: 100) |

**Response:**
```json
{
  "data": [
    {
      "id": 42,
      "title": "Website Redesign",
      "description": null,
      "status": "active",
      "organization": { "id": 1, "name": "Acme Corp" },
      "contact": { "id": 5, "fullName": "John Doe", "email": "john@acme.com" },
      "manager": { "id": 10, "fullName": "Jane Smith", "email": "jane@company.com" },
      "board": { "id": 1, "name": "Client Projects" },
      "stage": { "id": 11, "title": "In Progress", "menuOrder": 1 },
      "priority": null,
      "startDate": "2025-04-01",
      "endDate": "2025-06-30",
      "isArchived": false,
      "completedAt": null,
      "cancelledAt": null,
      "createdAt": "2025-04-01T10:00:00+00:00",
      "sections": []
    }
  ],
  "pagination": { "total": 50, "perPage": 20, "currentPage": 1, "lastPage": 3 }
}
```

---

### POST /projects
Create a new project.

**Body:**
```json
{
  "title": "Website Redesign",
  "orgId": 1,
  "contactId": 5,
  "managerId": 10,
  "boardId": 1,
  "stageId": 11,
  "startDate": "2025-04-01",
  "endDate": "2025-06-30",
  "description": "Full redesign of the company website."
}
```
Required: `title`, `orgId`.

**Response:** `201 Created` — full ProjectResource (see GET /projects/{id})

---

### GET /projects/{id}
Full project detail with all sections → steps → sub-steps → assignees.

**Response:**
```json
{
  "id": 42,
  "title": "Website Redesign",
  "description": "Full redesign of the company website.",
  "status": "active",
  "organization": { "id": 1, "name": "Acme Corp" },
  "contact": { "id": 5, "fullName": "John Doe", "email": "john@acme.com" },
  "manager": { "id": 10, "fullName": "Jane Smith", "email": "jane@company.com" },
  "board": { "id": 1, "name": "Client Projects" },
  "stage": { "id": 11, "title": "In Progress", "menuOrder": 1 },
  "priority": null,
  "startDate": "2025-04-01",
  "endDate": "2025-06-30",
  "isArchived": false,
  "completedAt": null,
  "cancelledAt": null,
  "createdAt": "2025-04-01T10:00:00+00:00",
  "sections": [
    {
      "id": 100,
      "name": "Design Phase",
      "menuOrder": 0,
      "dueDate": "2025-04-30",
      "steps": [
        {
          "id": 200,
          "sectionId": 100,
          "parentId": null,
          "name": "Create wireframes",
          "description": "<p>Use Figma to create wireframes for all pages.</p>",
          "priority": "high",
          "isCompleted": false,
          "completedAt": null,
          "completedBy": null,
          "dueDate": "2025-04-15",
          "timeEstimate": 120,
          "timeTrackedSeconds": null,
          "clickupId": "abc123xyz",
          "menuOrder": 0,
          "assignees": [
            { "id": 10, "name": "Jane Smith", "type": "user" }
          ],
          "subSteps": [
            {
              "id": 201,
              "sectionId": 100,
              "parentId": 200,
              "name": "Homepage wireframe",
              "description": null,
              "priority": "normal",
              "isCompleted": true,
              "completedAt": "2025-04-10T14:00:00+00:00",
              "completedBy": "Jane Smith",
              "dueDate": null,
              "timeEstimate": null,
              "timeTrackedSeconds": null,
              "clickupId": null,
              "menuOrder": 0,
              "assignees": [],
              "subSteps": [],
              "createdAt": "2025-04-01T10:00:00+00:00"
            }
          ],
          "createdAt": "2025-04-01T10:00:00+00:00"
        }
      ]
    }
  ]
}
```

---

### PATCH /projects/{id}/stage
Move a project to a different stage on the board.

**Body:** `{ "stageId": 12 }`

**Response:** Updated ProjectResource (without sections)

---

### PATCH /projects/{id}/status
Change a project's status. **There is no DELETE endpoint for projects** — use archived or completed to remove from active view.

**Body:** `{ "status": "completed" }`

Valid values: `active` | `completed` | `archived` | `unarchived`

- `active` — reopens: clears completedAt and cancelledAt, sets isArchived=false
- `completed` — sets completedAt to now
- `archived` — sets isArchived=true (hides from active board)
- `unarchived` — sets isArchived=false

**Response:** Updated ProjectResource

---

### PATCH /projects/{id}/manager
Update the account manager (project manager).

**Body:** `{ "managerId": 10 }` — set to `null` to clear.

**Response:** Updated ProjectResource

---

## Sections

### GET /projects/{id}/sections
List sections with their top-level steps and one level of sub-steps. Same structure as the `sections` array in GET /projects/{id}.

---

### POST /projects/{id}/sections
Create a new section. Sections are appended at the end (highest menuOrder + 1).

**Body:** `{ "name": "Development Phase", "dueDate": "2025-05-31" }`

**Response:** `201 Created` — SectionResource

---

### PATCH /projects/{id}/sections/{sectionId}
Update a section's name or due date. All fields optional.

**Body:** `{ "name": "Updated Name", "dueDate": null }` — set `dueDate` to `null` to clear it.

**Response:** Updated SectionResource

---

### DELETE /projects/{id}/sections/{sectionId}

**⚠️ Requires confirmation if active steps exist.**

The API blocks deletion when the section has incomplete steps and returns a 422. Agent must:
1. Surface the warning to the user (e.g. "Section has 3 incomplete steps — delete anyway?")
2. Only if user confirms, resend with `force: true`

**Body (normal attempt, no force):** no body needed

**Body (after user confirms):** `{ "force": true }`

**422 response (active steps exist):**
```json
{ "success": false, "message": "This section has 3 incomplete step(s). Pass force=true to delete anyway. Confirm with the user first." }
```

**200 response (deleted):**
```json
{ "success": true, "message": "Section deleted" }
```

---

## Steps (Todos)

Steps are todos within a section. Sub-steps have `parentId` set to their parent step's `id`. **Only one level of nesting is supported** — sub-steps never have children.

### POST /projects/{id}/sections/{sectionId}/steps
Create a top-level step or a sub-step.

**Body:**
```json
{
  "name": "Create homepage wireframe",
  "description": "<p>Use Figma. Export as PNG when done.</p>",
  "priority": 2,
  "dueDate": "2025-04-15",
  "timeEstimate": 120,
  "parentId": null
}
```
- Required: `name`
- `parentId`: set to a step's `id` to create it as a sub-step under that step. Use the **same `sectionId`** as the parent step.
- `priority`: integer in request (0=none, 1=urgent, 2=high, 3=normal, 4=low)
- `timeEstimate`: minutes (integer)
- `description`: HTML supported — use `<p>` tags for paragraphs

**Response:** `201 Created` — StepResource

---

### GET /projects/{id}/steps/{stepId}
Full step detail including sub-steps, assignees, and `timeTrackedSeconds` (sum of all time logged on this step).

Note: `timeTrackedSeconds` is only populated here — it is `null` in list/section responses.

**Response:** StepResource (see GET /projects/{id} for full shape)

---

### PATCH /projects/{id}/steps/{stepId}
Update a step or sub-step. All fields optional — only sent fields are updated.

**Body:** `{ "name": "...", "description": "...", "priority": 3, "dueDate": "2025-04-20", "timeEstimate": 60 }`

Set any field to `null` to clear it (e.g. `"dueDate": null`, `"timeEstimate": null`).

**Response:** Updated StepResource

---

### DELETE /projects/{id}/steps/{stepId}
Delete a step and all its sub-steps permanently.

**Response:** `{ "success": true, "message": "Step deleted" }`

---

### PATCH /projects/{id}/steps/{stepId}/complete
Toggle step or sub-step completion.

**Body:** `{ "completed": true }` — set `false` to reopen.

When completing a parent step, sub-steps are **not** automatically completed — complete them individually if needed.

**Response:** Updated StepResource with `isCompleted` and `completedAt` updated.

---

### PATCH /projects/{id}/steps/{stepId}/move
Move a step to a different section within the same project. Sub-steps cannot be moved independently — they follow their parent.

**Body:** `{ "sectionId": 101 }`

**Response:** Updated StepResource with new `sectionId`.

---

## Step Comments

Comments support HTML content and have a `visibility` flag for public/private.

### GET /projects/{id}/steps/{stepId}/comments
List comments on a step (or sub-step), oldest first.

**Query params:** `visibility` (optional) — `public` | `private`

**Response:**
```json
[
  {
    "id": 50,
    "stepId": 200,
    "comment": "<p>Wireframes approved by client.</p>",
    "visibility": "public",
    "authorId": 10,
    "authorName": "Jane Smith",
    "clickupId": null,
    "createdAt": "2025-04-12T09:00:00+00:00"
  }
]
```

`clickupId` is non-null if the comment was synced from ClickUp.

---

### POST /projects/{id}/steps/{stepId}/comments
Add a comment to a step or sub-step.

- `visibility: "public"` (default) — visible to the client/project team
- `visibility: "private"` — internal note, staff only

**Body:**
```json
{ "comment": "<p>Wireframes approved by client.</p>", "visibility": "public" }
```

HTML is supported. Use `<p>` for paragraphs. Do not use `<h1>`–`<h6>` or inline styles.

**Response:** `201 Created` — StepCommentResource

---

### DELETE /projects/{id}/steps/{stepId}/comments/{commentId}
Delete a comment permanently.

**Response:** `{ "success": true, "message": "Comment deleted" }`

---

## Time Tracking

Time is stored in **seconds** (`timeSeconds`). Divide by 3600 for hours.

### GET /projects/{id}/time
All time entries for a project — project-level entries (`type: "project"`) and step-level entries (`type: "projectTask"`).

**Response:**
```json
[
  {
    "id": 300,
    "type": "projectTask",
    "stepId": 200,
    "userId": 10,
    "userName": "Jane Smith",
    "timeSeconds": 3600,
    "notes": "Completed the homepage wireframe",
    "isBillable": true,
    "isManual": false,
    "status": "completed",
    "forDate": "2025-04-10",
    "createdAt": "2025-04-10T17:00:00+00:00"
  }
]
```

`status: "started"` means the timer is still running. `status: "completed"` means it was stopped.

---

### GET /projects/{id}/steps/{stepId}/time
Time entries on a specific step or sub-step only.

**Response:** Same format. All entries will have `type: "projectTask"` and `stepId` set.

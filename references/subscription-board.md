# Subscription Board Reference

> **Naming note:** `/subscription-board` refers to the operational marketing delivery boards — checklists, setup tasks, and recurring delivery tracking. It is **NOT** related to billing subscriptions (`OrganizationSubscription`), which are not exposed via this API.

All endpoints require:
```
Authorization: Bearer {GRANDCENTRAL_API_KEY}
Content-Type: application/json
Base URL: {GRANDCENTRAL_API_URL}/api/gc/v1
```

---

## Data Model Overview

```
SubscriptionBoard (read-only)
└── SubscriptionBoardItem (a client's active subscription/service)
    └── SubscriptionBoardItemChecklist (one per service deliverable)
        ├── ItemSetupChecklist (one-time setup work)
        │   └── ItemSetupChecklistSection
        │       └── ItemSetupChecklistStep (completable, commentable)
        │           └── subSteps (one level only)
        └── ItemRecurringChecklist (repeating monthly/weekly delivery)
            └── ItemRecurringChecklistSection
                └── ItemRecurringChecklistStep (completable, commentable)
```

---

## Enum Reference

### Item Status (`status`)
| Value       | Meaning                       |
|-------------|-------------------------------|
| `Pending`   | Not yet started               |
| `Active`    | In progress                   |
| `Overdue`   | Past due date                 |
| `Delivered` | Completed and delivered       |

### Step Priority (`priority`)
| Value    | Meaning     |
|----------|-------------|
| `none`   | No priority |
| `urgent` | Urgent      |
| `high`   | High        |
| `normal` | Normal      |
| `low`    | Low         |

### Recurring Type (`recurringType`)
Values: `Monthly` | `Weekly` | `Quarterly` | `Bi-Yearly` | `Yearly`

### Feedback (`feedback`)
| Value     | Meaning                              |
|-----------|--------------------------------------|
| `happy`   | Client satisfied                     |
| `sad`     | Client not satisfied                 |
| `neutral` | Client neutral / no strong opinion   |
| `null`    | Feedback not yet submitted           |

> **Important:** A recurring checklist instance's `isCompleted` will remain `false` even when all steps are done, until feedback is submitted via the feedback endpoint.

---

## Boards

### GET /subscription-board/boards
List all subscription boards (global to the tenant).

**Response:**
```json
{
  "data": [
    { "id": 1, "name": "SEO Delivery Board" },
    { "id": 2, "name": "PPC Management Board" }
  ]
}
```

---

### GET /subscription-board/boards/{id}
Show a board with the count of items belonging to the current company.

**Response:**
```json
{
  "id": 1,
  "name": "SEO Delivery Board",
  "itemCount": 42
}
```

---

## Items

### GET /subscription-board/items
List subscription board items (client subscriptions) for the current company.

**Query params:**
| Param        | Type    | Description                                          |
|--------------|---------|------------------------------------------------------|
| `boardId`    | int     | Filter by board                                      |
| `status`     | string  | Pending \| Active \| Overdue \| Delivered            |
| `overdue`    | bool    | Shortcut: filter to Overdue items only               |
| `assignedTo` | int     | Filter by assigned user ID (from GET /users)         |
| `search`     | string  | Search item name (partial, case-insensitive)         |
| `perPage`    | int     | Results per page (default: 20, max: 100)             |

**Response:**
```json
{
  "data": [
    {
      "id": 101,
      "name": "Acme Corp — Monthly SEO",
      "status": "Active",
      "progress": 65.5,
      "dueDate": "2025-04-30",
      "startDate": "2025-04-01",
      "organization": { "id": 50, "name": "Acme Corp" },
      "contact": { "id": 12, "fullName": "John Doe", "email": "john@acme.com" },
      "assignee": { "id": 10, "fullName": "Jane Smith" },
      "board": { "id": 1, "name": "SEO Delivery Board" },
      "checklistCount": 3,
      "createdAt": "2025-01-15T10:00:00+00:00",
      "checklists": []
    }
  ],
  "pagination": {
    "total": 80,
    "perPage": 20,
    "currentPage": 1,
    "lastPage": 4
  }
}
```

> Note: `checklists` is only populated on the item detail endpoint (GET /items/{id}). In the list view it is an empty array.

---

### GET /subscription-board/items/{id}
Show a single board item with all checklist summaries.

**Response:**
```json
{
  "id": 101,
  "name": "Acme Corp — Monthly SEO",
  "status": "Active",
  "progress": 65.5,
  "dueDate": "2025-04-30",
  "startDate": "2025-04-01",
  "organization": { "id": 50, "name": "Acme Corp" },
  "contact": { "id": 12, "fullName": "John Doe", "email": "john@acme.com" },
  "assignee": { "id": 10, "fullName": "Jane Smith" },
  "board": { "id": 1, "name": "SEO Delivery Board" },
  "checklistCount": 2,
  "createdAt": "2025-01-15T10:00:00+00:00",
  "checklists": [
    {
      "id": 201,
      "name": "On-Page SEO Setup",
      "status": "Active",
      "progress": 80.0,
      "recurringType": "Monthly",
      "dueDate": "2025-04-30",
      "startDate": "2025-04-01",
      "setupProgress": 100.0,
      "setupIsCompleted": true,
      "currentRecurringDueDate": "2025-04-30",
      "currentRecurringIsCompleted": false
    }
  ]
}
```

---

## Checklists

### GET /subscription-board/items/{item}/checklists/{checklist}/setup
Get the full setup checklist with all sections, steps, sub-steps, and assignees.

**Response:**
```json
{
  "id": 301,
  "title": "Initial Setup",
  "progress": 75.0,
  "status": "Active",
  "isCompleted": false,
  "sections": [
    {
      "id": 401,
      "title": "Technical SEO",
      "menuOrder": 1,
      "steps": [
        {
          "id": 501,
          "title": "Set up Google Search Console",
          "description": "Verify ownership and submit sitemap.",
          "link": "https://search.google.com/search-console",
          "isCompleted": true,
          "completedAt": "2025-04-02T09:00:00+00:00",
          "completedBy": "Jane Smith",
          "priority": "high",
          "dueDate": "2025-04-05",
          "menuOrder": 1,
          "parentId": null,
          "assignees": [{ "id": 10, "name": "Jane Smith" }],
          "subSteps": [],
          "createdAt": "2025-01-15T10:00:00+00:00"
        }
      ]
    }
  ]
}
```

---

### GET /subscription-board/items/{item}/checklists/{checklist}/recurring
List all recurring instances for a checklist (newest first by due date). Returns summary without steps.

**Response:**
```json
{
  "data": [
    {
      "id": 601,
      "title": "April 2025 Delivery",
      "progress": 50.0,
      "status": "Active",
      "isCompleted": false,
      "dueDate": "2025-04-30",
      "feedback": null,
      "feedbackComments": null,
      "feedbackAt": null,
      "createdAt": "2025-04-01T00:00:00+00:00"
    }
  ]
}
```

---

### GET /subscription-board/items/{item}/checklists/{checklist}/recurring/{recurring}
Show a single recurring instance with full sections and steps.

**Response:** Same shape as above, but includes a `sections` array (same structure as setup checklist sections/steps, without `subSteps`).

---

### PATCH /subscription-board/items/{item}/checklists/{checklist}/recurring/{recurring}/feedback
Submit client feedback on a recurring checklist instance.

> **Required before `isCompleted` can be true.** Call this after all steps are marked complete to finalize the delivery.

**Body:**
```json
{
  "feedback": "happy",
  "comments": "Client approved all deliverables."
}
```
`feedback` is required: `happy` | `sad` | `neutral`. `comments` is optional (max 2000 chars).

**Response:** Updated recurring checklist resource.

---

## Setup Steps

### PATCH /subscription-board/setup-steps/{step}/complete
Toggle a setup step's completion status (complete ↔ incomplete).

Automatically cascades progress recalculation up:
`step → section → setup checklist → SubscriptionBoardItemChecklist → SubscriptionBoardItem`

**No request body required.**

**Response:**
```json
{
  "id": 501,
  "title": "Set up Google Search Console",
  "description": "Verify ownership and submit sitemap.",
  "link": "https://search.google.com/search-console",
  "isCompleted": true,
  "completedAt": "2025-04-08T10:00:00+00:00",
  "completedBy": "Jane Smith",
  "priority": "high",
  "dueDate": "2025-04-05",
  "menuOrder": 1,
  "parentId": null,
  "assignees": [{ "id": 10, "name": "Jane Smith" }],
  "subSteps": [],
  "createdAt": "2025-01-15T10:00:00+00:00"
}
```

---

### GET /subscription-board/setup-steps/{step}/comments
List all comments on a setup step, oldest first.

**Response:**
```json
{
  "data": [
    {
      "id": 99,
      "stepId": 501,
      "comment": "Completed verification with client.",
      "authorId": 10,
      "authorName": "Jane Smith",
      "createdAt": "2025-04-08T11:00:00+00:00"
    }
  ]
}
```

---

### POST /subscription-board/setup-steps/{step}/comments
Add a comment to a setup step.

**Body:**
```json
{ "comment": "Completed verification with client." }
```

**Response:** `201 Created` — StepCommentResource

---

## Recurring Steps

### PATCH /subscription-board/recurring-steps/{step}/complete
Toggle a recurring step's completion status (complete ↔ incomplete).

Same cascade behavior as setup steps. Note: even after all recurring steps are complete, `isCompleted` on the recurring checklist instance remains `false` until feedback is submitted.

**No request body required.**

**Response:** Same shape as setup step response, without `subSteps`.

---

### GET /subscription-board/recurring-steps/{step}/comments
List all comments on a recurring step, oldest first.

**Response:** Same shape as setup step comments.

---

### POST /subscription-board/recurring-steps/{step}/comments
Add a comment to a recurring step.

**Body:**
```json
{ "comment": "Links confirmed with client." }
```

**Response:** `201 Created` — StepCommentResource

---

## Agent Guidance

- **To find overdue items:** `GET /subscription-board/items?overdue=true`
- **To complete a delivery cycle:** Mark all recurring steps complete via PATCH, then submit feedback via the feedback endpoint
- **No create/delete endpoints:** Boards, items, and checklists are managed in the GrandCentral UI — the agent API is for reporting and task completion only
- **Step IDs are global:** Setup step and recurring step IDs are unique across all checklists — you don't need to know which item or checklist they belong to when toggling completion
- **Progress is pre-computed:** The `progress` float on items, checklists, and instances reflects the current state — no need to calculate it yourself

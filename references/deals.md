# Deals & Activities Reference

All endpoints require:
```
Authorization: Bearer {GRANDCENTRAL_API_KEY}
Content-Type: application/json
Base URL: {GRANDCENTRAL_API_URL}/api/gc/v1
```

---

## Agent Guidance

### Data Model Overview

```
DealBoard
  └── DealStage (ordered by menuOrder, left = early, right = late in pipeline)
        └── Deal (status: in_progress | won | lost)
              ├── Organization
              ├── Contact (primary contact for this deal)
              ├── User (owner — the sales rep)
              └── Proposal[] (linked via JSON field, summary only)

ContactAction (object_type='deal', object_id=deal.id)
  └── ContactConversation (message body, plain text)
```

### Working with Deals

- **Default status is `in_progress`** — always pass `status` filter explicitly if you want won/lost deals.
- **Board stages flow left-to-right** by `menuOrder`. A deal moves forward by calling PATCH /deals/{id}/stage.
- **`daysInStage`** — how many days since the deal entered the current stage. Use this to flag stalled deals.
- **`dealAgeDays`** — total days since the deal was opened/created.
- **There is no DELETE endpoint.** To remove a deal from the active pipeline, set status to `won` or `lost`.
- **`no_value`** — when true, the deal's value is intentionally zero (e.g. a non-monetary opportunity). Don't flag this as a missing value.

### Working with Proposals

- Proposals are shown as a summary only on `GET /deals/{id}`. Do not try to load full proposal content from this endpoint.
- `isSigned: true` means the proposal has a digital signature.
- `isDeposited: true` means a deposit has been collected.
- For full proposal management, a dedicated Proposals module will be added separately.

### Working with Activities

- **Email activities are read-only.** They are synced from Gmail and cannot be created via this API.
- **Supported create types:** `note`, `call`, `sms`
- `notesResult` is a human-readable label shown in the activity feed (e.g. "Phone Call", "Left Voicemail", "Sent Proposal"). Defaults to the capitalized `activityType` if not provided.
- `forDate` is when the activity happened (not when you're logging it). Use ISO 8601 datetime.
- `direction` applies to calls and SMS: `incoming` = customer called us, `outgoing` = we called them.
- Activity `body` is always plain text. If the original content was HTML (email), it is stripped and newlines are preserved.
- For deal activities: `contactId` is optional (defaults to the deal's primary contact).
- For org activities: `contactId` is required — the org may have multiple contacts.

### Proposal Status Values

| Value      | Meaning                       |
|------------|-------------------------------|
| `draft`    | Not yet sent to customer      |
| `sent`     | Sent, not yet viewed          |
| `viewed`   | Customer has opened it        |
| `accepted` | Customer accepted             |
| `archived` | Removed from active pipeline  |

---

## Enum Reference

### Deal Status
| Value         | Meaning                        |
|---------------|--------------------------------|
| `in_progress` | Active deal in the pipeline    |
| `won`         | Deal closed as won             |
| `lost`        | Deal closed as lost            |

### Deal Interval
| Value      | Meaning         |
|------------|-----------------|
| `one time` | One-time deal   |
| `monthly`  | Monthly recurring |
| `quarterly`| Quarterly recurring |
| `yearly`   | Annual recurring  |

### Activity Type
| Value   | Meaning                  |
|---------|--------------------------|
| `note`  | Text note or memo        |
| `call`  | Phone call log           |
| `sms`   | SMS/text message log     |
| `email` | Email (read-only, synced from Gmail) |

### Organization Status
| Int | String            |
|-----|-------------------|
| 1   | `customer`        |
| 2   | `lead`            |
| 3   | `past_customer`   |
| 4   | `do_not_contact`  |
| 5   | `spam`            |
| 6   | `do_not_sell`     |
| 7   | `not_interested`  |
| 8   | `out_of_business` |

---

## Deal Boards

### GET /deals/boards
List all deal pipeline boards with their ordered stages.

**Response:**
```json
[
  {
    "id": 1,
    "name": "Main Sales Pipeline",
    "settings": { "dangerDays": 14, "warningDays": 7, "successDays": 3 },
    "stages": [
      { "id": 10, "title": "Prospect",      "menuOrder": 0, "expectedDays": 7 },
      { "id": 11, "title": "Qualified",     "menuOrder": 1, "expectedDays": 14 },
      { "id": 12, "title": "Proposal Sent", "menuOrder": 2, "expectedDays": 10 },
      { "id": 13, "title": "Negotiation",   "menuOrder": 3, "expectedDays": 7 }
    ]
  }
]
```

---

### GET /deals/boards/{id}
Get a single board with stages. Same response shape, single object.

---

## Deals

### GET /deals
List deals with optional filters. Default: `in_progress`.

**Query params:**
| Param     | Type   | Description |
|-----------|--------|-------------|
| `status`  | string | in_progress \| won \| lost (default: in_progress) |
| `boardId` | int    | Filter by board |
| `stageId` | int    | Filter by stage |
| `search`  | string | Search deal name OR org name |
| `ownerId` | int    | Filter by assigned sales rep |
| `perPage` | int    | Default: 20 |

**Response:**
```json
{
  "data": [
    {
      "id": 42,
      "name": "Acme Corp Website Redesign",
      "value": 15000.00,
      "noValue": false,
      "interval": "one time",
      "status": "in_progress",
      "statusAt": null,
      "lostReason": null,
      "lostExplain": null,
      "expectedClosed": "2025-12-31",
      "daysInStage": 5,
      "dealAgeDays": 12,
      "lastActivity": "2025-04-05T14:30:00+00:00",
      "board": { "id": 1, "name": "Main Sales Pipeline" },
      "stage": { "id": 11, "title": "Qualified", "menuOrder": 1 },
      "owner": { "id": 5, "name": "Jane Smith", "email": "jane@company.com" },
      "organization": { "id": 50, "name": "Acme Corp" },
      "contact": { "id": 20, "name": "John Doe" },
      "proposals": null,
      "createdAt": "2025-03-27T09:00:00+00:00"
    }
  ],
  "pagination": { "total": 45, "perPage": 20, "currentPage": 1, "lastPage": 3 }
}
```

---

### POST /deals
Create a new deal.

**Body:**
```json
{
  "name": "Acme Corp Website Redesign",
  "orgId": 50,
  "contactId": 20,
  "boardId": 1,
  "stageId": 10,
  "ownerId": 5,
  "value": 15000,
  "interval": "one time",
  "expectedClosed": "2025-12-31"
}
```
Required: `name`, `orgId`.

**Response:** `201 Created` — DealResource

---

### GET /deals/{id}
Get full deal detail including proposal summaries.

`proposals` array is populated here (it's `null` in list responses).

**Response:**
```json
{
  "id": 42,
  "name": "Acme Corp Website Redesign",
  "value": 15000.00,
  "status": "in_progress",
  "daysInStage": 5,
  "dealAgeDays": 12,
  "proposals": [
    {
      "id": 7,
      "title": "Website Redesign Proposal v2",
      "status": "sent",
      "amount": 15000.00,
      "deposit": 3000.00,
      "isDeposited": false,
      "isSigned": false,
      "signedAt": null,
      "viewedAt": "2025-04-02T16:00:00+00:00",
      "createdAt": "2025-04-01T10:00:00+00:00"
    }
  ]
}
```

---

### PATCH /deals/{id}/stage
Move a deal to a different stage.

**Body:** `{ "stageId": 12 }`

Use stage IDs from GET /deals/boards.

**Response:** Updated DealResource

---

### PATCH /deals/{id}/status
Change deal status.

**Body:** `{ "status": "won" }`

For lost deals:
```json
{
  "status": "lost",
  "lostReason": "Price too high",
  "lostExplain": "Customer chose a cheaper competitor."
}
```

**Response:** Updated DealResource

---

### PATCH /deals/{id}/value
**Body:** `{ "value": 18000, "noValue": false }`
**Response:** Updated DealResource

---

### PATCH /deals/{id}/owner
**Body:** `{ "ownerId": 5 }` — pass `null` to unassign
**Response:** Updated DealResource

---

### PATCH /deals/{id}/close-date
**Body:** `{ "closeDate": "2025-12-31" }` — pass `null` to clear
**Response:** Updated DealResource

---

## Activities (Deals)

### GET /deals/{id}/activities
List all activities on a deal, newest first. `perPage` default: 20.

**Response:**
```json
{
  "data": [
    {
      "id": 101,
      "activityType": "call",
      "notesResult": "Phone Call - Left Voicemail",
      "direction": "outgoing",
      "body": "Called John about the proposal. Left voicemail to follow up.",
      "forDate": "2025-04-05T14:00:00+00:00",
      "author": { "id": 5, "name": "Jane Smith" },
      "contact": null,
      "isPinned": false,
      "createdAt": "2025-04-05T14:30:00+00:00"
    },
    {
      "id": 99,
      "activityType": "email",
      "notesResult": null,
      "direction": "outgoing",
      "body": "Hi John,\n\nPlease find the proposal attached...",
      "forDate": "2025-04-03T10:00:00+00:00",
      "author": null,
      "contact": null,
      "isPinned": false,
      "createdAt": "2025-04-03T10:00:00+00:00"
    }
  ],
  "pagination": { "total": 8, "perPage": 20, "currentPage": 1, "lastPage": 1 }
}
```

Note: `email` activities are read-only (Gmail-synced).

---

### POST /deals/{id}/activities
Create a note, call, or SMS on a deal.

**Body:**
```json
{
  "activityType": "note",
  "body": "Customer confirmed budget is approved.",
  "notesResult": "Meeting Notes",
  "forDate": "2025-04-05T14:00:00+00:00",
  "direction": "outgoing"
}
```
Required: `activityType`, `body`.

**Response:** `201 Created` — ActivityResource

---

## Activities (Projects)

### GET /projects/{id}/activities
Same format as deal activities.

### POST /projects/{id}/activities
Same format. Additional optional `contactId` field.

---

## Organizations

### GET /organizations
List/search organizations. Use to find `orgId` before creating deals.

**Query params:** `search`, `status` (int 1-8), `ownerId`, `perPage` (default: 20)

**Response:**
```json
{
  "data": [
    {
      "id": 50,
      "name": "Acme Corp",
      "orgStatus": { "id": 1, "name": "Customer" },
      "lastActivity": "2025-04-05T14:30:00+00:00"
    }
  ],
  "pagination": { ... }
}
```

Note: `orgStatus` is returned as an object `{id, name}` from a company-defined status list (not a fixed enum).

---

### GET /organizations/{id}
Get a single organization.

---

### GET /organizations/{id}/activities
List activities across all contacts of the organization, newest first.

Same response format as deal activities. `perPage` default: 20.

---

### POST /organizations/{id}/activities
Create a note/call/SMS on an organization. `contactId` is **required**.

**Body:**
```json
{
  "contactId": 20,
  "activityType": "call",
  "body": "Initial discovery call with the founder.",
  "notesResult": "Discovery Call",
  "forDate": "2025-04-05T10:00:00+00:00",
  "direction": "outgoing"
}
```

**Response:** `201 Created` — ActivityResource

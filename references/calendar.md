# Calendar Events Reference

All endpoints require:
```
Authorization: Bearer {GRANDCENTRAL_API_KEY}
Content-Type: application/json
Base URL: https://api-v2.grandcentr.al/api
```

Calendar events are company-scoped and support internal users and external email guests as participants. All datetimes are expressed in the **company's configured timezone** (typically `America/New_York`).

---

## Datetime Format

**Input (what you send):** ISO 8601 string with an explicit timezone offset.
- ✅ `"2026-05-11T14:00:00-04:00"` — Eastern Daylight Time
- ✅ `"2026-05-11T18:00:00Z"` — UTC
- ❌ `1746000000` — Unix timestamps are rejected
- ❌ `"2026-05-11 14:00:00"` — naive strings without timezone are rejected

**Output (what you receive):** ISO 8601 string in company timezone, plus a `timezone` field on every event so you always know which timezone is in use.

---

## Event Response Shape

```json
{
  "id": 101,
  "name": "Kickoff Call",
  "description": "Discuss project scope.",
  "location": "Zoom",
  "startTime": "2026-05-11T14:00:00-04:00",
  "endTime": "2026-05-11T15:00:00-04:00",
  "timezone": "America/New_York",
  "status": "accepted",
  "color": "#3b82f6",
  "isSynced": false,
  "objectId": 55,
  "objectType": "deal",
  "orgId": 7,
  "createdBy": { "id": 5, "fullName": "Jane Smith" },
  "participants": [
    {
      "id": 12,
      "userId": 5,
      "email": "jane@company.com",
      "userType": "user",
      "fullName": "Jane Smith",
      "acceptanceStatus": "accepted"
    },
    {
      "id": 13,
      "userId": null,
      "email": "client@acme.com",
      "userType": "client",
      "fullName": null,
      "acceptanceStatus": "pending"
    }
  ]
}
```

> `timezone` tells you which timezone `startTime` and `endTime` are in. Always read this field — do not assume a timezone.

---

## Endpoints

### GET /calendar-events
List calendar events for the company with optional filters.

**Query params:**
| Param        | Type     | Description |
|--------------|----------|-------------|
| `startTime`  | string   | ISO 8601 with offset — lower bound (inclusive) |
| `endTime`    | string   | ISO 8601 with offset — upper bound (inclusive) |
| `objectId`   | int      | Filter by linked object ID |
| `objectType` | string   | `deal` \| `project` \| `timeMaterial` \| `organization` |
| `orgId`      | int      | Filter by organization ID |
| `userIds[]`  | int[]    | Filter to events where any of these users are assignees |

**Response:** `200` — array of event objects

---

### GET /calendar-events/free-busy
Check which time slots are already booked for one or more users. **Call this before creating an event** to avoid scheduling conflicts.

**Query params:**
| Param       | Type   | Required | Description |
|-------------|--------|----------|-------------|
| `startTime` | string | yes      | ISO 8601 with offset — start of window |
| `endTime`   | string | yes      | ISO 8601 with offset — end of window (must be after startTime) |
| `userIds[]` | int[]  | no       | Limit to specific user IDs. Omit to check all company users. |

**Response: `200`**
```json
{
  "timezone": "America/New_York",
  "startTime": "2026-05-11T00:00:00-04:00",
  "endTime": "2026-05-11T23:59:59-04:00",
  "busy": [
    {
      "eventId": 101,
      "name": "Team Standup",
      "startTime": "2026-05-11T09:00:00-04:00",
      "endTime": "2026-05-11T09:30:00-04:00",
      "participants": [
        { "userId": 5, "fullName": "Jane Smith" }
      ]
    }
  ]
}
```

An empty `busy` array means the window is fully free for the requested users.

---

### POST /calendar-events
Create a new calendar event.

**Body:**
```json
{
  "name": "Kickoff Call",
  "startTime": "2026-05-11T14:00:00-04:00",
  "endTime": "2026-05-11T15:00:00-04:00",
  "description": "Discuss project scope.",
  "location": "Zoom",
  "orgId": 7,
  "objectId": 55,
  "objectType": "deal",
  "color": "#3b82f6",
  "participants": [
    { "userId": 5 },
    { "email": "client@acme.com" }
  ],
  "force": false
}
```

| Field          | Required | Description |
|----------------|----------|-------------|
| `name`         | yes      | Event title |
| `startTime`    | yes      | ISO 8601 with timezone offset |
| `endTime`      | yes      | ISO 8601 with timezone offset, must be >= startTime |
| `description`  | no       | Free-text description |
| `location`     | no       | Location or meeting link |
| `orgId`        | no       | Link to an organization |
| `objectId`     | no       | Link to a related object (deal, project, etc.) |
| `objectType`   | no       | `deal` \| `project` \| `timeMaterial` \| `organization` |
| `color`        | no       | Hex color string |
| `participants` | no       | Array of `{ "userId": int }` or `{ "email": "string" }`. Creator is always included. |
| `force`        | no       | Pass `true` to create despite scheduling conflicts (see Conflict Detection below) |

**Response:** `201 Created` — event object

**Conflict response:** `409` — see Conflict Detection section below.

---

### GET /calendar-events/{id}
Get a single event with all participants.

**Response:** `200` — event object

---

### PATCH /calendar-events/{id}
Update a calendar event. Only fields present in the body are updated, **except `participants`** — if provided, it fully replaces the assignee list.

**Body (all fields optional except name, startTime, endTime which are required):**
```json
{
  "name": "Updated Call",
  "startTime": "2026-05-11T15:00:00-04:00",
  "endTime": "2026-05-11T16:00:00-04:00",
  "description": "Updated notes.",
  "location": "Google Meet",
  "color": "#10b981",
  "participants": [
    { "userId": 5 },
    { "userId": 8 }
  ],
  "force": false
}
```

Omitting `participants` leaves existing assignees untouched. Passing an empty array `[]` removes all assignees except the creator.

**Response:** `200` — updated event object

**Conflict response:** `409` — see Conflict Detection section below.

---

### DELETE /calendar-events/{id}
Delete a calendar event.

**Response:** `200`
```json
{ "success": true }
```

---

### PATCH /calendar-events/{id}/status
Update only the event's status field.

**Body:**
```json
{ "status": "rescheduled" }
```

Status values: `pending` | `accepted` | `declined` | `rescheduled` | `out_of_office`

**Response:** `200` — updated event object

---

### GET /calendar-events/{id}/participants
List all participants (assignees) for an event.

**Response:** `200` — array of participant objects

---

### PATCH /calendar-events/{id}/participants/{participantId}/status
Update a participant's RSVP acceptance status. Notifies the event organizer by email. If all client-type participants share the same status, the event's own status is updated automatically.

**Body:**
```json
{ "status": "accepted" }
```

RSVP values: `accepted` | `declined` | `maybe`

**Response:** `200` — updated event object

---

## Conflict Detection

When creating or updating an event, the API checks whether any **internal user participants** are already booked during the requested time window. External email guests are not checked.

### 409 Conflict Response
```json
{
  "success": false,
  "message": "One or more participants are already booked during this time.",
  "conflicts": [
    {
      "eventId": 88,
      "name": "Team Standup",
      "startTime": "2026-05-11T14:00:00-04:00",
      "endTime": "2026-05-11T14:30:00-04:00",
      "conflictingParticipants": [
        { "userId": 5, "fullName": "Jane Smith" }
      ]
    }
  ],
  "hint": "To create this event despite the conflict, pass \"force\": true in your request."
}
```

### Handling Conflicts

**If the user wants to see the conflicts:** present the `conflicts` array and ask whether to proceed anyway.

**If the user explicitly wants to override:** re-send the same request with `"force": true` added to the body.

**If you are autonomously scheduling:** always check free-busy first (`GET /calendar-events/free-busy`) to avoid hitting the 409. Only use `force: true` if the user has explicitly asked you to create the event despite knowing about the conflict.

---

## Agent Guidance

### Check availability before booking
Always call `GET /calendar-events/free-busy` before creating an event for a user:
```
GET /calendar-events/free-busy?startTime=2026-05-11T00:00:00-04:00&endTime=2026-05-11T23:59:59-04:00&userIds[]=5
```
An empty `busy` array = the window is free.

### Book an event
1. Check free-busy for the requested window + participants
2. If no conflicts, `POST /calendar-events` with the details
3. If there's a conflict, show the user what's already booked and ask whether to proceed

### Find a user's events for a day
```
GET /calendar-events?startTime=2026-05-11T00:00:00-04:00&endTime=2026-05-11T23:59:59-04:00&userIds[]=5
```

### Link an event to a deal or project
Include `objectId` and `objectType` in the create/update body:
```json
{ "objectId": 55, "objectType": "deal" }
```

### Update RSVP for a participant
1. `GET /calendar-events/{id}/participants` — find the participant's `id`
2. `PATCH /calendar-events/{id}/participants/{participantId}/status` with `{ "status": "accepted" }`

### Force-create despite conflict
Only do this when the user has explicitly acknowledged the conflict:
```json
{ "name": "...", "startTime": "...", "endTime": "...", "force": true }
```

### Datetime rules
- Always include a timezone offset in the datetime string: `-04:00`, `-05:00`, or `Z`
- Use the `timezone` field from any event response to know which timezone the company is in
- Never send Unix timestamps — they will be rejected with a 422

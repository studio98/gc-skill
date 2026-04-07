# GrandCentral Agent API Reference

Base URL: `{GRANDCENTRAL_API_URL}` (e.g. `https://api-v2.yourdomain.com/api`)

All requests require:
```
Authorization: Bearer {GRANDCENTRAL_API_KEY}
Content-Type: application/json
```

## Modules

| Module                | Base Path           | Reference File                     |
|-----------------------|---------------------|------------------------------------|
| Users (staff, r/o)    | /users              | references/tickets.md              |
| Support Tickets       | /tickets            | references/tickets.md              |
| Support Tasks         | /tickets/{id}/tasks | references/tickets.md              |
| Organizations         | /organizations      | references/organizations.md        |
| Contacts              | /contacts           | references/organizations.md        |
| Deals / Sales         | /deals              | references/deals.md _(coming soon)_ |

## Authentication

API keys are created in GrandCentral → Settings → API Keys.
Keys are prefixed `gc_` and are scoped to a specific company within a tenant.

Pass the key as a Bearer token:
```
Authorization: Bearer gc_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

## Response Format

All responses are JSON. List endpoints return arrays directly or paginated objects:
```json
{
  "data": [...],
  "pagination": { "total": 100, "perPage": 20, "currentPage": 1, "lastPage": 5 }
}
```

Error responses (HTTP 4xx):
```json
{ "message": "Descriptive error message" }
```

Success responses for non-resource actions (e.g. delete):
```json
{ "success": true, "message": "Task deleted" }
```

## Timestamps

All timestamps are ISO 8601 format: `"2025-04-01T10:00:00+00:00"`

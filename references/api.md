# GrandCentral Agent API Reference

Base URL: `{GRANDCENTRAL_API_URL}` (e.g. `https://yourdomain.com/api/gc/v1`)

All requests require:
```
Authorization: Bearer {GRANDCENTRAL_API_KEY}
Content-Type: application/json
```

## Modules

| Module                     | Base Path          | Reference File              |
|----------------------------|--------------------|-----------------------------|
| Support Tickets            | /tickets           | references/tickets.md       |
| Users (read-only)          | /users             | references/tickets.md       |
| Projects                   | /projects          | references/projects.md      |
| Deals & Activities         | /deals             | references/deals.md         |
| Organizations & Activities | /organizations     | references/deals.md         |
| Proposals                  | /proposals         | references/proposals.md _(coming soon)_ |

## Authentication

API keys are created in GrandCentral → Settings → API Keys.
Keys are prefixed `gc_` and scoped to a specific company within a tenant.

## Response Format

All responses are JSON. List endpoints return arrays directly or paginated objects:
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

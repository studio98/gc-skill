# Organizations & Contacts Reference

All endpoints require:
```
Authorization: Bearer {GRANDCENTRAL_API_KEY}
Content-Type: application/json
Base URL: {GRANDCENTRAL_API_URL}  (e.g. https://api-v2.yourdomain.com/api)
```

---

## Statuses (Reference Data)

### GET /organizations/statuses
List all organization status labels (tenant-defined, dynamic — not a fixed enum).

**Response:**
```json
[
  { "id": 1, "name": "Customer" },
  { "id": 2, "name": "Lead" },
  { "id": 3, "name": "Past Customer" }
]
```

---

### GET /contacts/statuses
List all contact status labels.

**Response:**
```json
[
  { "id": 1, "name": "Active" },
  { "id": 2, "name": "Inactive" }
]
```

---

## Organizations

### GET /organizations
Paginated list of organizations.

**Query params:**
| Param     | Type   | Description |
|-----------|--------|-------------|
| `search`  | string | Filter by name (partial, case-insensitive) |
| `status`  | int    | Filter by org status ID |
| `tags`    | string | Comma-separated tag names — returns orgs with ANY of those tags |
| `perPage` | int    | Results per page (default: 20, max: 100) |

**Response:**
```json
{
  "data": [
    {
      "id": 1,
      "name": "Acme Corp",
      "status": { "id": 1, "name": "Customer" },
      "isVip": false,
      "currency": "USD",
      "website": "https://acme.com",
      "owner": { "id": 10, "fullName": "Jane Smith" },
      "tags": [{ "id": 5, "name": "enterprise" }],
      "lastActivity": "2025-04-01T08:00:00-04:00",
      "createdAt": "2024-01-15T10:00:00-05:00"
    }
  ],
  "pagination": { "total": 50, "perPage": 20, "currentPage": 1, "lastPage": 3 }
}
```

---

### GET /organizations/{id}
Full organization detail including contacts, active subscriptions, and custom fields.

**Query params (all default true):**
| Param                  | Type | Description |
|------------------------|------|-------------|
| `includeContacts`      | bool | Load linked contacts (default: true) |
| `includeSubscriptions` | bool | Load active subscriptions (default: true) |
| `includeCustomFields`  | bool | Load custom field values (default: true) |

**Response:**
```json
{
  "id": 1,
  "name": "Acme Corp",
  "status": { "id": 1, "name": "Customer" },
  "isVip": false,
  "currency": "USD",
  "website": "https://acme.com",
  "owner": { "id": 10, "fullName": "Jane Smith" },
  "accountManager": { "id": 11, "fullName": "Bob Jones" },
  "tags": [{ "id": 5, "name": "enterprise" }],
  "contacts": [
    {
      "id": 5,
      "firstName": "John",
      "lastName": "Doe",
      "fullName": "John Doe",
      "title": "CEO",
      "status": { "id": 1, "name": "Active" },
      "primaryEmail": "john@acme.com",
      "primaryPhone": "+15551234567",
      "tags": [{ "id": 8, "name": "decision-maker" }],
      "createdAt": "2024-01-15T10:00:00-05:00"
    }
  ],
  "subscriptions": [
    {
      "id": 12,
      "title": "Growth Plan",
      "amount": 499.00,
      "cycle": "monthly",
      "status": "active",
      "contactEmail": "billing@acme.com",
      "startDate": "2024-01-01",
      "nextBilling": "2025-05-01",
      "cancelledAt": null,
      "createdAt": "2024-01-01T00:00:00-05:00"
    }
  ],
  "customFields": [
    { "fieldId": 3, "label": "Industry", "type": "dropdown", "value": "Technology" },
    { "fieldId": 7, "label": "Account Tier", "type": "singleOption", "value": "Gold" },
    { "fieldId": 9, "label": "Notes", "type": "text", "value": "Referred by Jane" }
  ],
  "lastActivity": "2025-04-01T08:00:00-04:00",
  "createdAt": "2024-01-15T10:00:00-05:00"
}
```

Subscription `cycle`: `monthly` | `quarterly` | `bi-annually` | `yearly`
Subscription `status`: `active` (unsub_at is null) | `cancelled`

---

## Contacts

### GET /contacts
Search all contacts company-wide.

**Query params:**
| Param     | Type   | Description |
|-----------|--------|-------------|
| `search`  | string | Search by name, email, or phone number (partial match) |
| `orgId`   | int    | Filter to a specific organization |
| `status`  | int    | Filter by contact status ID |
| `tags`    | string | Comma-separated tag names — ANY match |
| `perPage` | int    | Results per page (default: 20, max: 100) |

**Response:**
```json
{
  "data": [
    {
      "id": 5,
      "firstName": "John",
      "lastName": "Doe",
      "fullName": "John Doe",
      "title": "CEO",
      "status": { "id": 1, "name": "Active" },
      "organization": { "id": 1, "name": "Acme Corp" },
      "primaryEmail": "john@acme.com",
      "primaryPhone": "+15551234567",
      "tags": [{ "id": 8, "name": "decision-maker" }],
      "lastActivity": "2025-04-01T08:00:00-04:00",
      "createdAt": "2024-01-15T10:00:00-05:00"
    }
  ],
  "pagination": { "total": 100, "perPage": 20, "currentPage": 1, "lastPage": 5 }
}
```

---

### GET /organizations/{id}/contacts
List contacts within a specific org. Same query params as GET /contacts (except `orgId`).

---

### GET /contacts/{id}
Full contact detail with all emails, phones, custom fields, and tags.

**Response:**
```json
{
  "id": 5,
  "firstName": "John",
  "lastName": "Doe",
  "fullName": "John Doe",
  "title": "CEO",
  "status": { "id": 1, "name": "Active" },
  "organization": { "id": 1, "name": "Acme Corp" },
  "emails": [
    { "value": "john@acme.com", "type": "work" },
    { "value": "john.personal@gmail.com", "type": "personal" }
  ],
  "phones": [
    { "value": "+15551234567", "type": "mobile" },
    { "value": "+15559876543", "type": "work" }
  ],
  "tags": [{ "id": 8, "name": "decision-maker" }],
  "customFields": [
    { "fieldId": 4, "label": "LinkedIn URL", "type": "text", "value": "https://linkedin.com/in/johndoe" },
    { "fieldId": 6, "label": "Lead Source", "type": "dropdown", "value": "Referral" }
  ],
  "owner": { "id": 10, "fullName": "Jane Smith" },
  "lastActivity": "2025-04-01T08:00:00-04:00",
  "createdAt": "2024-01-15T10:00:00-05:00"
}
```

---

## Tags

Organization tags and contact tags use the same tag pool within a tenant.
Tag matching is exact and case-sensitive.

Filter by single tag:
```
GET /organizations?tags=enterprise
```

Filter by multiple tags (ANY match):
```
GET /contacts?tags=vip,decision-maker
```

---

## Custom Fields

Custom fields appear only in detail endpoints.
Each field has:
- `fieldId` — the field definition ID
- `label` — display name (e.g. "Industry", "LinkedIn URL")
- `type` — `text` | `dropdown` | `singleOption` | `multiOptions`
- `value` — string for text/dropdown/singleOption (label returned), array for multiOptions

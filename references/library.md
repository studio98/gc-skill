# Library Reference

All endpoints require:
```
Authorization: Bearer {GRANDCENTRAL_API_KEY}
Content-Type: application/json
Base URL: https://api-v2.grandcentr.al/api
```

The library is a hierarchical document store. Folders can be nested. Items live inside folders and hold SOPs, policies, how-to guides, job descriptions, custom documents, and uploaded files.

---

## Item Types

| Type        | Has Readable Content | Description |
|-------------|----------------------|-------------|
| `custom`    | Yes (Markdown)       | Template-driven documents with custom fields (most common) |
| `policy`    | Yes (Markdown)       | Company policies — HTML body |
| `howTo`     | Yes (Markdown)       | How-to guides — HTML body |
| `article`   | Yes (Markdown)       | Articles — HTML body |
| `job`       | Yes (Markdown)       | Job descriptions with structured sections |
| `file`      | No                   | Uploaded binary file |
| `orgChart`  | No                   | Visual org chart |
| `flowChart` | No                   | Visual flow chart |

Only `custom`, `policy`, `howTo`, `article`, and `job` return readable content from `GET /library/items/{id}/content`.

---

## Folders

### GET /library/folders
List library folders. Defaults to root-level folders (no parent).

**Query params:**
| Param      | Type | Description |
|------------|------|-------------|
| `parentId` | int  | List children of this folder. Omit for root folders. |

**Response:**
```json
[
  { "id": 324, "name": "SOPs",    "parentId": null, "itemCount": 0,  "createdAt": "...", "updatedAt": "..." },
  { "id": 92,  "name": "S98 Company", "parentId": null, "itemCount": 3, "createdAt": "...", "updatedAt": "..." }
]
```

`parentId: null` means root-level folder.

---

### GET /library/folders/{id}
Get folder detail including breadcrumbs, subfolders, and items directly in the folder.

**Response:**
```json
{
  "folder": { "id": 363, "name": "NEW SOP's", "parentId": 324, "itemCount": 0, "createdAt": "...", "updatedAt": "..." },
  "breadcrumbs": [
    { "id": 324, "name": "SOPs" },
    { "id": 363, "name": "NEW SOP's" }
  ],
  "subfolders": [
    { "id": 391, "name": "GrandCentral", "parentId": 363, "itemCount": 0 }
  ],
  "items": [
    { "id": 509, "name": "1. UNDERSTANDING THE PROJECTS BOARD", "type": "custom", "folderId": 363, ... }
  ]
}
```

`breadcrumbs` is ordered root → current folder.

---

## Items

### GET /library/items
Search and list items across the whole library.

**Query params:**
| Param      | Type    | Description |
|------------|---------|-------------|
| `search`   | string  | Partial, case-insensitive name search |
| `type`     | string  | Filter by type (see Item Types table above) |
| `folderId` | int     | Restrict to a specific folder |
| `perPage`  | int     | Results per page (default: 20, max: 100) |

**Response:**
```json
{
  "data": [
    {
      "id": 1208,
      "name": "1. MANAGING DEALS IN GRANDCENTR.AL",
      "type": "custom",
      "folderId": 391,
      "status": "public",
      "documentType": "SOP",
      "hasFile": false,
      "author": { "id": 10237, "fullName": "Jaen Saunders" },
      "updatedBy": { "id": 10237, "fullName": "Jaen Saunders" },
      "createdAt": "2024-12-25T01:41:52+00:00",
      "updatedAt": "2026-01-26T23:47:00+00:00"
    }
  ],
  "pagination": { "total": 899, "perPage": 20, "currentPage": 1, "lastPage": 45 }
}
```

`documentType` is the template name for `custom` items (e.g. "SOP", "Document"). `hasFile` is true when a binary file is attached.

---

### GET /library/items/{id}
Get item metadata and its full breadcrumb path. Does not include document content.

**Response:**
```json
{
  "item": {
    "id": 1208,
    "name": "1. MANAGING DEALS IN GRANDCENTR.AL",
    "type": "custom",
    "folderId": 391,
    "status": "public",
    "documentType": "SOP",
    "hasFile": false,
    "author": { "id": 10237, "fullName": "Jaen Saunders" },
    "updatedBy": { "id": 10237, "fullName": "Jaen Saunders" },
    "createdAt": "2024-12-25T01:41:52+00:00",
    "updatedAt": "2026-01-26T23:47:00+00:00"
  },
  "breadcrumbs": [
    { "id": 324, "name": "SOPs" },
    { "id": 363, "name": "NEW SOP's" },
    { "id": 365, "name": "SaaS" },
    { "id": 391, "name": "GrandCentral" }
  ]
}
```

---

### GET /library/items/{id}/content
Get the agent-readable Markdown content of an item.

Only works for text-based types: `custom`, `policy`, `howTo`, `article`, `job`.

**Response (text type):**
```json
{
  "id": 1208,
  "name": "1. MANAGING DEALS IN GRANDCENTR.AL",
  "type": "custom",
  "content": "# 1. MANAGING DEALS IN GRANDCENTR.AL\n\n### Vision\n\nTo provide users with..."
}
```

**Response (non-text type, e.g. `file`):**
```json
{
  "id": 900,
  "name": "recovery-codes.txt",
  "type": "file",
  "content": null,
  "note": "This item type (file) does not have text-readable content."
}
```

---

## Agent Guidance

### Finding a document by name
```
GET /library/items?search=managing+deals
```
Returns all items whose name contains "managing deals". Then use the item `id` to fetch content.

### Reading a document
```
GET /library/items/{id}/content
```
Returns full Markdown. Check `type` first — if it's `file`, `flowChart`, or `orgChart`, `content` will be null.

### Browsing a folder tree
1. `GET /library/folders` — get root folders
2. `GET /library/folders/{id}` — drill into a folder (returns subfolders + items)
3. Repeat step 2 for subfolders as needed

### Finding all SOPs
```
GET /library/items?type=custom&search=SOP
```
Or browse the SOPs folder: `GET /library/folders` → find the "SOPs" folder → `GET /library/folders/{id}` to see its subfolders, then drill in.

### Filtering by folder
If you already know which folder holds the documents:
```
GET /library/items?folderId=391
```

### item.documentType vs item.type
- `type` = the underlying data model (`custom`, `policy`, etc.)
- `documentType` = the human-readable template name for `custom` items (e.g. `"SOP"`, `"Document"`, `"Job Description"`)
- Most company documents are `type=custom` with various `documentType` labels

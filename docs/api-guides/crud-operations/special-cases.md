---
sidebar_position: 6
---

# Special Cases in Entity Representations

This section describes special requirements when working with certain entity types or data structures in the iDempiere REST API.

---

## Image Field Representation

When you are **creating or modifying values of an Image type column/field**, the JSON entity representation must follow this structure:

```json
{
  "id": 123,  // Optional: the foreign record ID
  "file_name": "example.jpg",  // Required: readable file name
  "url": "https://example.com/image.jpg",  // Optional: URL for the image
  "data": "base64-encoded-string"  // Required if 'url' is not used
}
```

- `id`: The foreign record ID. If not provided, a new foreign record will be created and linked to the main record.
- `file_name`: The readable file name to be stored at the foreign table.
- `url`: An optional URL from which the image can be fetched.
- `data`: A string containing the base64-encoded image data.

---

## doc-action

**Type:** `string` (optional, request body field)

**Used in:** `POST` (create) and `PUT` (update) requests to a model/table endpoint (e.g. `/api/v1/models/c_order/{id}`)

**Description:**

doc-action is a field you include in the JSON request body when creating or updating a document record (Order, Invoice, Payment, etc.) to trigger a document action - i.e. advance the document through its workflow/status lifecycle in the same request that saves the record. When creating or updating a record, you can add a field "doc-action" with the action you want to apply on the document.

**Value:** The standard iDempiere DocAction status codes, for example:

| Code | Action            |
|------|-------------------|
| PR   | Prepare           |
| CO   | Complete          |
| CL   | Close             |
| VO   | Void              |
| RE   | Re-activate       |

NOTE: This list is incomplete, just for reference, the accepted values are the same values as the List for DocAction.

**Example request body:**
```json
json
{
  "doc-action": "CO"
}
```
sent as part of a `PUT` to complete a document (e.g. move it from Draft/In Progress to Completed).
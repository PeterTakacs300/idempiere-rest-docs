---
sidebar_position: 7
---

# Batch Request

**Endpoint:** `POST /api/v1/batch`

This endpoint allows executing one or more sub-requests.

**Optional parameter:**

The `transaction` parameter determines whether all batch requests should be processed within a single database transaction.  The default is `true`

If transaction is `true`, the method will roll back all changes if any request in the batch fails, ensuring atomicity.  All requests are executed in single transaction. Processing of request stop when it hit the first error.

If transaction is `false`, each request is processed in its own transaction, so failures in one request do not affect others.  All request will be processed even when some has error. Caller needs to check the status code of each response item to find out whether the execution of a particular request is success or fail.

**Example:** `POST /api/v1/batch?transaction=false`

## Request Body

An array of sub-request objects. Each object includes:

- `method`: HTTP method `"POST"`, `"PUT"` or `"DELETE"`
- `path`: Sub-request resource path, e.g. `"v1/models/C_Order"`
- `body`: JSON body for the sub-request (if applicable)
- `responseAlias`: Optional, unique alias to chain a later sub-request off this response — see [Chaining sub-requests](#chaining-sub-requests) below.

### Example

```json
[
  {
    "method": "POST",
    "path": "v1/models/C_Order",
    "body": {
      "DocumentNo": "ORD001",
      "C_BPartner_ID": 1000000
    }
  },
  {
    "method": "PUT",
    "path": "v1/models/C_Order/1000012",
    "body": {
      "DocStatus": "CO"
    }
  },
  {
    "method": "DELETE",
    "path": "v1/models/C_Order/1000013"
  }
]
```

## Response

An array of results from each sub-request. Each object includes:

- `status`: HTTP status text (e.g., "OK", "Created")
- `statusCode`: HTTP status code (e.g., 200, 201, 400)
- `body`: JSON return from the sub-request (if available)

## Chaining sub-requests

A sub-request's `body` can reference a value from an earlier sub-request's response, or from the caller's session/context, instead of a literal value. References are wrapped in `@...@`, the same marker iDempiere uses for context variables.

References only resolve inside `body`, not `path`, and the whole field value must be the reference.

### Referencing a prior sub-request's response

Give the earlier sub-request a `responseAlias`, unique within the batch. A later sub-request can then reference a value from its response using a JSONPath (RFC 9535) expression:

```text
@alias$.jsonPathExpr@
```

A record's primary key is available at `$.id`.

```json
[
  {
    "method": "POST",
    "path": "v1/models/C_BPartner",
    "responseAlias": "bpartner",
    "body": { "Value": "CUST-1001", "Name": "Acme", "IsCustomer": "Y" }
  },
  {
    "method": "POST",
    "path": "v1/models/C_BPartner_Location",
    "body": { "C_BPartner_ID": "@bpartner$.id@", "Name": "Main", "IsShipTo": "Y" }
  }
]
```

JSONPath array indexing and filters are supported too, e.g. `@order$.Lines[0].C_OrderLine_ID@`.

If the same table is used more than once in a batch, give each occurrence a different `responseAlias` so later sub-requests can pick the right one.

### Referencing the caller's session/context

A value can also be pulled from the caller's session/context, independent of any sub-request, using iDempiere's own context variable syntax: `@#GlobalVar@`, `@$GlobalVar@`, or `@+GlobalVar@`.

```json
{
  "method": "POST",
  "path": "v1/models/C_BPartner",
  "body": { "Value": "CUST-1001", "Name": "Acme", "IsCustomer": "Y", "AD_Org_ID": "@#AD_Org_ID@" }
}
```

### Errors

- Referencing a `responseAlias` that wasn't declared (or whose sub-request failed), or a JSONPath expression that matches nothing, returns `400 Bad Request`.
- Reusing a `responseAlias` already claimed earlier in the batch also returns `400 Bad Request`.

## Notes

- You can include various types of requests in the same batch, including creation, update, deletion, and even running processes.
- To use data created earlier in the batch (e.g. a business partner) in a later request (e.g. an order), give the earlier sub-request a `responseAlias` and reference it — see [Chaining sub-requests](#chaining-sub-requests) above.

---
sidebar_position: 1
---

# Authentication

To access any endpoint in the iDempiere REST API, you must first authenticate and obtain a **Bearer token**. This token must be sent in the `Authorization` header of every request.

---

## Login Overview

You can log in to the REST API in two main ways:

- **One-step login** — if you already know all the session parameters  
- **Normal login flow** — a step-by-step process similar to the iDempiere UI  

> **Important:** You can only log in using roles where `Role Type` is set to `WebService` or is left blank. By default, the `System` role is not permitted unless you clear its `Role Type`, or better: create a dedicated WebService role.

---

## One-Step Login

When you know all the values required to open a session (user, client, role, org, warehouse, language), use:

### `POST /api/v1/auth/tokens`

**Request body:**
```json
{
  "userName": "GardenAdmin",
  "password": "GardenAdmin",
  "parameters": {
    "clientId": 11,
    "roleId": 2000001,
    "organizationId": 11,
    "warehouseId": 103,
    "language": "en_US"
  }
}
```

---

## Normal Login Flow

This flow mimics the login process of the iDempiere UI.

### Step 1: `POST /api/v1/auth/tokens`

**Body:**
```json
{
  "userName": "your-username",
  "password": "your-password"
}
```

**Response:**
```json
{
  "clients": [{ "id": 11, "name": "GardenWorld" }],
  "token": "eyJraWQiOiJpZGVtcGllcmUi..."
}
```

### Step 2: Use the Token to Retrieve Options

Use the returned token in your request headers:

```
Authorization: Bearer YOUR_TOKEN
```

Then query the following endpoints in order:

#### Get Roles
```http
GET /api/v1/auth/roles?client=11
```

#### Get Organizations
```http
GET /api/v1/auth/organizations?client=11&role=2000001
```

#### Get Warehouses
```http
GET /api/v1/auth/warehouses?client=11&role=2000001&organization=11
```

#### Get Languages
```http
GET /api/v1/auth/language?client=11
```

### Step 3: Finalize Login

#### `PUT /api/v1/auth/tokens`

**Body:**
```json
{
  "clientId": 11,
  "roleId": 2000001,
  "organizationId": 11,
  "warehouseId": 103,
  "language": "en_US"
}
```

:::tip 
Fields like `language`, `organizationId`, and `warehouseId` are optional. If omitted, defaults will be used. 
:::

---

### Response Payload

```json
{
  "userId": 101,
  "language": "en_US",
  "token": "eyJraWQiOiJpZGVtcGllcmUi...",
  "refresh_token": "eyJraWQiOiJpZGVtcGllcmUi..."
}
```

Use the `token` in your headers to authenticate future requests.

---

## ⚡ Abbreviated Login

If the user has access to only one client, role, and organization, the initial `POST /auth/tokens` may return the final token directly, without further steps.

---

## Refresh & Logout

### 🔄 Refresh Token

The login process returns a `token` and a `refresh_token`. By default:
- `token` expires in 1 hour
- `refresh_token` expires in 24 hours

These defaults can be changed using the SysConfig keys:
- `REST_TOKEN_EXPIRE_IN_MINUTES`
- `REST_REFRESH_TOKEN_EXPIRE_IN_MINUTES`

:::tip Security Tip

It's recommended to store the `refresh_token` in secure storage (e.g. cookies), and keep the `token` in memory only.
:::

#### POST `/api/v1/auth/refresh`

**Body:**
```json
{
  "refresh_token": "your-refresh-token",
  "clientId": 11,
  "userId": 101
}
```

> `clientId` and `userId` are optional unless required via:
> - `REST_MANDATORY_CLIENT_ID_ON_REFRESH`
> - `REST_MANDATORY_USER_ID_ON_REFRESH`

**Response:**
```json
{
  "token": "new-token",
  "refresh_token": "new-refresh-token"
}
```

> ⚠️ Refresh tokens can only be used **once**. Reuse triggers a security breach and invalidates all related tokens.

To invalidate tokens, use the "Expire Refresh Tokens" process in iDempiere.

---

### Logout

To log out and revoke the token:

#### POST `/api/v1/auth/logout`

**Body:**
```json
{
  "token": "your-auth-token"
}
```

This ends the session and invalidates both the token and its refresh token.

---

## Password Reset (Forgot Password)

A code-based flow that lets a user who **cannot log in** reset their password. Unlike the rest of the
API, these endpoints are **unauthenticated** — do **not** send an `Authorization` header.

The flow has three steps:

1. **Request** a one-time code, delivered by email.
2. **Verify** the code to obtain a short-lived, single-use `verifiedToken`.
3. **Complete** the reset by setting a new password with that token.

### Step 1: Request a code

#### POST `/api/v1/auth/password-reset/request`

**Body:**
```json
{
  "email": "user@example.com",
  "language": "en_US"
}
```

> `language` is optional and only selects the locale of the email template.

**Response:** always a neutral `200`, whether or not the email matches an account (so the endpoint
cannot be used to discover which addresses are registered):
```json
{
  "summary": "If the email is registered, a code has been sent."
}
```

| Status | Meaning |
| --- | --- |
| `200` | Request accepted (neutral — sent only if the account exists). |
| `400` | `email` is missing or blank. |
| `429` | Too many reset requests for this identifier — try again later. |

### Step 2: Verify the code

#### POST `/api/v1/auth/password-reset/verify`

**Body:**
```json
{
  "email": "user@example.com",
  "code": "123456"
}
```

**Response:**
```json
{
  "verifiedToken": "3f9a0b7c..."
}
```

| Status | Meaning |
| --- | --- |
| `200` | Code verified — returns the `verifiedToken` for Step 3. |
| `400` | Missing fields, or an invalid/expired code, or too many attempts. |

> The `400` response is uniform and does **not** reveal whether the email is registered.

### Step 3: Set the new password

#### POST `/api/v1/auth/password-reset/complete`

**Body:**
```json
{
  "verifiedToken": "3f9a0b7c...",
  "newPassword": "MyNewPassw0rd!"
}
```

**Response:**
```json
{
  "summary": "Password updated successfully."
}
```

| Status | Meaning |
| --- | --- |
| `200` | Password changed. The account's existing sessions are invalidated. |
| `400` | Missing fields, an invalid/expired `verifiedToken`, or the new password violates the password policy. |

:::warning Security

- Responses are **neutral** so the flow cannot be used to enumerate accounts - an unknown email behaves
  exactly like a registered one at every step.
- Codes are **rate-limited** and **lock** after a configurable number of wrong attempts; requesting a new
  code invalidates the previous one.
- The `verifiedToken` is **single-use** and short-lived.

The flow is tuned through the core `PASSWORD_RESET_*` System Configurator keys (documented with the
core *Code-based password reset* feature), not through REST-specific SysConfig keys.
:::
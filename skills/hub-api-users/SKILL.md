---
name: hub-api-users
description: >
  Manage users on Fingerspot devices: get all users, get user IDs,
  get/create/update/delete individual users. Use when the user asks
  about user management, employee registration, or user CRUD operations.
---

# Hub API — Users

All user endpoints are **async** — they return an ACK immediately and deliver results via webhook.

## Headers

| Header | Required |
|--------|----------|
| `Authorization` | Yes |
| `X-Reference-ID` | Yes |
| `X-Webhook-URL` | No |

## Endpoints

### Get All Users

```
GET /v1/{cloud_id}/users
```

Returns all users on the device via webhook. Response `data` is an array of user objects.

**Supported brands:** vida, vivo

**Webhook data:** `Array<UserInfo>` — each contains `employeeNo`, `name`, `userType`, `valid`, etc.

---

### Get All User IDs (Lightweight)

```
GET /v1/{cloud_id}/user-ids
```

Returns only user IDs/PINs and count, without full user data. More efficient than `getAllUsers` for large lists.

**Supported brands:** revo, vega, vivo, vida, zkteco

**Webhook data:**
```json
{
  "total": 50,
  "pin_arr": ["101", "102", "103"]
}
```

---

### Get User by ID

```
GET /v1/{cloud_id}/users/{id}
```

Returns full user info including credentials (fingerprint, card, face, password, veins).

**Path params:**
- `id` — employee number

**Supported brands:** vivo, vida, zkteco

---

### Create/Update User

```
POST /v1/{cloud_id}/users
```

Sets user profile and all credentials in a single command. Creates the user if they don't exist, updates if they do.

**Request body:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `employeeNo` | string | **yes** | User ID on device |
| `name` | string | **yes** | User name |
| `userType` | string | no | `normal`, `admin`, `visitor` (Revo: also `user`, `operator`, `manager`, `supervisor`) |
| `password` | string | no | Plaintext password |
| `fingerprints` | array | no | `[{ fingerNo, fingerData }]` |
| `cards` | array | no | `[{ cardNo }]` |
| `faceData` | string | no | Base64 face data |
| `veins` | array | no | Array of base64 vein templates |
| `valid` | object | no | `{ beginDate, endDate }` format `YYYY-MM-DD` |

**Supported brands:** vivo, vida, revo, vega, zkteco

**Example:**
```json
{
  "employeeNo": "101",
  "name": "John",
  "userType": "admin",
  "password": "123456",
  "fingerprints": [
    { "fingerNo": 1, "fingerData": "base64..." }
  ],
  "cards": [
    { "cardNo": "12345678" }
  ]
}
```

---

### Delete User

```
DELETE /v1/{cloud_id}/users/{id}
```

Deletes a user from the device.

**Path params:**
- `id` — employee number

**Supported brands:** vida, vivo, zkteco

## Brand Notes

- `getAllUsers` only supports vida and vivo. For other brands, use `getAllUserIds`.
- `setUser` works across all brands but `userType` values differ — Revo supports extended roles.
- Always check `x-supported-brands` in the OpenAPI spec for the specific endpoint before calling.

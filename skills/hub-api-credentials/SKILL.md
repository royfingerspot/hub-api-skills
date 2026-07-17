---
name: hub-api-credentials
description: >
  Manage biometric and non-biometric credentials on Fingerspot devices:
  fingerprints, face data, cards, passwords, and QR codes. Use when the
  user asks about credential enrollment, fingerprint management, face
  registration, card setup, or QR code operations.
---

# Hub API — Credentials

All credential endpoints are **async** — return ACK immediately, result via webhook.

## Headers

| Header | Required |
|--------|----------|
| `Authorization` | Yes |
| `X-Reference-ID` | Yes |
| `X-Webhook-URL` | No |

## Fingerprint

### Get All User Fingerprints

```
GET /v1/{cloud_id}/users/{id}/finger
```

Returns all fingerprints for a user.

**Path params:** `id` — employee number
**Supported brands:** vivo, vida, zkteco

---

### Get Specific Fingerprint

```
GET /v1/{cloud_id}/users/{id}/finger/{fingerNo}
```

Returns a single fingerprint by number.

**Path params:** `id` — employee number, `fingerNo` — finger slot (1-10)
**Supported brands:** vivo, vida, zkteco

---

### Set All Fingerprints (Bulk)

```
PUT /v1/{cloud_id}/users/{id}/finger
```

Replaces all fingerprints for a user with the provided array.

**Request body:**
```json
{
  "fingerprints": [
    { "fingerNo": 1, "fingerData": "base64..." },
    { "fingerNo": 2, "fingerData": "base64..." }
  ]
}
```

**Supported brands:** vida (native bulk), vivo/zkteco (via FingerCapable)

---

### Set Single Fingerprint

```
PUT /v1/{cloud_id}/users/{id}/finger/{fingerNo}
```

Sets a single fingerprint by number.

**Request body:**
```json
{
  "fingerData": "base64..."
}
```

**Supported brands:** vivo, vida, zkteco

---

### Delete All Fingerprints

```
DELETE /v1/{cloud_id}/users/{id}/finger
```

Removes all fingerprints for a user.

**Supported brands:** vida (native bulk)

---

### Delete Single Fingerprint

```
DELETE /v1/{cloud_id}/users/{id}/finger/{fingerNo}
```

Removes a single fingerprint.

**Supported brands:** vivo, vida, zkteco

## Face

### Get User Face

```
GET /v1/{cloud_id}/users/{id}/face
```

Returns face data for a user.

**Supported brands:** vida, vivo

---

### Set User Face

```
PUT /v1/{cloud_id}/users/{id}/face
```

Registers face data for a user.

**Request body:**
```json
{
  "faceData": "base64..."
}
```

**Supported brands:** vida, vivo

---

### Delete User Face

```
DELETE /v1/{cloud_id}/users/{id}/face
```

Removes face data from a user.

**Supported brands:** vida, vivo

## Card

### Get User Card

```
GET /v1/{cloud_id}/users/{id}/card
```

Returns all cards registered to a user.

**Supported brands:** vivo, vida, zkteco

---

### Set User Card

```
PUT /v1/{cloud_id}/users/{id}/card
```

Registers a card for a user.

**Request body:**
```json
{
  "cardNo": "1234567890",
  "cardType": "normalCard"
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `cardNo` | string | **yes** | Card number |
| `cardType` | string | no | `normalCard` (default) or `hijackCard` |

**Supported brands:** vivo, vida, zkteco

---

### Delete User Card

```
DELETE /v1/{cloud_id}/users/{id}/card/{cardNo}
```

Removes a specific card from a user.

**Path params:** `id` — employee number, `cardNo` — card number
**Supported brands:** vivo, vida, zkteco

## Password

### Get User Password

```
GET /v1/{cloud_id}/users/{id}/password
```

Returns the stored password for a user.

**Supported brands:** vivo, vida, zkteco

---

### Set User Password

```
PUT /v1/{cloud_id}/users/{id}/password
```

Sets a new password for a user.

**Request body:**
```json
{
  "password": "newpassword123"
}
```

**Supported brands:** vivo, vida, zkteco

## QR Code

### Get All QR Codes

```
GET /v1/{cloud_id}/qrcode
```

Returns all QR codes on the device.

---

### Get QR Code by Record Number

```
GET /v1/{cloud_id}/qrcode/{recno}
```

Returns a specific QR code by its record number.

---

### Delete QR Code

```
DELETE /v1/{cloud_id}/qrcode/{recno}
```

Deletes a QR code from the device.

---

### Get All QR Codes for User

```
GET /v1/{cloud_id}/users/{id}/qrcode
```

Returns all QR codes registered to a specific user.

---

### Add QR Code to User

```
POST /v1/{cloud_id}/users/{id}/qrcode
```

Registers a new QR code for a user.

**Request body:**
```json
{
  "qrcode": "QR_VALUE_HERE"
}
```

---

### Set User QR Code

```
PUT /v1/{cloud_id}/users/{id}/qrcode
```

Replaces a user's QR code.

**Request body:**
```json
{
  "qrcode": "NEW_QR_VALUE"
}
```

---

### Revoke User QR Code

```
DELETE /v1/{cloud_id}/users/{id}/qrcode
```

Revokes all QR codes for a user.

## Notes

- Fingerprint templates are base64-encoded binary data from the device.
- Face data format varies by brand — check device documentation.
- Card `cardType` defaults to `normalCard` if not specified.
- QR code endpoints are async — results delivered via webhook like other commands.

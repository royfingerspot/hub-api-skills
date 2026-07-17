---
name: hub-api-device
description: >
  Manage Fingerspot devices: get device info, check connectivity, set time,
  reboot, configure timezone, and manage user validity periods. Use when
  the user asks about device status, configuration, or time management.
---

# Hub API — Device

Mix of **sync** and **async** endpoints. Check each endpoint below.

## Headers

| Header | Required |
|--------|----------|
| `Authorization` | Yes |
| `X-Reference-ID` | Yes (async endpoints) |
| `X-Webhook-URL` | No |

## Device Info & Status

### Get Device Info (Sync)

```
GET /v1/{cloud_id}
```

Returns device brand, serial, and capabilities directly.

**Supported brands:** vivo, vida, revo, vega, zkteco

**Response:**
```json
{
  "cloudId": "R118000104",
  "brand": "zkteco",
  "serial": "ABC123",
  "capabilities": { ... }
}
```

---

### Get Device Activity (Sync)

```
GET /v1/{cloud_id}/activity
```

Returns device connectivity status and last-seen timestamp.

**Response:**
```json
{
  "status": "connected",
  "lastSeen": "2026-06-15T10:30:00Z"
}
```

## Time

### Set Device Time (Async)

```
PUT /v1/{cloud_id}/time
```

Sets device date/time and/or timezone. Accepts IANA timezone names or ISO 8601 offsets.

**Request body:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `dateTime` | string | no | Format `YYYY-MM-DDTHH:MM:SS` |
| `timezone` | string | no | IANA (`Asia/Jakarta`) or offset (`+07:00`) |
| `mode` | string | no | `ntp` or `manual` |

At least one field must be provided.

**Supported brands:** vivo, vida, revo, vega, zkteco

**Example:**
```json
{
  "dateTime": "2026-06-15T10:30:00",
  "timezone": "Asia/Jakarta"
}
```

---

## Reboot

### Reboot Device (Async)

```
POST /v1/{cloud_id}/reboot
```

Restarts the device. No request body needed.

**Supported brands:** vida only

**Webhook data:**
```json
{
  "message": "reboot_initiated"
}
```

## Timezone

### Get Timezone (Async)

```
GET /v1/{cloud_id}/timezone
```

Returns the device's current timezone setting.

**Supported brands:** vivo, vida, zkteco

---

### Set Timezone (Async)

```
PUT /v1/{cloud_id}/timezone
```

Sets the device timezone.

**Request body:**
```json
{
  "timezone": "Asia/Jakarta"
}
```

**Supported brands:** vivo, vida, zkteco

---

### Lock/Unlock Timezone (Async)

```
PUT /v1/{cloud_id}/timezone/lock
```

Locks or unlocks timezone settings to prevent changes from the physical device.

**Request body:**
```json
{
  "lock": true
}
```

**Supported brands:** vivo, vida, zkteco

## Valid Date

### Get User Valid Date (Async)

```
GET /v1/{cloud_id}/users/{id}/valid
```

Returns the validity period for a user.

**Path params:**
- `id` — employee number

---

### Set User Valid Date (Async)

```
PUT /v1/{cloud_id}/users/{id}/valid
```

Sets the validity period for a user.

**Request body:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `employeeNo` | string | **yes** | Employee number |
| `beginDate` | string | **yes** | Format `YYYY-MM-DD` |
| `endDate` | string | **yes** | Format `YYYY-MM-DD` |
| `weekTimeZone` | array of int | **yes** | 7 values (Mon-Sun), `1`=active, `0`=inactive |

**Example:**
```json
{
  "employeeNo": "101",
  "beginDate": "2026-01-01",
  "endDate": "2026-12-31",
  "weekTimeZone": [1, 1, 1, 1, 1, 1, 1]
}
```

## Live Capture

### Capture Fingerprint (Async)

```
POST /v1/{cloud_id}/capture/finger
```

Triggers live fingerprint capture on the device.

**Request body:**
```json
{
  "fingerNo": 1,
  "employeeNo": "101"
}
```

**Supported brands:** vivo, vida, revo, vega, zkteco

---

### Capture Card (Async)

```
POST /v1/{cloud_id}/capture/card
```

Triggers live card capture on the device.

**Request body:**
```json
{
  "employeeNo": "101"
}
```

**Supported brands:** vivo, vida, revo, vega, zkteco

## Enrollment (Device-Initiated)

Enrollment commands trigger the device to enter enrollment mode. The user must physically interact with the device (place finger, look at camera, etc.) after the command is sent.

### Enroll Fingerprint

```
POST /v1/{cloud_id}/enroll/finger
```

**Request body:**
```json
{
  "employeeNo": "101",
  "fingerNo": 1
}
```

**Supported brands:** revo, vega

---

### Enroll Face

```
POST /v1/{cloud_id}/enroll/face
```

**Request body:**
```json
{
  "employeeNo": "101"
}
```

**Supported brands:** revo, vega

---

### Enroll Vein

```
POST /v1/{cloud_id}/enroll/vein
```

**Request body:**
```json
{
  "employeeNo": "101"
}
```

**Supported brands:** revo, vega

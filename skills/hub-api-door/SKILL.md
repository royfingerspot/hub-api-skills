---
name: hub-api-door
description: >
  Control doors connected to Fingerspot devices: open, close, and check
  status. Use when the user asks about door control, access control,
  or door status monitoring.
---

# Hub API — Door

Door endpoints are **async** — return ACK immediately, result via webhook.

## Headers

| Header | Required |
|--------|----------|
| `Authorization` | Yes |
| `X-Reference-ID` | Yes |
| `X-Webhook-URL` | No |

## Endpoints

### Open Door

```
POST /v1/{cloud_id}/door/open
```

Opens the door connected to the device.

**Request body (optional):**
```json
{
  "channel": 1
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `channel` | int | no | Door channel number, defaults to `1` |

**Supported brands:** vida

---

### Close Door

```
POST /v1/{cloud_id}/door/close
```

Closes the door connected to the device.

**Request body (optional):**
```json
{
  "channel": 1
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `channel` | int | no | Door channel number, defaults to `1` |

**Supported brands:** vida

---

### Get Door Status (Async)

```
GET /v1/{cloud_id}/door/status
```

Returns the current status of the door.

**Webhook data:**
```json
{
  "channel": 1,
  "status": "open"
}
```

**Supported brands:** vida

## Notes

- Door control is currently only supported on **Vida** brand devices.
- The `channel` parameter is for devices with multiple door controllers. Most devices have a single door (channel 1).
- Door events (open/close/alarm) are also pushed as realtime webhook events — see `hub-api-webhook` skill.

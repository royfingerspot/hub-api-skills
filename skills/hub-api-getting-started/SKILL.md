---
name: hub-api-getting-started
description: >
  Core integration patterns for Fingerspot Hub API: authentication,
  async command flow, webhook handling, error codes, and how to use
  the OpenAPI spec. Start here before using any other hub-api skill.
---

# Fingerspot Hub API — Getting Started

## Overview

Fingerspot Hub API is an **async device command API**. Most endpoints respond `200` immediately with an acknowledgement; the actual result is delivered later via a POST to the device's registered webhook URL. Some endpoints (device info, attendance, webhook logs) return data synchronously.

## Source of Truth

The OpenAPI spec is the single source of truth for all endpoints, schemas, and auth:

```
https://raw.githubusercontent.com/GemaMagang/hub-api-skills/main/openapi.yaml
```

Fetch this spec to discover available endpoints, request/response schemas, and supported brands. The `servers.url` field in the spec contains the base URL.

## Authentication

All requests require a Bearer token:

```
Authorization: Bearer <your-api-token>
```

The same token (`V2_API_TOKEN`) is used for both API authentication and HMAC webhook signature signing.

## Base URL

The base URL is defined in `servers.url` of the OpenAPI spec. Read it from there — it may change between versions. The current format is:

```
{servers.url}/v1/{cloud_id}/...
```

## Async Command Pattern

1. **Send request** → Server returns `200` with `AckResponse`:
   ```json
   {
     "referenceId": "your-tracking-id",
     "status": "queued",
     "deviceStatus": "connected",
     "message": "command queued successfully",
     "webhookUrls": ["https://your-webhook.com/callback"]
   }
   ```

2. **Wait for webhook** → Server POSTs the result to your webhook URL:
   ```json
   {
     "referenceId": "your-tracking-id",
     "status": "success",
     "commandType": "setUser",
     "data": { ... }
   }
   ```

3. **Correlate** — Match `referenceId` from the ack to the webhook callback.

### Synchronous Endpoints

These return data directly (no webhook):

- `GET /v1/{cloud_id}` — device info
- `GET /v1/{cloud_id}/activity` — device connectivity status
- `GET /v1/{cloud_id}/attendance` — attendance logs
- `PUT /v1/{cloud_id}/webhook` — set webhook URL

## Required Headers

| Header | Required | Description |
|--------|----------|-------------|
| `Authorization` | Yes | `Bearer <token>` |
| `X-Reference-ID` | Yes (async) | Your tracking ID, returned in webhook callback |
| `X-Webhook-URL` | No | Comma-separated extra webhook URLs (max 3) |
| `Content-Type` | Yes (POST/PUT) | `application/json` |

## Webhook URL Resolution

The webhook URL is resolved in this order:

1. `X-Webhook-URL` header (per-request override)
2. Per-device default (set via `PUT /v1/{cloud_id}/webhook`)
3. `V2_DEFAULT_WEBHOOK_URL` env var (global fallback)

## Webhook Payloads

**Success:**
```json
{
  "referenceId": "abc-123",
  "status": "success",
  "commandType": "setUser",
  "data": { ... }
}
```

**Failed:**
```json
{
  "referenceId": "abc-123",
  "status": "failed",
  "commandType": "setUser",
  "error": {
    "code": "ERR_DEVICE_TIMEOUT",
    "message": "device did not respond within 30m0s"
  }
}
```

## Device Offline Behavior

When a command is sent to an offline device:

- Request returns `200` with `deviceStatus: "offline"`
- Command is queued and executed when device comes online
- Result is delivered via webhook as usual

> **Note:** There may be a TTL on queued commands for devices that stay offline for extended periods. Confirm with Fingerspot team for long-term offline scenarios.

## Error Codes

| Code | Meaning |
|------|---------|
| `ERR_DEVICE_NOT_FOUND` | Device not found in registry |
| `ERR_DEVICE_OFFLINE` | Device not connected |
| `ERR_DEVICE_TIMEOUT` | Command timeout (default 30 min) |
| `ERR_DEVICE_BUSY` | Device processing another command |
| `ERR_INVALID_PARAMS` | Invalid request body or query params |
| `ERR_DATE_RANGE` | Invalid date range |
| `ERR_UNSUPPORTED` | Device doesn't support this feature |
| `ERR_INTERNAL` | Unexpected server error |
| `ERR_UNAUTHORIZED` | Invalid or missing token |

## Webhook Retry

Failed webhook deliveries are retried with **exponential backoff**:

| Attempt | Delay |
|---------|-------|
| 1 | 1s |
| 2 | 2s |
| 3 | 4s |
| 4 | 8s |
| 5 | 16s |

After 5 failed attempts, the error is logged and the payload is **not retried further**. There is no dead-letter queue or manual retry mechanism.

## Webhook Signature Verification

Every webhook request includes HMAC-SHA256 signatures for verification:

- `X-Hub-API-Timestamp` — Unix timestamp (seconds)
- `X-Hub-API-Signature-256` — `sha256=<hex-digest>`

**Verification steps:**
1. Check timestamp — reject if `now - timestamp > 5 minutes`
2. Compute `HMAC-SHA256(request_body, your_api_token)`
3. Compare with `X-Hub-API-Signature-256` using **constant-time comparison** (not `==`)
4. Match → process; no match → reject

See `hub-api-webhook` skill for full implementation details and code examples.

## Brand Capability

Every endpoint in the OpenAPI spec has an `x-supported-brands` field listing which device brands support it. Always check this before calling an endpoint — if the customer's device brand is not listed, the endpoint will return `ERR_UNSUPPORTED`.

Supported brands: `vivo`, `vida`, `revo`, `vega`, `zkteco`

## Related Skills

| Skill | Covers |
|-------|--------|
| `hub-api-users` | Get/set/delete users, user IDs |
| `hub-api-credentials` | Fingerprint, card, face, password, QR code |
| `hub-api-attendance` | Attendance log queries |
| `hub-api-device` | Device info, activity, time, reboot, timezone, valid date |
| `hub-api-door` | Door open/close/status |
| `hub-api-webhook` | Webhook config, signature verification, realtime events |

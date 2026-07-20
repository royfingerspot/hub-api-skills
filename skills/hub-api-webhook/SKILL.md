---
name: hub-api-webhook
description: >
  Configure webhooks, verify signatures, and handle realtime events from
  Fingerspot devices. Use when the user asks about webhook setup, signature
  verification, or event handling.
---

# Hub API — Webhook

## Webhook Configuration

### Set Default Webhook URL (Sync)

```
PUT /v1/{cloud_id}/webhook
```

Sets the default webhook URL for async command results on this device.

**Request body:**
```json
{
  "url": "https://your-server.com/webhook"
}
```

**Response:**
```json
{
  "referenceId": "abc-123",
  "status": "ok",
  "message": "webhook URL updated"
}
```

---

### Test Webhook Reachability (Sync)

```
POST /v1/{cloud_id}/webhook/test
```

Tests whether the configured webhook URL is reachable. Sends a GET request and returns the status.

No request body needed.

## Webhook URL Resolution

The webhook URL is resolved in this order:

1. `X-Webhook-URL` header (per-request override, comma-separated)
2. Per-device default (set via `PUT /v1/{cloud_id}/webhook`)
3. `V2_DEFAULT_WEBHOOK_URL` env var (global fallback)

## Webhook Signature Verification

Every webhook request includes HMAC-SHA256 signatures. **Always verify** to prevent spoofed callbacks.

### Headers Sent

| Header | Description |
|--------|-------------|
| `X-Hub-API-Timestamp` | Unix timestamp (seconds, UTC) |
| `X-Hub-API-Signature-256` | `sha256=<hex-digest>` |

### Verification Steps

1. **Check timestamp** — reject if `now - X-Hub-API-Timestamp > 300` (5 minutes)
2. **Compute HMAC** — `HMAC-SHA256(request_body, your_api_token)`
3. **Compare** — use **constant-time comparison** (never `==`)
4. **Match** → process; **no match** → reject

### Code Examples

**Node.js:**
```javascript
const crypto = require('crypto');

function verifyWebhook(body, timestamp, signature, secret) {
  // 1. Check timestamp
  const age = Math.floor(Date.now() / 1000) - parseInt(timestamp);
  if (age > 300) return false;

  // 2. Compute HMAC
  const expected = 'sha256=' + crypto
    .createHmac('sha256', secret)
    .update(body)
    .digest('hex');

  // 3. Constant-time compare
  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expected)
  );
}
```

**Python:**
```python
import hmac, hashlib, time

def verify_webhook(body: bytes, timestamp: str, signature: str, secret: str) -> bool:
    # 1. Check timestamp
    if time.time() - int(timestamp) > 300:
        return False

    # 2. Compute HMAC
    expected = "sha256=" + hmac.new(
        secret.encode(), body, hashlib.sha256
    ).hexdigest()

    # 3. Constant-time compare
    return hmac.compare_digest(signature, expected)
```

**PHP (official SDK):**
```php
use Fingerspot\HubClient\Webhook\WebhookVerifier;

$verifier = new WebhookVerifier($yourApiToken);
$isValid = $verifier->verify($requestBody, $_SERVER, 300);
```

**Go:**
```go
func verifyWebhook(body []byte, timestamp, signature, secret string) bool {
    age := time.Now().Unix() - parseInt(timestamp)
    if age > 300 { return false }

    mac := hmac.New(sha256.New, []byte(secret))
    mac.Write(body)
    expected := "sha256=" + hex.EncodeToString(mac.Sum(nil))

    return subtle.ConstantTimeCompare([]byte(signature), []byte(expected)) == 1
}
```

## Webhook Retry Behavior

Failed deliveries are retried with **exponential backoff**:

| Attempt | Delay |
|---------|-------|
| 1 | 1s |
| 2 | 2s |
| 3 | 4s |
| 4 | 8s |
| 5 | 16s |

After 5 failed attempts, the error is logged and the payload is **not retried further**. There is no dead-letter queue or manual retry mechanism.

## Realtime Events

These are **device-initiated** events pushed to your webhook URL — they are NOT results of commands you sent. They fire independently.

### Event Types

| Event | Description |
|-------|-------------|
| `attendance` | User scanned at device (check-in/out) |
| `verificationFailed` | Verification attempt rejected |
| `doorEvent` | Door state changed (open/close/alarm) |
| `doorbell` | Doorbell button pressed |
| `tamper` | Physical tamper detected |
| `alarm` | Device alarm triggered |
| `exception` | Device exception occurred |
| `operation` | Device operation event |
| `userChange` | User data changed directly on device |
| `deviceActivity` | Device connectivity status changed (online/offline) |

### Attendance Event Payload

```json
{
  "event": "attendance",
  "cloudId": "R118000104",
  "employeeNo": "101",
  "name": "John",
  "cardNo": "12345678",
  "verifyMode": "fingerprint",
  "attendanceStatus": "check_in",
  "eventTime": "2026-06-15T08:00:00+07:00",
  "userType": "admin",
  "picture": "",
  "workCode": ""
}
```

### Door Event Payload

```json
{
  "event": "doorOpen",
  "cloudId": "R118000104",
  "eventTime": "2026-06-15T08:00:00+07:00",
  "doorNo": 1,
  "status": "open"
}
```

### User Change Event Payload

```json
{
  "event": "userChange",
  "cloudId": "R118000104",
  "eventTime": "2026-06-15T08:00:00+07:00",
  "employeeNo": "101",
  "changeType": "add",
  "credentials": ["face", "finger", "card"]
}
```

### Device Activity Payload

```json
{
  "event": "deviceActivity",
  "cloudId": "R118000104",
  "brand": "zkteco",
  "serial": "ABC123",
  "status": "connected",
  "lastSeen": "2026-06-15T10:30:00Z"
}
```

### Alarm/Exception/Operation Payload

```json
{
  "event": "alarm",
  "cloudId": "R118000104",
  "eventTime": "2026-06-15T08:00:00+07:00",
  "majorEventType": "alarm",
  "subEventType": "intrusion",
  "description": "Intrusion detected"
}
```

## Notes

- Realtime events require a webhook URL configured on the device.
- The `deviceActivity` event is based on Hub's connectivity polling, not a direct device push.
- All event payloads include `event` and `cloudId` fields for routing.
- Signature verification applies to ALL webhook deliveries — both command callbacks and realtime events.

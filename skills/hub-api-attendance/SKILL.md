---
name: hub-api-attendance
description: >
  Query attendance logs from Fingerspot devices. Use when the user asks
  about attendance records, scan history, check-in/check-out logs, or
  attendance reports.
---

# Hub API — Attendance

Attendance endpoint is **synchronous** — returns data directly, no webhook.

## Headers

| Header | Required |
|--------|----------|
| `Authorization` | Yes |
| `X-Reference-ID` | No (sync endpoint, but still recommended for tracking) |

## Endpoint

### Get Attendance Logs

```
GET /v1/{cloud_id}/attendance
```

Returns attendance log entries from the device database.

**Query params:**

| Param | Type | Required | Notes |
|-------|------|----------|-------|
| `startDate` | string | **yes** | Format `YYYY-MM-DD` |
| `endDate` | string | **yes** | Format `YYYY-MM-DD` |
| `page` | int | no | Default `1` |
| `limit` | int | no | Default `100`, max `100` |

**Supported brands:** vivo, vida, revo, vega, zkteco

**Response:**
```json
{
  "records": [
    {
      "employeeNo": "101",
      "name": "John",
      "cardNo": "12345678",
      "eventTime": "2026-06-15T08:00:00+07:00",
      "verifyMode": "fingerprint",
      "attendanceStatus": "check_in",
      "userType": "admin",
      "serialNumber": "ABC123",
      "cloudId": "GQ5778408",
      "image": null,
      "workCode": ""
    }
  ],
  "total": 50,
  "page": 1,
  "limit": 100
}
```

**Example:**
```bash
curl -H "Authorization: Bearer $TOKEN" \
  "https://your-api.com/v1/R118000104/attendance?startDate=2026-07-01&endDate=2026-07-17"
```

## Notes

- `verifyMode` values: `fingerprint`, `password`, `card`, `face`, `vein`
- `attendanceStatus` values: `check_in`, `check_out`

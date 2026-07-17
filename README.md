# Fingerspot Hub API — Agent Skills

Agent skills yang mengajarkan AI coding agent cara integrasi ke Fingerspot Hub API.

## Install

```bash
# Install semua skill sekaligus
npx skills add GemaMagang/hub-api-skills --all

# Install skill tertentu saja
npx skills add GemaMagang/hub-api-skills --skill hub-api-getting-started

# Preview skill apa aja yang tersedia sebelum install
npx skills add GemaMagang/hub-api-skills --list

# Install ke agent spesifik (kalau pakai lebih dari satu)
npx skills add GemaMagang/hub-api-skills --all -a claude-code
```

Setelah terinstall, tiap skill otomatis juga tersedia sebagai slash command di Claude Code (contoh: `/hub-api-webhook`), jadi kamu bisa memanggilnya langsung tanpa menunggu AI mendeteksi konteksnya sendiri. Perilaku ini spesifik untuk Claude Code — di agent lain (Codex, OpenCode, dll) skill tetap bekerja lewat auto-detect biasa.

## Skills

| Skill | Description |
|---|---|
| `hub-api-getting-started` | Auth, async flow, error handling, webhook signature overview |
| `hub-api-users` | Get/set/delete user, user IDs |
| `hub-api-credentials` | Fingerprint, card, face, password, QR code management |
| `hub-api-attendance` | Attendance log queries |
| `hub-api-device` | Device info, activity, time, reboot, timezone, valid date |
| `hub-api-door` | Door open/close/status |
| `hub-api-webhook` | Webhook config, signature verification, logs, realtime events |

## Source of Truth

`openapi.yaml` di root repo ini adalah source of truth untuk semua endpoint, schema, dan auth. Semua skill fetch langsung dari file ini.

## Development

Belum dipublikasikan. Masih dalam tahap development.

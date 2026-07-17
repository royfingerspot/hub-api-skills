# Hub API Skills — Ringkasan Desain & Keputusan

> Dokumen ini merangkum hasil diskusi desain untuk membangun "Agent Skills" publik yang mengajarkan AI coding agent (Claude Code, Codex, OpenCode, Cursor, dll) cara integrasi ke Fingerspot Hub API. Ditulis untuk dilanjutkan oleh agent/developer lain — bagian "belum selesai" perlu di-audit dan dieksekusi sebelum publish ke publik.

---

## 1. Tujuan Proyek

Membuat kumpulan **Agent Skills** (folder `SKILL.md` + file pendukung) yang bisa di-install developer eksternal via `npx skills add` sehingga AI coding agent mereka langsung paham cara integrasi ke Fingerspot Hub API — mirip pola skill yang dipakai Stripe, Midtrans, dll. Target audiens: **developer eksternal (publik)**, bukan tim internal Fingerspot.

---

## 2. Keputusan Repo & Distribusi

| Item | Keputusan |
|---|---|
| Tool distribusi | `npx skills` (vercel-labs/skills) — CLI open source untuk install/manage skill lintas agent |
| Platform hosting | **GitHub** (bukan Bitbucket) |
| Alasan GitHub, bukan Bitbucket | `npx skills` saat ini **hanya support GitHub dan GitLab**. Bitbucket adalah open feature request yang belum diimplementasikan (vercel-labs/skills issue #277). Ini keharusan teknis, bukan preferensi. |
| Repo saat ini (sementara) | `https://github.com/GemaMagang/hub-api-skills.git` — masih di akun personal, kemungkinan dipindah ke org resmi Fingerspot nanti setelah matang |
| Visibility | **Private** untuk sekarang, selama masih development. Publish ke publik setelah checklist di Bagian 6 selesai. |
| Catatan penting | Selama repo private, `raw.githubusercontent.com/...` **tidak bisa diakses tanpa token** oleh AI agent eksternal. Jangan share skill ke customer sebelum repo di-public-kan. |

---

## 3. Struktur Repo & Skill

```
hub-api-skills/
├── openapi.yaml                    ← source of truth, di-raw-fetch oleh semua skill
├── README.md                       ← dokumentasi repo untuk manusia
└── skills/
    ├── hub-api-getting-started/    ← WAJIB, skill sentral: auth, async flow, cara fetch
    │                                  openapi.yaml, X-Reference-ID, error handling umum,
    │                                  overview singkat webhook signature
    ├── hub-api-users/              ← get/set/delete user, user-ids
    ├── hub-api-credentials/        ← finger, face, card, password, qrcode (7 endpoint qrcode: get all, get by recno, delete, get user qr, add user qr, set user qr, revoke user qr)
    ├── hub-api-attendance/         ← attendance log queries
    ├── hub-api-device/             ← device info, activity, time, reboot
    │                                  (timezone & valid-date bisa digabung sini kalau kecil)
    ├── hub-api-door/               ← door open/close/status
    └── hub-api-webhook/            ← webhook config, signature verification detail,
                                       webhook logs, 10 realtime event types
```

**Alasan struktur ini:**
- Dipecah **per resource** (bukan per use-case atau satu skill besar) — mengikuti tag yang sudah ada persis di `openapi.yaml` (`Device`, `Users`, `Credentials`, `Timezone`, `Door`, `Valid Date`, `Attendance`, `Webhook Logs`, `Webhook Config`), jadi mapping-nya natural.
- Description trigger tiap skill jadi lebih spesifik → akurasi trigger AI lebih tinggi (referensi: philschmid.de mencatat peningkatan hingga 50% akurasi trigger hanya dari perbaikan description).
- `hub-api-getting-started` dibuat **sentral/tunggal** untuk pola inti (auth, async flow) — skill lain **reference** ke sini alih-alih mengulang penjelasan. Alasan: lebih mudah maintain, konsisten dengan prinsip *progressive disclosure* dan *"keep reference tree flat, one level deep"* dari best practice resmi skill authoring.
- Pola umum di industri: `{org}/skills` sebagai satu repo berisi banyak skill kecil (bukti: `anthropics/skills`, `vercel-labs/agent-skills`, `google/skills`, `indykite/skills`). Nama `hub-api-skills` valid karena lebih deskriptif untuk scope produk saat ini.

---

## 4. Source of Truth: `openapi.yaml`

### Keputusan kunci
- **`openapi.yaml` (bukan docs Docusaurus prosa) adalah source of truth utama** untuk endpoint, schema, auth, dan parameter.
- **Alasan konkret** (bukan teori): saat proses diskusi, ditemukan bahwa halaman `getting-started` di `dev.int.docs-hub-api.fingerspot.io` **sudah outdated** — menyebutkan auth `X-API-Key` padahal spec YAML yang benar menyebutkan `BearerAuth`. Endpoint path juga berbeda (`/v2/{cloud_id}` di docs vs `/v1/{cloud_id}` di YAML). Ini bukti nyata kenapa docs prosa tidak boleh dipercaya sebagai source utama.
- **Docs Docusaurus tetap dipakai**, tapi khusus untuk **penjelasan konseptual** yang tidak ada di YAML (misal rate limiting policy, narasi "kenapa" desain tertentu). **Kalau ada konflik antara YAML dan docs soal endpoint/auth/schema, YAML yang menang.**

### Cara expose
- Repo `hub-api-skills` di GitHub **berisi `openapi.yaml` langsung di root** (bukan repo terpisah), karena update endpoint jarang terjadi sehingga tidak menambah beban maintenance untuk digabung.
- Diakses via GitHub raw URL, contoh (sesuaikan branch/org bila berubah):
  ```
  https://raw.githubusercontent.com/GemaMagang/hub-api-skills/main/openapi.yaml
  ```
- Preseden industri: Stripe mempublikasikan OpenAPI spec mereka (JSON & YAML) di repo GitHub publik `stripe/openapi`, diupdate tiap rilis. Pola publikasi OpenAPI di GitHub ini umum di industri API.

### Aturan penanganan base URL (`servers.url`)
- **JANGAN hardcode base URL di badan `SKILL.md`/`references/`.** Field `servers.url` di `openapi.yaml` tetap menjadi satu-satunya sumber kebenaran base URL — kalau berubah, cukup update YAML, semua skill otomatis ikut benar (karena mereka fetch live, bukan menyalin nilai).
- **Base URL sudah terbukti berubah 3 kali** selama diskusi ini saja (`/v2/{cloud_id}`, `/api/external/v1/{cloud_id}/...`, `/v1/{cloud_id}`) — desain harus asumsikan ini akan terus berubah.
- Instruksi di `SKILL.md` harus eksplisit memberi tahu agent: field `servers.url` bisa berubah, jangan mengandalkan nilai yang di-cache dari sesi/fetch sebelumnya, selalu fetch ulang sebelum generate kode integrasi baru.
- **Saat ini `servers.url` di YAML masih menunjuk domain dev/internal** (`dev.int.api-new.fingerspot.io`) — lihat checklist Bagian 6.

---

## 5. Detail Teknis Kunci (Hasil Ekstraksi dari `openapi.yaml` & Diskusi)

### 5.1 Autentikasi
- **Bearer token** (`Authorization: Bearer <token>`), skema `BearerAuth` di `securitySchemes`.
- Token yang sama (`V2_API_TOKEN`) dipakai **dua fungsi**: (a) auth outbound request customer ke Hub API, dan (b) secret untuk HMAC signing webhook. Ini penting dikomunikasikan eksplisit ke customer di skill.

### 5.2 Pola Async Command + Webhook
- Sebagian besar endpoint command (contoh: `setUser`, `getUser`, `setUserFinger`, dll) bersifat **asynchronous**: request langsung dibalas `200` dengan `AckResponse` (`status: queued`), hasil sebenarnya dikirim belakangan via **POST ke webhook URL** milik device.
- Beberapa endpoint bersifat **synchronous** (langsung return data): `getDevice`, `getDeviceActivity`, `getAttendance`, `getWebhookLogs`, `resendWebhook`, `setWebhook`, `updateDevicePassword`.
- **`X-Reference-ID`** (header, wajib untuk endpoint async) — dipakai untuk korelasi antara ack response dan webhook callback yang datang belakangan.
- **`X-Webhook-URL`** (header, opsional) — comma-separated, mengirim callback tambahan ke URL lain di luar webhook default device, maksimum `V2_WEBHOOK_MAX_URLS` (default 3).
- **Webhook URL Resolution Order** (terkonfirmasi dari kode `webhook.go:70-75`):
  1. `X-Webhook-URL` header (per-request override)
  2. Per-device default (set via `PUT /v2/{cloud_id}/webhook`)
  3. `V2_DEFAULT_WEBHOOK_URL` env var (fallback global)
- **`x-supported-brands`** — setiap endpoint di YAML punya field ini (contoh: `[vivo, vida, revo, vega, zkteco]`). Skill **harus selalu cek capability per brand** sebelum menyarankan endpoint tertentu ke customer, agar tidak menyarankan endpoint yang device mereka tidak dukung.

### 5.3 Device Offline Behavior
- Ketika command dikirim ke device yang sedang offline: request tetap dibalas `200` dengan `deviceStatus: "offline"` di `AckResponse` — ini terkonfirmasi ada di spec.
- **Command diperkirakan tetap di-queue** dan akan dieksekusi begitu device online kembali, dengan hasil dikirim via webhook seperti biasa.
- **⚠️ BELUM DIVERIFIKASI:** Kemungkinan ada TTL/masa berlaku pada command yang di-queue (device developer sendiri menyebut "mungkin ada TTL" tapi tidak yakin angka pastinya). **Skill harus ditulis dengan disclaimer eksplisit**, bukan sebagai fakta pasti — sarankan customer konfirmasi ke tim Fingerspot untuk kasus device offline jangka panjang.

### 5.4 Error Handling
- `ErrorResponse` schema dengan `Error.code` enum tertutup: `ERR_DEVICE_NOT_FOUND`, `ERR_DEVICE_OFFLINE`, `ERR_DEVICE_TIMEOUT`, `ERR_DEVICE_BUSY`, `ERR_INVALID_PARAMS`, `ERR_DATE_RANGE`, `ERR_UNSUPPORTED`, `ERR_INTERNAL`, `ERR_UNAUTHORIZED`.
- Webhook callback yang gagal terkirim: retried up to 5 times dengan **exponential backoff** (1s, 2s, 4s, 8s, 16s) — terkonfirmasi dari kode `internal/v2/webhook.go:126-134`.
- **Setelah 5x gagal:** error di-log, function return error. **Tidak ada dead-letter queue** — payload hilang. Customer harus cek manual via `getWebhookLogs` untuk mendeteksi webhook yang gagal.

### 5.5 Realtime Webhook Events (terpisah dari command-callback)
Ada 10 event push standalone (bukan hasil dari command yang dikirim customer, melainkan device-initiated), penting dijelaskan terpisah agar tidak tertukar dengan pola command-callback biasa:
1. `attendance` — user scan di device
2. `verificationFailed` — percobaan verifikasi ditolak
3. `doorEvent` — perubahan state pintu (open/close/alarm)
4. `doorbell` — tombol bel ditekan
5. `tamper` — deteksi tamper fisik
6. `alarm` — alarm device
7. `exception` — exception device
8. `operation` — operation event device
9. `userChange` — user data berubah langsung di device (bukan lewat hub)
10. `deviceActivity` — perubahan status koneksi device (online/offline) — **ini bukan realtime push dari device, melainkan perubahan activity status yang di-poll oleh Hub**

### 5.6 Webhook Signature Verification (LENGKAP, siap ditulis ke skill)

**Signing (sisi Hub API / producer):**
- Algorithm: **HMAC-SHA256**
- Yang di-sign: **body JSON raw bytes**
- Secret: **`V2_API_TOKEN`** (sama dengan Bearer token auth)
- Header yang dikirim:
  - `X-Hub-API-Timestamp`: unix seconds
  - `X-Hub-API-Signature-256`: format `sha256=<hex>`

**Verification (sisi consumer/customer):**
1. Cek timestamp — jika `now - X-Hub-API-Timestamp > 5 menit` → **REJECT** (expired/replay protection)
2. Compute ulang `HMAC-SHA256(body, secret)` di sisi consumer
3. Bandingkan signature **menggunakan `hash_equals()`** (atau ekuivalen constant-time comparison di bahasa lain) — **BUKAN** operator `==` biasa, untuk mencegah timing attack
4. Jika match → proses payload; jika tidak → reject (invalid signature)

**Status implementasi saat ini (dari internal Fingerspot):**
| Komponen | Status |
|---|---|
| Signing (Go, `internal/v2/webhook.go`) | ✅ Selesai |
| Verifier resmi | ✅ Hanya **PHP** (`team-device-hub-client-php/src/Webhook/WebhookVerifier.php`) |
| Verifier Go/Node/Python | ❌ Belum ada SDK resmi |
| Verifier di developer website (demo) | ⚠️ Masih `// TODO`, belum aktif |
| Demo receiver | ⚠️ Tidak melakukan verifikasi signature — **jangan dijadikan contoh produksi tanpa warning eksplisit** |

**Implikasi untuk skill:** karena hanya PHP yang punya SDK resmi, skill `hub-api-webhook` harus menjelaskan alur di atas secara **bahasa-agnostic (pseudocode/step-by-step)**, bukan hanya menunjuk ke kode PHP — supaya AI agent bisa membantu generate implementasi di bahasa apa pun (Node, Python, Go, dll) berdasarkan pola yang sama.

---

## 6. Strategi Multi-Bahasa (SDK vs Skill)

- **Keputusan:** tidak buru-buru membuat SDK resmi di banyak bahasa sekaligus. SDK resmi PHP tetap menjadi prioritas (karena mayoritas customer saat ini pakai PHP), sementara bahasa lain **"berkembang sesuai demand"**.
- **Alasan skill bisa menutup gap ini tanpa perlu SDK banyak bahasa:** skill berisi instruksi/pola (bukan kode terkompilasi), sehingga secara alami bahasa-agnostic. AI agent yang membaca skill (misal alur HMAC verification di atas) bisa men-generate implementasi di bahasa apa pun on-the-fly, tanpa Fingerspot perlu menulis dan memelihara SDK resmi di bahasa tersebut.
- Ini adalah solusi langsung untuk gap yang sudah teridentifikasi sendiri oleh tim: "No Go/Node/Python verifier."

---

## 7. Audit Keamanan `openapi.yaml` — Temuan

### Aman untuk dipublikasikan
- Semua `example`/`description` bersih, tidak ada token, cloud_id asli, atau kredensial nyelip di dalam spec.
- Struktur schema, endpoint, error code — semua memang ditujukan untuk dibaca publik.

### ⚠️ HARUS ditangani sebelum publish (lihat Bagian 8 — Checklist)
1. **`servers.url` masih menunjuk domain dev/internal** (`dev.int.api-new.fingerspot.io`). Jika file di-expose apa adanya, customer publik akan mencoba call ke domain internal ini.
2. **Tag `Devices (DEV ONLY)`** (endpoint `GET/POST /v1/devices`, `DELETE /v1/devices/{cloud_id}`) — endpoint administratif untuk register/hapus device, **bukan untuk customer eksternal**.
3. **`PUT/DELETE /v1/{cloud_id}/credentials`** — endpoint untuk update/hapus password login device di NATS registry. Ini juga administratif internal, berpotensi disalahgunakan jika ter-expose ke publik (bisa mengubah/menghapus kredensial device milik orang lain).

> **Developer (Gema) sudah menyatakan akan menghapus endpoint dev-only ini sendiri sebelum publish** — item ini di-track di checklist tapi eksekusinya di luar scope kerja skill authoring.

---

## 8. Checklist — Status Saat Ini

### ✅ Selesai / Settled
- [x] Nama repo & lokasi: `github.com/GemaMagang/hub-api-skills` (sementara, personal account)
- [x] Platform distribusi: GitHub (bukan Bitbucket — alasan teknis, lihat Bagian 2)
- [x] Struktur skill: per resource + satu skill sentral `hub-api-getting-started`
- [x] Source of truth: `openapi.yaml` di-raw-fetch dari GitHub, bukan docs Docusaurus
- [x] Kebijakan base URL: tetap variable di YAML, tidak boleh di-hardcode di skill
- [x] Detail lengkap webhook signature verification (HMAC-SHA256, header, replay window, hash_equals)
- [x] Strategi multi-bahasa: skill sebagai pengganti SDK, SDK resmi berkembang sesuai demand
- [x] Audit keamanan awal `openapi.yaml` selesai dilakukan (lihat Bagian 7)
- [x] Webhook retry: exponential backoff 1s/2s/4s/8s/16s, max 5 retries, no dead-letter — terkonfirmasi dari kode
- [x] Realtime events: 10 tipe event (termasuk `deviceActivity`), bukan 7 — terkonfirmasi dari kode
- [x] Webhook URL resolution order: header → per-device → env fallback — terkonfirmasi dari kode
- [x] Auth middleware: fixed timing attack vulnerability dengan `crypto/subtle.ConstantTimeCompare`

### ⏳ Belum Selesai — Perlu Dieksekusi/Diaudit Sebelum Publish Publik
- [ ] **Strip endpoint dev-only** dari `openapi.yaml` versi publik (`Devices (DEV ONLY)` tag + `PUT/DELETE /v1/{cloud_id}/credentials`)
- [ ] **Tentukan/perbaiki `servers.url`** — arahkan ke domain publik final, atau beri disclaimer jelas jika masih dev
- [ ] **Fix halaman `getting-started` di Docusaurus** (`dev.int.docs-hub-api.fingerspot.io`) — auth header salah (tertulis `X-API-Key`, seharusnya `Bearer`)
- [ ] **Fix frontend endpoint path** — frontend saat ini hit `/v2/` padahal live server `/v1/`. `openapi.yaml` sudah benar (`/v1/`), yang perlu diubah frontend-nya
- [ ] Repo GitHub masih **private** — publish ke public setelah semua poin di atas selesai
- [ ] Draft `SKILL.md` untuk `hub-api-getting-started` **belum ditulis sama sekali** — ini next step setelah repo local ter-setup
- [ ] Draft `SKILL.md` untuk skill per-resource lainnya (`hub-api-users`, `hub-api-credentials`, dll) — belum dimulai, menyusul setelah `getting-started` selesai dan divalidasi
- [ ] Testing skill ke agent nyata (Claude Code/OpenCode) — belum dilakukan sama sekali, ini validasi wajib sebelum publish (rujukan best practice: proses ideal melibatkan satu agent untuk menyusun skill, agent lain untuk menguji di tugas nyata)

### ❓ Informasi Belum Lengkap (bisa ditulis dengan disclaimer di skill, tidak blocking)
- [ ] TTL/masa berlaku command yang di-queue saat device offline dalam waktu lama
- [ ] Definisi presisi tiap error code (kapan tepatnya `ERR_DEVICE_BUSY` dsb muncul) — nice-to-have, tidak blocking v1

---

## 9. Next Step yang Disarankan (Urutan)

1. Selesaikan setup repo local (`GemaMagang/hub-api-skills`) — copy `openapi.yaml` ke root repo
2. Eksekusi 3 item checklist "belum selesai" di Bagian 8 (strip dev-only endpoint, fix server URL, fix docs Docusaurus)
3. Mulai tulis draft `skills/hub-api-getting-started/SKILL.md` — ini fondasi yang direferensikan semua skill lain, isi mencakup: overview async pattern, auth Bearer, cara fetch `openapi.yaml`, `X-Reference-ID`, `X-Webhook-URL`, device offline behavior (dengan disclaimer TTL), overview singkat webhook signature (link ke skill `hub-api-webhook` untuk detail)
4. Test draft ke agent nyata (Claude Code/OpenCode) sebelum lanjut ke skill resource lainnya
5. Setelah `getting-started` solid, lanjut ke skill per-resource lain mengikuti pola yang sama
6. Publish repo ke public setelah semua checklist wajib selesai

---

*Dokumen ini adalah ringkasan hasil diskusi desain, bukan spesifikasi final. Bagian dengan tanda ⚠️ atau ❓ memerlukan verifikasi/keputusan lebih lanjut sebelum dianggap sebagai fakta yang bisa ditulis di skill publik.*
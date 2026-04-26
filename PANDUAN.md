# Panduan Penggunaan: Go WhatsApp Web Multi-Device (GoWA)

> Versi: v8.x | Bahasa: Go | Platform: Linux, macOS, Windows

GoWA adalah aplikasi open-source yang mengekspos WhatsApp Web sebagai **REST API** atau **MCP Server** (Model Context Protocol). Cocok untuk otomatisasi pesan, integrasi CRM, atau AI agent yang perlu berinteraksi dengan WhatsApp.

---

## Daftar Isi

1. [Persiapan & Instalasi](#1-persiapan--instalasi)
2. [Konfigurasi](#2-konfigurasi)
3. [Cara Menjalankan](#3-cara-menjalankan)
4. [Login WhatsApp](#4-login-whatsapp)
5. [REST API — Panduan Lengkap](#5-rest-api--panduan-lengkap)
6. [MCP Server (untuk AI Agent)](#6-mcp-server-untuk-ai-agent)
7. [Webhook](#7-webhook)
8. [Multi-Device](#8-multi-device)
9. [Integrasi Chatwoot](#9-integrasi-chatwoot)
10. [Tips Produksi](#10-tips-produksi)

---

## 1. Persiapan & Instalasi

### Opsi A — Docker (Direkomendasikan, tidak perlu install Go)

```bash
# Clone repository
git clone https://github.com/aldinokemal/go-whatsapp-web-multidevice
cd go-whatsapp-web-multidevice

# Salin file konfigurasi
cp src/.env.example src/.env

# Jalankan
docker-compose up -d --build

# Akses UI: http://localhost:3000
```

### Opsi B — Jalankan dari Source Code

**Requirement:**
- Go 1.24.0 atau lebih tinggi
- FFmpeg
- libwebp (untuk sticker)

**Instalasi dependency sistem:**

```bash
# macOS
brew install ffmpeg webp
export CGO_CFLAGS_ALLOW="-Xpreprocessor"  # wajib di macOS

# Linux (Ubuntu/Debian)
sudo apt update && sudo apt install ffmpeg webp

# Windows (WSL sangat direkomendasikan)
# - Download FFmpeg: https://www.ffmpeg.org/download.html#build-windows
# - Download libwebp: https://developers.google.com/speed/webp/download
# - Tambahkan folder bin keduanya ke PATH environment variable
```

**Jalankan aplikasi:**

```bash
cd src
cp .env.example .env   # salin template konfigurasi
go run . rest          # mode REST API
# Akses: http://localhost:3000
```

### Opsi C — Download Binary (Tercepat)

1. Download binary dari [GitHub Releases](https://github.com/aldinokemal/go-whatsapp-web-multidevice/releases/latest)
2. Pilih file sesuai OS:
   - Linux x86_64: `whatsapp_linux_amd64`
   - Linux ARM64: `whatsapp_linux_arm64`
   - macOS Intel: `whatsapp_darwin_amd64`
   - macOS Apple Silicon: `whatsapp_darwin_arm64`
   - Windows: `whatsapp_windows_amd64.exe`
3. Jalankan:
   ```bash
   # Linux/macOS
   chmod +x whatsapp_linux_amd64
   ./whatsapp_linux_amd64 rest

   # Windows
   .\whatsapp_windows_amd64.exe rest
   ```

### Opsi D — Docker Run (Produksi)

```bash
docker run --detach \
  --publish=3000:3000 \
  --name=whatsapp \
  --restart=always \
  --volume=$(docker volume create --name=whatsapp):/app/storages \
  aldinokemal2104/go-whatsapp-web-multidevice rest --port=3000
```

---

## 2. Konfigurasi

Buat file `src/.env` (copy dari `src/.env.example`) dan sesuaikan nilainya.

### Pengaturan Aplikasi

```env
APP_PORT=3000                         # Port server (default: 3000)
APP_HOST=0.0.0.0                      # Bind address
APP_DEBUG=false                       # Mode debug (tampilkan log detail)
APP_OS=Chrome                         # Nama perangkat di WhatsApp (misal: MyApp)
APP_BASIC_AUTH=user1:pass1,user2:pass2  # Proteksi Basic Auth (opsional)
APP_BASE_PATH=/gowa                   # Sub-path deployment (opsional)
APP_TRUSTED_PROXIES=0.0.0.0/0         # IP proxy yang dipercaya
```

### Database

```env
# SQLite (default, tidak perlu setup)
DB_URI=file:storages/whatsapp.db?_foreign_keys=on

# PostgreSQL (opsional, untuk skala besar)
DB_URI=postgres://user:password@localhost:5432/whatsapp_db?sslmode=disable
```

### Pengaturan WhatsApp

```env
WHATSAPP_AUTO_REPLY="Maaf, saya sedang tidak aktif."   # Pesan auto-reply (opsional)
WHATSAPP_AUTO_MARK_READ=false          # Tandai pesan masuk sebagai dibaca otomatis
WHATSAPP_AUTO_REJECT_CALL=false        # Tolak panggilan masuk otomatis
WHATSAPP_AUTO_DOWNLOAD_MEDIA=true      # Download media otomatis (gambar, video, dll)
WHATSAPP_ACCOUNT_VALIDATION=true       # Validasi nomor sebelum kirim pesan
WHATSAPP_PRESENCE_ON_CONNECT=unavailable  # Status saat connect: available/unavailable/none
WHATSAPP_CHAT_STORAGE=true             # Simpan riwayat chat ke database lokal
```

### Webhook

```env
# Satu atau beberapa URL (pisahkan dengan koma)
WHATSAPP_WEBHOOK=https://yourapp.com/webhook,https://backup.com/webhook

WHATSAPP_WEBHOOK_SECRET=super-secret-key          # Kunci HMAC SHA256
WHATSAPP_WEBHOOK_INSECURE_SKIP_VERIFY=false        # Skip verifikasi TLS (jangan di produksi)
WHATSAPP_WEBHOOK_INCLUDE_OUTGOING=false            # Sertakan pesan yang Anda kirim

# Filter event (kosong = semua event dikirim)
WHATSAPP_WEBHOOK_EVENTS=message,message.reaction,message.ack
```

### Prioritas Konfigurasi

```
Flag CLI  >  Environment Variable  >  File .env
```

Contoh: `./whatsapp rest --port=8080` akan mengabaikan `APP_PORT` di `.env`.

---

## 3. Cara Menjalankan

### Mode REST API

```bash
# Dari source
cd src && go run . rest

# Dari binary
./whatsapp rest

# Dengan flag custom
./whatsapp rest \
  --port=8080 \
  --debug=true \
  --os="BotSaya" \
  --basic-auth="admin:rahasia123" \
  --webhook="https://yourapp.com/webhook"

# Lihat semua flag yang tersedia
./whatsapp rest --help
```

### Mode MCP Server (untuk AI Agent)

```bash
# Dari source
cd src && go run . mcp

# Dari binary (default: host=localhost, port=8080)
./whatsapp mcp

# Dengan custom host/port
./whatsapp mcp --host=0.0.0.0 --port=9090
```

### Build Binary Sendiri

```bash
cd src

# Linux/macOS
go build -o whatsapp

# Windows
go build -o whatsapp.exe

# Cross-compile untuk Linux dari macOS/Windows
GOOS=linux GOARCH=amd64 go build -o whatsapp_linux
```

---

## 4. Login WhatsApp

Setelah aplikasi berjalan, buka browser dan akses `http://localhost:3000`.

### Login via QR Code

1. Buka `http://localhost:3000`
2. Klik tombol **Login**
3. Scan QR code dengan aplikasi WhatsApp di ponsel
4. Atau via API: `GET /app/login`

### Login via Kode Pairing

Cocok untuk server headless (tanpa UI):

```bash
# Minta kode pairing untuk nomor +628xxxxxxxxxx
GET /app/login-with-code?phone=628xxxxxxxxxx
```

Masukkan 8-digit kode yang muncul ke WhatsApp di ponsel:
**WhatsApp > Perangkat Tertaut > Tautkan Perangkat > Tautkan dengan Nomor Telepon**

### Cek Status Koneksi

```bash
GET http://localhost:3000/app/status
```

Response:
```json
{
  "code": 200,
  "message": "Success",
  "results": {
    "connected": true,
    "phone_number": "628xxxxx",
    "name": "Nama Anda"
  }
}
```

---

## 5. REST API — Panduan Lengkap

Base URL: `http://localhost:3000`

Semua request dan response menggunakan format JSON (kecuali upload file menggunakan `multipart/form-data`).

### 5.1 Format Nomor WhatsApp (JID)

Nomor WhatsApp menggunakan format **JID (Jabber ID)**:

| Tipe | Format | Contoh |
|------|--------|--------|
| Personal | `{kode_negara}{nomor}@s.whatsapp.net` | `6281234567890@s.whatsapp.net` |
| Grup | `{id_grup}@g.us` | `120363xxxxxx@g.us` |
| Newsletter | `{id}@newsletter` | `120363xxxxxx@newsletter` |

> **Penting:** Selalu gunakan kode negara tanpa `+` atau `0` di depan. Indonesia: `62`, bukan `+62` atau `0`.

---

### 5.2 Kirim Pesan

#### Kirim Teks

```bash
POST /send/message
Content-Type: application/json

{
  "phone": "6281234567890@s.whatsapp.net",
  "message": "Halo! Ini pesan dari API.",
  "reply_message_id": ""  # opsional: ID pesan yang dibalas
}
```

#### Mention Seseorang dalam Pesan

```json
{
  "phone": "120363xxxxxx@g.us",
  "message": "Hei @6281234567890 tolong cek ini ya!",
  "mentions": ["6281234567890@s.whatsapp.net"]
}
```

#### Mention Semua Anggota Grup (Ghost Mention)

```json
{
  "phone": "120363xxxxxx@g.us",
  "message": "Pengumuman penting!",
  "mentions": ["@everyone"]
}
```

---

#### Kirim Gambar

```bash
POST /send/image
Content-Type: multipart/form-data

phone=6281234567890@s.whatsapp.net
caption=Foto produk terbaru
image=@/path/ke/gambar.jpg   # upload file
# atau
image=https://example.com/gambar.jpg  # dari URL
compress=true   # kompres gambar sebelum kirim
view_once=false # gambar hilang setelah dilihat sekali
```

---

#### Kirim Video

```bash
POST /send/video
Content-Type: multipart/form-data

phone=6281234567890@s.whatsapp.net
caption=Video promo
video=@/path/ke/video.mp4
compress=true
```

---

#### Kirim Audio

```bash
POST /send/audio
Content-Type: multipart/form-data

phone=6281234567890@s.whatsapp.net
audio=@/path/ke/audio.mp3
```

---

#### Kirim File/Dokumen

```bash
POST /send/file
Content-Type: multipart/form-data

phone=6281234567890@s.whatsapp.net
caption=Dokumen penting
file=@/path/ke/dokumen.pdf
```

---

#### Kirim Sticker

Gambar akan otomatis dikonversi ke format WebP sticker:

```bash
POST /send/sticker
Content-Type: multipart/form-data

phone=6281234567890@s.whatsapp.net
sticker=@/path/ke/gambar.jpg   # JPG, PNG, GIF didukung
# Untuk animated sticker (GIF/WebP): ukuran 512x512px, <500KB, <10 detik
```

---

#### Kirim Lokasi

```bash
POST /send/location
Content-Type: application/json

{
  "phone": "6281234567890@s.whatsapp.net",
  "latitude": -6.2088,
  "longitude": 106.8456,
  "address": "Jakarta, Indonesia"
}
```

---

#### Kirim Kontak

```bash
POST /send/contact
Content-Type: application/json

{
  "phone": "6281234567890@s.whatsapp.net",
  "contact_name": "Budi Santoso",
  "contact_phone": "6289876543210"
}
```

---

#### Kirim Link dengan Preview

```bash
POST /send/link
Content-Type: application/json

{
  "phone": "6281234567890@s.whatsapp.net",
  "link": "https://example.com/artikel",
  "caption": "Baca artikel menarik ini!"
}
```

---

#### Kirim Poll/Voting

```bash
POST /send/poll
Content-Type: application/json

{
  "phone": "120363xxxxxx@g.us",
  "question": "Pilih jadwal meeting:",
  "options": ["Senin 09.00", "Selasa 14.00", "Rabu 10.00"],
  "max_answer": 1
}
```

---

### 5.3 Manajemen Pesan

#### Hapus Pesan (untuk Semua)

```bash
POST /message/{message_id}/revoke
Content-Type: application/json

{
  "phone": "6281234567890@s.whatsapp.net"
}
```

#### Edit Pesan

```bash
POST /message/{message_id}/update
Content-Type: application/json

{
  "phone": "6281234567890@s.whatsapp.net",
  "message": "Isi pesan yang sudah diedit"
}
```

#### React ke Pesan (Emoji)

```bash
POST /message/{message_id}/reaction
Content-Type: application/json

{
  "phone": "6281234567890@s.whatsapp.net",
  "emoji": "👍"
}
```

#### Tandai Pesan Dibaca

```bash
POST /message/{message_id}/read
Content-Type: application/json

{
  "phone": "6281234567890@s.whatsapp.net"
}
```

#### Download Media dari Pesan

```bash
GET /message/{message_id}/download?phone=6281234567890@s.whatsapp.net
```

---

### 5.4 Manajemen Grup

#### Buat Grup Baru

```bash
POST /group
Content-Type: application/json

{
  "title": "Tim Project Alpha",
  "participants": [
    "6281234567890@s.whatsapp.net",
    "6289876543210@s.whatsapp.net"
  ]
}
```

Response menyertakan `group_id` (JID grup).

---

#### Info Grup

```bash
GET /group/info?group_id=120363xxxxxx@g.us
```

---

#### Daftar Anggota Grup

```bash
GET /group/participants?group_id=120363xxxxxx@g.us
```

---

#### Tambah Anggota

```bash
POST /group/participants
Content-Type: application/json

{
  "group_id": "120363xxxxxx@g.us",
  "participants": ["6281234567890@s.whatsapp.net"]
}
```

---

#### Hapus Anggota

```bash
POST /group/participants/remove
Content-Type: application/json

{
  "group_id": "120363xxxxxx@g.us",
  "participants": ["6281234567890@s.whatsapp.net"]
}
```

---

#### Promosi/Demosi Admin

```bash
# Jadikan admin
POST /group/participants/promote
# Turunkan dari admin
POST /group/participants/demote

{
  "group_id": "120363xxxxxx@g.us",
  "participants": ["6281234567890@s.whatsapp.net"]
}
```

---

#### Get Link Undangan Grup

```bash
GET /group/invite-link?group_id=120363xxxxxx@g.us
```

---

#### Bergabung via Link

```bash
POST /group/join-with-link
Content-Type: application/json

{
  "link": "https://chat.whatsapp.com/AbCdEf123456"
}
```

---

#### Export Anggota Grup (CSV)

```bash
GET /group/participants/export?group_id=120363xxxxxx@g.us
```

---

#### Atur Nama / Deskripsi Grup

```bash
# Ganti nama
POST /group/name
{ "group_id": "120363xxxxxx@g.us", "name": "Nama Baru" }

# Ganti deskripsi
POST /group/topic
{ "group_id": "120363xxxxxx@g.us", "topic": "Deskripsi grup baru" }
```

---

#### Mode Announce (Hanya Admin yang Bisa Kirim)

```bash
POST /group/announce
{ "group_id": "120363xxxxxx@g.us", "announce": true }
```

---

#### Keluar dari Grup

```bash
POST /group/leave
{ "group_id": "120363xxxxxx@g.us" }
```

---

### 5.5 Informasi User

```bash
# Info profil sendiri
GET /user/info

# Avatar kontak
GET /user/avatar?phone=6281234567890@s.whatsapp.net

# Ganti nama tampilan
POST /user/pushname
{ "name": "Bot Saya" }

# Daftar kontak
GET /user/my/contacts

# Daftar grup (maks 500)
GET /user/my/groups

# Privasi akun
GET /user/my/privacy

# Cek apakah nomor ada di WhatsApp
GET /user/check?phone=6281234567890@s.whatsapp.net
```

---

### 5.6 Manajemen Chat

```bash
# Daftar chat terbaru
GET /chats

# Riwayat pesan chat tertentu
GET /chat/6281234567890@s.whatsapp.net/messages?limit=20

# Archive chat
POST /chat/6281234567890@s.whatsapp.net/archive
{ "archive": true }

# Pin chat
POST /chat/6281234567890@s.whatsapp.net/pin
{ "pin": true }

# Pesan hilang otomatis
POST /chat/6281234567890@s.whatsapp.net/disappearing
{ "disappearing": 86400 }  # dalam detik: 86400=1hari, 604800=1minggu, 0=nonaktif
```

---

### 5.7 Typing Indicator & Presence

```bash
# Tampilkan "Mengetik..."
POST /send/chat-presence
{
  "phone": "6281234567890@s.whatsapp.net",
  "presence": "composing",  # atau "paused"
  "media": "text"           # atau "audio"
}

# Set status online/offline
POST /send/presence
{ "presence": "available" }  # atau "unavailable"
```

---

### 5.8 Contoh Lengkap dengan cURL

```bash
# Kirim pesan teks
curl -X POST http://localhost:3000/send/message \
  -H "Content-Type: application/json" \
  -d '{"phone":"6281234567890@s.whatsapp.net","message":"Halo dari cURL!"}'

# Kirim gambar dari URL
curl -X POST http://localhost:3000/send/image \
  -F "phone=6281234567890@s.whatsapp.net" \
  -F "caption=Gambar dari internet" \
  -F "image=https://example.com/foto.jpg"

# Kirim file lokal
curl -X POST http://localhost:3000/send/file \
  -F "phone=6281234567890@s.whatsapp.net" \
  -F "caption=Laporan bulanan" \
  -F "file=@/home/user/laporan.pdf"

# Dengan Basic Auth
curl -u "user1:pass1" -X POST http://localhost:3000/send/message \
  -H "Content-Type: application/json" \
  -d '{"phone":"6281234567890@s.whatsapp.net","message":"Pesan terautentikasi"}'
```

---

## 6. MCP Server (untuk AI Agent)

MCP (Model Context Protocol) memungkinkan AI agent seperti **Cursor**, **Claude Desktop**, atau tools lain berinteraksi langsung dengan WhatsApp melalui protokol standar.

### Menjalankan MCP Server

```bash
./whatsapp mcp
# Server berjalan di http://localhost:8080
```

### Konfigurasi di Cursor

Tambahkan ke `~/.cursor/mcp.json` (atau settings MCP di Cursor):

```json
{
  "mcpServers": {
    "whatsapp": {
      "url": "http://localhost:8080/sse"
    }
  }
}
```

### Konfigurasi di Claude Desktop

Tambahkan ke file config Claude Desktop:

```json
{
  "mcpServers": {
    "whatsapp": {
      "url": "http://localhost:8080/sse"
    }
  }
}
```

### Tools yang Tersedia via MCP

| Tool | Fungsi |
|------|--------|
| `whatsapp_connection_status` | Cek status koneksi |
| `whatsapp_login_qr` | Login dengan QR code |
| `whatsapp_login_with_code` | Login dengan kode pairing |
| `whatsapp_logout` | Logout |
| `whatsapp_reconnect` | Reconnect ke WhatsApp |
| `whatsapp_send_text` | Kirim pesan teks |
| `whatsapp_send_image` | Kirim gambar |
| `whatsapp_send_sticker` | Kirim sticker |
| `whatsapp_send_contact` | Kirim kartu kontak |
| `whatsapp_send_link` | Kirim link |
| `whatsapp_send_location` | Kirim lokasi |
| `whatsapp_list_contacts` | Daftar kontak |
| `whatsapp_list_chats` | Daftar chat terbaru |
| `whatsapp_get_chat_messages` | Ambil riwayat pesan |
| `whatsapp_download_message_media` | Download media dari pesan |
| `whatsapp_archive_chat` | Archive chat |
| `whatsapp_group_create` | Buat grup |
| `whatsapp_group_leave` | Keluar grup |
| `whatsapp_group_participants` | Daftar anggota grup |
| `whatsapp_group_manage_participants` | Kelola anggota grup |
| `whatsapp_group_invite_link` | Get/reset link undangan |
| `whatsapp_group_info` | Info grup |
| `whatsapp_group_set_name` | Ganti nama grup |
| `whatsapp_group_set_topic` | Ganti deskripsi grup |
| `whatsapp_group_join_requests` | Lihat permintaan bergabung |

### Endpoint MCP

- **SSE (Server-Sent Events):** `http://localhost:8080/sse`
- **Message endpoint:** `http://localhost:8080/message`

> **Catatan:** Anda hanya bisa menjalankan satu mode (REST atau MCP) dalam satu instance. Untuk menjalankan keduanya, gunakan dua instance terpisah.

---

## 7. Webhook

Webhook memungkinkan aplikasi Anda menerima notifikasi real-time saat ada event di WhatsApp (pesan masuk, reaksi, dll).

### Konfigurasi Webhook

```env
WHATSAPP_WEBHOOK=https://yourapp.com/api/webhook
WHATSAPP_WEBHOOK_SECRET=kunci-rahasia-anda
WHATSAPP_WEBHOOK_EVENTS=message,message.reaction,group.participants
```

### Event yang Tersedia

| Event | Trigger |
|-------|---------|
| `message` | Pesan teks/media/kontak/lokasi masuk |
| `message.reaction` | Emoji reaction pada pesan |
| `message.revoked` | Pesan dihapus oleh pengirim |
| `message.edited` | Pesan diedit |
| `message.ack` | Konfirmasi terkirim/dibaca |
| `message.deleted` | Pesan dihapus dari sisi Anda |
| `group.participants` | Anggota grup bergabung/keluar/naik/turun |
| `group.joined` | Anda ditambahkan ke grup |
| `newsletter.joined` | Berlangganan newsletter |
| `newsletter.left` | Berhenti berlangganan |
| `newsletter.message` | Pesan baru di newsletter |
| `call.offer` | Panggilan masuk |

### Format Payload Webhook

```json
{
  "event": "message",
  "device_id": "628xxxxxxx@s.whatsapp.net",
  "payload": {
    "id": "3EB0XXXXXXXXXXXXXX",
    "from": "6281234567890@s.whatsapp.net",
    "pushname": "Budi Santoso",
    "message": {
      "conversation": "Halo!"
    },
    "timestamp": 1714000000,
    "is_from_me": false,
    "is_group": false
  }
}
```

### Verifikasi HMAC (Keamanan Webhook)

Setiap request webhook menyertakan header `X-Hub-Signature-256`. Verifikasi di server Anda:

```python
# Python
import hmac, hashlib

def verify_webhook(payload_body: bytes, signature_header: str, secret: str) -> bool:
    expected = "sha256=" + hmac.new(
        secret.encode(), payload_body, hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature_header)
```

```javascript
// Node.js
const crypto = require('crypto');

function verifyWebhook(payloadBody, signatureHeader, secret) {
  const expected = 'sha256=' + crypto
    .createHmac('sha256', secret)
    .update(payloadBody)
    .digest('hex');
  return crypto.timingSafeEqual(
    Buffer.from(expected),
    Buffer.from(signatureHeader)
  );
}
```

---

## 8. Multi-Device

Sejak v8, GoWA mendukung mengelola **beberapa akun WhatsApp** dalam satu server.

### Tambah Device Baru

```bash
# Buat device baru
POST /devices
{ "device_id": "akun-bisnis" }

# Login device tersebut
GET /devices/akun-bisnis/login

# Cek status
GET /devices/akun-bisnis/status
```

### Kirim Pesan dari Device Tertentu

Tambahkan header `X-Device-Id` atau query parameter `device_id`:

```bash
# Via header
curl -H "X-Device-Id: akun-bisnis" \
  -X POST http://localhost:3000/send/message \
  -d '{"phone":"6281234567890@s.whatsapp.net","message":"Dari akun bisnis"}'

# Via query parameter
curl -X POST "http://localhost:3000/send/message?device_id=akun-bisnis" \
  -d '{"phone":"6281234567890@s.whatsapp.net","message":"Dari akun bisnis"}'
```

> **Catatan:** Jika hanya ada satu device terdaftar, tidak perlu menyertakan `device_id` — otomatis menggunakan device tersebut.

### WebSocket per Device

```javascript
const ws = new WebSocket('ws://localhost:3000/ws?device_id=akun-bisnis');
ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log('Event dari akun bisnis:', data);
};
```

### Manajemen Device via API

```bash
# Daftar semua device
GET /devices

# Info device tertentu
GET /devices/akun-bisnis

# Logout device
POST /devices/akun-bisnis/logout

# Hapus device
DELETE /devices/akun-bisnis

# Reconnect device
POST /devices/akun-bisnis/reconnect
```

---

## 9. Integrasi Chatwoot

Chatwoot adalah platform CRM open-source. GoWA dapat mensinkronisasi pesan WhatsApp ke Chatwoot secara otomatis.

### Konfigurasi

```env
CHATWOOT_ENABLED=true
CHATWOOT_URL=https://app.chatwoot.com          # atau URL self-hosted Anda
CHATWOOT_API_TOKEN=token-dari-chatwoot          # Profile > Access Token
CHATWOOT_ACCOUNT_ID=12345                       # ID akun Chatwoot
CHATWOOT_INBOX_ID=67890                         # ID inbox yang dibuat untuk WhatsApp
CHATWOOT_DEVICE_ID=628xxxxxxx@s.whatsapp.net   # JID device (untuk multi-device)

# Import riwayat chat saat pertama kali connect
CHATWOOT_IMPORT_MESSAGES=true
CHATWOOT_DAYS_LIMIT_IMPORT_MESSAGES=7          # Import 7 hari terakhir
```

### Setup di Chatwoot

1. Buat akun di [Chatwoot](https://app.chatwoot.com) atau deploy sendiri
2. Buat **Inbox** baru bertipe "API"
3. Salin **Inbox ID** dan **API Access Token**
4. Isi nilai tersebut di `.env`
5. Restart GoWA

---

## 10. Tips Produksi

### Docker Compose untuk Produksi

Buat `docker-compose.prod.yml`:

```yaml
services:
  whatsapp:
    image: aldinokemal2104/go-whatsapp-web-multidevice
    container_name: whatsapp
    restart: always
    ports:
      - "3000:3000"
    volumes:
      - whatsapp_storage:/app/storages
      - whatsapp_statics:/app/statics
    environment:
      - APP_PORT=3000
      - APP_BASIC_AUTH=admin:password_kuat_anda
      - APP_OS=BotPerusahaan
      - WHATSAPP_WEBHOOK=https://api.perusahaan.com/webhook
      - WHATSAPP_WEBHOOK_SECRET=kunci_rahasia_sangat_panjang
      - WHATSAPP_WEBHOOK_EVENTS=message,message.reaction
      - WHATSAPP_AUTO_REJECT_CALL=true
    command:
      - rest
      - --port=3000

volumes:
  whatsapp_storage:
  whatsapp_statics:
```

```bash
docker-compose -f docker-compose.prod.yml up -d
```

### Deployment di Balik Reverse Proxy (Nginx)

Contoh konfigurasi Nginx:

```nginx
server {
    listen 443 ssl;
    server_name wa.perusahaan.com;

    ssl_certificate /etc/letsencrypt/live/wa.perusahaan.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/wa.perusahaan.com/privkey.pem;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Tambahkan ke `.env`:
```env
APP_TRUSTED_PROXIES=127.0.0.1
```

### Deployment di Sub-Path

Jika ingin akses di `https://domain.com/whatsapp`:

```env
APP_BASE_PATH=/whatsapp
```

```nginx
location /whatsapp {
    proxy_pass http://localhost:3000/whatsapp;
}
```

### Keamanan

- **Selalu aktifkan Basic Auth** di lingkungan produksi: `APP_BASIC_AUTH=user:password_kuat`
- **Gunakan webhook secret** yang kuat dan verifikasi HMAC di sisi penerima
- **Jangan nonaktifkan TLS** (`WHATSAPP_WEBHOOK_INSECURE_SKIP_VERIFY=false` di produksi)
- **Simpan session data** dengan volume Docker agar tidak perlu scan QR ulang setelah restart

### Storage & Backup

Data penting tersimpan di:
- `storages/` — database SQLite (sesi WhatsApp, chat history)
- `statics/media/` — media yang didownload
- `statics/senditems/` — file yang pernah dikirim

Backup rutin folder `storages/` untuk mencegah kehilangan sesi.

### Monitoring

```bash
# Cek log container
docker logs -f whatsapp

# Cek status koneksi via API
curl http://localhost:3000/app/status

# Health check endpoint (untuk load balancer)
curl http://localhost:3000/app/status
```

---

## Referensi Cepat

| Aksi | Method | Endpoint |
|------|--------|----------|
| Status koneksi | GET | `/app/status` |
| Login QR | GET | `/app/login` |
| Logout | GET | `/app/logout` |
| Kirim pesan teks | POST | `/send/message` |
| Kirim gambar | POST | `/send/image` |
| Kirim video | POST | `/send/video` |
| Kirim file | POST | `/send/file` |
| Kirim audio | POST | `/send/audio` |
| Kirim sticker | POST | `/send/sticker` |
| Kirim lokasi | POST | `/send/location` |
| Kirim kontak | POST | `/send/contact` |
| Kirim poll | POST | `/send/poll` |
| Hapus pesan | POST | `/message/:id/revoke` |
| React pesan | POST | `/message/:id/reaction` |
| Daftar chat | GET | `/chats` |
| Riwayat chat | GET | `/chat/:jid/messages` |
| Buat grup | POST | `/group` |
| Daftar device | GET | `/devices` |

**Dokumentasi API lengkap:** Buka [SwaggerEditor](https://editor.swagger.io) dan import file `docs/openapi.yaml` dari project ini.

**Webhook payload lengkap:** Lihat `docs/webhook-payload.md`

**Integrasi Chatwoot:** Lihat `docs/chatwoot.md`

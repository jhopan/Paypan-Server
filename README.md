# Paypan-Server

Payment gateway QRIS mandiri — satu binary Go, tanpa framework. Menerima notifikasi pembayaran dari [NotifListen-Payment](https://github.com/jhopan/NotifListen-Payment) (app Android yang merelay notif e-wallet), mencocokkan ke order, dan mengubah QRIS statis menjadi QRIS dinamis per order.

## Arsitektur

```
HP merchant (app NotifListen) ──POST /api/notif──┐
                                                 ▼
                                     ┌──────────────────┐
Customer → scan QRIS dinamis ──────► │  Paypan-Server    │
                                     │  - ingest + match │
Dashboard admin ◄──────────────────► │  - SQLite         │
Bot Telegram    ◄──────────────────► │  - QR dinamis     │
Webhook (site lain) ◄─────────────── └──────────────────┘
```

## Fitur

- **QRIS dinamis dari statis** — inject tag 54 (nominal) + recompute CRC16 ke payload QRIS statis; satu QRIS untuk semua nominal
- **Kode unik berurutan 001-999** — `total = harga + kode`, unik di antara semua order pending
- **Match exact** — notif `amount` dicocokkan ke order pending dengan total identik; satu pembayaran = satu order (race-safe)
- **Expiry 5 menit** — kode otomatis kembali ke pool
- **Web admin** (`/admin`) — kelola token per-aplikasi (scope notif/order), QRIS (gambar+payload tersinkron), webhook, Telegram, audit log
- **Multi-app token** — tiap app punya token sendiri (rotate/on-off/hapus)
- **Webhook** — POST `order.paid` + **HMAC signature** + retry
- **Telegram notify** — LUNAS / unmatched, multi-chat, retry
- **Keamanan** — rate limit login & API, security headers, HTML escape, audit log, backup DB otomatis tiap 6 jam

## API

| Endpoint | Auth | Fungsi |
|---|---|---|
| `POST /api/notif` | Bearer (scope notif) | ingest notif dari device |
| `POST /api/order` | Bearer (scope order) | buat order → `{id, code, total, qr}` |
| `GET /api/order/{id}` | Bearer | status order |
| `GET /pay/{id}` | publik | halaman checkout + polling |
| `GET /pay/{id}/qr.png` | publik | gambar QR dinamis |
| `GET /kasir` | publik | halaman kasir (input nominal → QR) |
| `/admin` | session | dashboard admin |

## Setup

```bash
# 1. siapkan payload QRIS statis (scan QR lo dengan scanner, simpan raw text)
echo "000201010211..." > qris_base.txt

# 2. jalankan
./paypan-server -addr :9090

# 3. buka admin
# http://localhost:9090/admin  (default: admin / admin123 — segera ganti)
```

Flag: `-addr`, `-db`, `-token` (seed token master), `-tgtoken`, `-tgchat`.
Setting via web tersimpan di SQLite dan menimpa flag.

## Deploy (production)

- Jalankan di balik reverse proxy / CF tunnel (HTTPS wajib)
- Ganti password admin, rotate token master
- DB + backup otomatis di `paypan.db` dan `backup/`
- Systemd unit contoh:

```ini
[Service]
ExecStart=/opt/paypan/paypan-server -addr :9090
WorkingDirectory=/opt/paypan
Restart=always
User=paypan
```

## Verifikasi webhook (penerima)

```python
import hmac, hashlib
expected = hmac.new(secret.encode(), body, hashlib.sha256).hexdigest()
assert hmac.compare_digest(expected, request.headers["X-Paypan-Signature"])
```

## Lisensi
MIT

---

## Kredit

**Dikembangkan oleh [JhopanStore](https://github.com/jhopan)**

© 2026 JhopanStore. Dibangun dengan bantuan AI (Hermes Agent).

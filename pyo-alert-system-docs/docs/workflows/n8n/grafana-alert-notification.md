# Dokumentasi Workflow: Grafana Alert Notification Engine (n8n)

Dokumentasi teknis alur pengiriman notifikasi alert Grafana secara dinamis ke multi-channel (WhatsApp & Telegram) menggunakan n8n.

## 1. System Overview

Workflow ini berfungsi sebagai middleware pencatat dan pengirim notifikasi dari Grafana Alerting. Daftar penerima aktif ditarik dari database MySQL, dialirkan sesuai channel target (`WHATSAPP` / `TELEGRAM`), diformat berdasarkan jenis metrik serta status alert (`FIRING` / `RESOLVED`), lalu dikirim via HTTP API.

```
[ Grafana Alert ]
       │
       ▼ (POST Webhook)
[ Webhook Node ]
       │
       ▼
[ MySQL Query ] ──► Filter active recipients (Category: GRAFANA)
       │
       ▼
[ Wait Node ] ────► Pause 2 Seconds
       │
       ▼
[ Switch Node ] ──► Dynamic Channel Routing
       ├── Output 0 (TELEGRAM) ──► [ Parsing Tele ] ──► [ HTTP Telegram ]
       └── Output 1 (WHATSAPP) ──► [ Parsing WA ]   ──► [ HTTP WA ]
```

## 2. Rincian Konfigurasi Node

### 2.1 Webhook Node

| Parameter | Nilai |
|-----------|-------|
| Endpoint URL | `http://172.24.82.64:5678/webhook/grafana-pyo` |
| HTTP Method | `POST` |
| Authentication | None |
| Response Mode | Immediately (200 OK langsung dikembalikan ke Grafana tanpa menunggu pengerjaan selesai) |

### 2.2 MySQL Node (`executeQuery`)

Mengambil target penerima notifikasi secara dinamis berdasarkan kategori `GRAFANA`. Struktur tabel: [param_global_notifications](../../database/param_global_notifications.md).

```sql
SELECT CAST(chat_id AS CHAR) AS chat_id, recipient_name, channel
FROM param_global_notifications
WHERE category = 'GRAFANA' AND is_active = 1;
```

### 2.3 Wait Node

- **Duration:** 2 detik
- **Fungsi (menurut dokumentasi awal):** menunda eksekusi sejenak untuk memastikan sinkronisasi data stream sebelum routing.

> **TODO: perlu verifikasi.** Penjelasan "sinkronisasi" kurang meyakinkan secara teknis, karena data dari node MySQL sudah lengkap saat node berikutnya berjalan. Kemungkinan fungsi sebenarnya adalah *rate limiting* ke API WhatsApp/Telegram. Konfirmasi dengan pembuat workflow.

### 2.4 Switch Node (Routing)

Mengarahkan cabang eksekusi berdasarkan field `channel` hasil query MySQL.

| Parameter | Nilai |
|-----------|-------|
| Mode | Rules |
| Rule 0 (Output 0) | `{{ $json.channel }}` equals `TELEGRAM` |
| Rule 1 (Output 1) | `{{ $json.channel }}` equals `WHATSAPP` |

## 3. Logika Parsing & Transformation (Parsing WA)

Node **Parsing WA** mengekstrak payload Grafana, memformat pesan teks WhatsApp, merincikan metrik, memetakan IP khusus, menghitung Time to Recovery (TTR), dan membersihkan nomor penerima.

### 3.1 Mapping Metrik & Sub-Sistem

| Metric Type (`labels.metric_type`) | Label Tampilan | Unit | Decimals |
|------------------------------------|----------------|------|----------|
| `success_rate` | Success Rate | `%` | 2 |
| `failed_count` | Jumlah Gagal | (kosong) | 0 |
| `rc99_count` | Total RC 99 | (kosong) | 0 |
| Fallback (legacy) | `values.F`, `values.B`, `values.A` | variatif | - |

### 3.2 Mapping IP RC 99 (khusus `rc99_count`)

Jika `labels.metric_type = 'rc99_count'`, key dinamis berikut dipetakan ke penamaan IP internal:

| Key | Nama |
|-----|------|
| R105 | BL105 |
| R106 | BL106 |
| R148 | BL148 |
| R160 | BL160 |
| R161 | BL161 |
| R44 | BL44 |
| R45 | BL45 |
| R248 | BL248 |

### 3.3 Kalkulasi Time to Recovery (TTR)

Status `RESOLVED` menghitung selisih durasi antara `startsAt` dan `endsAt`:

```
TTR = endsAt - startsAt
```

Output diformat ke kombinasi jam, menit, dan detik (contoh: `1 jam 12 menit 30 detik`).

> **TODO: perlu verifikasi.** Pastikan `startsAt` dari Grafana mewakili waktu alert mulai *firing*, bukan waktu evaluasi pertama. Ini menentukan akurasi TTR.

### 3.4 Format Pesan Output WhatsApp

**Contoh pesan FIRING:**

```
🚨 *High Failure Rate - PYO Monitoring*
Status: *FIRING*

*Service:* FLIP Remittance
*Jumlah Gagal:* 150
*Waktu:* 28/10/2026, 23.00.00
*Summary:* Service mengalami lonjakan error.
*Description:* Exceeds threshold > 100
```

**Contoh pesan RESOLVED:**

```
✅ *High Failure Rate - PYO Monitoring*
Status: *RESOLVED*

*Service:* FLIP Remittance
*Jumlah Gagal:* Normal
*Waktu:* 28/10/2026, 23.05.00
*Time to Recovery:* 5 menit 0 detik
```

### 3.5 Pembersihan Nomor Telepon Target

Setiap nomor telepon disesuaikan dengan standar WhatsApp API:

1. Menghapus karakter non-digit.
2. Mengubah prefix `0` menjadi `62`.
3. Menambahkan suffix `@s.whatsapp.net`.

## 4. Delivery Node (HTTP WA)

| Parameter | Nilai |
|-----------|-------|
| Endpoint URL | `http://172.24.82.64:8089/send/message` |
| HTTP Method | `POST` |
| Authentication | Basic Auth (credential: `PYO WA Credential`) |
| Body Format | JSON |

Body parameters:

| Key | Value |
|-----|-------|
| `phone` | `{{ $json.phone }}` |
| `message` | `{{ $json.message }}` |

## 5. Known Limitations & TODO

- **Webhook tanpa autentikasi.** Siapa pun di jaringan internal yang bisa menjangkau `172.24.82.64:5678` dapat memicu notifikasi palsu. Pertimbangkan header secret atau pembatasan jaringan.
- **Sisi Telegram belum terdokumentasi.** Node `Parsing Tele` dan `HTTP Telegram` belum dijelaskan: format pesan, endpoint, dan credential-nya.
- **Wait Node 2 detik**: tujuan sebenarnya perlu dikonfirmasi (lihat 2.3).
- **Validasi TTR**: lihat 3.3.
- **Penanganan error** (API WA/Telegram gagal, query kosong) belum terdokumentasi.

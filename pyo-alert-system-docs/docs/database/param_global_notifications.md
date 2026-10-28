# Tabel `param_global_notifications`

Daftar penerima notifikasi. Digunakan oleh workflow [Grafana Alert Notification Engine](../workflows/n8n/grafana-alert-notification.md).

## Kolom yang diketahui

| Kolom | Keterangan |
|-------|------------|
| `chat_id` | ID chat Telegram / nomor WhatsApp penerima. Di-cast ke `CHAR` saat query agar tidak terpotong sebagai angka besar. |
| `recipient_name` | Nama penerima. |
| `channel` | `TELEGRAM` atau `WHATSAPP`. |
| `category` | Kategori alert. Workflow ini memakai `GRAFANA`. |
| `is_active` | `1` = aktif menerima notifikasi. |

## Query yang dipakai workflow

```sql
SELECT CAST(chat_id AS CHAR) AS chat_id, recipient_name, channel
FROM param_global_notifications
WHERE category = 'GRAFANA' AND is_active = 1;
```

## TODO: perlu verifikasi
- Tipe data, primary key, dan index lengkap.
- Kategori lain selain `GRAFANA`.
- Prosedur menambah/menonaktifkan penerima.

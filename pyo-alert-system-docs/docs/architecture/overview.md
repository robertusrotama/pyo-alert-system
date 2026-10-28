# Arsitektur Alert System PYO

Ringkasan alur yang saat ini terdokumentasi.

```
Grafana Alerting
      │  POST webhook
      ▼
n8n (Grafana Alert Notification Engine)
      │  baca penerima aktif dari MySQL
      ▼
WhatsApp API  /  Telegram API
```

Detail: [Grafana Alert Notification Engine](../workflows/n8n/grafana-alert-notification.md).

## TODO: perlu verifikasi
- Daftar alert rule Grafana dan `metric_type` yang dipakai.
- Sumber datasource alert.
- Alert source selain Grafana (jika ada).

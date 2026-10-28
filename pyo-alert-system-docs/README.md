# PYO Alert System

Dokumentasi teknis untuk Alert System di PYO: alur notifikasi, workflow, database, dan runbook.

> **Repo private.** Dokumen berisi alamat internal dan skema database. Jangan dijadikan public tanpa menyamarkan isinya, dan jangan commit password, token, atau export credential.

## Peta Dokumen

| Area | Dokumen |
|------|---------|
| Arsitektur | [Overview alur end-to-end](docs/architecture/overview.md) |
| Workflow n8n | [Grafana Alert Notification Engine](docs/workflows/n8n/grafana-alert-notification.md) |
| Database | [param_global_notifications](docs/database/param_global_notifications.md) |
| Runbook | [docs/runbooks/](docs/runbooks/) (belum ada isi) |

## Cara Menambah Dokumen

1. Salin template di [docs/_templates/workflow-template.md](docs/_templates/workflow-template.md).
2. Taruh di folder sesuai area, lalu tambahkan ke tabel di atas.
3. Tandai informasi yang belum pasti dengan `TODO: perlu verifikasi`. Jangan menulis tebakan sebagai fakta.

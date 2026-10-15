# Alert Telegram dari Rancher Monitoring

Fleet memasang Rancher Alerting Drivers `106.0.0` di cluster downstream
`altera-rke`, namespace `cattle-monitoring-system`, dan mengaktifkan Sachet,
driver yang dipakai untuk Telegram. Nama cluster ini mengikuti konfigurasi
`clusterName` pada bundle Rancher Monitoring. Tunggu bundle
`monitoring-monitoring-15-alerting-drivers` berstatus Ready sebelum mengatur
receiver.

## Lokasi konfigurasi

Konfigurasi bot berada di ConfigMap **`rancher-alerting-drivers-sachet`** pada
namespace **`cattle-monitoring-system`** di cluster **`altera-rke`**. Pastikan
konteks `kubectl` menunjuk ke cluster itu sebelum mengedit:

```sh
kubectl config current-context
kubectl -n cattle-monitoring-system edit configmap rancher-alerting-drivers-sachet
```

Edit key `config.yaml` pada ConfigMap tersebut. Jangan mengedit ConfigMap di
cluster Rancher local atau cluster lain. Jika ConfigMap belum ada, pastikan
Fleet bundle sudah Ready lalu periksa kembali konteks dan namespace.

## Mengatur bot Telegram

Buat bot Telegram dan dapatkan bot token serta chat ID tujuan. Ganti isi key
`config.yaml` dengan konfigurasi berikut, ganti nilai placeholder, dan
sesuaikan nama receiver dengan nama penuh yang dipakai Alertmanager:

```yaml
providers:
  telegram:
    token: "<TELEGRAM_BOT_TOKEN>"
receivers:
  - name: "cattle-monitoring-system/<alertmanagerconfig-name>/<receiver-name>"
    provider: "telegram"
    to:
      - "<TELEGRAM_CHAT_ID>"
    text: '{{ .GroupLabels.alertname }} @ {{ .Labels.instance }}: {{ .Status | toUpper }}'
```

> **Penyimpanan token:** chart upstream membaca konfigurasi ini dari ConfigMap,
> bukan Kubernetes Secret. Token tidak dimasukkan ke Git, tetapi akan tersimpan
> sebagai data ConfigMap di cluster. Sachet memiliki config reloader sehingga
> perubahan ConfigMap dimuat otomatis.

## Apakah `values.yaml` perlu diubah?

Tidak untuk mengisi token atau chat ID. File
`monitoring/15-alerting-drivers/values.yaml` hanya mengaktifkan Sachet dan
mematikan prom2teams. Biarkan kredensial di ConfigMap cluster seperti langkah
di atas; jangan commit token ke file values.

Chart membuat ConfigMap saat instalasi awal dan mempertahankannya saat upgrade.
Karena itu, perubahan pada `values.yaml` tidak memperbarui ConfigMap yang sudah
ada. Edit ConfigMap langsung di cluster. Jika mengubah opsi instalasi Sachet,
barulah ubah values lalu biarkan Fleet menerapkan upgrade.

## Menghubungkan Alertmanager

Di Rancher, buka **Monitoring → Alerting → AlertmanagerConfigs**, buat atau
pilih AlertmanagerConfig, lalu tambahkan receiver Webhook dengan tipe **SMS**.
Target-nya:

```text
http://rancher-alerting-drivers-sachet.cattle-monitoring-system.svc:9876/alert
```

Buat route pada AlertmanagerConfig yang mengarah ke receiver tersebut dan atur
matcher sesuai alert yang ingin dikirim. Nama konfigurasi Sachet harus mengikuti
format `<namespace>/<alertmanagerconfig-name>/<receiver-name>` seperti contoh
di atas. Dokumentasi Rancher menjelaskan bahwa Telegram memakai driver SMS dan
receiver Webhook bertipe SMS.

Referensi: [Rancher Receiver Configuration](https://ranchermanager.docs.rancher.com/v2.12/reference-guides/monitoring-v2-configuration/receivers).

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

Bundle Fleet `monitoring-monitoring-16-telegram-alerting` membuat resource
`AlertmanagerConfig` bernama `telegram-alerts` di namespace
`cattle-monitoring-system`. Receiver Webhook-nya bernama `telegram` dan
mengarah ke service Sachet. Bundle ini menunggu stack monitoring dan Alerting
Drivers siap.

Route di AlertmanagerConfig ini sengaja hanya mencocokkan label
`test=telegram`. Dengan demikian, ia tidak mengirim seluruh alert cluster ke
Telegram. Nama receiver yang harus dimasukkan ke ConfigMap Sachet adalah:

```yaml
name: "cattle-monitoring-system/telegram-alerts/telegram"
```

Formatnya `<namespace>/<alertmanagerconfig-name>/<receiver-name>`. Resource
AlertmanagerConfig dipilih oleh Rancher Monitoring 106 karena selector chart
menerima konfigurasi di namespace cluster monitoring.

## Mengirim alert uji

Setelah mengisi ConfigMap Sachet dan bundle AlertmanagerConfig berstatus Ready,
pastikan konteks `kubectl` masih menunjuk ke cluster `altera-rke`, lalu terapkan
contoh rule uji:

```sh
kubectl apply -f examples/telegram-test-alert.yaml
```

Rule ini memakai `vector(1)`, sehingga langsung firing dan berulang selama
resource ada. Route `test=telegram` akan mengirimkannya ke Telegram. Hapus rule
setelah pesan uji diterima agar alert berhenti:

```sh
kubectl delete -f examples/telegram-test-alert.yaml
```

Rule contoh sengaja disimpan di `examples/`, di luar bundle Fleet, sehingga
tidak dipasang otomatis.

Referensi: [Rancher Receiver Configuration](https://ranchermanager.docs.rancher.com/v2.12/reference-guides/monitoring-v2-configuration/receivers).

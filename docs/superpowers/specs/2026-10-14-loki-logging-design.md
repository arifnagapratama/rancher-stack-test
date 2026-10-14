# Desain integrasi Loki

## Tujuan

Menambahkan Loki bersama stack Rancher Monitoring dan Rancher Logging yang sudah ada, mengarahkan log cluster ke Loki, dan mendaftarkan Loki sebagai datasource Grafana agar log dapat dicari melalui Grafana Explore.

## Konteks saat ini

- Rancher Monitoring dipasang dari `monitoring/10-stack`, di namespace `cattle-monitoring-system`.
- Grafana merupakan bagian dari Rancher Monitoring dan menyimpan datanya pada PVC 5Gi.
- Rancher Logging beserta chart CRD-nya dipasang di `cattle-logging-system` dengan versi chart `106.0.8+up4.10.0-rancher.24`.
- Rancher Logging menggunakan Logging Operator jalur versi 4.10, yang mendukung output Loki dan alur log cluster.
- Versi Grafana yang digunakan diperkirakan 11.5.5. Dukungan datasource Loki sudah tersedia di Grafana, jadi plugin tambahan tidak diperlukan.

## Desain

### Pemasangan Loki

Tambahkan bundle Fleet untuk Loki di `monitoring/40-loki`. Gunakan chart Loki Grafana Community versi `18.7.6` (versi aplikasi Loki `3.7.6`) dalam mode monolitik/single-binary. Jalankan di namespace `cattle-logging-system` agar dekat dengan Rancher Logging. Repo chart ini merupakan jalur OSS yang dipelihara setelah migrasi chart Loki.

Gunakan penyimpanan filesystem pada PVC `ReadWriteOnce` berukuran 20Gi dan simpan log selama 15 hari. Ini mengikuti periode retensi monitoring dan pola storage yang sudah ada. Loki hanya dapat diakses dari dalam cluster; ingress eksternal dan lapisan autentikasi tidak termasuk perubahan ini.

Konfigurasi bundle dan nilai chart Loki disimpan di repo sebagai `monitoring/40-loki/fleet.yaml` dan `monitoring/40-loki/values.yaml`. Resource Kubernetes seperti Deployment, Service, dan PVC dirender oleh chart Loki yang versinya dipatok di Fleet; konfigurasi values yang menghasilkan resource tersebut tetap dikelola dari repo.

### Pengiriman log

Tambahkan bundle Fleet di `monitoring/50-logging-pipeline` yang berisi:

- `ClusterOutput` di `cattle-logging-system` untuk mengirim log ke service Loki di dalam cluster.
- `ClusterFlow` yang memilih log dari semua namespace dan meneruskannya ke output tersebut.

Bundle harus bergantung pada bundle Rancher Logging dan Loki agar CRD Logging serta service Loki tersedia sebelum pipeline diterapkan. Pertahankan metadata Kubernetes dan gunakan buffer Fluentd dengan batas serta retry, supaya gangguan sementara pada Loki tidak langsung membuang log yang antre.

Simpan deklarasi `ClusterOutput` dan `ClusterFlow` sebagai manifest YAML di bundle tersebut. Dengan demikian, alur pengiriman log dikelola melalui Git dan Fleet, bukan dibuat manual lewat Rancher UI.

### Datasource Grafana

Tambahkan datasource terprovisi bernama `Loki` ke `monitoring/10-stack/values.yaml`, dengan alamat service query Loki di dalam cluster. Jangan jadikan datasource ini default agar datasource Prometheus yang ada tetap menjadi default Grafana. Setelah dipasang, Loki dapat langsung dipilih di Explore dan dashboard tanpa konfigurasi manual di Grafana.

Semua konfigurasi yang ditambahkan disimpan di repo sebagai YAML dan diterapkan oleh Fleet. Tidak ada resource tambahan yang perlu dibuat manual melalui Rancher UI.

## Perilaku saat terjadi gangguan dan batasan

- Loki berjalan sebagai satu binary dengan satu PVC; konfigurasi ini bukan backend logging high availability.
- Jika Loki tidak tersedia, Fluentd akan mencoba lagi menggunakan buffer yang dikonfigurasi. Buffer yang penuh tetap dapat menyebabkan log hilang, sehingga ukuran buffer dan pengaturan retry perlu dibatasi.
- Kapasitas PVC awal ditetapkan 20Gi. Jika kebutuhan melebihi kapasitas tersebut, PVC perlu diperbesar dan migrasi storage mungkin diperlukan, bergantung pada storage class cluster.
- Perubahan ini mengirim log container cluster yang dikoleksi Rancher Logging. Parsing khusus, dashboard retensi, aturan alert, dan akses eksternal tidak termasuk.

## Kriteria validasi

- Fleet dapat merender dan menerima konfigurasi Loki, pipeline logging, dan datasource.
- Loki siap digunakan dan PVC-nya terikat.
- Rancher Logging melaporkan `ClusterOutput` dan `ClusterFlow` aktif serta meneruskan record ke Loki.
- Grafana melaporkan datasource Loki sehat dan Grafana Explore dapat mencari log Kubernetes terbaru.

## Referensi

- [Migrasi chart Helm Loki](https://grafana.com/docs/loki/latest/setup/upgrade/)
- [Memasang chart Helm Loki monolitik](https://grafana.com/docs/loki/latest/setup/install/helm/install-monolithic/)
- [Datasource Loki Grafana](https://grafana.com/docs/grafana/latest/datasources/loki/)
- [Tipe output Logging Operator 4.10](https://kube-logging.dev/4.10/docs/configuration/crds/v1beta1/output_types/)

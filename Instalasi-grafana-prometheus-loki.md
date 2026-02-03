# 🔭 Kubernetes Observability Stack Setup
> **Prometheus • Grafana • Loki • Promtail**

Dokumen ini berisi panduan teknis langkah-demi-langkah untuk mengimplementasikan *Full Observability Stack* pada cluster Kubernetes. Stack ini mencakup pemantauan metrik (Metrics) dan sentralisasi log (Logging).

## 📋 Daftar Isi
- [Prasyarat](#-prasyarat)
- [1. Persiapan Helm Package Manager](#1-persiapan-helm-package-manager)
- [2. Instalasi Metrics Stack (Prometheus & Grafana)](#2-instalasi-metrics-stack-prometheus--grafana)
- [3. Akses Dashboard Grafana](#3-akses-dashboard-grafana)
- [4. Instalasi Logging Stack (Loki & Promtail)](#4-instalasi-logging-stack-loki--promtail)
- [5. Integrasi Data Source](#5-integrasi-data-source)
- [6. Konfigurasi Lanjutan (Log Volume Enable)](#6-konfigurasi-lanjutan-log-volume-enable)

---

## 🛠 Prasyarat
Sebelum memulai, pastikan Anda memiliki akses ke cluster dan terminal:
* **Kubernetes Cluster** (Local/Cloud) sudah berjalan.
* **Kubectl** sudah terkonfigurasi ke cluster tujuan.
* **Koneksi Internet** stabil untuk menarik image container.

---

## 1. Persiapan Helm Package Manager
Helm digunakan sebagai "App Store" untuk mempermudah instalasi aplikasi kompleks di Kubernetes.

**Cek instalasi Helm:**

helm version
Jika belum terinstall (Linux/WSL):

```
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3| bash
```
## 2. Instalasi Metrics Stack (Prometheus & Grafana)
Kita menggunakan paket kube-prometheus-stack yang mencakup Prometheus, Grafana, Node Exporter, dan Kube State Metrics dalam satu paket.

### 2.1 Tambahkan Repository

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```
```
helm repo update
```
### 2.2 Install Stack
Kita akan menempatkan stack ini di namespace khusus bernama monitoring.

Buat namespace

```
kubectl create namespace monitoring
```

> #### Install paket (Proses ini memakan waktu 1-3 menit)

```
helm install monitoring-stack prometheus-community/kube-prometheus-stack -n monitoring
```
Verifikasi: Pastikan semua Pod berstatus Running:
```
kubectl get pods -n monitoring -w
```
## 3. Akses Dashboard Grafana
Secara default, Grafana tidak diekspos ke publik demi keamanan. Kita menggunakan metode Port-Forwarding untuk mengaksesnya.

### 3.1 Port Forwarding
Jalankan perintah ini di terminal terpisah (biarkan berjalan):

```
kubectl port-forward svc/monitoring-stack-grafana -n monitoring 3000:80
```
### 3.2 Login

> AKSES URL: http://localhost:3000

> JALANKAN PERINTAH BERIKUT UNTUK MENDAPATKAN PASSWORD
```
kubectl --namespace monitoring get secrets monitoring-stack-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo
```

> Username: admin </br>
Password: (masukkan token yang didapat dari perintah diatas) 

### 3.3 Navigasi Dashboard
- Klik menu Dashboards (icon kotak empat) → Browse.

- Buka folder General atau Kubernetes.

- Pilih dashboard: "Kubernetes / Compute Resources / Namespace (Pods)".

- Ubah filter Namespace di bagian atas menjadi namespace aplikasi Anda (misal: cetakan-wahana).

## 4. Instalasi Logging Stack (Loki & Promtail)
Loki berfungsi sebagai penyimpan log ("Gudang"), dan Promtail berfungsi sebagai pengirim log ("Kurir").

>Catatan Penting: Karena Grafana sudah terinstall pada Langkah 2, kita tidak menginstall Grafana lagi di tahap ini.

### 4.1 Tambahkan Repository Grafana
```
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```
### 4.2 Install Loki Stack (Tanpa Grafana)
```
helm upgrade --install loki grafana/loki-stack \
  --namespace monitoring \
  --set grafana.enabled=false \
  --set promtail.enabled=true \
  --set loki.isDefault=false
```
 Tunggu hingga pod loki-0 dan promtail-xxx berstatus Running.

## 5. Integrasi Data Source
Hubungkan Loki ke Grafana secara manual agar log dapat divisualisasikan.

- Buka Dashboard Grafana http://localhost:3000.

- Masuk ke Connections / Administration ⚙️ → Data Sources.

- Klik + Add new data source.

- Cari dan pilih Loki.

- Isi konfigurasi URL:

> http://loki:3100
Klik Save & Test. Pastikan muncul pesan hijau: "Data source connected and labels found."

## 6. Konfigurasi Lanjutan (Log Volume Enable)
Jika grafik volume (histogram) tidak muncul pada log, lakukan konfigurasi kustom berikut.

### 6.1 Buat File Konfigurasi
Buat file bernama loki-custom.yaml di direktori kerja Anda:

loki-custom.yaml
```
loki:
  isDefault: false
  image:
    tag: 2.9.3 # Mengunci versi stabil
  config:
    limits_config:
      volume_enabled: true  # Mengaktifkan grafik volume log
      retention_period: 48h # Retensi log 48 jam untuk hemat storage
    # Menghapus structured_metadata untuk kompatibilitas
    
promtail:
  enabled: true

grafana:
  enabled: false
```
### 6.2 Terapkan Konfigurasi (Upgrade)
Update instalasi Loki menggunakan file tersebut:

```
helm upgrade --install loki grafana/loki-stack \
  --namespace monitoring \
  -f loki-custom.yaml
```
### 6.3 Verifikasi Hasil
- Tunggu pod Loki restart.
- Buka menu Explore di Grafana.
- Pilih source Loki.
- Jalankan query log (contoh: {app="cetakan-app"}).

- Pastikan error "Failed to load log volume" hilang dan grafik batang muncul.
# POC Migrasi Aplikasi WAFR: Production Legacy → Kubernetes

| | |
|---|---|
| **Dokumen** | Proof of Concept (POC) Migrasi WAFR |
| **Author** | Elang |
| **Tanggal** | 2025-05-05 |
| **Update Terakhir** | 2025-05-11 |
| **Status** | Draft / Testing Phase |

---

## Daftar Isi

1. [Tujuan POC](#1-tujuan-poc)
2. [Ruang Lingkup](#2-ruang-lingkup)
3. [Arsitektur & Environment](#3-arsitektur--environment)
4. [Skenario Pengujian](#4-skenario-pengujian)
   - [TC-01: Validasi Data (Data Parity)](#tc-01-validasi-data-data-parity)
   - [TC-02: Pod Failure — Ketersediaan Aplikasi saat Pod Dimatikan](#tc-02-pod-failure--ketersediaan-aplikasi-saat-pod-dimatikan)
   - [TC-03: Node Failure — Pod Rescheduling & Zero Downtime](#tc-03-node-failure--pod-rescheduling--zero-downtime)
   - [TC-04: Rolling Update tanpa Downtime](#tc-04-rolling-update-tanpa-downtime)
   - [TC-05: Load & Performance Baseline](#tc-05-load--performance-baseline)
   - [TC-06: Persistent Storage & Koneksi Database](#tc-06-persistent-storage--koneksi-database)
5. [Alat & Prasyarat](#5-alat--prasyarat)
6. [Kriteria Keberhasilan (Success Criteria)](#6-kriteria-keberhasilan-success-criteria)
7. [Template Hasil Pengujian](#7-template-hasil-pengujian)
8. [Risiko & Mitigasi](#8-risiko--mitigasi)

---

## 1. Tujuan POC

POC ini bertujuan membuktikan bahwa aplikasi **WAFR** yang dijalankan di atas cluster **Kubernetes** mampu:

1. **Menampilkan data yang identik** dengan aplikasi WAFR di environment Production Legacy.
2. **Tetap tersedia (highly available)** meskipun salah satu pod dimatikan, mengingat deployment dikonfigurasi dengan **2 replica (hingga 10 dengan HPA)**.
3. **Melakukan rescheduling pod secara otomatis** tanpa downtime ketika sebuah node Kubernetes mati.
4. _(Tambahan)_ Mendukung rolling update tanpa downtime.
5. _(Tambahan)_ Memberikan performa yang setara atau lebih baik dari legacy.
6. _(Tambahan)_ Memastikan health check, probe, dan koneksi database berjalan dengan benar.

---

## 2. Ruang Lingkup

| Item | Termasuk | Tidak Termasuk |
|---|---|---|
| Aplikasi WAFR | ✅ | |
| Kubernetes cluster (existing) | ✅ | |
| Production Legacy WAFR | ✅ (read-only, hanya perbandingan) | |
| Migrasi data/database | ❌ | Tidak dilakukan pada POC |
| Setup CI/CD pipeline | ❌ | Di luar scope POC |
| Konfigurasi monitoring (Prometheus/Grafana) | ✅ | |

---

## 3. Arsitektur & Environment

```
┌─────────────────────────────────────────────────────────────────┐
│                        PRODUCTION LEGACY                        │
│   [WAFR App Server]  ──►  [Database Legacy]                     │
│   (Bare metal / VM)                                             │
└─────────────────────────────────────────────────────────────────┘
                             │ Perbandingan Data (TC-01)
┌─────────────────────────────────────────────────────────────────┐
│                     KUBERNETES CLUSTER (Target)                  │
│                                                                  │
│  ┌─────────────┐   ┌─────────────┐                              │
│  │  Node-1     │   │  Node-2     │   (... Node-N)               │
│  │ ┌─────────┐ │   │ ┌─────────┐ │                              │
│  │ │Pod WAFR │ │   │ │Pod WAFR │ │  ← Deployment (replica: 2)   │
│  │ │(replica1│ │   │ │(replica2│ │                              │
│  │ └─────────┘ │   │ └─────────┘ │                              │
│  └─────────────┘   └─────────────┘                              │
│         │                 │                                      │
│         └────────┬────────┘                                      │
│              [Service / Ingress / LoadBalancer]                  │
│                      │                                           │
│              [Database / PVC / ConfigMap]                        │
└─────────────────────────────────────────────────────────────────┘
```

### Environment Detail

| Parameter | Production Legacy | Kubernetes (Target) |
|---|---|---|
| Host/Node | **nanti** | 192.168.200.101 / 102 / 103 / 181 / 182 |
| Database | 120 | 120 |
| URL Akses | wafr | wafr-prod-k8s.wahana.com |
| Replica | N/A | 2 (sampai 10) |
| Namespace | N/A | wafr-wahana |

---

## 4. Skenario Pengujian

---

### TC-01: Validasi Data (Data Parity)

**Tujuan:** Memastikan data yang ditampilkan oleh aplikasi WAFR di Kubernetes identik dengan data di Production Legacy.

**Prasyarat:**
- Aplikasi WAFR di Kubernetes sudah running dan terhubung ke database yang sama atau database yang telah disinkronisasi.
- Akses ke URL Legacy dan URL Kubernetes tersedia.

**Langkah Pengujian:**

**Checklist Validasi Manual (UI):**

| No. | Halaman / Fitur | Legacy | Kubernetes | Sama? |
|---|---|---|---|---|
| 1 | Halaman utama / dashboard | | | ☐ |
| 2 | Daftar data report | | | ☐ |
| 3 | Detail isi report | | | ☐ |
| 4 | Pencarian / filter | | | ☐ |

**Kriteria Lulus:** Semua data yang ditampilkan di UI dan API response identik antara Legacy dan Kubernetes.

---

### TC-02: Pod Failure — Ketersediaan Aplikasi saat Pod Dimatikan

**Tujuan:** Membuktikan bahwa dengan 2 replica, mematikan 1 pod tidak menyebabkan downtime.

**Prasyarat:**
- Deployment WAFR berjalan dengan `replicas: 2`.
- Tool monitoring aktif (curl loop / k6 / watch).

**Langkah Pengujian:**

```bash
# Terminal 1 — Monitor ketersediaan aplikasi secara terus-menerus
watch kubectl get pods -n wafr-wahana

# Terminal 2 — Cek pod yang sedang running
kubectl get pods -n wafr-wahana -l app=wafr -o wide

# Catat nama pod
POD_TO_KILL="wafr-6d4f8b-xxxx"

# Matikan 1 pod
kubectl delete pod $POD_TO_KILL -n wafr-wahana

# Pantau pod baru yang dibuat otomatis
kubectl get pods -n wafr-wahana -l app=wafr -w
```

**Observasi yang Dicatat:**

| Parameter | Nilai |
|---|---|
| Waktu pod didelete | _(catat timestamp)_ |
| Waktu pod baru Ready | _(catat timestamp)_ |
| Jumlah request gagal (HTTP non-2xx) selama proses | _(catat jumlah)_ |
| Apakah aplikasi tetap bisa diakses? | ✅ / ❌ |

**Kriteria Lulus:** Monitor di Terminal 1 tidak menunjukkan error/timeout selama proses delete pod. Kubernetes secara otomatis membuat pod pengganti hingga kembali ke 2 replica.

---

### TC-03: Node Failure — Pod Rescheduling & Zero Downtime

**Tujuan:** Membuktikan bahwa ketika sebuah node mati, pod yang ada di node tersebut akan dijadwalkan ulang ke node lain secara otomatis tanpa downtime.

**Prasyarat:**
- Minimal 2 node worker tersedia di cluster.
- Pod WAFR tersebar di node yang berbeda.
- Akses SSH ke node atau hak akses untuk `kubectl cordon/drain`.

**Langkah Pengujian:**

```bash
# Terminal 1 — Monitor ketersediaan (jalankan sepanjang test)
watch kubectl get pods -n wafr-wahana -o wide

# Terminal 2 — Identifikasi pod dan node-nya
kubectl get pods -n wafr-wahana -l app=wafr -o wide
# Output: NAME | READY | STATUS | NODE
# Pilih node yang akan "dimatikan"
TARGET_NODE="node-1"

# Simulasi node failure — Opsi A: Drain node (graceful, direkomendasikan untuk test awal)
kubectl drain $TARGET_NODE --ignore-daemonsets --delete-emptydir-data --force

# Simulasi node failure — Opsi B: Cordon node (node tidak menerima pod baru)
kubectl cordon $TARGET_NODE

# Simulasi node failure — Opsi C: Matikan node secara hard (simulasi crash)
# HATI-HATI: lakukan ini hanya di cluster non-prod atau node yang memang boleh dimatikan
# ssh user@$TARGET_NODE "sudo shutdown -h now"

# Pantau rescheduling pod
kubectl get pods -n wafr-wahana -l app=wafr -o wide -w

# Setelah test selesai, kembalikan node
kubectl uncordon $TARGET_NODE
```

**Observasi yang Dicatat:**

| Parameter | Nilai |
|---|---|
| Node yang dimatikan | _(nama node)_ |
| Pod yang terdampak | _(nama pod)_ |
| Waktu node tidak tersedia | _(timestamp)_ |
| Waktu pod baru dijadwalkan di node lain | _(timestamp)_ |
| Waktu pod baru berstatus Running/Ready | _(timestamp)_ |
| Total waktu rescheduling | _(dalam detik)_ |
| Aplikasi tetap bisa diakses? | ✅ / ❌ |

**Kriteria Lulus:**
- Pod secara otomatis dijadwalkan ke node lain dalam waktu < 60 detik (atau sesuai toleransi yang disepakati).
- Aplikasi masih bisa diakses saat terjadi restart pod.

---

### TC-04: Rolling Update tanpa Downtime

**Tujuan:** Membuktikan bahwa update image/konfigurasi aplikasi WAFR dapat dilakukan tanpa downtime menggunakan strategi Rolling Update.

**Prasyarat:**
- Tersedia image versi baru.
- Strategy deployment diset ke `RollingUpdate`.

**Konfigurasi Deployment yang Direkomendasikan:**

```yaml
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Boleh ada 1 pod extra saat update
      maxUnavailable: 0  # Tidak boleh ada pod yang unavailable saat update
```

**Langkah Pengujian:**

```bash
# Terminal 1 — Monitor ketersediaan
watch kubectl get pods -n wafr-wahana -o wide

# Terminal 2 — Trigger rolling update (ganti image/tag)
kubectl set image deployment/wafr wafr=<image>:<new-tag> -n wafr-wahana

# Pantau proses rollout
kubectl rollout status deployment/wafr -n wafr-wahana
```

**Kriteria Lulus:** Tidak ada error selama proses rolling update berlangsung. Rollout selesai dengan status `successfully rolled out`.

---

### TC-05: Load & Performance Baseline

**Tujuan:** Memastikan performa aplikasi WAFR di Kubernetes cukup untuk handle Production, serta mengetahui batas kapasitas.

**Tool yang Digunakan:** Locust

**Script locust:**

```py
asda
```

```bash
# Jalankan locust
locust -f locust.py
```

**Metrik yang Diperhatikan:**

| Metrik | Kubernetes | Target | Keterangan |
| :--- | :--- | :--- | :--- |
| Avg Response Time | _ ms | ≤ Legacy | Performa rata-rata |
| P95 Response Time | _ ms | < 2000ms | Latensi persentil 95 |
| Throughput (req/s) | _ | cukup | Kapasitas beban |
| Error Rate | _ % | < 20% | Batas kegagalan |

**Kriteria Lulus:** Performa Kubernetes tidak lebih buruk dari 20%.

---

### TC-06: Koneksi Database

**Tujuan:** Memastikan aplikasi WAFR di Kubernetes dapat membaca data ke database dengan benar.

**Langkah Pengujian:**

```bash
# 1. Verifikasi environment variable / secret koneksi DB
kubectl get secret wafr-db-secret -n wafr-wahana -o yaml
kubectl describe configmap wafr-config -n wafr-wahana

# 2. Test melihat data yang ada di database melalui UI
# - Hit endpoint aplikasi wafr di Kubernetes environment
# - Verifikasi apakah data muncul atau tidak
```

**Kriteria Lulus:**
- Koneksi ke database berhasil dari dalam pod.
- Bisa mengakses data yang ada di database.
- Tidak ada data yang hilang atau kurang.

---

## 5. Alat & Prasyarat

| No. | Alat | Kegunaan | Wajib |
|---|---|---|---|
| 1 | `kubectl` | Manajemen cluster Kubernetes | ✅ |
| 2 | Akses cluster (kubeconfig) | Eksekusi perintah kubectl | ✅ |
| 3 | `curl` | Monitoring HTTP sederhana | ✅ |
| 4 | `locust` | Load testing & performance baseline | Disarankan |
| 5 | Browser | Validasi UI manual | ✅ |
| 6 | Terminal dengan `watch` | Monitoring pod secara real-time | Disarankan |
| 7 | Akses SSH ke node | Simulasi node failure (TC-03 Opsi C) | Opsional |
| 8 | Prometheus + Grafana | Monitoring metrics | Opsional |

---

## 6. Kriteria Keberhasilan (Success Criteria)

| Test Case | Kriteria Lulus | Bobot |
|---|---|---|
| TC-01: Validasi Data | Data identik antara Legacy dan Kubernetes | **Wajib** |
| TC-02: Pod Failure | 0 downtime saat 1 dari 2 pod dimatikan | **Wajib** |
| TC-03: Node Failure | Pod rescheduling < 60 detik, aplikasi tetap aksesibel | **Wajib** |
| TC-04: Rolling Update | 0 downtime saat rolling update | Disarankan |
| TC-05: Performance | Error rate < 20% | Disarankan |
| TC-06: Persistent Storage | Koneksi DB stabil, tidak ada data loss | **Wajib** |

**POC dinyatakan LULUS** apabila semua item **Wajib** terpenuhi.

---

## 7. Hasil Pengujian

### Ringkasan Hasil

| Test Case | Tanggal Test | PIC | Status | Catatan |
|---|---|---|---|---|
| TC-01: Validasi Data | | | ⬜ Belum / ✅ Lulus / ❌ Gagal | |
| TC-02: Pod Failure | | | ⬜ Belum / ✅ Lulus / ❌ Gagal | |
| TC-03: Node Failure | | | ⬜ Belum / ✅ Lulus / ❌ Gagal | |
| TC-04: Rolling Update | | | ⬜ Belum / ✅ Lulus / ❌ Gagal | |
| TC-05: Performance | | | ⬜ Belum / ✅ Lulus / ❌ Gagal | |
| TC-06: Persistent Storage | | | ⬜ Belum / ✅ Lulus / ❌ Gagal | |

### Detail Temuan

```
TC-02: Pod Failure
- Pod didelete pada: HH:MM:SS
- Pod baru Ready pada: HH:MM:SS  
- Jumlah error selama proses: 0
- Kesimpulan: LULUS / GAGAL
- Catatan: ...
```

---

## 8. Risiko & Mitigasi

| Risiko | Dampak | Kemungkinan | Mitigasi |
|---|---|---|---|
| Pod WAFR dijadwalkan di node yang sama | Jika node itu mati, kedua pod ikut mati (downtime) | Sedang | Pastikan HPA aktif agar kubernetes langsung melakukan assign pod baru |
| Koneksi database putus saat pod restart | Data tidak bisa diakses | Rendah | Pastikan `readinessProbe` sudah di-set |
| Image pull lambat saat rescheduling | Pod baru lama menjadi Ready | Sedang | Pre-pull image di semua node, atau gunakan registry lokal |
| Resource limit tidak sesuai | Pod OOMKilled atau throttled | Sedang | Set `requests` dan `limits` sesuai hasil TC-05 |
| Perbedaan konfigurasi (env var, secret) | Perilaku aplikasi berbeda dari legacy | Sedang | Audit semua env var legacy vs ConfigMap/Secret Kubernetes |

---

> **Catatan:** Dokumen ini bersifat living document dan dapat diperbarui seiring progress pengujian.
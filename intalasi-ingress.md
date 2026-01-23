# 📖 Panduan Instalasi NGINX Ingress Controller

Dokumen ini menjelaskan langkah-langkah instalasi dan konfigurasi **NGINX Ingress Controller** pada cluster Kubernetes berbasis **k3d**. Panduan ini mencakup instalasi controller, penyesuaian *port mapping*, dan konfigurasi Ingress resource.

## 🧠 Konsep Dasar

Sebelum memulai, pahami perbedaan dua komponen berikut:

1. **Nginx Web Server (Aplikasi):** Melayani konten web (berjalan di dalam pod `cetakan-app`).
2. **Nginx Ingress Controller (Gateway):** Bertugas sebagai router/"satpam" yang mengatur lalu lintas dari luar cluster menuju aplikasi.

---

## 🛠️ Langkah 1: Instalasi Controller (Baremetal)

Kita menggunakan manifest versi *baremetal* yang ringan dan kompatibel dengan k3d.

Jalankan perintah berikut di terminal:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/baremetal/deploy.yaml

```

Verifikasi instalasi (tunggu hingga status **Running**):

```bash
kubectl get pods -n ingress-nginx

```

---

## ⚙️ Langkah 2: Konfigurasi Port (Patching Service)

**PENTING:** Cluster k3d mengekspos port `30000-30026`. Kita harus memaksa Ingress Controller untuk menggunakan port **30000** (HTTP) agar sesuai dengan *binding* port host.

### 2.1. Buat file `patch-service.yaml`

Buat file baru bernama `patch-service.yaml` dengan isi sebagai berikut:

```yaml
spec:
  type: NodePort
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: http
    nodePort: 30000  # <--- Mengunci akses HTTP di port 30000
  - name: https
    port: 443
    protocol: TCP
    targetPort: https
    nodePort: 30001  # Opsional: Mengunci akses HTTPS di port 30001

```

### 2.2. Terapkan Patch

Aplikasikan konfigurasi tersebut ke service ingress yang sedang berjalan:

```bash
kubectl patch svc ingress-nginx-controller -n ingress-nginx --patch-file patch-service.yaml

```

---

## 🚀 Langkah 3: Setup Ingress Resource Project

Update konfigurasi Ingress pada project `cetakan-wahana` agar dikenali oleh controller yang baru diinstall.

### 3.1. Buat/Update `ingress-cetakan.yaml`

### 3.1.1 Jika Project Menggunakan Http

Pastikan `ingressClassName` diatur ke `nginx`.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cetakan-ingress
  namespace: cetakan-wahana
  annotations:
    # Mencegah konflik dengan Traefik (default k3d)
    kubernetes.io/ingress.class: nginx
spec:
  ingressClassName: nginx # WAJIB: Agar dikenali oleh NGINX Controller
  rules:
  - host: cetakan.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: cetakan-app
            port:
              number: 80 # Port internal service aplikasi

```

### 3.1.2 Jika Project Menggunakan Https
Pastikan `ingressClassName` diatur ke `nginx`.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cetakan-ingress
  namespace: cetakan-wahana
  annotations:
    # Mencegah konflik dengan Traefik (default k3d)
    kubernetes.io/ingress.class: nginx
    # tambahkan 3 baris dibawah untuk HTTPS
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS" 
    nginx.ingress.kubernetes.io/proxy-ssl-verify: "off"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

spec:
  ingressClassName: nginx # WAJIB: Agar dikenali oleh NGINX Controller 
  # Tambahkan baris tls untuk Https
  tls:
  - hosts:
    - cetakan-local.192.168.4.100.sslip.io
  rules:
  - host: cetakan-local.192.168.4.100.sslip.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: cetakan-app
            port:
              number: 80 # Port internal service aplikasi

```

### 3.2. Terapkan Ingress

```bash
kubectl apply -f ingress-cetakan.yaml

```

---

## 🌐 Langkah 4: Akses Aplikasi

Karena ini berjalan di *local environment* dengan k3d, akses dilakukan melalui **Port 30000**.

1. **Cek DNS Lokal (`/etc/hosts`):**
Pastikan domain sudah terdaftar.
```text
127.0.0.1  cetakan.local

```
> **NOTE !<br>SETIAP PROJECT HARUS MEMILIKI 1 DOMAIN/HOST, DOMAIN cetakan-local HANYA UNTUK PROJECT BERNAMA CETAKAN**


2. **Buka Browser:**
Akses alamat berikut (wajib menyertakan port):
👉 **[http://cetakan-local.192.168.4.100.sslip.io:30000](http://cetakan-local.192.168.4.100.sslip.io:30000)**

# Deploy Aplikasi SWB di Kubernetes

## Apa yang Akan Anda Dapatkan?

Setelah mengikuti panduan ini:

- Aplikasi SWB deploy di Kubernetes dengan 2 replicas  
- NGINX frontend untuk serve static files  
- Backend API dengan auto-scaling (HPA)  
- SSL/TLS certificate untuk HTTPS  
- High Availability dengan multiple pods  
- Akses via domain 

---

## Berapa Lama Prosesnya?

- **Waktu:** 30-60 menit
- **Tingkat Kesulitan:** Menengah
- **Yang Dibutuhkan:**
  - Cluster Kubernetes yang sudah jalan. Jika belum ada, bisa melihat proses instalasi cluster di [ dokumentasi ini ]( 1.%20Instalasi-kubernetes-3-node-cluster.md )
  - MetalLB & NGINX Ingress sudah terinstall. Jika belum, lihat di [ dokumentasi ini ]( 2.%20instalasi-ingress-metallb.md )
  - Docker image SWB sudah di-build
  - SSL certificate (tls.crt & tls.key)
  - Akses ke Docker Hub atau registry lain (untuk push image)

---

## Persiapan Sebelum Deploy

### Hal yang Harus Disiapkan:

1. **Domain/hostname** untuk aplikasi (contoh: `swb-docker100.wahana.com`)
2. **SSL Certificate** (file `tls.crt` dan `tls.key`)
3. **Docker Hub atau registry lain** (untuk upload image)
4. **Versi aplikasi** yang akan di-deploy (cek di `package.json`)
5. **IP Hosts Database** yang bisa diakses dari cluster (untuk extra hosts)

---

## BAGIAN 1: Build & Push Docker Image

### LANGKAH 1: Persiapan Source Code

#### Langkah 1.1: Clone/Download Source Code

Pastikan sudah punya source code aplikasi SWB di device lokal. Jika sudah ada, buka source code dengan VSCode atau IDE lainnya.

#### Langkah 1.2: Cek Versi Aplikasi

Lihat versi aplikasi di file `package.json`:

```bash
web/dev/dcaf/package.json
```

**Contoh isi file:**
```json
"version": "0.2.66",
```

📝 **CATAT versi ini!** Misal: `0.2.66`

---

### LANGKAH 2: Konfigurasi Aplikasi (PENTING!)

**Sebelum build image**, Anda harus edit beberapa file konfigurasi agar aplikasi tahu domain yang akan dipakai.

#### Langkah 2.1: Ubah File axios.js (Konfigurasi API Backend)

File ini menentukan kemana frontend akan request data API.

**Buka file:**

```bash
web/dev/dcaf/src/boot/axios.js
```

**Cari baris yang ada `baseURL`**, lalu ubah domain-nya:


**Sesuaikan dengan domain Anda:**
```javascript
const axiosInstance = axios.create({
  baseURL: 'https://swb-docker100.wahana.com/a'
})
```

**⚠️ Ganti `swb-docker100.wahana.com` dengan domain yang akan Anda pakai!**

---

#### Langkah 2.2: Ubah File API YAML (Konfigurasi API Endpoint)

Ada beberapa file `.yml` di folder `etc/api` yang perlu diubah (ubah semua file yml yang ada di etc/api).

**beberapa file yang perlu diubah:**
- `etc/api/3pl.yml`
- `etc/api/dcaf.yml`
- `etc/api/drp.yml`
- (dan file `.yml` lainnya di folder `etc/api/`)

**Cara ubah (contoh untuk `3pl.yml`):**

```bash
etc/api/3pl.yml
```

**Cari baris yang ada `servers:` dan `url:`**, lalu tambahkan domain:
```yaml
servers:
  - url: https://swb-docker100.wahana.com/a/3pl
```

**Ulangi untuk SEMUA file `.yml` di folder `etc/api/`!**


**⚠️ Sesuaikan `swb-docker100.wahana.com` dengan domain yang dipunya!**

---

#### Langkah 2.3: Sesuaikan Versi Aplikasi di dcaf.yml

File `etc/dcaf.yml` harus punya versi yang sama dengan `package.json`.

**Buka file:**

```bash
etc/dcaf.yml
```

**Cari bagian `versions:`**, lalu sesuaikan:

**SEBELUM:**
```yaml
app:
  code: swb
  name: Wahana SWB App
  from: <email>
  versions:
    swb: 0.2.65
```

**SESUDAH (sesuai package.json tadi):**
```yaml
app:
  code: swb
  name: Wahana SWB App
  from: <email>
  versions:
    swb: 0.2.66
```

**Simpan:**

✅ **Konfigurasi selesai! Siap build image!**

---

### LANGKAH 3: Build Docker Image Backend

#### Langkah 3.1: Buat Dockerfile untuk Backend

Pastikan telah membuat file `Dockerfile` di root folder lalu isi dengan:

```dockerfile
FROM node:16-alpine AS builder
USER root
WORKDIR /usr/src/app
RUN npm install -g @quasar/cli
COPY ./web/dev/dcaf .
RUN npm install
RUN quasar build -m pwa

FROM docker-registry.stage.wahana.com/swb_base:1.0.3 AS base
# define arg
ARG UWSGI_PATH_CONF=/etc/uwsgi/apps-enabled/
ARG SWB_DIR=/opt/pm/apps/swb/

# set apps saas dir
RUN mkdir -p ${SWB_DIR} 
WORKDIR ${SWB_DIR}

COPY . .
COPY --from=builder /usr/src/app/dist ${SWB_DIR}web/dev/dcaf/dist 

ADD lib/cf/swb.ini ${UWSGI_PATH_CONF}

EXPOSE 3189

ENTRYPOINT ["uwsgi", "--ini", "/etc/uwsgi/apps-enabled/swb.ini"]
```

**Simpan:**

#### Langkah 3.2: Build Image Backend

Dari root folder `swb`, jalankan:

```bash
docker build -t elangptra/nama-image:tag .
```

**⚠️ Ubah:**
- `elangptra` → username Dockerhub atau registry yang anda gunakan
- `nama-image:tag` → nama image dan tag yang ingin disimpan ke registry

**Penjelasan:**
- `-t` = tag/nama image
- `.` = build dari folder saat ini

**Proses build akan memakan waktu 5-15 menit** tergantung koneksi internet.

**Output yang diharapkan di akhir:**
```
Successfully built **********
Successfully tagged elangptra/nama-image:tag
```

✅ **Image backend berhasil di-build!**

---

### LANGKAH 4: Build Docker Image Frontend (NGINX)

#### Langkah 4.1: Buat Dockerfile frontend + nginx

Buat file baru untuk build frontend:

```bash
fe.Dockerfile
```

Isi dengan:

```dockerfile
FROM node:16-alpine AS builder
USER root
WORKDIR /usr/src/app
RUN npm install -g @quasar/cli
COPY ./web/dev/dcaf .
RUN npm install
RUN quasar build -m pwa

FROM nginx:alpine AS production
ARG SWB_DIR=/opt/pm/apps/swb/

COPY --from=builder /usr/src/app/dist/pwa ${SWB_DIR}web/dev/dcaf/dist

EXPOSE 443

CMD ["nginx", "-g", "daemon off;"]
```

**Simpan:**

#### Langkah 4.2: Build Image Frontend

```bash
docker build -f fe.Dockerfile -t elangptra/nama-image:tag .
```

**⚠️ Ubah:**
- `elangptra` → username DockerHub atau image registry yang digunakan
- `nama-image:tag` → nama image dan tag yang ingin disimpan ke registry

**Tunggu 5-15 menit.**

**Output yang diharapkan:**
```
Successfully built **************
Successfully tagged elangptra/nama-image:tag
```

✅ **Image frontend berhasil di-build!**

---

### LANGKAH 5: Push Image ke Docker Hub (disini menggunakan dockerhub, kalau pakai registry lain bisa disesuaikan)

#### Langkah 5.1: Login ke Docker Hub

```bash
docker login
```

**Masukkan:**
- **Username:** Docker Hub username Anda
- **Password:** Docker Hub password Anda

**Output:**
```
Login Succeeded
```

#### Langkah 5.2: Push Image Backend

```bash
docker push elangptra/nama-image:tag
```

**⚠️ Sesuaikan nama image!**

**Tunggu 5-10 menit** (tergantung ukuran image dan koneksi).

#### Langkah 5.3: Push Image Frontend

```bash
docker push elangptra/nama-image:tag
```

**Tunggu 5-10 menit.**

**Output di akhir:**
```
v0.2.66: digest: sha256:1234567890abcdef... size: 1234
```

✅ **Kedua image sudah di-upload ke Docker Hub!**

---

## BAGIAN 2: Persiapan Manifest Kubernetes

### LANGKAH 6: Buat Folder Untuk Deployment

Di server master Kubernetes, buat struktur folder untuk menyimpan file YAML.

```bash
cd ~
mkdir -p swb-deployment/{app,ingress,nginx,config}
cd swb-deployment
```

**Struktur folder:**
```
swb-deployment/
├── app/
│   ├── swb-deployment.yaml
│   └── swb-service.yaml
├── ingress/
│   └── ingress-config.yaml
├── nginx/
│   ├── nginx-deployment.yaml
│   ├── nginx-service.yaml
│   └── nginx-config.yaml
└── config/
    ├── regcred-swb.yaml
    ├── swb-hpa.yaml
    └── swb-ssl-secret.yaml
```

---

### LANGKAH 7: Buat Namespace

Aplikasi SWB akan jalan di namespace `swb-wahana`.

```bash
kubectl create namespace swb-wahana
```

**Output:**
```
namespace/swb-wahana created
```

---

### LANGKAH 8: Buat File Konfigurasi

#### Langkah 8.1: Buat Docker Registry Secret

File ini berisi credential untuk pull image dari Docker Hub (atau registry lain yang digunakan).

```bash
vim config/regcred-swb.yaml
```

Isi dengan:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: regcred-swb
  namespace: swb-wahana
type: kubernetes.io/dockerconfigjson
stringData:
  .dockerconfigjson: |
    {
      "auths": {
        "https://index.docker.io/v1/": {
          "username": "elangptra",
          "password": "password",
          "email": "email@email.com",
          "auth": "echo -n "username:password" | base64"
        }
      }
    }
```

**⚠️ PENTING - Ubah:**
- `username` → Username registry
- `password` → Password registry
- `email` → Email (jika menggunakan dockerhub)
- `auth` → Base64 encode dari `username:password`

**Cara generate `auth` (jalankan di terminal):**

```bash
echo -n "username:password" | base64
```

Copy output-nya dan paste ke bagian `auth`.

**Simpan:** Tekan `Esc`, ketik `:wq`, lalu `Enter` untuk menyimpan dan keluar.

---

#### Langkah 8.2: Buat SSL Secret

File ini berisi SSL certificate untuk HTTPS.

**⚠️ Pastikan sudah punya file `tls.crt` dan `tls.key`!**

##### Cara A: Generate Base64 Manual

**1. Encode certificate dan key:**

```bash
base64 -w 0 /path/to/tls.crt > tls-crt-base64.txt
base64 -w 0 /path/to/tls.key > tls-key-base64.txt
```

**⚠️ Ganti `/path/to/` dengan lokasi file SSLnya!**

**2. Buat file secret:**

```bash
vim config/swb-ssl-secret.yaml
```

Isi dengan:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: swb-ssl
  namespace: swb-wahana
type: kubernetes.io/tls
data:
  tls.crt: <ISI_BASE64_TLS_CRT>
  tls.key: <ISI_BASE64_TLS_KEY>
```

**3. Copy-paste base64:**
- Buka file `tls-crt-base64.txt`, copy isinya
- Paste di tempat `<ISI_BASE64_TLS_CRT>`
- Buka file `tls-key-base64.txt`, copy isinya
- Paste di tempat `<ISI_BASE64_TLS_KEY>`

**Simpan:** Tekan `Esc`, ketik `:wq`, lalu `Enter` untuk menyimpan dan keluar.

##### Cara B: Gunakan kubectl create secret (Lebih Mudah)

**Jika punya file `tls.crt` dan `tls.key`, langsung:**

```bash
kubectl create secret tls swb-ssl \
  --cert=/path/to/tls.crt \
  --key=/path/to/tls.key \
  -n swb-wahana
```

**Output:**
```
secret/swb-ssl created
```

✅ **Kalau pakai cara B, SKIP file `swb-ssl-secret.yaml`!**

---

#### Langkah 8.3: Buat HPA (Horizontal Pod Autoscaler)

File ini untuk auto-scaling backend pods berdasarkan CPU usage.

```bash
vim config/swb-hpa.yaml
```

Isi dengan:

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: swb-app
  namespace: swb-wahana
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: swb-app
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

**Penjelasan:**
- **minReplicas: 2** → Minimal selalu ada 2 pods jalan
- **maxReplicas: 10** → Maksimal bisa scale sampai 10 pods
- **targetCPUUtilizationPercentage: 70** → Kalau CPU usage >70%, tambah pods

**Simpan:** Tekan `Esc`, ketik `:wq`, lalu `Enter` untuk menyimpan dan keluar.

---

#### Langkah 8.4: Buat Backend Deployment

```bash
vim app/swb-deployment.yaml
```

Isi dengan:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: swb-app
  namespace: swb-wahana
spec:
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: swb-app
  template:
    metadata:
      labels:
        app: swb-app
    spec:
      hostAliases:
        - ip: <ip-hosts-tambahan>
          hostnames:
            - "hosts"
            - "hosts"
            - "hosts"
      imagePullSecrets:
        - name: regcred-swb
      containers:
        - name: swb-app
          image: user/nama-image:tag
          ports:
            - containerPort: 3189
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          readinessProbe:
            tcpSocket:
              port: 3189
            initialDelaySeconds: 10
            periodSeconds: 10
          livenessProbe:
            tcpSocket:
              port: 3189
            initialDelaySeconds: 30
            periodSeconds: 20
```

**⚠️ PENTING - Ubah:**
- `hostAliases` → IP: `ip-hosts-tambahan` (IP hosts database di luar cluster)
- `image` → `user/nama-image:tag` (sesuaikan dengan image yang telah dibuild dan dipush sebelumnya)

**Penjelasan:**
- **hostAliases:** Mapping hostname ke IP (seperti /etc/hosts di container)
- **imagePullSecrets:** Credential untuk pull image dari Dockerhub (atau registry lain yang digunakan)
- **resources:** Batasan CPU dan Memory yang bisa digunakan oleh container
- **readinessProbe:** Cek apakah pod sudah siap terima traffic atau belum
- **livenessProbe:** Cek apakah pod masih hidup (kalau mati, restart otomatis)

**Simpan:** Tekan `Esc`, ketik `:wq`, lalu `Enter` untuk menyimpan dan keluar.

---

#### Langkah 8.5: Buat Backend Service

```bash
vim app/swb-service.yaml
```

Isi dengan:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: swb-app-service
  namespace: swb-wahana
spec:
  ports:
    - port: 3189
      targetPort: 3189
  selector:
    app: swb-app
```

**Penjelasan:**
- Service ini expose backend pods di port 3189
- Service type `ClusterIP` (default) → hanya bisa diakses dari dalam cluster

**Simpan:** Tekan `Esc`, ketik `:wq`, lalu `Enter` untuk menyimpan dan keluar.

---

#### Langkah 8.6: Buat NGINX ConfigMap

```bash
vim nginx/nginx-config.yaml
```

Isi dengan:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: swb-wahana
data:
  default.conf: |
    server {

    # ini hanya contoh, banyak yang dihilangkan!

      server_name localhost;
      root       /opt/pm/apps/swb/web/dev/dcaf/dist/pwa;
      access_log /var/log/nginx-ssl-access.log;
      error_log  /var/log/nginx-ssl-error.log;
      index index.html index.htm;
      resolver 8.8.8.8 8.8.4.4 valid=300s;
      resolver_timeout 5s;
      add_header Strict-Transport-Security "max-age=63072000; includeSubdomains";
      add_header X-Frame-Options "SAMEORIGIM";
      add_header X-Content-Type-Options nosniff;
      gzip on;
      gzip_types *;
      gzip_comp_level 9;
      gzip_http_version 1.0;

      listen 443 ssl;
      ssl_certificate /ssl/ssl-wahana/tls.crt;
      ssl_certificate_key /ssl/ssl-wahana/tls.key;

      location /a/ {
        
        # disini header request param

        include uwsgi_params;
        uwsgi_pass swb-app-service:3189;
        uwsgi_modifier1 5;
      }
    }
```

**⚠️ Ubah:**
- `uwsgi_pass` → URL backend service

**Penjelasan:**
- **listen 443 ssl:** NGINX listen di port 443 (HTTPS)
- **ssl_certificate:** Path ke SSL certificate
- **root:** Path ke static files frontend
- **location /a/:** Proxy API request ke backend
- **gzip on:** Compress response untuk hemat bandwidth

**Simpan:** Tekan `Esc`, ketik `:wq`, lalu `Enter` untuk menyimpan dan keluar.

---

#### Langkah 8.7: Buat NGINX Deployment

```bash
vim nginx/nginx-deployment.yaml
```

Isi dengan:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: swb-nginx
  namespace: swb-wahana
spec:
  replicas: 2
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: swb-nginx
  template:
    metadata:
      labels:
        app: swb-nginx
    spec:
      containers:
      - name: swb-nginx
        image: username/nama-image:tag
        ports:
          - containerPort: 443
        volumeMounts:
        - name: nginx-config-volume
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: default.conf
        
        - name: swb-ssl-volume
          mountPath: /ssl/ssl-wahana
          readOnly: true

      volumes:
      - name: nginx-config-volume
        configMap:
          name: nginx-config
      
      - name: swb-ssl-volume
        secret:
          secretName: swb-ssl
```

**⚠️ Ubah:**
- `image` → `username/nama-image:tag` (sesuai image frontend Anda)

**Penjelasan:**
- **replicas: 2:** Jalan 2 pods NGINX (untuk High Availability)
- **volumeMounts:** Mount konfigurasi NGINX dan SSL certificate ke dalam container
- **volumes:** Ambil data dari ConfigMap (`nginx-config`) dan Secret (`swb-ssl`)

**Simpan:** Tekan `Esc`, ketik `:wq`, lalu `Enter` untuk menyimpan dan keluar.

---

#### Langkah 8.8: Buat NGINX Service

```bash
vim nginx/nginx-service.yaml
```

Isi dengan:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: swb-nginx-service
  namespace: swb-wahana
spec:
  type: ClusterIP
  ports:
    - port: 443
      targetPort: 443
  selector:
    app: swb-nginx
```

**Penjelasan:**
- Service ini expose NGINX pods di port 443
- Ingress akan connect ke service ini

**Simpan:** Tekan `Esc`, ketik `:wq`, lalu `Enter` untuk menyimpan dan keluar.

---

#### Langkah 8.9: Buat Ingress

```bash
vim ingress/ingress-config.yaml
```

Isi dengan:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: swb-ingress
  namespace: swb-wahana
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/proxy-ssl-verify: "off"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/use-gzip: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - swb-docker100.wahana.com
    secretName: swb-ssl
  rules:
  - host: swb-docker100.wahana.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: swb-nginx-service
            port:
              number: 443
```

**⚠️ Ubah:**
- `host` → Domain yang akan digunakan (`swb-docker100.wahana.com`)
- `secretName` → Nama SSL secret yang telah dibuat (`swb-ssl`)

**Penjelasan:**
- **tls:** Aktifkan HTTPS dengan certificate dari secret `swb-ssl`
- **rules:** Traffic untuk domain `swb-docker100.wahana.com` diarahkan ke service `swb-nginx-service`

**Simpan:** Tekan `Esc`, ketik `:wq`, lalu `Enter` untuk menyimpan dan keluar.

✅ **Semua file konfigurasi sudah siap!**

---

## BAGIAN 3: Deploy ke Kubernetes

### LANGKAH 9: Deploy Aplikasi

#### Langkah 9.1: Apply Secrets

Deploy secrets terlebih dahulu (Docker registry & SSL):

```bash
kubectl apply -f config/regcred-swb.yaml
```

**Kalau pakai cara A (manual YAML) untuk SSL:**
```bash
kubectl apply -f config/swb-ssl-secret.yaml
```

**Kalau pakai cara B (kubectl create), skip command di atas!**

**Verifikasi:**
```bash
kubectl get secrets -n swb-wahana
```

**Output:**
```
NAME          TYPE                             DATA   AGE
regcred-swb   kubernetes.io/dockerconfigjson   1      10s
swb-ssl       kubernetes.io/tls                2      10s
```

---

#### Langkah 9.2: Apply ConfigMap

```bash
kubectl apply -f nginx/nginx-config.yaml
```

**Verifikasi:**
```bash
kubectl get configmap -n swb-wahana
```

**Output:**
```
NAME           DATA   AGE
nginx-config   1      5s
```

---

#### Langkah 9.3: Deploy Backend

```bash
kubectl apply -f app/swb-deployment.yaml
kubectl apply -f app/swb-service.yaml
```

**Tunggu 1-2 menit**, lalu verifikasi:

```bash
kubectl get pods -n swb-wahana
```

**Output yang diharapkan:**
```
NAME                       READY   STATUS    RESTARTS   AGE
swb-app-xxxxxxxxxx-xxxxx   1/1     Running   0          1m
swb-app-yyyyyyyyyy-yyyyy   1/1     Running   0          1m
```

**⚠️ Kalau status `ImagePullBackOff`:**
- Cek apakah `regcred-swb` sudah benar (username/password Docker Hub)
- Cek apakah image sudah di-push ke Docker Hub (atau registry lain yang digunakan)

**⚠️ Kalau status `CrashLoopBackOff`:**
- Cek log pods: `kubectl logs swb-app-xxxxxxxxxx-xxxxx -n swb-wahana`

---

#### Langkah 9.4: Deploy Frontend (NGINX)

```bash
kubectl apply -f nginx/nginx-deployment.yaml
kubectl apply -f nginx/nginx-service.yaml
```

**Tunggu 1-2 menit**, lalu verifikasi:

```bash
kubectl get pods -n swb-wahana
```

**Output:**
```
NAME                         READY   STATUS    RESTARTS   AGE
swb-app-xxxxxxxxxx-xxxxx     1/1     Running   0          3m
swb-app-yyyyyyyyyy-yyyyy     1/1     Running   0          3m
swb-nginx-xxxxxxxxxx-xxxxx   1/1     Running   0          1m
swb-nginx-yyyyyyyyyy-yyyyy   1/1     Running   0          1m
```

✅ **Sekarang ada 4 pods: 2 backend + 2 frontend!**

---

#### Langkah 9.5: Deploy Ingress

```bash
kubectl apply -f ingress/ingress-config.yaml
```

**Verifikasi:**
```bash
kubectl get ingress -n swb-wahana
```

**Output:**
```
NAME          CLASS   HOSTS                        ADDRESS         PORTS     AGE
swb-ingress   nginx   swb-docker100.wahana.com     192.168.2.240   80, 443   30s
```

**Lihat kolom `ADDRESS`:**
- Harus terisi dengan **IP LoadBalancer** dari MetalLB
- Kalau masih kosong, tunggu 1-2 menit lagi

---

#### Langkah 9.6: Deploy HPA (Auto Scaling)

```bash
kubectl apply -f config/swb-hpa.yaml
```

**Verifikasi:**
```bash
kubectl get hpa -n swb-wahana
```

**Output:**
```
NAME      REFERENCE            TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
swb-app   Deployment/swb-app   <unknown>/70%   2         10        2          30s
```

**Catatan:** `<unknown>` akan berubah jadi angka CPU percentage setelah beberapa menit (metrics server butuh waktu collect data).

✅ **Semua komponen sudah di-deploy!**

---

### LANGKAH 10: Setup /etc/hosts

Agar bisa akses aplikasi via domain, tambahkan IP LoadBalancer Ingress dan nama domain di /etc/hosts.

#### Langkah 10.1: Cari IP LoadBalancer

```bash
kubectl get ingress -n swb-wahana
```

Catat IP di kolom `ADDRESS`, misal: `192.168.2.240`

#### Langkah 10.2: Tambahkan ke /etc/hosts

**Di Laptop/Komputer Windows:**

1. Buka Notepad **sebagai Administrator**
2. File → Open → `C:\Windows\System32\drivers\etc\hosts`
3. Ubah file type ke "All Files"
4. Tambahkan:
   ```
   192.168.2.240  swb-docker100.wahana.com
   ```
5. Save

**Di Laptop/Komputer Linux/Mac:**

```bash
sudo vim /etc/hosts
```

Tambahkan:
```
192.168.2.240  swb-docker100.wahana.com
```

---

### LANGKAH 11: Akses Aplikasi!

#### Langkah 11.1: Buka Browser

Buka browser (Chrome/Firefox/Edge), ketik:

```
https://swb-docker100.wahana.com
```

**⚠️ Ganti dengan domain Anda!**

#### Langkah 11.2: Bypass SSL Warning (Kalau Pakai Self-Signed Certificate)

Kalau pakai self-signed certificate, akan muncul warning.

**Di Chrome:**
- Klik "Advanced" → "Proceed to ... (unsafe)"

**Di Firefox:**
- Klik "Advanced" → "Accept the Risk and Continue"

#### Langkah 11.3: Proses deployment selesai!

Kalau berhasil, Anda akan melihat **halaman login aplikasi SWB!**

🎉 **SELAMAT! Aplikasi SWB sudah berhasil di-deploy!**

---

## Verifikasi dan logging aplikasi

### Cek Semua Resource

```bash
kubectl get all -n swb-wahana
```

**Output yang diharapkan:**

```
NAME                             READY   STATUS    RESTARTS   AGE
pod/swb-app-xxxxxxxxxx-xxxxx     1/1     Running   0          10m
pod/swb-app-yyyyyyyyyy-yyyyy     1/1     Running   0          10m
pod/swb-nginx-xxxxxxxxxx-xxxxx   1/1     Running   0          10m
pod/swb-nginx-yyyyyyyyyy-yyyyy   1/1     Running   0          10m

NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/swb-app-service    ClusterIP   10.96.123.45     <none>        3189/TCP   10m
service/swb-nginx-service  ClusterIP   10.96.234.56     <none>        443/TCP    10m

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/swb-app     2/2     2            2           10m
deployment.apps/swb-nginx   2/2     2            2           10m

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/swb-app-xxxxxxxxxx     2         2         2       10m
replicaset.apps/swb-nginx-xxxxxxxxxx   2         2         2       10m

NAME                                      REFERENCE            TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/swb-app   Deployment/swb-app   15%/70%   2         10        2          10m
```

### Cek Log Aplikasi

**Log backend:**
```bash
kubectl logs -f deployment/swb-app -n swb-wahana
```

**Log frontend (NGINX):**
```bash
kubectl logs -f deployment/swb-nginx -n swb-wahana
```

**Tekan `Ctrl+C` untuk keluar dari log.**

---

## Troubleshooting - Kalau Ada Masalah

### Masalah 1: Pods Stuck di "ImagePullBackOff"

**Gejala:**

```bash
kubectl get pods -n swb-wahana
```

Output:
```
NAME                       READY   STATUS             RESTARTS   AGE
swb-app-xxxxx              0/1     ImagePullBackOff   0          2m
```

**Penyebab:**
- Credential Docker Hub (atau image registry lain) salah
- Image tidak ada di registry
- Image name/tag salah

**Solusi:**

1. **Cek detail error:**
```bash
kubectl describe pod swb-app-xxxxx -n swb-wahana
```

Lihat bagian `Events`, biasanya ada pesan error seperti:
```
Failed to pull image "user/nama-iamge:tag": rpc error: code = Unknown desc = Error response from daemon: pull access denied
```

2. **Cek secret regcred:**
```bash
kubectl get secret regcred-swb -n swb-wahana -o yaml
```

Pastikan username/password/email benar.

3. **Generate ulang auth base64:**
```bash
echo -n "username:password" | base64
```

Update di file `regcred-swb.yaml`, lalu apply ulang:
```bash
kubectl delete secret regcred-swb -n swb-wahana
kubectl apply -f config/regcred-swb.yaml
```

4. **Restart deployment:**
```bash
kubectl rollout restart deployment swb-app -n swb-wahana
```

---

### Masalah 2: Pods Stuck di "CrashLoopBackOff"

**Gejala:**

```
NAME                       READY   STATUS             RESTARTS   AGE
swb-app-xxxxx              0/1     CrashLoopBackOff   5          5m
```

**Penyebab:**
- Aplikasi crash saat start
- Hosts tidak bisa diakses
- Konfigurasi salah

**Solusi:**

1. **Cek log pods:**
```bash
kubectl logs swb-app-xxxxx -n swb-wahana
```

Lihat error message di paling bawah.

2. **Cek koneksi database:**

Exec ke dalam pod:
```bash
kubectl exec -it swb-app-xxxxx -n swb-wahana -- sh
```

lihat apakah ada hosts di dalam pod (gunakan ping jika command ada):
```bash
getent hosts galera-cluster
```

Jika muncul IP maka sudah Ok. Jika tidak ada, lakukan update hostAliases di deployment backend dan sesuaikan IP dan nama host dengan hosts yang akan digunakan.

3. **Update hostAliases:**

Edit deployment:
```bash
kubectl edit deployment swb-app -n swb-wahana
```

Pastikan IP database benar di bagian `hostAliases`.

---

### Masalah 3: NGINX Pods Error "403 Forbidden"

**Gejala:**

Bisa akses domain, tapi muncul "403 Forbidden" dari NGINX.

**Penyebab:**
- Path ke static files salah
- Permission issue
- ConfigMap NGINX salah

**Solusi:**

1. **Cek log NGINX:**
```bash
kubectl logs swb-nginx-xxxxx -n swb-wahana
```

2. **Exec ke pod NGINX:**
```bash
kubectl exec -it swb-nginx-xxxxx -n swb-wahana -- sh
```

3. **Cek apakah static files ada:**
```bash
ls -la /opt/pm/apps/swb/web/dev/dcaf/dist/pwa
```

Harusnya ada file `index.html` dan folder `js`, `css`, dll.

4. **Kalau folder kosong:**
- Image frontend salah
- Build frontend gagal

Rebuild kembali image frontend hingga static file ada.

---

### Masalah 4: 502 Bad Gateway

**Gejala:**

Saat akses aplikasi di browser, muncul error 502 Bad Gateway.

**Penyebab:**
- Backend pods tidak jalan
- NGINX proxy_pass salah

**Solusi:**

1. **Cek backend pods:**
```bash
kubectl get pods -n swb-wahana | grep swb-app
```

Pastikan `Running`.

2. **Cek ConfigMap NGINX:**

Pastikan `proxy_pass` URL benar:
```
proxy_pass <nama-service-aplikasi>;
```

---

### Masalah 5: HPA Tidak Auto-Scale

**Gejala:**

CPU usage tinggi (>70%), tapi pods tidak bertambah.

**Penyebab:**
- Metrics Server belum di install
- HPA target salah

**Solusi:**

1. **Cek Metrics Server:**
```bash
kubectl get pods -n kube-system | grep metrics-server
```

Kalau tidak ada, install:
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

```bash
kubectl patch deployment metrics-server -n kube-system --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'
```

2. **Cek HPA:**
```bash
kubectl describe hpa swb-app -n swb-wahana
```

Lihat bagian `Conditions` - kalau ada error, perbaiki sesuai pesan.

3. **Test manual scale:**
```bash
kubectl scale deployment swb-app --replicas=5 -n swb-wahana
```

Kalau bisa scale manual, berarti HPA yang bermasalah.

---

## Perintah Penting untuk Dicatat

```bash
# Cek semua resource
kubectl get all -n swb-wahana

# Cek pods
kubectl get pods -n swb-wahana -o wide

# Cek log backend
kubectl logs -f deployment/swb-app -n swb-wahana

# Cek log frontend
kubectl logs -f deployment/swb-nginx -n swb-wahana

# Cek Ingress
kubectl get ingress -n swb-wahana

# Cek HPA
kubectl get hpa -n swb-wahana

# Describe pod (untuk debugging)
kubectl describe pod <pod-name> -n swb-wahana

# Exec ke pod
kubectl exec -it <pod-name> -n swb-wahana -- sh

# Restart deployment
kubectl rollout restart deployment swb-app -n swb-wahana
kubectl rollout restart deployment swb-nginx -n swb-wahana

# Scale manual
kubectl scale deployment swb-app --replicas=5 -n swb-wahana

# Delete deployment (hati-hati!)
kubectl delete deployment swb-app -n swb-wahana

# Apply semua manifest di folder
kubectl apply -f app/
kubectl apply -f nginx/
kubectl apply -f ingress/
kubectl apply -f config/
```

---

## Update Aplikasi (Deploy Versi Baru)

Kalau ada update kode, ikuti langkah ini:

### 1. Update Kode & Konfigurasi

- Edit file yang perlu diubah di source code
- Ubah versi di `package.json` (misal: `0.2.66` → `0.2.67`)
- Ubah versi di `dcaf.yml`
- Update file `axios.js` dan API `.yml` kalau ada perubahan domain

### 2. Rebuild Image

#### Build backend
```bash
docker build -t elangptra/nama-image-backend:tag .
```

#### Build frontend
```bash
docker build -f fe.Dockerfile -t elangptra/nama-image-frontend:tag .
```

### 3. Push Image

```bash
docker push elangptra/nama-image-backend:tag
docker push elangptra/nama-image-frontend:tag
```

### 4. Update Deployment

Edit file `swb-deployment.yaml` dan `nginx-deployment.yaml`, ubah tag image ke versi baru:

```yaml
image: elangptra/nama-image:tag  # <-- Update ini
```

### 5. Apply Update

```bash
kubectl apply -f app/swb-deployment.yaml
kubectl apply -f nginx/nginx-deployment.yaml
```

**Atau gunakan `kubectl set image` (lebih cepat):**

```bash
kubectl set image deployment/swb-app swb-app=elangptra/nama-image-backend:tag -n swb-wahana
kubectl set image deployment/swb-nginx swb-nginx=elangptra/nama-image-frontend:tag -n swb-wahana
```

### 6. Monitor Rollout

```bash
kubectl rollout status deployment/swb-app -n swb-wahana
kubectl rollout status deployment/swb-nginx -n swb-wahana
```

**Output:**
```
deployment "swb-app" successfully rolled out
```

### 7. Rollback (Kalau Update Bermasalah)

```bash
kubectl rollout undo deployment/swb-app -n swb-wahana
kubectl rollout undo deployment/swb-nginx -n swb-wahana
```

✅ **Aplikasi sudah update!**

---

## Konsep Penting (Nice to Have)

### 1. Zero-Downtime Deployment

Kubernetes melakukan rolling update:
1. Start pod baru dengan image baru
2. Tunggu pod baru `Ready` (readiness probe pass)
3. Matikan pod lama
4. Ulangi untuk pod berikutnya

User tidak akan merasakan downtime!

### 2. Auto-Scaling dengan HPA

HPA memantau CPU usage:
- CPU >70% → Tambah pods (sampai max 10)
- CPU <70% → Kurangi pods (sampai min 2)

### 3. Health Checks

**Liveness Probe:**
- Cek apakah aplikasi masih hidup
- Kalau gagal → Restart pod

**Readiness Probe:**
- Cek apakah aplikasi siap terima traffic
- Kalau gagal → Jangan route traffic ke pod ini

### 4. ConfigMap vs Secret

**ConfigMap:**
- Untuk konfigurasi non-sensitive (NGINX config, environment variables)
- Data dalam plain text

**Secret:**
- Untuk data sensitive (password, SSL certificate, token)
- Data di-encode base64

---

## Kesimpulan

**Apa yang sudah tercapai:**

✅ Build Docker image untuk backend dan frontend  
✅ Push image ke Docker Hub (atau registry lainnya)  
✅ Deploy aplikasi SWB dengan 2 replicas  
✅ Setup SSL/TLS untuk HTTPS  
✅ Setup auto-scaling dengan HPA  
✅ Setup Ingress untuk akses dari luar cluster  
✅ High Availability dengan multiple pods  

**Setup sekarang:**
- **Backend:** 2+ pods (bisa auto-scale sampai 10)
- **Frontend:** 2 pods NGINX
- **SSL:** Certificate untuk HTTPS
- **Domain:** Akses via `https://swb-docker100.wahana.com`

## Referensi

>[Kubernetes Tasks - Pull an Image from a Private Registry](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)
>
>[Kubernetes Concepts - TLS Secrets](https://kubernetes.io/docs/concepts/configuration/secret/#tls-secrets)
>
>[Kubernetes Tasks - Adding entries to Pod /etc/hosts with HostAliases](https://kubernetes.io/docs/tasks/network/customize-hosts-file-for-pods/)
>
>[Kubernetes Tasks - Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
>
>[Kubernetes Tasks - Horizontal Pod Autoscaling](https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/)


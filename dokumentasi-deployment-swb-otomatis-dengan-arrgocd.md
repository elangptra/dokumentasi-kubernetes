# SETUP DEPLOYMENT SWB KE KUBERNETES MENGGUNAKAN ARGOCD
> 

Dokumen ini menjelaskan alur kerja deployment aplikasi (GitOps Workflow) ke dalam cluster Kubernetes. Panduan ini mencakup persiapan *image*, konfigurasi manifest, manajemen *secrets*, hingga sinkronisasi otomatis menggunakan ArgoCD.

## 📋 Daftar Isi
- [SETUP DEPLOYMENT SWB KE KUBERNETES MENGGUNAKAN ARGOCD](#setup-deployment-swb-ke-kubernetes-menggunakan-argocd)
  - [📋 Daftar Isi](#-daftar-isi)
  - [1. PRASYARAT (MANDATORY)](#1-prasyarat-mandatory)
  - [2. SETUP CONFIGURASI GITLAB](#2-setup-configurasi-gitlab)
    - [2.1 Buat Repository GITLAB](#21-buat-repository-gitlab)
    - [2.2 Buat Personal Access Token (PAT) Gitlab](#22-buat-personal-access-token-pat-gitlab)
      - [Navigasi Menu](#navigasi-menu)
      - [Definisi Token Baru](#definisi-token-baru)
      - [Simpan Personal Access Token](#simpan-personal-access-token)
      - [Validasi Akses](#validasi-akses)
    - [2.3 Manajemen Variabel CI/CD (Project Variables)](#23-manajemen-variabel-cicd-project-variables)
      - [Navigasi Project Settings](#navigasi-project-settings)
      - [Penambahan Variabel (Add Variable)](#penambahan-variabel-add-variable)
      - [Variabel Aplikasi (Opsional - Sesuai Kebutuhan)](#variabel-aplikasi-opsional---sesuai-kebutuhan)
  - [3. PEMBUATAN MANIFEST KUBERNETES](#3-pembuatan-manifest-kubernetes)
    - [3.1 swb-deployment.yaml](#31-swb-deploymentyaml)
    - [3.2 swb-service.yaml](#32-swb-serviceyaml)
    - [3.3 swb-gitlab.yaml](#33-swb-gitlabyaml)
    - [3.4 swb-secret-store.yaml](#34-swb-secret-storeyaml)
    - [3.5 swb-ssl-secret.yaml](#35-swb-ssl-secretyaml)
    - [3.6 ingress-config.yaml](#36-ingress-configyaml)
    - [3.7 nginx-deployment.yaml](#37-nginx-deploymentyaml)
    - [3.8 nginx-service.yaml](#38-nginx-serviceyaml)
    - [3.9 nginx-config.yaml](#39-nginx-configyaml)
    - [3.10 regcred-swb.yaml](#310-regcred-swbyaml)
    - [3.11 swb-hpa.yaml](#311-swb-hpayaml)
  - [4. DEPLOYMENT PROJECT MENGGUNAKAN ARGOCD](#4-deployment-project-menggunakan-argocd)
    - [4.1 Publikasi Manifest (Git Push)](#41-publikasi-manifest-git-push)
    - [4.2 Konfigurasi aplikasi ARGOCD](#42-konfigurasi-aplikasi-argocd)
      - [Inisialisasi Repository](#inisialisasi-repository)
      - [Tambahkan Repositori Baru](#tambahkan-repositori-baru)
      - [Inisialisasi Aplikasi](#inisialisasi-aplikasi)
      - [Buka Menu Pembuatan](#buka-menu-pembuatan)
      - [Eksekusi](#eksekusi)
      - [Status Indikator](#status-indikator)
  - [Referensi](#referensi)


## 1. PRASYARAT (MANDATORY)
**Sangat Penting:** Sebelum melanjutkan tahap ini, pastikan infrastruktur berikut **sudah terinstall dan berjalan** di cluster:

> 1.  **[Cluster Kubernetes](1.%20Instalasi-kubernetes-3-node-cluster.md)** (Running & Healthy)
> 2.  **[Akun GitLab](https://gitlab.com)** (Untuk Repository & Registry)
> 3.  **[ArgoCD](5.%20instalasi-dan-penggunaan-argocd-dan-eso.md)** (CD Controller)
> 4.  **[MetalLB](2.%20instalasi-ingress-metallb.md)** (Load Balancer Controller)
> 5.  **[Ingress NGINX](2.%20instalasi-ingress-metallb.md)** (Ingress Controller)
> 6.  **[External Secret Operator](3.%20instalasi-external-secret-operator.md)** (Secret Management)
> 7.  **[Images](https://dockerhub.com)** (Builded Image)
> 8.  **[Headlamp](4.%20instalasi-headlamp-dashboard.md)** *(Opsional - UI Dashboard)*
> 9.  **[Grafana](6.%20Instalasi-grafana-prometheus-loki.md)** *(Opsional - Monitoring)*


## 2. SETUP CONFIGURASI GITLAB

### 2.1 Buat Repository GITLAB
membuat wadah penyimpanan kode (repository) di [GitLab](github.com).

1.  Masuk ke dashboard GitLab.
2.  Navigasi ke menu **Projects** > **New project**.
3.  Pilih opsi **Create blank project**.

4.  Isi konfigurasi proyek sebagai berikut:
    * **Project name:** `swb-wahana` (Sesuaikan dengan nama aplikasi).
    * **Project slug:** (Otomatis terisi).
    * **Visibility Level:** **Private** (Wajib, untuk keamanan kredensial).
    * **Project Configuration:** Centang **Initialize repository with a README**.
5.  Klik tombol **Create project**.


Setelah repositori terbentuk, lakukan *cloning* ke mesin lokal dan bangun struktur direktori sesuai standar arsitektur.

Buka terminal dan jalankan perintah berikut (ganti URL dengan URL repositori Anda):

```bash
git clone https://gitlab.com/USERNAME_ANDA/swb-wahana.git
cd swb-wahana
```

### 2.2 Buat Personal Access Token (PAT) Gitlab
Personal Access Token (PAT) berfungsi sebagai pengganti *password* akun pengguna, dengan cakupan akses (*scope*) yang dapat dibatasi.

#### Navigasi Menu
1.  Klik pada **Avatar Profil** pengguna (pojok kanan atas).
2.  Pilih **Edit profile**.
3.  Pada menu navigasi kiri, pilih **Access Tokens**.


#### Definisi Token Baru
Isi formulir pembuatan token dengan parameter berikut:

* **Token name:** `argocd-access-token` (Deskriptif untuk audit log).
* **Expiration date:** Opsional (Saran: Setel 1 tahun untuk stabilitas produksi, atau sesuai kebijakan keamanan perusahaan).
* **Select scopes:**
* >**NOTE: Bisa disesuaikan lagi sesuai keinginan**
    * [x] `read_repository` (Wajib: Izinkan akses baca kode).
    * [x] `read_registry` (Wajib: Izinkan akses tarik *image* container).
    * [x] `write_repository` (opsional: jika ingin menggunkaan image updater version di argo, centang).


#### Simpan Personal Access Token
1.  Klik tombol **Generate token**.
2.  **PENTING:** Salin token yang muncul di bagian atas layar (`glpat-xxxxxxxxxxxx`).
    > **PERINGATAN! :** Token ini hanya ditampilkan **sekali**. Jika halaman di-refresh sebelum disalin, token tidak dapat dilihat kembali dan harus dibuat ulang.

#### Validasi Akses
Pastikan token berfungsi dengan mencoba melakukan login ke registry via terminal lokal (memerlukan Docker terinstall):

```bash
docker login registry.gitlab.com -u <USERNAME_GITLAB> -p <PAT_TOKEN_ANDA>
```

### 2.3 Manajemen Variabel CI/CD (Project Variables)

Langkah ini bertujuan menyimpan data sensitif (seperti kredensial registry) agar dapat digunakan oleh pipeline atau External Secrets tanpa terekspos dalam kode sumber.

#### Navigasi Project Settings
1.  Buka halaman utama proyek `swb-wahana`.
2.  Pada sidebar kiri, pilih **Settings** > **CI/CD**.
3.  Cari bagian **Variables** dan klik tombol **Expand**.

#### Penambahan Variabel (Add Variable)
Klik **Add variable** dan buat entri berikut:

#### Variabel Aplikasi (Opsional - Sesuai Kebutuhan)
Jika aplikasi membutuhkan *Database Password* atau *API Key*, tambahkan dengan cara seperti ini.
* **Key:** `DB_PASSWORD`
* **Value:** `RahasiaSuperAman123!`
* **Flags:** Mask variable (Wajib).
  
pada applikasi **SWB-wahana** Variable hanya diisi dengan ssl certificate crt dan ssl certificate key
* **Key:** `LOCALHOST_CRT`
* **Value:** `HASH*****`
* **Flags:** Mask variable (Wajib).
---
* **Key:** `LOCALHOST_DECRYPTED_KEY`
* **Value:** `HASH*****`
* **Flags:** Mask variable (Wajib).

## 3. PEMBUATAN MANIFEST KUBERNETES
Untuk mempermudah pembacaan file manifest, kita akan mengikuti struktur directory di dalam folder swb-wahana yang sudah di clone seperti berikut

```text
.
├── App
│   └── swb-deployment.yaml
|   └── swb-service.yaml
├── Eso
|   └── swb-gitlab.yaml
|   └── swb-secret-store.yaml
|   └── swb-ssl-secret.yaml
├── Ingress
|   └── ingress-config.yaml
├── Nginx
|   └── nginx-deployment.yaml
|   └── nginx-service.yaml
|   └── nginx-config.yaml
├── Secret
|   └── regcred-swb.yaml
├── HPA
|   └── swb-hpa.yaml
```
untuk isi setiap file ikuti code berikut :

### 3.1 swb-deployment.yaml
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
        - ip: 192.168.2.23
          hostnames:
            - "galera-cluster"
            - "galera-cluster23"
            - "galera-cluster-drc" 
            - "redis-local"
            - "redis106"
      imagePullSecrets:
        - name: regcred-swb
      containers:
        - name: swb-app
          image: elangptra/swb-wahana:v1.4
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
### 3.2 swb-service.yaml
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
    app: swb-app #harus mengikuti nama deployment
```
### 3.3 swb-gitlab.yaml
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: swb-gitlab-token
  namespace: swb-wahana
type: Opaque
stringData:
  token: #isi dengan Personal Access Token Gitlab
```
### 3.4 swb-secret-store.yaml
```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: gitlab-store
  namespace: swb-wahana
spec:
  provider:
    gitlab:
      url: https://gitlab.com
      auth:
        SecretRef:
          accessToken:
           name: swb-gitlab-token
           key: token
      projectID: "78080012" # sesuaikan project ID
```
### 3.5 swb-ssl-secret.yaml
```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: swb-ssl-secret
  namespace: swb-wahana
spec:
  refreshInterval: 300s
  secretStoreRef:
    name: gitlab-store
    kind: SecretStore
  target:
    name: swb-ssl
    template: 
      type: kubernetes.io/tls
      data:
        tls.crt: "{{ .crt_file }}"
        tls.key: "{{ .key_file }}"
  data:
    - secretKey: crt_file
      remoteRef:
        key: LOCALHOST_CRT
    - secretKey: key_file
      remoteRef:
        key: LOCALHOST_DECRYPTED_KEY
```
### 3.6 ingress-config.yaml
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
    - swb-docker100.wahana.com #sesuaikan dengan domain yang tersedia
    secretName: swb-ssl
  rules:
  - host: swb-docker100.wahana.com #sesuaikan dengan domain yang tersedia
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
### 3.7 nginx-deployment.yaml
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
        image: elangptra/swb-quasar:v1.2
        ports:
          - containerPort: 80
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
### 3.8 nginx-service.yaml
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
### 3.9 nginx-config.yaml
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: swb-wahana
data:
  default.conf: |
    server {
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

      location / {
        # try_files $uri $uri/ =404;
        # proxy_pass http://localhost:8088;
      }

      location /lib/ {
          try_files $uri $uri/ =404;
          # if ($http_referer !~ "^$uri.*$"){
          #      return 403;
          # }
          # proxy_pass http://127.0.0.1:8091;
      }

      location /a/ {
        add_header 'Content-Security-Policy' "script-src 'unsafe-eval' 'self' *.pirantimaya.id http://localhost:*; worker-src blob:; child-src blob:; style-src 'unsafe-inline' 'self' *.pirantimaya.id http://localhost:*;" always;
        client_max_body_size 5M;

        if ($request_method = 'OPTIONS') {

            add_header 'Access-Control-Allow-Origin' '*' always;
            add_header 'Access-Control-Allow-Methods' 'GET, POST, DELETE, PUT, PATCH, OPTIONS';
            #
            # Custom headers and headers various browsers *should* be OK with but aren't
            #
            add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization,X-API-Key,D-Request-Id';
            #
            # Tell client that this pre-flight info is valid for 20 days
            #
            add_header 'Access-Control-Max-Age' 1728000;
            add_header 'Content-Type' 'text/plain; charset=utf-8';
            add_header 'Content-Length' 0;
            return 204;
        }
        if ($request_method = 'POST') {
            add_header 'Access-Control-Allow-Origin' '*' always;
            add_header 'Access-Control-Allow-Methods' 'GET, POST, DELETE, PUT, PATCH, OPTIONS';
            add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization,X-API-Key,D-Request-Id';
            add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Range,D-Request-Id';
        }
        if ($request_method = 'PUT') {
            add_header 'Access-Control-Allow-Origin' '*' always;
            add_header 'Access-Control-Allow-Methods' 'GET, POST, DELETE, PUT, PATCH, OPTIONS';
            add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization,X-API-Key,D-Request-Id';
            add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Range,D-Request-Id';
        }
        if ($request_method = 'DELETE') {
            add_header 'Access-Control-Allow-Origin' '*' always;
            add_header 'Access-Control-Allow-Methods' 'GET, POST, DELETE, PUT, PATCH, OPTIONS';
            add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization,X-API-Key,D-Request-Id';
            add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Range,D-Request-Id';
        }
        if ($request_method = 'GET') {
            add_header 'Access-Control-Allow-Origin' '*' always;
            add_header 'Access-Control-Allow-Methods' 'GET, POST, DELETE, PUT, PATCH, OPTIONS';
            add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization,X-API-Key,D-Request-Id';
            add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Range,D-Request-Id';
        }

        include uwsgi_params;
        uwsgi_pass swb-app-service:3189;
        uwsgi_modifier1 5;
      }

      location /nginx_status {
        stub_status;
        allow 127.0.0.1;  #only allow requests from localhost
        allow 192.168.88.105;  #only allow requests from localhost
        deny all;   #deny all other hosts
      }

      location /doc/ {
        auth_basic "Restricted Access";
        auth_basic_user_file /opt/pm/apps/swb/etc/.user;
      }

      listen 443 ssl;
      ssl_certificate /ssl/ssl-wahana/tls.crt;
      ssl_certificate_key /ssl/ssl-wahana/tls.key;

      ssl_session_cache shared:le_nginx_SSL:10m;
      ssl_session_timeout 1440m;
      ssl_session_tickets off;

      ssl_protocols TLSv1.2 TLSv1.3;
      ssl_prefer_server_ciphers off;

      ssl_ciphers "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384";

    }

    server {
      if ($host = localhost) {
        return 301 https://$host$request_uri;
      } # managed by Certbot

      server_name localhost;
      listen 80;
      return 404; # managed by Certbot
    }
```
### 3.10 regcred-swb.yaml
> **PERINGATAN !**
> **regcred digunakan untuk deployment pull images dari private repo, dan configurasi dibawah ini mengarah ke dockerhub**
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
          "password": "*****", # gunakan Personal Access Token dockerhub
          "email": "elangptra17@gmail.com",
          "auth": "echo -n 'username:password' | base64"
        }
      }
    }
```
>**jika ingin menggunakan wahana registry bisa menggunakan yaml berikut:**
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
            "docker-registry.stage.wahana.com": {
            "auth": "xxxxxxxx" #Sesuaikan Auth nya
            }
        }
    }
```
### 3.11 swb-hpa.yaml
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
    name: swb-app # harus sama seperti nama deployment
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70 # Bisa disesuaikan batas nya, tergantung penggunaan
```

## 4. DEPLOYMENT PROJECT MENGGUNAKAN ARGOCD

kita akan menggunakan ArgoCD sebagai ***Continuous Delivery Controller*** yang secara otomatis menyinkronkan keadaan cluster dengan repositori Git, tanpa harus menyentuh CLI seperti `kubectl apply`.

### 4.1 Publikasi Manifest (Git Push)

Sebelum ArgoCD dapat membaca konfigurasi, seluruh file YAML yang telah Anda letakkan di folder lokal harus diunggah ke repositori utama (GitLab).

jalankan perintah berikut untuk menambahkan directory ke repository gitlab

```bash
git add .
git commit -m "feat: push files into gitlab repo"
git push origin main
```
> **PERINGATAN:** <br>
> **PASTIKAN FILE SUDAH DIPUSH KE REPOSITORY GITLAB, JIKA ADA 1 FILE YANG TERTINGGAL, DEPLOYMENT PASTI `ERROR`**


### 4.2 Konfigurasi aplikasi ARGOCD 
Langkah ini dilakukan melalui antarmuka grafis (Web UI) ArgoCD. Kita akan mendaftarkan repositori GitLab sebagai sumber file.

#### Inisialisasi Repository

1.  Login ke Dashboard ArgoCD.
2.  Klik ikon **Gear ⚙️ (Settings)** pada menu *sidebar* sebelah kiri.
3.  Pilih menu **Repositories**.

> ![connect ke repo](images/choose%20settings.png)

#### Tambahkan Repositori Baru
1.  Klik tombol **+ CONNECT REPO** di bagian atas.
2.  Formulir koneksi akan muncul. Isi dengan detail presisi berikut:

| Parameter | Nilai / Instruksi |
| :--- | :--- |
| **Choose your connection method** | `VIA HTTPS` Karena koneksi kita menggunakan gitlab
| **Type** | `git` (Biarkan default) |
| **Project** | `default` |
| **Repository URL** | Masukkan URL HTTPS GitLab Anda. |
| **Username** | Ketik username akun GitLab Anda. |
| **Password** | **PENTING:** Tempelkan **Personal Access Token (PAT)** di sini. <br>⚠️ *JANGAN masukkan password login GitLab biasa.* |
| **TLS Client Cert** | (Kosongkan) |
| **TLS Client Key** | (Kosongkan) |

![alt](images/connect%20repos.png)

> **PENTING !** <br>
> **PASTIKAN STATUS REPO SUCCESSFUL** <br>
> ![check status repo](images/pastikan%20connection%20status%20nya%20successful.png)
   
#### Inisialisasi Aplikasi

Setelah repositori terhubung, langkah selanjutnya adalah mendefinisikan spesifikasi aplikasi. Kita akan memberitahu ArgoCD **apa** yang harus dideploy (Source) dan **ke mana** tujuannya (Destination).

#### Buka Menu Pembuatan
1.  Klik tombol **+ NEW APP** di pojok kiri atas dashboard ArgoCD.
2.  Akan muncul panel formulir panjang, ikuti panduan tabel di bawah ini.


| Parameter | Nilai / Instruksi | Keterangan Penting |
| :--- | :--- | :--- |
| **Application Name** | `swb-wahana` | Gunakan huruf kecil, tanpa spasi. |
| **Project** | `default` | Project bawaan ArgoCD. |
| **Sync Policy** | `manual` | **Wajib.** Agar ArgoCD otomatis mendeteksi update dari Git. |
| **Sync Options** | ☑️ `Prune Resources` <br> ☑️ `Self Heal` | • **Prune:** Hapus resource di cluster jika file di Git dihapus. <br> • **Self Heal:** Koreksi otomatis jika ada perubahan manual di cluster. |
| **Source - Repository URL** | *(Pilih URL Repo Anda)* | Pilih dari dropdown (karena sudah di-connect di [pembuatan koneksi repo](#inisialisasi-repository)). |
| **Source - Revision** | `main` | Atau sesuaikan dengan branch untuk production. |
| **Source - Path** | `.` | **Ketik tanda TITIK saja.** <br> Artinya "Root Directory". |
| **Destination - Cluster URL** | `https://kubernetes.default.svc` | Pilih URL cluster lokal tempat ArgoCD berjalan. |
| **Destination - Namespace** | `swb-wahana` | Namespace tujuan deployment. |
| **Namespace Option** | ☑️ `Auto-Create Namespace` | **PENTING:** Centang kotak kecil ini agar Namespace dibuat otomatis. |

#### Eksekusi
1.  Cek kembali isian Anda.
2.  Klik tombol **CREATE** di bagian atas panel.
3.  Anda akan diarahkan ke tampilan *Card* aplikasi. Tunggu hingga status berubah dari **Processing** 🔄 menjadi **Healthy** 💚.

> **Tips:**
> Jika status tetap **OutOfSync** (Kuning) meski sudah pilih auto-sync, klik tombol **SYNC** > **SYNCHRONIZE** secara manual untuk pemicu awal.

![check status app swb](images/check%20status%20app%20swb.png)

#### Status Indikator

Perhatikan ikon status pada kartu aplikasi:

    🔄 Processing (Biru): ArgoCD sedang membaca Git dan menerapkan manifest ke cluster.

    💚 Healthy (Hijau): Semua komponen (Pod, Service, Ingress) berjalan normal.

    ✅ Synced (Hijau): Konfigurasi di cluster sudah 100% sama dengan di Git.

    💔 Degraded (Merah): Ada error pada konfigurasi (misal: Image tidak ditemukan atau Secret salah).

## Referensi

>[GitLab - Personal Access Tokens](https://docs.gitlab.com/user/profile/personal_access_tokens/)
>
>[GitLab - CI/CD Variables](https://docs.gitlab.com/ci/variables/)
>
>[K8s Tasks - Adding entries to Pod /etc/hosts](https://kubernetes.io/docs/tasks/network/customize-hosts-file-for-pods/)
>
>[K8s Tasks - Configure Liveness/Readiness Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
>
>[K8s Tasks - Pull Image from Private Registry](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)
>
>[External Secrets - Advanced Templating](https://external-secrets.io/latest/guides/templating/)
>
>[NGINX Ingress - Backend Protocol](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#backend-protocol)
>
>[NGINX Ingress - Proxy SSL](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#proxy-ssl)
>
>[ArgoCD - Connecting to Private Repos](https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/#connecting-via-https)
>
>[ArgoCD - Auto Sync](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/)
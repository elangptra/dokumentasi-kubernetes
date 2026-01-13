# Panduan Instalasi Kubernetes (Multi-Node Simulation) pada Single Bare Metal

Dokumentasi ini membahas cara membangun cluster Kubernetes 3-Nodes (1 Master, 2 Workers) di atas **satu server fisik (Bare Metal)** dengan spesifikasi sumber daya terbatas (3-4 Core CPU).

Metode yang digunakan adalah **"Nodes-in-Containers"** menggunakan `k3d`. Ini memungkinkan kita mensimulasikan arsitektur production tanpa membebani server dengan virtualisasi OS yang berat.

## 📋 Prasyarat Sistem

>[!warning] Sebelum memulai instalasi, pastikan lingkungan "Bare Metal" Anda memenuhi spesifikasi minimum berikut agar cluster berjalan stabil.

| Komponen | Spesifikasi Minimum | Rekomendasi Senior |
| :--- | :--- | :--- |
| **🖥️ Hardware** | 4 Core CPU, 8GB RAM | Gunakan **SSD** agar performa etcd/database cepat. |
| **🐧 Sistem Operasi** | Debian 13 (Trixie) / Ubuntu 22.04+ | Pastikan update kernel terbaru sudah terinstall. |
| **🛡️ Hak Akses** | User dengan `sudo` / Root | Jangan gunakan user root langsung, gunakan `sudo`. |
| **🌐 Koneksi** | Internet Stabil | Diperlukan untuk pull image Docker & K3s. |

<br>

>[!important] : Spesifikasi CPU 4 Core sangat krusial karena akan menjalankan 3 Node (Virtual) dalam satu mesin.




---

## 🚀 Langkah 1: Persiapan Sistem Operasi

Pilih panduan di bawah ini sesuai dengan OS yang terinstall di server Anda.

### A. Untuk Pengguna Debian 13 (Trixie)
Debian Trixie adalah versi *testing/next-stable*. Karena sangat baru, repository Docker resmi mungkin belum menyediakan folder khusus untuknya. Kita perlu trik khusus.

3.  **Siapkan Keyring Keamanan:**
    ```bash
    sudo apt-get update 
    sudo apt-get install -y ca-certificates curl gnupg iptables
    sudo install -m 0755 -d /etc/apt/keyrings
    curl -fsSL [https://download.docker.com/linux/debian/gpg](https://download.docker.com/linux/debian/gpg) | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    sudo chmod a+r /etc/apt/keyrings/docker.gpg
    ```
    > [!WARNING] JIKA MUNCUL ERROR syntax error near unexpected token 
    > - berarti anda kemungkinan menggunakan CLI Linux
    > - ubah curl -fsSL menjadi seperti ini  curl -fsSL "https://download.docker.com/linux/debian/gpg" | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

4.  **Tambahkan Repository (Compatibility Mode):**
    Kita menggunakan repo `bookworm` (Debian 12) jika repo `trixie` belum tersedia, agar instalasi stabil.
    ```bash
    echo \
      "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] [https://download.docker.com/linux/debian](https://download.docker.com/linux/debian) \
      bookworm stable" | \
      sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    ```

5.  **Update Paket:**
    ```bash
    sudo apt-get update
    ```

### B. Untuk Pengguna Ubuntu (22.04 / 24.04)
Instalasi di Ubuntu lebih *straightforward* karena dukungan resminya sangat luas.

1.  **Siapkan Keyring:**
    ```bash
    sudo apt-get update
    sudo apt-get install -y ca-certificates curl gnupg
    sudo install -m 0755 -d /etc/apt/keyrings
    curl -fsSL [https://download.docker.com/linux/ubuntu/gpg](https://download.docker.com/linux/ubuntu/gpg) | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    sudo chmod a+r /etc/apt/keyrings/docker.gpg
    ```

2.  **Tambahkan Repository:**
    ```bash
    echo \
      "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] [https://download.docker.com/linux/ubuntu](https://download.docker.com/linux/ubuntu) \
      "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
      sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    ```

3.  **Update Paket:**
    ```bash
    sudo apt-get update
    ```

---

## 🐳 Langkah 2: Instalasi Docker Engine & K3d

Docker berfungsi sebagai fondasi, sedangkan K3d adalah alat untuk membuat "node palsu" (container) yang berperilaku seperti server asli.

**Jalankan perintah ini di semua jenis OS:**

1.  **Install Docker:**
    ```bash
    sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    ```

2.  **Install K3d (Cluster Manager):**
    ```bash
    curl -s [https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh](https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh) | bash
    ```

3.  **Install Kubectl (Remote Control Kubernetes):**
    ```bash
    curl -LO "[https://dl.k8s.io/release/$(curl](https://dl.k8s.io/release/$(curl) -L -s [https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl](https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl)"
    sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
    ```

---

## 🏗️ Langkah 3: Membuat Cluster (The Safe Way)

Kita akan membuat cluster dengan 1 Master dan 2 Worker.
**PENTING:** Kita membatasi *port mapping* agar server tidak *hang* karena kehabisan CPU.

Jalankan perintah berikut:

```bash
k3d cluster create my-cluster \
    --servers 1 \
    --agents 2 \
    --port "8080:80@loadbalancer" \
    --port "30000-30005:30000-30005@server:0" \
    --timeout 120s
```
> [!NOTE]  Penjelasan Parameter (Kenapa settingannya begini?):
> - ```--agents 2```: Menyiapkan 2 node worker tambahan (Total 3 node dalam cluster).
> - ```--port "8080:80..."```: Mengakses web server cluster (port 80) lewat port 8080 di browser laptop kita.
> - ```--port "30000-30005..."```: Hanya membuka 6 port eksternal. Jangan membuka range 30000-32767 secara penuh karena akan membuat ribuan proses proxy yang memakan habis RAM/CPU server fisik.
> - ```--timeout 120s```: Memberi waktu napas lebih panjang saat inisialisasi agar tidak error di hardware terbatas.

## ✅ Langkah 4: Validasi (Cek Apakah Cluster Hidup)

Saatnya memastikan bahwa "armada kapal" kita sudah siap berlayar.
1. **Cek Daftar Node (Server) :**
Ketik perintah "sakti" ini untuk melihat status server:

    ```bash
    kubectl get nodes
    ```
    Cara Membaca Output: Jika instalasi sukses, Anda akan melihat tampilan seperti ini:

    ```bash
    NAME                      STATUS   ROLES                  AGE     VERSION
    k3d-my-cluster-server-0   Ready    control-plane,master   2m      v1.27.x+k3s1
    k3d-my-cluster-agent-0    Ready    <none>                 2m      v1.27.x+k3s1
    k3d-my-cluster-agent-1    Ready    <none>                 2m      v1.27.x+k3s1
    ```
    > [!NOTE] Arti dari header pada output get nodes 
    > - STATUS Ready: Server sehat. Jika NotReady, tunggu 1-2 menit lagi.
    > - ROLES control-plane: Ini adalah Kapten kapal (Master).
    > - ROLES <none>: Ini adalah Awak kapal (Worker), tempat aplikasi Anda berjalan.

1. **Cek Kesehatan Sistem :**
    Pastikan organ dalam cluster (Network, DNS) berjalan:

    ```Bash
    kubectl get pods -A
    ```
    > [!warning] Pastikan mayoritas statusnya Running.
    



## 🔑 Langkah 5: Manajemen Akses User (Agar Bisa Dipakai User Biasa)

Secara default, hanya "Root" (Administrator) yang memegang kendali penuh. Langkah ini bertujuan memberikan "kunci" kepada user biasa (misalnya user `damar`, `elang`, atau user pribadi Anda) agar bisa mengelola cluster tanpa harus terus-menerus mengetik `sudo`.

Ikuti langkah ini dengan teliti:

### 1. Berikan Izin Menggunakan Docker
Docker adalah mesin utama. Kita perlu mendaftarkan user Anda ke dalam grup "khusus" agar diperbolehkan memerintah Docker.

Jalankan perintah ini:

```bash
sudo usermod -aG docker $USER
```

> [!warning] WAJIB DILAKUKAN
> Setelah menjalankan perintah di atas, perubahan BELUM aktif. Anda HARUS Logout (keluar akun) lalu Login kembali ke server agar izin tersebut diterapkan.

Setelah login kembali, kita akan menyalin kunci akses dari K3d ke user Anda.
Jalankan perintah ini satu per satu (sebagai user biasa/lain):

```bash
mkdir -p ~/.kube
k3d kubeconfig get my-cluster > ~/.kube/config

# 3. 
chmod 600 ~/.kube/config
```

> [!note] Arti dari perintah diatas
> - ```mkdir -p ~/.kube``` Membuat folder khusus untuk menyimpan konfigurasi
> - ```k3d kubeconfig get my-cluster > ~/.kube/config``` Mengambil konfigurasi cluster dan menyimpannya ke file config
> - ```chmod 600 ~/.kube/config``` Mengamankan file agar hanya user Anda yang bisa membacanya (Standar Keamanan)

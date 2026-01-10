# 📘 Panduan Setup Kubernetes 3-Node Cluster

### Kubernetes v1.35.0 + containerd + Calico Tigera

Dokumentasi ini menjelaskan langkah-demi-langkah membuat cluster Kubernetes dengan:

- 1 Control Plane (Master) - otak dari cluster yang mengatur semuanya
- 2 Worker Node - server yang menjalankan aplikasi Anda
- Container runtime: **containerd** - software untuk menjalankan container
- CNI: **Calico Tigera** - software untuk networking antar pod
- Pod CIDR: **10.244.0.0/16** - range IP address untuk pod

---

## 🧩 1. Arsitektur & Topologi

Cluster akan memiliki struktur seperti di bawah ini:

| Role         | Hostname      | IP Address      | OS               | CPU | RAM |
|-------------|---------------|----------------|------------------|-----|-----|
| Master Node | `k8s-master`  | `192.168.2.104` | Debian 13 Trixie | 2+  | 4GB |
| Worker 1    | `k8s-worker1` | `192.168.2.105` | Debian 13 Trixie | 2+  | 4GB |
| Worker 2    | `k8s-worker2` | `192.168.2.106` | Debian 13 Trixie | 2+  | 4GB |

**⚠️ PENTING: Sesuaikan IP Address di atas dengan IP server Anda!**

**Minimal Spesifikasi yang Direkomendasikan:**
- CPU: 2 core
- RAM: 4GB
- Disk: 20GB+
- Internet aktif

> ✅ Pastikan semua node bisa saling ping satu sama lain

---

## 🛠 2. Tujuan Setup

Dengan panduan ini, Anda akan mendapatkan:

✔ Sebuah cluster Kubernetes dengan 3 server (nodes)  
✔ Worker node siap menjalankan aplikasi  
✔ Jaringan pod menggunakan Calico  
✔ Runtime container menggunakan containerd  

---

## 🔑 3. Persiapan Dasar (Wajib di Semua Node)

**📍 Langkah ini dilakukan di semua server:**
- `k8s-master`
- `k8s-worker1`
- `k8s-worker2`

### 3.1 Set Hostname

Jalankan perintah sesuai dengan nama server masing-masing:

**Di Server Master:**
```bash
sudo hostnamectl set-hostname k8s-master
```

**Di Server Worker 1:**
```bash
sudo hostnamectl set-hostname k8s-worker1
```

**Di Server Worker 2:**
```bash
sudo hostnamectl set-hostname k8s-worker2
```

---

### 3.2 Tambahkan Hosts Mapping (di Semua Node)

Tujuannya agar setiap server bisa saling mengenali menggunakan nama, bukan hanya IP address.

Edit file hosts:
```bash
sudo nano /etc/hosts
```

Tambahkan baris ini di paling bawah (sesuaikan dengan IP server Anda):
```
192.168.2.104 k8s-master
192.168.2.105 k8s-worker1
192.168.2.106 k8s-worker2
```

**Tekan `Ctrl + X`, lalu `Y`, dan `Enter` untuk menyimpan.**

---

### 3.3 Disable Swap

Swap harus dimatikan karena Kubernetes memerlukan kontrol penuh terhadap memory.

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

**Penjelasan:** Perintah pertama mematikan swap sementara, perintah kedua memastikan swap tetap mati setelah reboot.

---

### 3.4 Aktifkan Kernel Modules

Kernel modules dibutuhkan agar container dapat berjalan dengan baik.

```bash
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
```

Kemudian load module tersebut:

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

---

### 3.5 Konfigurasi Networking (Sysctl)

```bash
sudo tee /etc/sysctl.d/kubernetes.conf <<EOT
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOT
```

Terapkan konfigurasi:

```bash
sudo sysctl --system
```

**Penjelasan:** Konfigurasi ini mengaktifkan IP forwarding agar Pod dapat berkomunikasi satu sama lain.

---

## 🐳 4. Install containerd (di Semua Node)

containerd adalah software yang menjalankan container di dalam Kubernetes.

### 4.1 Install Dependencies

Update package list dan install tools yang dibutuhkan:

```bash
sudo apt update
sudo apt install -y curl gnupg2 apt-transport-https ca-certificates
```

---

### 4.2 Tambahkan Repository Docker

Tambahkan GPG key dari Docker:

```bash
sudo curl -fsSL https://download.docker.com/linux/debian/gpg | \
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

Beri permission untuk membaca file:

```bash
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

Tambahkan repository Docker:

```bash
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/debian \
bookworm stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

**⚠️ Catatan:** Hapus `sudo` di awal perintah `echo`

---

### 4.3 Install containerd

Update package list dan install containerd:

```bash
sudo apt-get update
sudo apt-get install -y containerd.io
```

---

### 4.4 Generate Default Config

Buat folder konfigurasi dan generate config default:

```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

---

### 4.5 Gunakan Systemd Cgroup Driver

Edit konfigurasi agar menggunakan systemd:

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
```

Restart containerd dan cek statusnya:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd
```

> ✅ Pastikan status menunjukkan **active (running)**. Tekan `q` untuk keluar dari tampilan status.

---

## 🚀 5. Install Kubernetes (di Semua Node)

### 5.1 Tambahkan Repository Kubernetes

Update dan install tools yang dibutuhkan:

```bash
sudo apt-get update
sudo apt-get install -y curl gpg
```

Tambahkan GPG key Kubernetes:

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Tambahkan repository Kubernetes:

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

---

### 5.2 Install Package Kubernetes

Update dan install komponen Kubernetes:

```bash
sudo apt-get update
sudo apt-get install -y kubeadm kubectl kubelet
```

Kunci versi agar tidak auto-update:

```bash
sudo apt-mark hold kubeadm kubectl kubelet
```

**Penjelasan:**
- **kubeadm** - tool untuk setup cluster
- **kubectl** - command line untuk kontrol Kubernetes
- **kubelet** - service yang berjalan di setiap node

---

## 🧠 6. Inisialisasi Cluster (Master Node Saja)

**⚠️ HANYA jalankan di Master Node!**

Inisialisasi cluster dengan perintah berikut (sesuaikan IP master Anda):

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --control-plane-endpoint=192.168.2.104
```

**📝 PENTING:** Simpan output yang berisi perintah `kubeadm join ...` karena akan digunakan di worker node nanti!

Contoh output yang harus disimpan:
```
kubeadm join 192.168.2.104:6443 --token abc123.xyz789 \
--discovery-token-ca-cert-hash sha256:1234567890abcdef...
```

### 6.1 Konfigurasi kubectl

Agar kubectl bisa digunakan tanpa sudo:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verifikasi bahwa kubectl sudah bisa digunakan:

```bash
kubectl get nodes
```

Anda akan melihat master node dengan status `NotReady` - ini normal karena networking belum terinstall.

---

## 🌐 7. Install Calico Tigera (Master Node Saja)

Calico adalah plugin networking yang membuat pod bisa berkomunikasi.

### 7.1 Install Operator

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
```

---

### 7.2 Download Konfigurasi Jaringan

```bash
wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml
```

---

### 7.3 Edit File custom-resources.yaml

Buka file dengan nano:

```bash
nano custom-resources.yaml
```

Cari bagian `cidr: 192.168.0.0/16` dan **UBAH** menjadi `10.244.0.0/16`

Hasil akhir seharusnya seperti ini:

```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - blockSize: 26
      cidr: 10.244.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
```

**Tekan `Ctrl + X`, lalu `Y`, dan `Enter` untuk menyimpan.**

---

### 7.4 Apply Konfigurasi

```bash
kubectl create -f custom-resources.yaml
```

---

### 7.5 Verifikasi Pods Calico

Tunggu beberapa saat, lalu cek status pods Calico:

```bash
kubectl get pods -n calico-system
```

Tunggu sampai semua pods menunjukkan status `Running`. Ini bisa memakan waktu 2-5 menit.

Cek kembali nodes:

```bash
kubectl get nodes
```

Master node sekarang seharusnya sudah `Ready`!

---

## 🤝 8. Join Worker Node ke Master (Worker Node Saja)

**⚠️ Sebelum melakukan langkah ini, pastikan Worker Node sudah menyelesaikan langkah 3, 4, dan 5!**

### 8.1 Join Worker ke Cluster

Di setiap worker node, jalankan perintah `kubeadm join` yang Anda simpan dari langkah 6:

```bash
sudo kubeadm join 192.168.2.104:6443 --token abc123.xyz789 \
--discovery-token-ca-cert-hash sha256:1234567890abcdef...
```

**Jika lupa token**, jalankan perintah ini di master node:

```bash
kubeadm token create --print-join-command
```

### 8.2 Troubleshooting: Error CRI

Jika muncul error seperti ini:

```
[ERROR CRI]: could not connect to the container runtime...
unknown service runtime.v1.RuntimeService
```

**Solusi:**

Buka file config containerd:

```bash
sudo nano /etc/containerd/config.toml
```

Cari baris:

```toml
disabled_plugins = ["cri"]
```

Ubah menjadi (kosongkan kurungnya):

```toml
disabled_plugins = []
```

**Tekan `Ctrl + X`, lalu `Y`, dan `Enter` untuk menyimpan.**

Restart containerd:

```bash
sudo systemctl restart containerd
```

Ulangi perintah `kubeadm join`.

---

## 🧪 9. Verifikasi Cluster

Kembali ke master node, cek semua nodes:

```bash
kubectl get nodes
```

Output yang diharapkan:

```
NAME          STATUS   ROLES           AGE   VERSION
k8s-master    Ready    control-plane   10m   v1.35.0
k8s-worker1   Ready    <none>          5m    v1.35.0
k8s-worker2   Ready    <none>          5m    v1.35.0
```

✅ Semua node harus menunjukkan status `Ready`!

Cek semua pods di cluster:

```bash
kubectl get pods -A
```

Semua pods harus berstatus `Running`.

---

## 🎯 10. Hasil Akhir

Selamat! Anda sekarang memiliki:

✔ Kubernetes v1.35.0 yang berfungsi penuh  
✔ 1 Control Plane (Master Node)  
✔ 2 Worker Nodes  
✔ containerd sebagai Container Runtime  
✔ Calico Tigera untuk Networking  

Cluster siap digunakan untuk deploy aplikasi! 🚀

---

## 📊 11. Install Metrics Server (Master Node Saja)

Metrics Server adalah komponen yang mengumpulkan data penggunaan CPU dan Memory dari setiap node dan pod. Data ini berguna untuk:
- Melihat resource usage dengan `kubectl top`
- Auto-scaling (HPA - Horizontal Pod Autoscaler)
- Monitoring performa cluster

### 11.1 Install Metrics Server

Jalankan perintah berikut di master node:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

---

### 11.2 Konfigurasi Metrics Server

Karena kita menggunakan cluster self-hosted (bukan cloud provider), kita perlu menambahkan flag `--kubelet-insecure-tls` agar Metrics Server bisa berkomunikasi dengan kubelet.

**Ada 2 cara untuk melakukan ini:**

#### Cara 1: Menggunakan kubectl patch (Lebih Cepat)

```bash
kubectl patch deployment metrics-server -n kube-system --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'
```

#### Cara 2: Edit Manual (Lebih Jelas)

Jika cara 1 tidak berhasil atau Anda ingin lebih memahami konfigurasinya:

Edit deployment Metrics Server:

```bash
kubectl edit deployment metrics-server -n kube-system
```

Cari bagian `spec` → `containers` → `args`. Akan terlihat seperti ini:

```yaml
args:
- --cert-dir=/tmp
- --secure-port=4443
- --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
- --kubelet-use-node-status-port
- --metric-resolution=15s
```

Tambahkan satu baris `- --kubelet-insecure-tls` di dalam daftar args:

```yaml
args:
- --cert-dir=/tmp
- --secure-port=4443
- --kubelet-insecure-tls                                        # <-- TAMBAHKAN INI
- --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
- --kubelet-use-node-status-port
- --metric-resolution=15s
```

**⚠️ PENTING:** 
- Gunakan **SPASI untuk indentasi**, BUKAN TAB
- Pastikan posisi `-` sejajar dengan baris lainnya
- Tekan `Esc`, ketik `:wq`, lalu `Enter` untuk menyimpan dan keluar

---

### 11.3 Verifikasi Metrics Server

Tunggu beberapa saat, lalu cek status pod Metrics Server:

```bash
kubectl get pods -n kube-system | grep metrics
```

Output yang diharapkan:
```
metrics-server-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
```

**Tunggu sampai status menjadi `Running`!**

---

### 11.4 Test Metrics Server

Setelah pod Metrics Server running, test dengan melihat resource usage:

**Lihat penggunaan resource di semua nodes:**
```bash
kubectl top nodes
```

Output contoh:
```
NAME          CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
k8s-master    250m         12%    1500Mi          37%
k8s-worker1   150m         7%     800Mi           20%
k8s-worker2   180m         9%     900Mi           22%
```

**Lihat penggunaan resource semua pods:**
```bash
kubectl top pods -A
```

✅ Jika perintah di atas berhasil menampilkan data, berarti Metrics Server sudah berfungsi dengan baik!

---

### 11.5 Troubleshooting Metrics Server

**Jika muncul error "Metrics API not available":**

Tunggu 1-2 menit, karena Metrics Server perlu waktu untuk mengumpulkan data pertama kali.

**Jika tetap error setelah 5 menit:**

Cek log Metrics Server:
```bash
kubectl logs -n kube-system deployment/metrics-server
```

Pastikan tidak ada error yang berkaitan dengan certificate atau connection.

**Jika pod CrashLoopBackOff:**

Restart deployment:
```bash
kubectl rollout restart deployment metrics-server -n kube-system
```

---

## 🧪 12. Test Deployment Sederhana (Opsional)

Untuk memastikan cluster benar-benar berfungsi, coba deploy aplikasi sederhana:

```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get pods
kubectl get services
```

Akses nginx melalui browser: `http://<IP-worker-node>:<NodePort>`

Lihat resource usage nginx:
```bash
kubectl top pod
```

Hapus test deployment:

```bash
kubectl delete deployment nginx
kubectl delete service nginx
```

---

## 📚 13. Perintah Berguna

### Perintah Dasar
```bash
# Lihat semua nodes
kubectl get nodes

# Lihat semua pods di semua namespace
kubectl get pods -A

# Lihat pods di namespace tertentu
kubectl get pods -n <namespace>

# Lihat detail node
kubectl describe node k8s-master

# Lihat log pod
kubectl logs <nama-pod> -n <namespace>

# Restart deployment
kubectl rollout restart deployment <nama-deployment>
```

### Perintah Monitoring (Setelah Install Metrics Server)
```bash
# Lihat penggunaan CPU dan Memory nodes
kubectl top nodes

# Lihat penggunaan CPU dan Memory pods
kubectl top pods -A

# Lihat penggunaan resource pod tertentu
kubectl top pod <nama-pod> -n <namespace>

# Lihat penggunaan resource dengan sorting
kubectl top pods -A --sort-by=cpu
kubectl top pods -A --sort-by=memory
```

### Perintah Troubleshooting
```bash
# Lihat events cluster
kubectl get events -A --sort-by='.lastTimestamp'

# Lihat status semua resources
kubectl get all -A

# Cek konfigurasi cluster
kubectl cluster-info

# Cek versi Kubernetes
kubectl version --short

# Describe pod untuk melihat detail error
kubectl describe pod <nama-pod> -n <namespace>
```

---

## 📎 14. Referensi

- [Dokumentasi Kubernetes](https://kubernetes.io/docs)
- [Dokumentasi Calico](https://projectcalico.docs.tigera.io)
- [Dokumentasi containerd](https://containerd.io)
- [Dokumentasi Metrics Server](https://github.com/kubernetes-sigs/metrics-server)

---

**💡 Tips:**
- Simpan kubeconfig (`~/.kube/config`) sebagai backup
- Gunakan `kubectl get events` untuk troubleshooting
- Monitoring resource dengan `kubectl top nodes` setelah install Metrics Server
- Gunakan `kubectl explain <resource>` untuk melihat dokumentasi resource (contoh: `kubectl explain pod`)
- Install `kubectx` dan `kubens` untuk switching context dan namespace dengan mudah
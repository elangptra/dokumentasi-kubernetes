# Panduan Setup Kubernetes 3-Node Cluster

### Kubernetes v1.35.0 + containerd v2.2.1 + Calico Tigera (Binary Installation)

Dokumentasi ini menjelaskan langkah-demi-langkah membuat cluster Kubernetes di **CentOS 7 / RHEL 7** menggunakan **binary files (tar.gz)** karena OS ini sudah End of Life dan tidak lagi mendukung instalasi via package manager resmi.

- 1 Control Plane (Master) — otak dari cluster yang mengatur semuanya
- 2 Worker Node — server yang menjalankan aplikasi Anda
- Container runtime: **containerd v2.2.1** — diinstal via binary tar.gz
- CNI: **Calico Tigera** — software untuk networking antar pod
- Kubernetes: **v1.35.0** — diinstal via binary langsung
- Pod CIDR: **10.244.0.0/16** — range IP address untuk pod

> ⚠️ **PERHATIAN:** CentOS 7 dan RHEL 7 sudah **End of Life** sejak 30 Juni 2024. Kubernetes v1.35 membutuhkan kernel yang cukup baru. Pastikan kernel Anda minimal **3.10.x** (bawaan CentOS 7) dan pertimbangkan upgrade OS untuk environment produksi.

---

## 1. Arsitektur & Topologi

Cluster akan memiliki struktur seperti di bawah ini:

| Role         | Hostname      | IP Address      | OS           | CPU | RAM |
|-------------|---------------|----------------|--------------|-----|-----|
| Master Node | `k8s-master`  | `192.168.2.104` | CentOS 7 / RHEL 7 | 2+  | 4GB |
| Worker 1    | `k8s-worker1` | `192.168.2.105` | CentOS 7 / RHEL 7 | 2+  | 4GB |
| Worker 2    | `k8s-worker2` | `192.168.2.106` | CentOS 7 / RHEL 7 | 2+  | 4GB |

**⚠️ PENTING: Sesuaikan IP Address di atas dengan IP server Anda!**

**Minimal Spesifikasi yang Direkomendasikan:**
- CPU: 2 core
- RAM: 4GB
- Disk: 30GB+
- Internet aktif (untuk download binary)

> ✅ Pastikan semua node bisa saling ping satu sama lain

---

## 2. Tujuan Setup

Dengan panduan ini, Anda akan mendapatkan:

- ✔ Sebuah cluster Kubernetes v1.35.0 dengan 3 server (nodes)
- ✔ containerd v2.2.1 sebagai container runtime (binary installation)
- ✔ kubeadm, kubectl, kubelet v1.35.0 (binary installation)
- ✔ Worker node siap menjalankan aplikasi
- ✔ Jaringan pod menggunakan Calico Tigera

---

## 3. Persiapan Dasar (Wajib di Semua Node)

> 📍 **Langkah ini dilakukan di semua server:** `k8s-master`, `k8s-worker1`, `k8s-worker2`

### 3.1 Set Hostname (Di Semua Node)

Jalankan perintah sesuai dengan nama server masing-masing:

**Di Server Master:**
```bash
sudo hostnamectl set-hostname k8s-master
exec bash
```

**Di Server Worker 1:**
```bash
sudo hostnamectl set-hostname k8s-worker1
exec bash
```

**Di Server Worker 2:**
```bash
sudo hostnamectl set-hostname k8s-worker2
exec bash
```

---

### 3.2 Tambahkan Hosts Mapping (di Semua Node)

Tujuannya agar setiap server bisa saling mengenali menggunakan nama, bukan hanya IP address.

```bash
sudo vim /etc/hosts
```

Tambahkan baris ini di paling bawah (sesuaikan dengan IP server Anda):

```
192.168.2.104 k8s-master
192.168.2.105 k8s-worker1
192.168.2.106 k8s-worker2
```

Tekan `Esc`, ketik `:wq`, lalu `Enter` untuk menyimpan dan keluar.

---

### 3.3 Disable SELinux (di Semua Node)

SELinux perlu dinonaktifkan atau diset ke permissive agar Kubernetes dapat berjalan dengan baik.

```bash
# Set ke permissive sementara (langsung berlaku)
sudo setenforce 0

# Set permanen agar tetap disabled setelah reboot
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

Verifikasi:
```bash
getenforce
# Output harus: Permissive
```

---

### 3.4 Disable Firewalld (di Semua Node)

```bash
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```

> ⚠️ **Catatan:** Jika lingkungan Anda memerlukan firewall, Anda bisa mengkonfigurasi port yang diperlukan Kubernetes secara manual. Untuk setup lab/development, mematikan firewalld adalah cara termudah.

**Port yang dibutuhkan jika ingin tetap menggunakan firewall:**

| Node   | Port            | Keterangan                  |
|--------|-----------------|-----------------------------|
| Master | 6443/tcp        | Kubernetes API Server       |
| Master | 2379-2380/tcp   | etcd                        |
| Master | 10250/tcp       | Kubelet API                 |
| Master | 10251/tcp       | kube-scheduler              |
| Master | 10252/tcp       | kube-controller-manager     |
| Worker | 10250/tcp       | Kubelet API                 |
| Worker | 30000-32767/tcp | NodePort Services           |

---

### 3.5 Disable Swap (di Semua Node)

Swap harus dimatikan karena Kubernetes memerlukan kontrol penuh terhadap memory.

```bash
# Matikan swap sementara
sudo swapoff -a

# Matikan swap permanen (comment baris swap di fstab)
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

Verifikasi:
```bash
free -h
# Kolom Swap harus menunjukkan: 0
```

---

### 3.6 Aktifkan Kernel Modules (di Semua Node)

Buat file konfigurasi modul:

```bash
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
```

Load modul sekarang:

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Verifikasi modul sudah aktif:
```bash
lsmod | grep overlay
lsmod | grep br_netfilter
```

**Penjelasan:**
- **overlay:** Dibutuhkan agar containerd dapat mengelola sistem penyimpanan berlapis (layering) pada kontainer. Tanpa ini, kontainer tidak bisa dibuat.
- **br_netfilter:** Dibutuhkan agar trafik jaringan yang melewati bridge virtual (antar Pod) dapat diproses oleh iptables. Tanpa ini, komunikasi antar Pod akan terputus.

---

### 3.7 Konfigurasi Networking / Sysctl (di Semua Node)

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

Verifikasi:
```bash
sysctl net.ipv4.ip_forward
# Output: net.ipv4.ip_forward = 1
```

---

### 3.8 Install Dependencies Sistem (di Semua Node)

Install package yang dibutuhkan:

```bash
sudo yum install -y curl wget vim tar socat conntrack ipset
```

**Penjelasan package:**
- `socat` — dibutuhkan oleh kubelet untuk port-forwarding
- `conntrack` — dibutuhkan untuk connection tracking di networking
- `ipset` — dibutuhkan oleh kube-proxy
- `tar` — untuk mengekstrak binary tar.gz

---

## 4. Install containerd v2.2.1 via Binary (di Semua Node)

Karena CentOS 7 / RHEL 7 sudah tidak mendukung repository Docker/containerd resmi, kita menginstall containerd langsung dari binary tar.gz yang dirilis di GitHub.

### 4.1 Download Binary containerd (di Semua Node)

```bash
# Buat direktori kerja
mkdir -p /tmp/containerd-install
cd /tmp/containerd-install

# Download containerd v2.2.1 binary
wget https://github.com/containerd/containerd/releases/download/v2.2.1/containerd-2.2.1-linux-amd64.tar.gz

# Download file checksum untuk verifikasi
wget https://github.com/containerd/containerd/releases/download/v2.2.1/containerd-2.2.1-linux-amd64.tar.gz.sha256sum
```

Verifikasi checksum (pastikan output menunjukkan `OK`):
```bash
sha256sum -c containerd-2.2.1-linux-amd64.tar.gz.sha256sum
```

---

### 4.2 Ekstrak dan Install containerd (di Semua Node)

```bash
# Ekstrak ke /usr/local
sudo tar -C /usr/local -xzvf containerd-2.2.1-linux-amd64.tar.gz
```

Setelah ekstrak, binary containerd akan berada di:
- `/usr/local/bin/containerd`
- `/usr/local/bin/containerd-shim`
- `/usr/local/bin/containerd-shim-runc-v1`
- `/usr/local/bin/containerd-shim-runc-v2`
- `/usr/local/bin/ctr`

---

### 4.3 Install runc (di Semua Node)

runc adalah low-level container runtime yang dibutuhkan oleh containerd.

```bash
cd /tmp/containerd-install

# Download runc binary (versi terbaru yang kompatibel)
wget https://github.com/opencontainers/runc/releases/download/v1.2.3/runc.amd64

# Install runc ke /usr/local/sbin
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
```

Verifikasi:
```bash
runc --version
```

---

### 4.4 Install CNI Plugins (di Semua Node)

CNI plugins dibutuhkan agar containerd bisa mengatur jaringan dasar container.

```bash
cd /tmp/containerd-install

# Download CNI plugins
wget https://github.com/containernetworking/plugins/releases/download/v1.6.1/cni-plugins-linux-amd64-v1.6.1.tgz

# Buat direktori CNI
sudo mkdir -p /opt/cni/bin

# Ekstrak CNI plugins
sudo tar -C /opt/cni/bin -xzvf cni-plugins-linux-amd64-v1.6.1.tgz
```

---

### 4.5 Buat Systemd Service untuk containerd (di Semua Node)

Download file service resmi dari GitHub containerd:

```bash
sudo wget -O /etc/systemd/system/containerd.service \
  https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
```

Jika tidak ada akses internet atau ingin membuat manual:

```bash
sudo tee /etc/systemd/system/containerd.service <<EOF
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=infinity
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
EOF
```

---

### 4.6 Generate Konfigurasi Default containerd (di Semua Node)

```bash
# Buat direktori konfigurasi
sudo mkdir -p /etc/containerd

# Generate konfigurasi default
sudo /usr/local/bin/containerd config default | sudo tee /etc/containerd/config.toml
```

---

### 4.7 Aktifkan SystemdCgroup Driver (di Semua Node)

Kubernetes memerlukan cgroup driver yang sama antara containerd dan kubelet. Kita gunakan `systemd`.

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
```

Verifikasi perubahan:
```bash
grep 'SystemdCgroup' /etc/containerd/config.toml
# Output: SystemdCgroup = true
```

---

### 4.8 Jalankan dan Enable containerd (di Semua Node)

```bash
sudo systemctl daemon-reload
sudo systemctl enable containerd
sudo systemctl start containerd
sudo systemctl status containerd
```

> ✅ Pastikan status menunjukkan **active (running)**. Tekan `q` untuk keluar dari tampilan status.

---

### 4.9 Install crictl (di Semua Node)

`crictl` adalah CLI tool untuk berinteraksi dengan containerd. Ini dibutuhkan oleh kubeadm untuk verifikasi.

```bash
cd /tmp/containerd-install

# Download crictl
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.32.0/crictl-v1.32.0-linux-amd64.tar.gz

# Ekstrak
sudo tar -C /usr/local/bin -xzvf crictl-v1.32.0-linux-amd64.tar.gz
```

Konfigurasi crictl agar mengarah ke containerd:

```bash
sudo tee /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF
```

Verifikasi:
```bash
sudo crictl info
```

---

## 5. Install Kubernetes v1.35.0 via Binary (di Semua Node)

Karena CentOS 7 / RHEL 7 sudah tidak ada di repository Kubernetes resmi, kita download binary langsung dari server resmi Kubernetes.

### 5.1 Download Binary Kubernetes (di Semua Node)

```bash
mkdir -p /tmp/k8s-install
cd /tmp/k8s-install

K8S_VERSION="v1.35.0"

# Download kubeadm
wget https://dl.k8s.io/release/${K8S_VERSION}/bin/linux/amd64/kubeadm

# Download kubectl
wget https://dl.k8s.io/release/${K8S_VERSION}/bin/linux/amd64/kubectl

# Download kubelet
wget https://dl.k8s.io/release/${K8S_VERSION}/bin/linux/amd64/kubelet
```

---

### 5.2 Verifikasi Checksum (di Semua Node)

```bash
cd /tmp/k8s-install

# Download file checksum
wget https://dl.k8s.io/release/${K8S_VERSION}/bin/linux/amd64/kubeadm.sha256
wget https://dl.k8s.io/release/${K8S_VERSION}/bin/linux/amd64/kubectl.sha256
wget https://dl.k8s.io/release/${K8S_VERSION}/bin/linux/amd64/kubelet.sha256

# Verifikasi (semua harus menunjukkan: OK)
echo "$(cat kubeadm.sha256) kubeadm" | sha256sum --check
echo "$(cat kubectl.sha256) kubectl"  | sha256sum --check
echo "$(cat kubelet.sha256) kubelet"  | sha256sum --check
```

---

### 5.3 Install Binary ke Sistem (di Semua Node)

```bash
cd /tmp/k8s-install

# Install ke /usr/local/bin
sudo install -o root -g root -m 0755 kubeadm /usr/local/bin/kubeadm
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
sudo install -o root -g root -m 0755 kubelet /usr/local/bin/kubelet
```

Verifikasi versi:
```bash
kubeadm version
kubectl version --client
kubelet --version
```

---

### 5.4 Buat Systemd Service untuk kubelet (di Semua Node)

Download kubelet service file dari GitHub Kubernetes:

```bash
# Download kubelet.service
sudo wget -O /etc/systemd/system/kubelet.service \
  https://raw.githubusercontent.com/kubernetes/release/v0.18.0/cmd/krel/templates/latest/kubelet/kubelet.service

# Edit path binary karena kita install di /usr/local/bin bukan /usr/bin
sudo sed -i 's|/usr/bin/kubelet|/usr/local/bin/kubelet|g' /etc/systemd/system/kubelet.service
```

Buat direktori drop-in untuk konfigurasi tambahan:

```bash
sudo mkdir -p /etc/systemd/system/kubelet.service.d
```

Download kubeadm drop-in config:

```bash
sudo wget -O /etc/systemd/system/kubelet.service.d/10-kubeadm.conf \
  https://raw.githubusercontent.com/kubernetes/release/v0.18.0/cmd/krel/templates/latest/kubeadm/10-kubeadm.conf

# Edit path binary
sudo sed -i 's|/usr/bin/kubelet|/usr/local/bin/kubelet|g' \
  /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```

---

### 5.5 Enable kubelet (di Semua Node)

```bash
sudo systemctl daemon-reload
sudo systemctl enable kubelet
```

> ✅ **Catatan:** kubelet akan start otomatis setelah `kubeadm init` atau `kubeadm join` berhasil. Saat ini kubelet akan restart-loop — ini **NORMAL** karena belum ada konfigurasi cluster.

---

### 5.6 Konfigurasi PATH (Opsional, di Semua Node)

Jika binary tidak ditemukan, tambahkan `/usr/local/bin` ke PATH:

```bash
echo 'export PATH=$PATH:/usr/local/bin:/usr/local/sbin' >> ~/.bashrc
source ~/.bashrc
```

---

## 6. Inisialisasi Cluster (Master Node Saja)

> ⚠️ **HANYA jalankan di Master Node!**

### 6.1 Pre-flight Check (Master Node)

Sebelum inisialisasi, jalankan pre-flight check untuk memastikan tidak ada masalah:

```bash
sudo kubeadm init phase preflight
```

Perbaiki semua error yang muncul sebelum melanjutkan.

---

### 6.2 Inisialisasi Cluster (Master Node)

Jalankan perintah berikut (sesuaikan IP master Anda):

```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --control-plane-endpoint=192.168.2.104 \
  --cri-socket=unix:///run/containerd/containerd.sock \
  --kubernetes-version=v1.35.0
```

**Penjelasan flag:**
- `--pod-network-cidr` — range IP untuk pod network (harus match dengan konfigurasi Calico nanti)
- `--control-plane-endpoint` — IP atau hostname master node
- `--cri-socket` — path ke socket containerd
- `--kubernetes-version` — versi Kubernetes yang digunakan

> 📝 **PENTING:** Simpan output yang berisi perintah `kubeadm join ...` karena akan digunakan di worker node nanti!

Contoh output yang harus disimpan:
```
kubeadm join 192.168.2.104:6443 --token abc123.xyz789 \
    --discovery-token-ca-cert-hash sha256:1234567890abcdef...
```

---

### 6.3 Konfigurasi kubectl (Master Node)

Agar kubectl bisa digunakan tanpa sudo:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verifikasi kubectl sudah bisa digunakan:

```bash
kubectl get nodes
```

Anda akan melihat master node dengan status `NotReady` — ini **normal** karena networking (Calico) belum terinstall.

---

## 7. Install Calico Tigera (Master Node Saja)

Calico adalah plugin networking yang membuat pod bisa berkomunikasi antar node.

### 7.1 Install Tigera Operator (Master Node)

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
```

Tunggu operator pod berjalan:
```bash
kubectl get pods -n tigera-operator --watch
# Tunggu sampai STATUS menjadi Running, lalu tekan Ctrl+C
```

---

### 7.2 Download Custom Resources Calico (Master Node)

```bash
wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml
```

---

### 7.3 Edit Pod CIDR di custom-resources.yaml (Master Node)

```bash
vim custom-resources.yaml
```

Cari bagian `cidr: 192.168.0.0/16` dan **UBAH** menjadi `10.244.0.0/16` agar sesuai dengan `--pod-network-cidr` yang digunakan saat `kubeadm init`.

Atau gunakan sed langsung:
```bash
sed -i 's|192.168.0.0/16|10.244.0.0/16|g' custom-resources.yaml
```

Hasil akhir bagian yang berubah:

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

---

### 7.4 Apply Konfigurasi Calico (Master Node)

```bash
kubectl create -f custom-resources.yaml
```

---

### 7.5 Verifikasi Pods Calico (Master Node)

Tunggu beberapa saat, lalu cek status pods:

```bash
# Pantau pods calico-system
kubectl get pods -n calico-system --watch
```

Tunggu sampai semua pods menunjukkan status `Running`. Ini bisa memakan waktu **2-5 menit**.

Tekan `Ctrl+C` setelah semua running, lalu cek nodes:

```bash
kubectl get nodes
```

Master node seharusnya sudah berstatus `Ready`!

---

## 8. Join Worker Node ke Master (Worker Node Saja)

> ⚠️ **Pastikan Worker Node sudah menyelesaikan langkah 3, 4, dan 5 terlebih dahulu!**

### 8.1 Join Worker ke Cluster (Di Setiap Worker Node)

Jalankan perintah `kubeadm join` yang Anda simpan dari langkah 6.2:

```bash
sudo kubeadm join 192.168.2.104:6443 --token abc123.xyz789 \
    --discovery-token-ca-cert-hash sha256:1234567890abcdef... \
    --cri-socket=unix:///run/containerd/containerd.sock
```

> 📝 **Jika lupa token**, jalankan perintah ini di **master node**:
> ```bash
> kubeadm token create --print-join-command
> ```

---

### 8.2 Troubleshooting: Error CRI

Jika muncul error:
```
[ERROR CRI]: could not connect to the container runtime...
unknown service runtime.v1.RuntimeService
```

**Solusi — cek dan perbaiki konfigurasi containerd:**

```bash
sudo vim /etc/containerd/config.toml
```

Cari baris:
```toml
disabled_plugins = ["cri"]
```

Ubah menjadi:
```toml
disabled_plugins = []
```

Restart containerd:
```bash
sudo systemctl restart containerd
```

Ulangi perintah `kubeadm join`.

---

### 8.3 Troubleshooting: Error socat/conntrack Not Found

Jika muncul error terkait `socat` atau `conntrack`:

```bash
sudo yum install -y socat conntrack ipset
```

Kemudian ulangi `kubeadm join`.

---

### 8.4 Troubleshooting: Timeout saat Download Images

Jika kubeadm timeout saat pull image, pastikan internet aktif dan coba:

```bash
# Test koneksi ke registry
curl -I https://registry.k8s.io
```

Jika menggunakan proxy, set environment variable:
```bash
export HTTP_PROXY=http://proxy-server:port
export HTTPS_PROXY=http://proxy-server:port
export NO_PROXY=localhost,127.0.0.1,192.168.2.0/24
```

---

## 9. Verifikasi Cluster (Di Master Node)

Cek semua nodes:

```bash
kubectl get nodes -o wide
```

Output yang diharapkan:

```
NAME          STATUS   ROLES           AGE   VERSION   INTERNAL-IP      OS-IMAGE
k8s-master    Ready    control-plane   10m   v1.35.0   192.168.2.104    CentOS Linux 7
k8s-worker1   Ready    <none>          5m    v1.35.0   192.168.2.105    CentOS Linux 7
k8s-worker2   Ready    <none>          5m    v1.35.0   192.168.2.106    CentOS Linux 7
```

> ✅ Semua node harus menunjukkan status `Ready`!

Cek semua pods di cluster:

```bash
kubectl get pods -A
```

Semua pods harus berstatus `Running`.

---

## 10. Hasil Akhir

Selamat! Anda sekarang memiliki:

- ✔ Kubernetes v1.35.0 yang berfungsi penuh
- ✔ 1 Control Plane (Master Node)
- ✔ 2 Worker Nodes
- ✔ containerd v2.2.1 sebagai Container Runtime (binary installation)
- ✔ Calico Tigera untuk Networking

Cluster siap digunakan untuk deploy aplikasi! 🚀

---

## 11. Install Metrics Server (Master Node Saja)

Metrics Server mengumpulkan data penggunaan CPU dan Memory dari setiap node dan pod. Berguna untuk `kubectl top` dan Horizontal Pod Autoscaler (HPA).

### 11.1 Install Metrics Server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

---

### 11.2 Konfigurasi Flag TLS (Untuk Self-Hosted Cluster)

Karena cluster self-hosted, tambahkan flag `--kubelet-insecure-tls`:

```bash
kubectl patch deployment metrics-server -n kube-system \
  --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'
```

---

### 11.3 Verifikasi dan Test Metrics Server

```bash
# Cek status pod
kubectl get pods -n kube-system | grep metrics

# Tunggu sampai Running, lalu test
kubectl top nodes
kubectl top pods -A
```

Output contoh `kubectl top nodes`:
```
NAME          CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
k8s-master    250m         12%    1500Mi          37%
k8s-worker1   150m         7%     800Mi           20%
k8s-worker2   180m         9%     900Mi           22%
```

---

## 12. Test Deployment Sederhana (Opsional)

Untuk memastikan cluster benar-benar berfungsi:

```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get pods
kubectl get services
```

Akses nginx melalui browser: `http://<IP-worker-node>:<NodePort>`

Hapus test deployment:
```bash
kubectl delete deployment nginx
kubectl delete service nginx
```

---

## 13. Struktur Direktori Instalasi Binary

Karena menggunakan binary manual, berikut lokasi file-file penting:

### 13.1 Komponen Utama Kubernetes

| File/Direktori | Keterangan |
|---|---|
| `/usr/local/bin/kubeadm` | Binary kubeadm |
| `/usr/local/bin/kubectl` | Binary kubectl |
| `/usr/local/bin/kubelet` | Binary kubelet |
| `/etc/kubernetes/` | Konfigurasi cluster (admin.conf, manifests) |
| `/etc/kubernetes/manifests/` | Static pods (etcd, api-server, dll) |
| `/etc/systemd/system/kubelet.service` | Systemd unit kubelet |
| `/etc/systemd/system/kubelet.service.d/` | Drop-in configs kubelet |
| `/var/lib/kubelet/` | Data runtime kubelet |
| `/var/lib/etcd/` | Database cluster (hanya master) |
| `/var/log/pods/` | Log pod |
| `/var/log/containers/` | Log container |

### 13.2 Container Runtime (containerd)

| File/Direktori | Keterangan |
|---|---|
| `/usr/local/bin/containerd` | Binary utama containerd |
| `/usr/local/bin/containerd-shim-runc-v2` | Shim runtime |
| `/usr/local/sbin/runc` | Low-level container runtime |
| `/usr/local/bin/ctr` | CLI bawaan containerd |
| `/usr/local/bin/crictl` | CLI untuk CRI (Kubernetes-compatible) |
| `/etc/containerd/config.toml` | Konfigurasi containerd |
| `/etc/crictl.yaml` | Konfigurasi crictl |
| `/etc/systemd/system/containerd.service` | Systemd unit containerd |
| `/var/lib/containerd/` | Images, snapshots, data container |
| `/run/containerd/containerd.sock` | Unix socket containerd |

### 13.3 Networking (CNI / Calico)

| File/Direktori | Keterangan |
|---|---|
| `/opt/cni/bin/` | Binary CNI plugins |
| `/etc/cni/net.d/` | Konfigurasi jaringan (10-calico.conflist) |

### 13.4 Tabel Ringkasan

| Komponen | Jenis Data | Path |
|---|---|---|
| Kubernetes Binaries | Eksekusi | `/usr/local/bin/` |
| containerd Binary | Eksekusi | `/usr/local/bin/` |
| runc Binary | Eksekusi | `/usr/local/sbin/` |
| Kubernetes Config | Konfigurasi Cluster | `/etc/kubernetes/` |
| Kubelet Data | Pods & Volumes | `/var/lib/kubelet/` |
| Containerd Config | Konfigurasi Runtime | `/etc/containerd/` |
| Containerd Data | Image & Snapshots | `/var/lib/containerd/` |
| Containerd Socket | Runtime State | `/run/containerd/` |
| etcd | Database Cluster | `/var/lib/etcd/` |
| Networking Config | CNI Config | `/etc/cni/net.d/` |
| Networking Binary | CNI Plugins | `/opt/cni/bin/` |
| Log | Pod & Container | `/var/log/pods/` & `/var/log/containers/` |

> 📍 **Note:** Gunakan `sudo crictl images` untuk melihat image di node. Gunakan `sudo crictl rmi --prune` untuk menghapus image yang tidak terpakai.

---

## 14. Perintah Berguna

### Perintah Dasar kubectl

```bash
# Lihat semua nodes
kubectl get nodes -o wide

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

# Sort berdasarkan CPU atau Memory
kubectl top pods -A --sort-by=cpu
kubectl top pods -A --sort-by=memory
```

### Perintah crictl (Container Runtime Level)

> ⚠️ Semua perintah `crictl` memerlukan akses root (`sudo`).

```bash
# Lihat container yang sedang running
sudo crictl ps

# Lihat semua container termasuk yang stopped
sudo crictl ps -a

# Lihat semua image di node
sudo crictl images

# Pull image
sudo crictl pull nginx:latest

# Hapus image tidak terpakai
sudo crictl rmi --prune

# Lihat log container
sudo crictl logs <container-id>

# Masuk ke dalam container
sudo crictl exec -it <container-id> /bin/sh

# Lihat semua pod sandbox
sudo crictl pods

# Cek status runtime
sudo crictl info

# Lihat statistik resource container
sudo crictl stats
```

### Perintah Troubleshooting

```bash
# Lihat events cluster (sorted by time)
kubectl get events -A --sort-by='.lastTimestamp'

# Lihat status semua resources
kubectl get all -A

# Cek informasi cluster
kubectl cluster-info

# Cek versi Kubernetes
kubectl version

# Describe pod untuk melihat detail error
kubectl describe pod <nama-pod> -n <namespace>

# Cek status kubelet
sudo systemctl status kubelet

# Lihat log kubelet
sudo journalctl -u kubelet -f

# Cek status containerd
sudo systemctl status containerd

# Lihat log containerd
sudo journalctl -u containerd -f
```

---

## 15. Referensi

- [Kubernetes Binary Releases](https://dl.k8s.io/release/)
- [containerd GitHub Releases](https://github.com/containerd/containerd/releases)
- [runc GitHub Releases](https://github.com/opencontainers/runc/releases)
- [CNI Plugins Releases](https://github.com/containernetworking/plugins/releases)
- [crictl (cri-tools) Releases](https://github.com/kubernetes-sigs/cri-tools/releases)
- [Dokumentasi Kubernetes - Install kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm)
- [Dokumentasi Calico Tigera](https://projectcalico.docs.tigera.io)
- [Dokumentasi containerd](https://containerd.io)
- [Dokumentasi Metrics Server](https://github.com/kubernetes-sigs/metrics-server)

---

**💡 Tips:**
- Simpan kubeconfig (`~/.kube/config`) sebagai backup
- Gunakan `kubectl get events -A --sort-by='.lastTimestamp'` untuk troubleshooting di level Kubernetes
- Gunakan `sudo journalctl -u kubelet -f` untuk melihat log kubelet secara real-time
- Gunakan `sudo crictl ps` untuk troubleshooting di level container runtime
- Setelah reboot, verifikasi semua service running: `systemctl status containerd kubelet`
- Gunakan `kubectl explain <resource>` untuk melihat dokumentasi resource
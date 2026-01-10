# 📊 Instalasi Kubernetes Dashboard dengan NodePort

Dokumentasi ini menjelaskan cara menginstall Kubernetes Dashboard menggunakan Helm dan mengaksesnya melalui NodePort tanpa perlu port-forwarding atau tunneling.

---

## ✅ Prasyarat

Sebelum memulai, pastikan:
- Cluster Kubernetes sudah berjalan dan semua node `Ready`
- Metrics Server sudah terinstall (opsional, tapi direkomendasikan)
- Anda memiliki akses `kubectl` ke cluster

Cek status cluster:
```bash
kubectl get nodes
kubectl get pods -A
```

---

## 🛠 1. Install Helm

Helm adalah package manager untuk Kubernetes yang akan kita gunakan untuk menginstall Dashboard.

### 1.1 Download dan Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### 1.2 Verifikasi Instalasi Helm

```bash
helm version
```

Output yang diharapkan:
```
version.BuildInfo{Version:"v3.x.x", ...}
```

---

## 📊 2. Install Kubernetes Dashboard

### 2.1 Tambahkan Repository Helm Dashboard

```bash
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
```

### 2.2 Update Repository Helm

```bash
helm repo update
```

### 2.3 Install Dashboard dengan Helm

```bash
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard \
  --create-namespace \
  --namespace kubernetes-dashboard
```

**Penjelasan:**
- `upgrade --install` - install baru atau upgrade jika sudah ada
- `--create-namespace` - buat namespace otomatis jika belum ada
- `--namespace kubernetes-dashboard` - install di namespace bernama kubernetes-dashboard

### 2.4 Verifikasi Instalasi

Tunggu beberapa saat, lalu cek pods:

```bash
kubectl get pods -n kubernetes-dashboard
```

Output yang diharapkan:
```
NAME                                                  READY   STATUS    RESTARTS   AGE
kubernetes-dashboard-api-xxxxxxxxxx-xxxxx             1/1     Running   0          2m
kubernetes-dashboard-auth-xxxxxxxxxx-xxxxx            1/1     Running   0          2m
kubernetes-dashboard-kong-xxxxxxxxxx-xxxxx            1/1     Running   0          2m
kubernetes-dashboard-metrics-scraper-xxxxxxxxxx-xxxxx 1/1     Running   0          2m
kubernetes-dashboard-web-xxxxxxxxxx-xxxxx             1/1     Running   0          2m
```

✅ Pastikan semua pods berstatus `Running`!

---

## 🌐 3. Konfigurasi NodePort

Secara default, Dashboard menggunakan ClusterIP yang hanya bisa diakses dari dalam cluster. Kita akan mengubahnya menjadi NodePort agar bisa diakses dari luar cluster.

### 3.1 Edit Service Kong Proxy

Kong proxy adalah gateway yang menangani akses ke Dashboard. Kita akan mengubah service-nya menjadi NodePort.

```bash
kubectl edit service kubernetes-dashboard-kong-proxy -n kubernetes-dashboard
```

### 3.2 Ubah Type Service

Cari bagian `spec` → `type`. Akan terlihat seperti ini:

```yaml
spec:
  type: ClusterIP
  ports:
  - name: https
    port: 443
    protocol: TCP
    targetPort: 8443
```

Ubah menjadi:

```yaml
spec:
  type: NodePort
  ports:
  - name: https
    port: 443
    protocol: TCP
    targetPort: 8443
    nodePort: 30443      # <-- TAMBAHKAN BARIS INI
```

**⚠️ PENTING:**
- Gunakan **SPASI untuk indentasi**, BUKAN TAB
- NodePort harus dalam range 30000-32767
- Disini menggunakan 30443 untuk HTTPS dashboard

**Simpan dengan menekan `Esc`, ketik `:wq`, lalu `Enter`**

### 3.3 Verifikasi Perubahan Service

```bash
kubectl get svc -n kubernetes-dashboard
```

Output yang diharapkan:
```
NAME                                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
kubernetes-dashboard-api               ClusterIP   10.x.x.x         <none>        8000/TCP        5m
kubernetes-dashboard-auth              ClusterIP   10.x.x.x         <none>        8000/TCP        5m
kubernetes-dashboard-kong-proxy        NodePort    10.x.x.x         <none>        443:30443/TCP   5m
kubernetes-dashboard-metrics-scraper   ClusterIP   10.x.x.x         <none>        8000/TCP        5m
kubernetes-dashboard-web               ClusterIP   10.x.x.x         <none>        8000/TCP        5m
```

✅ Perhatikan `kubernetes-dashboard-kong-proxy` sudah berubah menjadi `NodePort` dengan port `443:30443/TCP`

---

## 👤 4. Membuat User Admin

Dashboard memerlukan token untuk login. Kita akan membuat ServiceAccount dengan role admin.

### 4.1 Buat ServiceAccount

```bash
kubectl create serviceaccount dashboard-admin -n kubernetes-dashboard
```

### 4.2 Buat ClusterRoleBinding

Berikan akses cluster-admin ke ServiceAccount:

```bash
kubectl create clusterrolebinding dashboard-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=kubernetes-dashboard:dashboard-admin
```

**⚠️ Perhatian Keamanan:**
- Role `cluster-admin` memberikan akses penuh ke seluruh cluster
- Untuk production, sebaiknya gunakan role yang lebih terbatas
- Jangan share token dengan sembarang orang

### 4.3 Generate Token untuk Login

Untuk Kubernetes versi 1.24+, gunakan perintah ini:

```bash
kubectl -n kubernetes-dashboard create token dashboard-admin
```

Output akan berupa token panjang seperti ini:
```
eyJhbGciOiJSUzI1NiIsImtpZCI6IjRhN... (sangat panjang)
```

**📝 PENTING: Simpan token ini!** Anda akan membutuhkannya untuk login ke Dashboard.

#### Cara Alternatif: Buat Token Permanen (Opsional)

Jika Anda ingin token yang tidak expire, buat Secret secara manual:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: dashboard-admin-secret
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/service-account.name: dashboard-admin
type: kubernetes.io/service-account-token
EOF
```

Lalu ambil token dari secret:

```bash
kubectl get secret dashboard-admin-secret -n kubernetes-dashboard -o jsonpath='{.data.token}' | base64 --decode
```

---

## 🌍 5. Akses Dashboard

### 5.1 Cari IP Node

Dapatkan IP salah satu node (Master atau Worker):

```bash
kubectl get nodes -o wide
```

Output contoh:
```
NAME          STATUS   ROLES           AGE   VERSION   INTERNAL-IP     EXTERNAL-IP
k8s-master    Ready    control-plane   1h    v1.35.0   192.168.2.104   <none>
k8s-worker1   Ready    <none>          1h    v1.35.0   192.168.2.105   <none>
k8s-worker2   Ready    <none>          1h    v1.35.0   192.168.2.106   <none>
```

Anda bisa menggunakan IP dari node mana saja.

### 5.2 Akses Dashboard via Browser

Buka browser dan akses:

```
https://<IP-NODE>:30443
```

Contoh:
```
https://192.168.2.104:30443
```

atau

```
https://192.168.2.105:30443
```

### 5.3 Bypass Warning SSL

Karena menggunakan self-signed certificate, browser akan menampilkan warning keamanan.

**Di Chrome/Edge:**
1. Klik "Advanced"
2. Klik "Proceed to ... (unsafe)"

**Di Firefox:**
1. Klik "Advanced"
2. Klik "Accept the Risk and Continue"

### 5.4 Login ke Dashboard

1. Pilih opsi **"Token"**
2. Paste token yang Anda simpan dari langkah 4.3
3. Klik **"Sign in"**

🎉 **Selamat! Anda sekarang sudah bisa mengakses Kubernetes Dashboard!**

---

## 🎨 6. Fitur Dashboard

Setelah login, Anda bisa:

✅ Melihat semua workload (Deployments, Pods, Services, dll)  
✅ Monitor resource usage (jika Metrics Server terinstall)  
✅ Melihat logs dari pods  
✅ Exec ke dalam container  
✅ Edit resource secara langsung  
✅ Deploy aplikasi baru via UI  
✅ Melihat events dan troubleshooting  

---

## 🔧 7. Troubleshooting

### 7.1 Dashboard Tidak Bisa Diakses

**Cek apakah semua pods running:**
```bash
kubectl get pods -n kubernetes-dashboard
```

**Cek service NodePort:**
```bash
kubectl get svc kubernetes-dashboard-kong-proxy -n kubernetes-dashboard
```

**Cek apakah port 30443 terbuka di firewall:**
```bash
sudo ufw status
sudo ufw allow 30443/tcp
```

### 7.2 Token Tidak Diterima

**Generate token baru:**
```bash
kubectl -n kubernetes-dashboard create token dashboard-admin
```

**Cek apakah ServiceAccount ada:**
```bash
kubectl get sa dashboard-admin -n kubernetes-dashboard
```

### 7.3 Metrics Tidak Muncul di Dashboard

Dashboard memerlukan Metrics Server untuk menampilkan grafik resource usage.

**Install Metrics Server jika belum:**
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

**Patch Metrics Server untuk self-hosted cluster:**
```bash
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'
```

### 7.4 Certificate Error Terus Muncul

Ini normal untuk self-signed certificate. Untuk production, sebaiknya gunakan certificate yang valid dari Let's Encrypt atau CA lainnya.

### 7.5 Restart Dashboard

Jika ada masalah, coba restart semua pods Dashboard:

```bash
kubectl rollout restart deployment -n kubernetes-dashboard
```

---

## 🔒 8. Keamanan (Production)

Untuk environment production, pertimbangkan:

### 8.1 Gunakan Role yang Lebih Terbatas

Jangan gunakan `cluster-admin` untuk production. Contoh role terbatas:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: dashboard-viewonly
rules:
- apiGroups: [""]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
```

### 8.2 Gunakan Valid SSL Certificate

Install cert-manager dan gunakan Let's Encrypt:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
```

### 8.3 Gunakan Ingress + NGINX

Untuk akses yang lebih aman, gunakan Ingress Controller dengan domain yang proper.

### 8.4 Enable Authentication External

Integrasikan dengan LDAP, OAuth2, atau OIDC untuk authentication yang lebih robust.

---

## 📚 9. Perintah Berguna

```bash
# Lihat semua resource Dashboard
kubectl get all -n kubernetes-dashboard

# Lihat logs Dashboard
kubectl logs -n kubernetes-dashboard deployment/kubernetes-dashboard-web

# Lihat events Dashboard
kubectl get events -n kubernetes-dashboard --sort-by='.lastTimestamp'

# Generate token baru
kubectl -n kubernetes-dashboard create token dashboard-admin

# Hapus token lama (jika ada)
kubectl delete secret dashboard-admin-secret -n kubernetes-dashboard

# Restart semua pods Dashboard
kubectl rollout restart deployment -n kubernetes-dashboard

# Uninstall Dashboard
helm uninstall kubernetes-dashboard -n kubernetes-dashboard
```

---

## 📎 10. Referensi

- [Kubernetes Dashboard Official](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/)
- [Kubernetes Dashboard GitHub](https://github.com/kubernetes/dashboard)
- [Helm Documentation](https://helm.sh/docs/)

---

## 🎯 Kesimpulan

Anda sekarang memiliki:

✅ Kubernetes Dashboard yang terinstall dengan Helm  
✅ Akses melalui NodePort tanpa port-forwarding  
✅ User admin dengan token untuk login  
✅ Dashboard yang bisa diakses dari browser di `https://<IP-NODE>:30443`  

**Dashboard siap digunakan untuk monitoring dan management cluster Kubernetes! 🚀**

---

**💡 Tips:**
- Bookmark URL dashboard Anda
- Simpan token di password manager yang aman
- Gunakan dashboard untuk monitoring, tapi tetap pelajari kubectl untuk automation
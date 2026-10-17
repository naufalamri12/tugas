link vidio (https://drive.google.com/file/d/1JrNmzmtUsgH2X7i4ETTj01lOnAPHnoUq/view?usp=sharing)


## 1. Membuat Kubernetes Cluster

### 1.1. Persiapan Server

Siapkan tiga server atau VM dengan konektivitas jaringan antar-node.

Contoh pembagian node:

| Node | Peran | Contoh hostname |
|---|---|---|
| Node 1 | K3s Server / Control Plane | `k3s-master` |
| Node 2 | K3s Agent / Worker | `k3s-worker-1` |
| Node 3 | K3s Agent / Worker | `k3s-worker-2` |

K3s menggunakan istilah **server** untuk node control plane dan **agent** untuk node worker. Jika tugas mensyaratkan satu master dan dua worker, konfigurasi ini sudah sesuai.


### 2.2. Instalasi K3s pada Master

Jalankan pada node pertama:

```bash
curl -sfL https://get.k3s.io | sh -
```

Periksa status service:

```bash
sudo systemctl status k3s
```

Periksa node cluster:

```bash
sudo kubectl get nodes -o wide
```

Ambil token untuk menggabungkan worker:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

Simpan token tersebut dengan aman.

### 2.3. Menggabungkan Worker

Pada masing-masing worker, jalankan perintah berikut setelah mengganti placeholder:

```bash
curl -sfL https://get.k3s.io | \
  K3S_URL=https://MASTER_PRIVATE_IP:6443 \
  K3S_TOKEN=TOKEN_DARI_MASTER \
  sh -
```

Pastikan port API K3s `6443` dapat dijangkau oleh worker dari jaringan yang sesuai.

Periksa status agent:

```bash
sudo systemctl status k3s-agent
```

### 2.4. Validasi Cluster

Pada master:

```bash
sudo kubectl get nodes -o wide
```

Hasil yang diharapkan adalah tiga node dengan status `Ready`.

Contoh:

```text
NAME           STATUS   ROLES
k3s-master     Ready    control-plane,master
k3s-worker-1   Ready    <none>
k3s-worker-2   Ready    <none>
```

Nama dan label role dapat berbeda tergantung konfigurasi instalasi.

---

## 3. Install Helm, Disable Ingress Bawaan K3s, dan Install Ingress NGINX

### 3.1 Install Helm

Jalankan perintah berikut pada server K3s (control-plane):

```bash
sudo apt-get update
sudo apt-get install -y curl
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Verifikasi instalasi:

```bash
helm version
```

### 3.2 Nonaktifkan Traefik Bawaan K3s

Edit konfigurasi K3s pada server:

```bash
sudo nano /etc/rancher/k3s/config.yaml
```

Tambahkan konfigurasi berikut:

```yaml
disable:
  - traefik
```

Simpan file, kemudian restart K3s:

```bash
sudo systemctl restart k3s
```

Verifikasi status:

```bash
sudo systemctl status k3s
sudo kubectl get pods -A
```

### 3.3 Hapus Resource Traefik yang Masih Tersisa

Periksa resource Traefik:

```bash
sudo kubectl get helmcharts -A
sudo kubectl get pods -A | grep -i traefik
```
Jika masih ada resource Traefik yang tertinggal setelah dinonaktifkan, periksa terlebih dahulu resource tersebut sebelum menghapusnya. dan pastika Pastikan Traefik sudah tidak berjalan sebelum melanjutkan.

### 4.4 Tambahkan Repository Helm Ingress NGINX

Tambahkan repository resmi komunitas Ingress NGINX:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

### 4.5 Install Ingress NGINX

Install Ingress NGINX ke namespace khusus:

```bash
sudo kubectl create namespace ingress-nginx
```

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx
```

Periksa hasil instalasi:

```bash
sudo kubectl get pods -n ingress-nginx
sudo kubectl get svc -n ingress-nginx
```

### 4.6 Verifikasi Ingress Controller

Jalankan:

```bash
sudo kubectl get ingressclass
sudo kubectl get pods -A
```

## 5. Deploy Aplikasi ke Kubernetes

### 5.1. Membuat Namespace

Buat file `kubernetes/namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dumbways
```

Terapkan:

```bash
kubectl apply -f kubernetes/namespace.yaml
```

### 5.2. Deployment Frontend

Contoh `kubernetes/frontend/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: dumbways
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: DOCKERHUB_USERNAME/FRONTEND_IMAGE:TAG
          ports:
            - containerPort: 80
```

### 5.3. Service Frontend

Buat `kubernetes/frontend/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: dumbways
spec:
  selector:
    app: frontend
  ports:
    - name: http
      port: 80
      targetPort: 80
  type: ClusterIP
```

### 5.4. Deployment Backend

Contoh `kubernetes/backend/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: dumbways
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: DOCKERHUB_USERNAME/BACKEND_IMAGE:TAG
          ports:
            - containerPort: 5000
          env:
            - name: DB_HOST
              value: mysql
            - name: DB_PORT
              value: "3306"
            - name: DB_NAME
              value: dumbways
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: password
```

### 5.5. Service Backend

Buat `kubernetes/backend/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: dumbways
spec:
  selector:
    app: backend
  ports:
    - name: http
      port: 5000
      targetPort: 5000
  type: ClusterIP
```

Terapkan manifest aplikasi:

```bash
kubectl apply -f kubernetes/namespace.yaml
kubectl apply -f kubernetes/database/
kubectl apply -f kubernetes/backend/
kubectl apply -f kubernetes/frontend/
```

Periksa status deployment:

```bash
kubectl get pods -n dumbways
kubectl get deployments -n dumbways
kubectl get svc -n dumbways
```

## 6. Persistent Volume

### 6.1. Memeriksa Storage Class

```bash
kubectl get storageclass
kubectl get pv
kubectl get pvc -A
```

### 6.2. Membuat PersistentVolumeClaim

Contoh `kubernetes/database/pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data
  namespace: dumbways
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

Jika menggunakan storage class tertentu, tambahkan `storageClassName` yang sesuai.

Terapkan dan periksa status:

```bash
kubectl apply -f kubernetes/database/pvc.yaml
kubectl get pvc -n dumbways
```

## 7. Deploy MySQL StatefulSet dan Secret

### 7.1. Membuat Secret

Secret menyimpan username dan password database.

Buat Secret langsung di cluster:

```bash
kubectl create secret generic mysql-secret \
  --namespace dumbways \
  --from-literal=username=appuser \
  --from-literal=password='GANTI_DENGAN_PASSWORD_KUAT'
```

Periksa:

```bash
kubectl get secret -n dumbways
```

### 7.2. Membuat Service MySQL

Buat `kubernetes/database/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: dumbways
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
    - name: mysql
      port: 3306
      targetPort: 3306
```


### 8.3. Membuat StatefulSet

Buat `kubernetes/database/statefulset.yaml`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: dumbways
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - name: mysql
              containerPort: 3306
          env:
            - name: MYSQL_DATABASE
              value: dumbways
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: username
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: password
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: root-password
          volumeMounts:
            - name: mysql-storage
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-storage
          persistentVolumeClaim:
            claimName: mysql-data
```

Sebelum menerapkan StatefulSet, pastikan Secret juga memiliki key `root-password`. Buat dengan perintah:

```bash
kubectl create secret generic mysql-secret \
  --namespace dumbways \
  --from-literal=username=appuser \
  --from-literal=password='GANTI_PASSWORD_APP' \
  --from-literal=root-password='GANTI_PASSWORD_ROOT'
```

Jika Secret sudah ada, perbarui menggunakan prosedur yang aman. Hindari menuliskan password sebenarnya di manifest yang akan dimasukkan ke repository.

Terapkan StatefulSet dan Service:

```bash
kubectl apply -f kubernetes/database/service.yaml
kubectl apply -f kubernetes/database/statefulset.yaml
```

Periksa status:

```bash
kubectl get pods -n dumbways
kubectl get statefulset -n dumbways
kubectl get pvc -n dumbways
kubectl logs mysql-0 -n dumbways
```

Pastikan MySQL berjalan, PVC terpasang, dan backend dapat melakukan koneksi ke `mysql:3306`.

## 9. Instalasi Cert-Manager

cert-manager digunakan untuk mengelola sertifikat TLS di Kubernetes secara otomatis.

### 9.1. Instalasi Menggunakan Helm

Tambahkan repository Helm:

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

Instal cert-manager:

```bash
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

Periksa status:

```bash
kubectl get pods -n cert-manager
kubectl get crds | grep cert-manager
```

Pastikan seluruh Pod cert-manager berjalan dengan baik.

### 9.2. Membuat ClusterIssuer

Untuk wildcard certificate, gunakan metode DNS-01 challenge. HTTP-01 tidak dapat menerbitkan wildcard certificate.

Contoh `cert-manager/clusterissuer.yaml` untuk Let's Encrypt staging dengan solver DNS Cloudflare:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token-secret
  namespace: cert-manager
type: Opaque
stringData:
  api-token: REPLACE_WITH_CLOUDFLARE_API_TOKEN
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: YOUR_EMAIL@example.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-staging-account-key
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token-secret
              key: api-token
```
### 9.3. Membuat Wildcard Certificate

Contoh `cert-manager/certificate.yaml`:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: kubernetes-wildcard
  namespace: dumbways
spec:
  secretName: kubernetes-wildcard-tls
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
  dnsNames:
    - "*.kubernetes.studentdumbways.my.id"
```

Terapkan konfigurasi:

```bash
kubectl apply -f cert-manager/clusterissuer.yaml
kubectl apply -f cert-manager/certificate.yaml
```

Periksa proses penerbitan sertifikat:

```bash
kubectl get clusterissuer
kubectl get certificate -n dumbways
kubectl describe certificate kubernetes-wildcard -n dumbways
kubectl get challenges -A
kubectl get orders -A
```

**Penting:** Secret TLS harus berada pada namespace yang sama dengan resource Ingress yang menggunakannya. Pada contoh ini, keduanya berada di namespace `dumbways`.

---

## 10. Konfigurasi Ingress Frontend dan Backend

### 10.1. DNS Record

Buat DNS record yang mengarah ke alamat IP Ingress Controller.

| Hostname | Tujuan |
|---|---|
| `name.kubernetes.studentdumbways.my.id` | Frontend |
| `api.name.kubernetes.studentdumbways.my.id` | Backend |

Kedua hostname dapat mengarah ke IP ingress yang sama karena routing ditentukan oleh hostname.

### 10.2. Ingress Frontend

Buat `kubernetes/frontend/ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-ingress
  namespace: dumbways
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - name.kubernetes.studentdumbways.my.id
      secretName: kubernetes-wildcard-tls
  rules:
    - host: name.kubernetes.studentdumbways.my.id
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

### 10.3. Ingress Backend

Buat `kubernetes/backend/ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: backend-ingress
  namespace: dumbways
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.name.kubernetes.studentdumbways.my.id
      secretName: kubernetes-wildcard-tls
  rules:
    - host: api.name.kubernetes.studentdumbways.my.id
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 5000
```

Terapkan:

```bash
kubectl apply -f kubernetes/frontend/ingress.yaml
kubectl apply -f kubernetes/backend/ingress.yaml
```

Periksa:

```bash
kubectl get ingress -n dumbways
kubectl describe ingress frontend-ingress -n dumbways
kubectl describe ingress backend-ingress -n dumbways
```

### 10.4. Testing HTTPS

Setelah DNS dan sertifikat siap, uji akses:

```bash
curl -I https://name.kubernetes.studentdumbways.my.id
curl -I https://api.name.kubernetes.studentdumbways.my.id
```
**Repository:** `devops21-dumbways-nama-kalian`

**Task:** Week 4 — Kubernetes

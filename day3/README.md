link tugas vidio( https://drive.google.com/file/d/1nX5DfMAgtviXLR7ftKBEx1-duNrmfIPl/view?usp=sharing)
# DevOps Automation — Terraform, Ansible & Monitoring Server

# 1. Terraform

## 1.1. Resource yang Dibutuhkan

Untuk AWS, beberapa resource yang dapat digunakan adalah:

| Resource | Fungsi |
|---|---|
| `aws_vpc` | Membuat jaringan virtual |
| `aws_subnet` | Membuat subnet |
| `aws_internet_gateway` | Menghubungkan VPC ke internet |
| `aws_route_table` | Mengatur routing jaringan |
| `aws_route_table_association` | Menghubungkan route table dengan subnet |
| `aws_instance` | Membuat virtual machine |
| `aws_eip` | Mengalokasikan Elastic IP |
| `aws_security_group` | Mengatur firewall instance |
| `aws_ebs_volume` | Membuat block storage |
| `aws_volume_attachment` | Memasang block storage ke instance |
| `aws_key_pair` | Menghubungkan SSH public key ke instance |

Sesuaikan resource dengan provider yang digunakan karena nama dan implementasinya berbeda pada AWS, GCP, dan Azure.

## 1.2. Tahapan Pengerjaan

1. Membuat konfigurasi provider.
2. Menentukan region dan availability zone.
3. Membuat VPC, subnet, Internet Gateway, route table, dan route table association.
4. Memilih image Ubuntu 24.04 dan Debian 11 yang tersedia pada provider.
5. Membuat kedua virtual machine.
6. Mengalokasikan dan memasang static IP.
7. Membuat inbound dan outbound firewall rules.
8. Membuat dua block storage, misalnya 10 GB atau 15 GB sesuai kebutuhan.
9. Memasang satu block storage ke server Ubuntu dan satu ke server Debian.
10. Menjalankan Terraform.
11. Memeriksa output dan menguji SSH.

**Catatan:** Pada AWS, Elastic IP dapat tetap menggunakan alamat IP yang sama ketika instance dihentikan sementara selama alamat tersebut tetap dialokasikan dan tidak dilepaskan. Elastic IP yang tidak digunakan juga dapat menimbulkan biaya. Periksa ketentuan provider yang dipakai.

## 1.3. Menjalankan Terraform

Masuk ke direktori provider yang digunakan, misalnya:

```bash
cd Terraform/aws
```

Jalankan perintah:

```bash
terraform fmt -recursive
terraform init
terraform validate
terraform plan
terraform apply
```

Keterangan:

- `terraform fmt -recursive`: merapikan format file Terraform.
- `terraform init`: menginisialisasi Terraform dan provider.
- `terraform validate`: memeriksa konfigurasi.
- `terraform plan`: melihat resource yang akan dibuat.
- `terraform apply`: menerapkan konfigurasi dan membuat infrastructure.

Setelah selesai, periksa output Terraform untuk mengetahui alamat IP server.

## 1.4. Testing SSH

Contoh koneksi ke server Ubuntu:

```bash
ssh -i ~/.ssh/aws-key.pem ubuntu@PUBLIC_IP_UBUNTU
```

Contoh koneksi ke server Debian:

```bash
ssh -i ~/.ssh/aws-key.pem admin@PUBLIC_IP_DEBIAN
```

Username `admin` merupakan contoh yang umum untuk image Debian AWS. Gunakan username sesuai image yang dipilih.

Pastikan permission private key sudah benar:

```bash
chmod 400 ~/.ssh/aws-key.pem
```

Jika terjadi kegagalan, periksa security group, IP address, username, lokasi private key, dan status instance.

## 1.5. Menghapus Infrastructure

Jika ingin menghapus resource yang dibuat oleh Terraform, jalankan dari direktori konfigurasi yang sesuai:

```bash
terraform plan -destroy
terraform destroy
```

Periksa resource yang akan dihapus sebelum menyetujui proses destroy. Perintah tersebut dapat menghapus VM, storage, dan resource lain yang dikelola oleh konfigurasi Terraform.

---

# 2. Ansible

Ansible digunakan untuk mengotomatisasi konfigurasi server, membuat user, menginstal Docker, melakukan deployment frontend, dan mengatur monitoring.

## 2.1. Struktur Direktori

```text
Automation/
└── Ansible/
    ├── ansible.cfg
    ├── Inventory
    ├── group_vars/
    │   └── all.yml
    └── playbooks/
        ├── create-user.yml
        ├── install-docker.yml
        ├── deploy-frontend.yml
        ├── install-node-exporter.yml
        ├── setup-monitoring.yml
        ├── setup-reverse-proxy.yml
        └── setup-ssl.yml
```

Nama playbook dapat disesuaikan dengan kebutuhan pengerjaan.

## 2.2. Membuat Inventory

File `Inventory` berisi alamat IP server yang akan dikelola oleh Ansible.

Contoh:

```ini
[gateway]
gateway-server ansible_host=GATEWAY_PUBLIC_IP

[appserver]
ubuntu-server ansible_host=APP_PRIVATE_IP

[database]
debian-server ansible_host=DB_PRIVATE_IP

[monitoring]
monitoring-server ansible_host=MONITORING_PUBLIC_IP

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

Ganti placeholder dengan IP server yang sebenarnya.

Jika username Debian berbeda, atur username khusus untuk host atau group tersebut. Jika server hanya menggunakan private IP, pastikan koneksi tersedia melalui jaringan internal atau gateway.

## 2.3. Membuat ansible.cfg

File `ansible.cfg` digunakan untuk menentukan konfigurasi dasar Ansible.

```ini
[defaults]
inventory = Inventory
host_key_checking = True
timeout = 30
interpreter_python = auto_silent

[privilege_escalation]
become = True
become_method = sudo
```

Jika private key disimpan di lokasi lain, sesuaikan konfigurasi dengan lokasi file tersebut.

## 2.4. Menyiapkan SSH Key

Agar Ansible dapat mengakses server, public key harus terpasang pada server tujuan di file:

```text
~/.ssh/authorized_keys
```

Pastikan permission SSH sudah benar dan Ansible menggunakan private key yang sesuai.

Pengujian koneksi:

```bash
ansible all -m ping
```

Pengujian hostname:

```bash
ansible all -m command -a "hostname"
```
## 2.5. Membuat User Baru

Ansible dapat digunakan untuk:

- Membuat user baru.
- Memberikan akses `sudo` sesuai kebutuhan.
- Memasang SSH public key.
- Mengatur password akun.
- Mengatur autentikasi SSH sesuai kebijakan keamanan.

Untuk autentikasi menggunakan SSH key dan password, konfigurasi server harus sesuai. Pengaturan `PasswordAuthentication yes` pada `/etc/ssh/sshd_config` hanya diperlukan jika login menggunakan password memang diizinkan.


## 2.6. Instalasi Docker

Buat playbook `install-docker.yml` untuk menginstal Docker pada server yang diperlukan.

Contoh perintah:

```bash
ansible-playbook playbooks/install-docker.yml
```

Pastikan Docker berjalan pada server target:

```bash
docker --version
docker ps
```

## 2.7. Deploy Frontend

Playbook `deploy-frontend.yml` digunakan untuk mengambil source code frontend, membangun Docker image, dan menjalankan container aplikasi.

Jalankan:

```bash
ansible appserver -m ping
ansible-playbook playbooks/deploy-frontend.yml
```

Sesuaikan playbook dengan aplikasi frontend yang digunakan sebelumnya, termasuk:

- Repository Git.
- Branch yang akan digunakan.
- Direktori deployment.
- Konfigurasi environment.
- Dockerfile dan proses build image.
- Port aplikasi.
- Perintah untuk menjalankan container.

Setelah deployment selesai, periksa container pada app-server:

```bash
docker ps
```

## 2.8. Instalasi Monitoring, Reverse Proxy, dan SSL

Ansible dapat digunakan untuk mengotomatisasi instalasi dan konfigurasi:

- Node Exporter pada seluruh server yang ingin dipantau.
- Prometheus pada monitoring server.
- Grafana untuk dashboard dan alerting.
- cAdvisor untuk monitoring container.
- Nginx sebagai reverse proxy.
- Certbot atau metode lain untuk membuat SSL certificate.

Dokumentasi resmi dan contoh konfigurasi dapat digunakan sebagai referensi. Sesuaikan setiap playbook dengan kebutuhan server dan versi software.

---

# 3. Monitoring Server

Monitoring digunakan untuk memantau seluruh server yang dibuat menggunakan Terraform serta server yang dimiliki di Biznet.

## 3.1. Komponen Monitoring

| Komponen | Fungsi |
|---|---|
| Node Exporter | Mengumpulkan metrik sistem operasi |
| Prometheus | Mengambil dan menyimpan metrik |
| Grafana | Menampilkan dashboard dan mengelola alert |
| cAdvisor | Mengumpulkan metrik container |
| Nginx | Menyediakan reverse proxy |
| Certbot / Let's Encrypt | Membuat dan memperbarui SSL certificate |
| Discord Webhook | Mengirimkan notifikasi alert |

## 3.2. Memasang Node Exporter

Pasang Node Exporter pada semua server yang ingin dipantau, termasuk server cloud dan Biznet.

Node Exporter mengumpulkan informasi seperti:

- CPU usage.
- RAM usage.
- Disk usage.
- Disk read/write.
- Network receive/transmit.
- Uptime server.

Pastikan service Node Exporter berjalan dan port metrics `9100` dapat diakses oleh Prometheus.

## 3.3. Konfigurasi Prometheus

Prometheus harus dikonfigurasi agar mengambil data dari seluruh server target.

Contoh konfigurasi `prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - /etc/prometheus/rules/*.yml

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets:
          - localhost:9090

  - job_name: dumbflix
    static_configs:
      - targets:
          - APP_PRIVATE_IP:9100
          - DB_PRIVATE_IP:9100
          - MONITORING_PRIVATE_IP:9100
          - BIZNET_SERVER_PRIVATE_IP:9100

  - job_name: cadvisor
    static_configs:
      - targets:
          - APP_PRIVATE_IP:8080
```
## 3.4. Membuka Prometheus

Buka Prometheus melalui browser:

```text
http://MONITORING_SERVER_IP:9090
```

Kemudian:

1. Masuk ke menu **Status → Targets**.
2. Periksa seluruh target Node Exporter.
3. Pastikan target yang dibutuhkan berstatus `UP`.
4. Jika target berstatus `DOWN`, periksa alamat IP, port, firewall, dan konfigurasi `prometheus.yml`.

Kamu juga bisa menguji status target menggunakan query:

```promql
up
```

Jika hasilnya `1`, target berhasil di-scrape. Jika hasilnya `0`, Prometheus tidak berhasil mengambil data dari target tersebut.

Jika target tidak terbaca, periksa konfigurasi terlebih dahulu. Untuk mengatasi masalah resolusi nama atau hostname, IP address dapat digunakan langsung pada `targets`, asalkan alamat tersebut dapat dijangkau oleh Prometheus.

---

# 4. Prometheus

Prometheus digunakan untuk mengambil, menyimpan, dan meng-query metrik dari Node Exporter serta cAdvisor.

Metrik yang dikumpulkan kemudian digunakan sebagai data source pada Grafana dan sebagai dasar pembuatan alert.

Hal yang perlu diperiksa:

- Target Node Exporter sudah terdaftar.
- Target cAdvisor sudah terdaftar untuk monitoring container.
- Status target menunjukkan `UP`.
- Query PromQL menghasilkan data.
- File alert rule sudah terdaftar pada konfigurasi Prometheus.

---

# 5. Grafana Dashboard

Setelah data Prometheus masuk, lanjutkan ke Grafana untuk membuat dashboard monitoring.

## 5.1. Membuka Grafana

Akses Grafana menggunakan alamat IP dan port yang dikonfigurasi.

Contoh:

```text
http://MONITORING_SERVER_IP:3000
```

Jika deployment menggunakan port mapping `3001:3000`, gunakan:

```text
http://MONITORING_SERVER_IP:3001
```

Jika sudah menggunakan reverse proxy dan SSL, akses melalui domain Grafana.

Pada instalasi baru, kredensial awal dapat berupa `admin` untuk username dan `admin` untuk password. Ini bergantung pada metode deployment. Ikuti konfigurasi yang diterapkan dan segera ganti password awal jika masih digunakan.

## 5.2. Menghubungkan Data Source

1. Login ke Grafana.
2. Buka **Connections → Data sources**.
3. Tambahkan Prometheus.
4. Masukkan URL Prometheus yang dapat dijangkau dari Grafana.
5. Klik **Save & test**.
6. Pastikan koneksi berhasil.

## 5.3. Membuat Dashboard

Buka **Dashboards → New → New dashboard**, kemudian tambahkan panel sesuai kebutuhan.

Dashboard yang dibuat menampilkan:

- CPU Usage (%).
- RAM Usage (%).
- Disk Usage (%).
- Disk Read/Write.
- Network Receive/Transmit.
- Uptime Server.
- Container CPU Usage.
- Container Memory Usage.
- Status target Prometheus.
- Status dan riwayat alert.

---

# 6. PromQL Documentation

PromQL adalah bahasa query untuk mengambil dan mengolah metrik dari Prometheus.

Contoh berikut menggunakan IP `15.232.204.64` dan job `dumbflix` sebagai contoh konfigurasi. Ganti dengan IP dan label milikmu jika berbeda.

## 6.1. CPU Usage

Query CPU yang digunakan:

```promql
100 * (
  1 - avg(
    rate(node_cpu_seconds_total{
      instance="15.232.204.64:9100",
      mode="idle"
    }[5m])
  )
)
```

**Penjelasan:**

- `node_cpu_seconds_total` adalah counter waktu CPU.
- `mode="idle"` memilih waktu CPU dalam kondisi idle.
- `rate(...[5m])` menghitung perubahan rata-rata selama lima menit.
- `avg(...)` menghitung rata-rata idle CPU.
- `1 - idle` menghasilkan estimasi bagian CPU yang digunakan.
- Dikalikan `100` untuk mendapatkan persentase.

Hasil query berupa estimasi persentase penggunaan CPU.


## 6.2. Status Target Prometheus

```promql
up
```

Nilai `1` menunjukkan target berhasil di-scrape, sedangkan nilai `0` menunjukkan kegagalan scrape.

## 6.3. Uptime Server

```promql
time() - node_boot_time_seconds
```

Query ini menghasilkan perkiraan lama waktu sejak server terakhir boot dalam satuan detik.

## 6.4. Disk Read/Write

Contoh disk read rate:

```promql
rate(node_disk_read_bytes_total[5m])
```

Contoh disk write rate:

```promql
rate(node_disk_written_bytes_total[5m])
```

Hasil query berupa byte per detik. Untuk mendapatkan MiB/s, bagi hasil dengan `1024 * 1024`.

---

# 7. Alerting dan Discord

Alerting digunakan untuk memberikan notifikasi ketika penggunaan resource melewati batas yang ditentukan atau ketika container frontend tidak terdeteksi.

Ketentuan tugas:

- CPU Usage lebih dari 20%.
- RAM Usage lebih dari 75%.
- Monitoring specific container.
- Notifikasi dikirimkan ke channel Discord yang disediakan.

## 7.1. Membuat Alert Rule

Buat file:

```text
Monitoring/prometheus/rules/server-alerts.yml
```

Isi file:

```yaml
groups:
  - name: server-resource-alerts
    rules:
      - alert: HighCPUUsage
        expr: |
          100 * (
            1 - avg by (instance) (
              rate(node_cpu_seconds_total{mode="idle"}[5m])
            )
          ) > 20
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "CPU usage tinggi pada {{ $labels.instance }}"
          description: "CPU usage melebihi 20% selama 5 menit."

      - alert: HighMemoryUsage
        expr: |
          100 * (
            1 -
            node_memory_MemAvailable_bytes
            /
            node_memory_MemTotal_bytes
          ) > 75
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "RAM usage tinggi pada {{ $labels.instance }}"
          description: "RAM usage melebihi 75% selama 5 menit."
```


## 7.2. Integrasi Discord

Langkah konfigurasi:

1. Buka Discord dan pilih server yang digunakan untuk alerting.
2. Masuk ke pengaturan channel yang ditentukan.
3. Buka bagian **Integrations → Webhooks**.
4. Buat webhook dan salin URL-nya.
5. Buka Grafana.
6. Masuk ke **Alerting → Contact points**.
7. Tambahkan contact point Discord jika didukung oleh versi Grafana yang digunakan.
8. Masukkan webhook melalui konfigurasi secret yang aman.
9. Buat notification policy untuk mengarahkan alert ke contact point.
10. Uji notifikasi dan pastikan pesan masuk ke channel Discord.

Jangan menyimpan URL webhook dalam repository GitHub karena siapa pun yang memilikinya dapat menggunakannya untuk mengirim pesan ke channel tersebut.

## 7.3. Pengujian Alert

Pastikan alert yang berikut sudah dikonfigurasi:

- [ ] HighCPUUsage: CPU lebih dari 20% selama 5 menit.
- [ ] HighMemoryUsage: RAM lebih dari 75% selama 5 menit.
- [ ] FrontendContainerMissing: metrik container frontend tidak ditemukan.
- [ ] Discord menerima notifikasi dari alert yang aktif.

Untuk menguji, gunakan lingkungan lab yang aman dan pantau status alert di Grafana. Jangan melakukan stress test pada server produksi tanpa izin.

---

# 8. Reverse Proxy dan SSL

Reverse proxy digunakan untuk mengakses aplikasi frontend dan layanan monitoring melalui domain, sedangkan SSL digunakan untuk mengenkripsi koneksi HTTPS.

## 8.1. Konfigurasi DNS

Buka pengaturan DNS pada domain atau Cloudflare, kemudian buat record yang mengarah ke IP server reverse proxy.

Contoh domain monitoring:

| Domain | Layanan |
|---|---|
| `exporter-mentor.studentdumbways.my.id` | Node Exporter |
| `prom-mentor.studentdumbways.my.id` | Prometheus |
| `monitoring-mentor.studentdumbways.my.id` | Grafana |

## 8.2. Konfigurasi Nginx

Jika frontend atau Grafana mengalami masalah origin, host header, atau reverse proxy, periksa konfigurasi Nginx.

Contoh header yang dapat digunakan pada blok `location`:

```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

Contoh konfigurasi reverse proxy:

```nginx
server {
    listen 80;
    server_name monitoring-mentor.studentdumbways.my.id;

    location / {
        proxy_pass http://MONITORING_PRIVATE_IP:3000;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Setelah perubahan, periksa konfigurasi:

```bash
sudo nginx -t
```

## 8.3. SSL Certificate

SSL dapat dibuat menggunakan:

- Certbot / Let's Encrypt.
- Wildcard SSL.
- Cloudflare SSL sesuai kebutuhan tugas.

Jika menggunakan Certbot, ikuti metode challenge yang sesuai dengan konfigurasi DNS dan web server. Setelah sertifikat berhasil dibuat, pasang path sertifikat dan private key pada konfigurasi HTTPS Nginx.

Jika memakai Ansible, proses penerbitan dan pemasangan sertifikat dapat diotomatisasi melalui playbook. Untuk DNS challenge, pastikan kredensial DNS dikelola secara aman.

## 8.4. Menggabungkan Playbook SSL

Jika terdapat banyak domain, konfigurasi reverse proxy dan SSL dapat dibuat menggunakan loop Ansible agar satu playbook mengelola beberapa domain sekaligus.

Gunakan variabel atau daftar domain untuk menyimpan konfigurasi yang berbeda, misalnya domain, port tujuan, dan service yang akan di-proxy. Dengan demikian, konfigurasi tidak perlu ditulis berulang kali untuk setiap domain.

---

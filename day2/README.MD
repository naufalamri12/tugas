vidio link (https://drive.google.com/file/d/1kv1HwS9YHiy1EZSzUcQL-AtVUbPmKQ_C/view?usp=sharing)

# DevOps Week 2 — Wayshub Deployment

## 1. Persiapan Server

Lakukan pada setiap VM yang digunakan.

Update package:

```bash
sudo apt update
sudo apt upgrade -y
```

## 2. Membuat User Baru

Jalankan pada setiap server:

```bash
sudo adduser amri
sudo usermod -aG sudo amri
```

Periksa user:

```bash
id amri
```

Setelah Docker terinstal, jika user `amri` perlu menjalankan Docker tanpa `sudo`, tambahkan ke grup Docker:

```bash
sudo usermod -aG docker amri
```

Keluar dan masuk kembali agar perubahan grup berlaku.

> Keanggotaan grup Docker memberikan hak istimewa tinggi pada server. Gunakan hanya untuk user tepercaya.

## 3. Membuat Bash Script Instalasi Docker

Buat file script:
```bash
nano install-docker.sh
```

Masukkan perintah berikut:
```bash
#!/bin/bash

# Add Docker's official GPG key:
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update

# Install Docker Engine
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Verify that Docker is running
sudo systemctl status docker
sudo systemctl start docker
```

### Menjalankan Script

Berikan izin eksekusi:
```bash
chmod +x install-docker.sh
```

Jalankan script:
```bash
./install-docker.sh
```
Periksa status Docker:

```bash
sudo systemctl status docker
```
Periksa Docker Compose:

```bash
docker compose version
```


## 4. Deployment Database MySQL
### 4.1 Instalasi MySQL pada Database Server

Instal MySQL langsung pada VM Database:
```bash
sudo apt update
sudo apt install mysql-server -y
```
Periksa status:

```bash
sudo systemctl status mysql
```

Aktifkan MySQL:

```bash
sudo systemctl enable --now mysql
```

### 4.2 Masuk ke MySQL

```bash
sudo mysql
```

### 4.3 Membuat Database dan User MySQL

Buat database Wayshub:

```sql
CREATE DATABASE wayshub;
```

Buat user untuk backend:

```sql
CREATE USER 'amri'@'PRIVATE_IP_BACKEND'
IDENTIFIED BY 'GANTI_DENGAN_PASSWORD_KUAT';
```

Ganti `PRIVATE_IP_BACKEND` dengan private IP Backend Server yang sebenarnya.

Berikan akses ke database:

```sql
GRANT ALL PRIVILEGES ON wayshub.* TO 'amri'@'PRIVATE_IP_BACKEND';
```

Periksa database:

```sql
SHOW DATABASES;
SELECT user, host FROM mysql.user;
```

Keluar:

```sql
EXIT;
```

### 4.4 Konfigurasi MySQL agar Dapat Diakses Backend

Buka konfigurasi MySQL:

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Sesuaikan `bind-address` dengan private IP Database Server:

```ini
bind-address = PRIVATE_IP_DATABASE
```
Ganti `PRIVATE_IP_DATABASE` dengan private IP Database Server yang sebenarnya.
Restart MySQL:

```bash
sudo systemctl restart mysql
```

Periksa port MySQL:

```bash
sudo ss -lntp | grep 3306
```

### 4.5 Menguji Koneksi dari Backend Server

Pada Backend Server, instal MySQL client:

```bash
sudo apt update
sudo apt install mysql-client -y
```

Uji koneksi:

```bash
mysql -h PRIVATE_IP_DATABASE -P 3306 -u amri -p wayshub
```

Masukkan password ketika diminta.

Jika berhasil, koneksi Backend Server ke Database Server sudah berfungsi.

### 4.6 merubah oackage,json

Pada Backend Server, clone repository:

```bash
git clone https://github.com/dumbwaysdev/wayshub-backend.git
cd wayshub-backend
```

Periksa struktur repository:

```bash
cat package.json
find . -maxdepth 3 -type f | grep -Ei 'sequelize|migration|config'
```

Jika repository menggunakan `config/config.json`, periksa file tersebut:

```bash
nano config/config.json
```

Sesuaikan konfigurasi database dengan MySQL:

- Host: private IP Database Server.
- Port: `3306`.
- Database: `wayshub`.
- Username: `amri`.
- Password: password yang telah dibuat.

Instal dependency sesuai kebutuhan repository:

```bash
npm install
```

Jika repository memiliki `package-lock.json`, kamu dapat menggunakan `npm ci` untuk instalasi dependency yang konsisten.

Jika proyek menggunakan Sequelize CLI dan menyediakan file migration, jalankan:

```bash
npx sequelize-cli db:migrate
```

Setelah migration berhasil, masuk kembali ke MySQL dan periksa tabel:

```sql
SHOW TABLES;
```

**Catatan:** Pastikan repository memang menggunakan Sequelize dan menyediakan migration sebelum menjalankan perintah tersebut. Jika struktur repository berbeda, sesuaikan dengan konfigurasi aslinya.

## 5. Deployment Backend Wayshub

### 5.1 Konfigurasi Database

Periksa file konfigurasi database:

```bash
nano config/config.json
```

Sesuaikan konfigurasi dengan database MySQL yang telah dibuat.

Pastikan backend dapat terhubung ke Database Server sebelum melakukan Dockerisasi.

Periksa script yang tersedia:

```bash
npm run
```

Jalankan backend secara langsung menggunakan script yang sesuai dengan repository, misalnya:

```bash
npm start
```

Perintah tersebut hanya contoh. Gunakan perintah yang tersedia di `package.json` dan hentikan proses setelah pengujian jika diperlukan.

### 5.2 Membuat Dockerfile Multistage

Buat file `Dockerfile`:

```dockerfile
FROM node:14-alpine AS builder

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

FROM node:14-alpine AS production

WORKDIR /app

ENV NODE_ENV=production

COPY --from=builder /app /app

EXPOSE 5000

CMD ["npm", "start"]
```

Dockerfile ini merupakan contoh multistage build. Pastikan versi Node.js sesuai dengan kebutuhan repository Wayshub.

### 5.3 Membuat `.dockerignore`

Buat file `.dockerignore`:

```text
node_modules
.git
.env
npm-debug.log
```

File `.env` tidak dimasukkan ke image. Konfigurasi database diberikan ketika container dijalankan.

### 5.4 Build Image Backend

```bash
docker build -t mentor/wayshub-backend:production .
```

Ganti `mentor` dengan username Docker Hub yang digunakan.

### 5.5 Membuat `compose.yaml`

Buat file `compose.yaml`:

```yaml
services:
  backend:
    image: mentor/wayshub-backend:production
    container_name: wayshub-backend
    restart: unless-stopped
    env_file:
      - .env
    ports:
      - "5000:5000"
    networks:
      - team1

networks:
  team1:
    name: team1
    driver: bridge
```

Buat file `.env` berdasarkan nama variabel yang benar-benar digunakan aplikasi. Pastikan konfigurasi database menunjuk ke private IP Database Server.

Jalankan backend:

```bash
docker compose up -d
docker ps
docker logs wayshub-backend
```

## 6. Dockerize Wayshub Frontend

### 6.1 Clone Repository

Pada Frontend Server:

```bash
git clone https://github.com/dumbwaysdev/wayshub-frontend.git
cd wayshub-frontend
```

### 6.2 Konfigurasi URL Backend

Periksa file konfigurasi API:

```bash
find . -maxdepth 3 -type f | grep -Ei 'config|api'
```

Jika repository menggunakan `src/config/api.js`, edit:

```bash
nano src/config/api.js
```

Contoh konfigurasi:

```javascript
const API = "https://api.mentor.studentdumbways.my.id";
export default API;
```

Sesuaikan dengan struktur repository yang sebenarnya. Pastikan URL API benar sebelum melakukan build image.

### 6.3 Membuat Dockerfile Multistage

Contoh untuk aplikasi Create React App:

```dockerfile
FROM node:14-alpine AS builder

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

RUN npm run build

FROM nginx:alpine AS production

COPY --from=builder /app/build /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

Pastikan aplikasi menghasilkan file build pada folder `build`. Jika proyek menggunakan tool berbeda, sesuaikan dengan hasil build-nya.

### 6.4 Membuat `.dockerignore`

```text
node_modules
.git
.env
build
npm-debug.log
```

### 6.5 Build Image Frontend

```bash
docker build -t mentor/wayshub-frontend:production .
```

### 6.6 Deployment Frontend dengan Docker Compose

Buat file `compose.yaml`:

```yaml
services:
  frontend:
    image: mentor/wayshub-frontend:production
    container_name: wayshub-frontend
    restart: unless-stopped
    ports:
      - "3000:80"
    networks:
      - team1

networks:
  team1:
    name: team1
    driver: bridge
```

Jalankan:

```bash
docker compose up -d
docker ps
docker logs wayshub-frontend
```

## 7. Deployment Web Server dan Reverse Proxy

### 7.1 Membuat Direktori Nginx

Pada Web Server:

```bash
mkdir -p ~/webserver/conf.d
mkdir -p ~/webserver/certbot/conf
mkdir -p ~/webserver/certbot/www

cd ~/webserver
```

### 7.2 Konfigurasi Reverse Proxy

Buat file `conf.d/wayshub.conf`:

```nginx
server {
    listen 80;
    server_name mentor.studentdumbways.my.id;

    location / {
        proxy_pass http://PRIVATE_IP_FRONTEND:3000;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name api.mentor.studentdumbways.my.id;

    location / {
        proxy_pass http://PRIVATE_IP_BACKEND:5000;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
### 7.3 Membuat Docker Compose Web Server

Buat file `compose.yaml`:

```yaml
services:
  nginx:
    image: nginx:alpine
    container_name: wayshub-webserver
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./conf.d:/etc/nginx/conf.d:ro
      - ./certbot/conf:/etc/letsencrypt:ro
      - ./certbot/www:/var/www/certbot:ro
    networks:
      - team1

networks:
  team1:
    name: team1
    driver: bridge
```

Volume digunakan untuk menyimpan konfigurasi Nginx dan mengakses sertifikat SSL dari host.

### 7.4 Menjalankan Nginx

```bash
docker compose up -d
docker exec wayshub-webserver nginx -t
docker ps
```

Jika konfigurasi berubah:

```bash
docker exec wayshub-webserver nginx -t
docker exec wayshub-webserver nginx -s reload
```

> Network `team1` pada Web Server juga bersifat lokal. Reverse proxy mengakses Frontend dan Backend menggunakan private IP karena setiap layanan berjalan di VM berbeda.

## 8. Konfigurasi SSL Wildcard

### 8.1 Instal Certbot

Pada Web Server:

```bash
sudo apt update
sudo apt install certbot -y
```

### 8.2 Meminta Wildcard Certificate

Wildcard certificate memerlukan validasi DNS-01. Jalankan:

```bash
sudo certbot certonly \
  --manual \
  --preferred-challenges dns \
  -d mentor.studentdumbways.my.id \
  -d '*.mentor.studentdumbways.my.id'
```

Ikuti instruksi Certbot untuk menambahkan TXT record pada DNS. Tunggu sampai validasi berhasil.

Metode manual biasanya membutuhkan pembaruan sertifikat secara manual. Perhatikan masa berlaku sertifikat dan siapkan proses renewal.

### 8.3 Menggunakan Sertifikat di Nginx

Tambahkan konfigurasi HTTPS untuk frontend:

```nginx
server {
    listen 443 ssl;
    server_name mentor.studentdumbways.my.id;

    ssl_certificate /etc/letsencrypt/live/mentor.studentdumbways.my.id/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mentor.studentdumbways.my.id/privkey.pem;

    location / {
        proxy_pass http://PRIVATE_IP_FRONTEND:3000;

        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

Tambahkan konfigurasi HTTPS untuk API:

```nginx
server {
    listen 443 ssl;
    server_name api.mentor.studentdumbways.my.id;

    ssl_certificate /etc/letsencrypt/live/mentor.studentdumbways.my.id/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mentor.studentdumbways.my.id/privkey.pem;

    location / {
        proxy_pass http://PRIVATE_IP_BACKEND:5000;

        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

Sertifikat wildcard `*.mentor.studentdumbways.my.id` mencakup subdomain API. Sertifikat untuk domain utama juga diminta dalam perintah Certbot di atas.

Pastikan sertifikat tersedia pada path yang di-mount ke container Nginx.

Setelah perubahan:

```bash
docker exec wayshub-webserver nginx -t
docker exec wayshub-webserver nginx -s reload
```

Pastikan DNS kedua domain mengarah ke IP publik Web Server. Jika tugas mensyaratkan Cloudflare proxy OFF, atur record DNS menjadi **DNS only**.

## 9. Push Image ke Docker Hub

Login:

```bash
docker login
```

Push image backend:

```bash
docker push mentor/wayshub-backend:production
```

Push image frontend:

```bash
docker push mentor/wayshub-frontend:production
```

Pastikan nama image menggunakan username Docker Hub yang benar dan repository image sudah tersedia.

## 10. Instalasi dan Konfigurasi Jenkins

### 10.1 Menjalankan Jenkins Menggunakan Docker

Pada Jenkins Server:

```bash
mkdir -p ~/jenkins
cd ~/jenkins
```

Buat file `compose.yaml`:

```yaml
services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins
    restart: unless-stopped
    ports:
      - "127.0.0.1:8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home

volumes:
  jenkins_home:
```

Jalankan Jenkins:

```bash
docker compose up -d
docker ps
docker logs jenkins
```

Ambil initial admin password:

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Selesaikan setup melalui browser dan instal plugin yang dibutuhkan.

### 10.2 Menyiapkan SSH Key untuk Jenkins

Jenkins membutuhkan akses SSH ke server tujuan deployment.

Siapkan SSH key pada lingkungan Jenkins, lalu tambahkan public key ke `~/.ssh/authorized_keys` user deployment pada server tujuan.

Uji koneksi SSH dari lingkungan yang menjalankan pipeline:

```bash
ssh amri@PRIVATE_IP_BACKEND
```

Jika Jenkins berjalan dalam container, pastikan SSH client dan private key tersedia untuk pipeline. Simpan private key menggunakan Jenkins Credentials dan gunakan SSH Agent plugin atau mekanisme credential yang sesuai.

Jangan memasukkan private key ke repository.

### 10.3 Reverse Proxy Jenkins

Tambahkan server block berikut pada konfigurasi Nginx Web Server:

```nginx
server {
    listen 80;
    server_name jenkins.mentor.studentdumbways.my.id;

    location / {
        proxy_pass http://PRIVATE_IP_JENKINS:8080;

        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

Ganti `PRIVATE_IP_JENKINS` dengan private IP Jenkins Server.

**Penting:** konfigurasi Jenkins di atas memetakan port `8080` ke `127.0.0.1`, sehingga hanya dapat diakses secara lokal di Jenkins Server. Jika Nginx berada pada VM berbeda, ubah port mapping Jenkins agar dapat dijangkau melalui private IP Jenkins Server, lalu batasi aksesnya agar hanya Web Server yang diizinkan.

Setelah perubahan konfigurasi Nginx:

```bash
docker exec wayshub-webserver nginx -t
docker exec wayshub-webserver nginx -s reload
```

## 11. Pipeline CI/CD Jenkins

Pipeline mencakup:

1. Pull source code dari GitHub.
2. Build Docker image.
3. Test aplikasi.
4. Push image ke Docker Hub.
5. Deploy image pada server tujuan.
6. Memeriksa status aplikasi.
7. Mengirim notifikasi Discord.

### 11.1 Membuat Pipeline Job

Di Jenkins:

1. Pilih **New Item**.
2. Masukkan nama job.
3. Pilih **Pipeline**.
4. Hubungkan repository GitHub.
5. Konfigurasikan branch yang digunakan.
6. Pilih Pipeline script dari `Jenkinsfile`.
7. Simpan konfigurasi.

### 11.2 Auto Trigger SCM

Aktifkan pemicu otomatis saat ada perubahan source code menggunakan:

- GitHub webhook ke Jenkins; atau
- Poll SCM pada konfigurasi job.

Webhook membutuhkan Jenkins dapat menerima request dari GitHub melalui endpoint yang sesuai. Konfigurasikan akses dengan aman.

Buat `Jenkinsfile` sesuai struktur backend dan frontend. Gunakan Jenkins Credentials untuk Docker Hub dan SSH.

Pastikan pipeline menjalankan build, test, push, dan deployment sesuai kebutuhan aplikasi.

## 12. GitLab Runner dan Pipeline CI/CD

### 12.1 Registrasi GitLab Runner

Buka GitLab project:

**Settings → CI/CD → Runners**

Ikuti instruksi GitLab untuk memasang dan mendaftarkan Runner sesuai executor yang digunakan.

Pastikan Runner memiliki akses Docker yang diperlukan untuk build image. Jangan mengekspos Docker daemon tanpa perlindungan.

### 12.2 Membuat Pipeline CI/CD GitLab

Buat file `.gitlab-ci.yml` pada root repository aplikasi.

Pipeline meliputi:

1. Mengambil source code dari repository.
2. Build Docker image.
3. Test aplikasi.
4. Push image ke Docker Hub.
5. Deploy ke server tujuan.
6. Memverifikasi status deployment.
7. Mengirim notifikasi Discord.

Gunakan GitLab CI/CD Variables untuk menyimpan credential Docker Hub, SSH key, dan webhook Discord. Gunakan pengaturan masked dan protected jika sesuai dengan kebutuhan branch.

Aktifkan pipeline untuk branch yang digunakan dan uji dengan commit baru.

## 13. Discord Notification

Simpan URL webhook Discord di Jenkins Credentials atau GitLab CI/CD Variables.

Notifikasi pipeline dapat berisi:

- Nama project.
- Nama branch.
- Status build.
- Status test.
- Status deployment.
- Link pipeline.

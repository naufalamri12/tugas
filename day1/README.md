
#link tugas vidio 
(https://drive.google.com/file/d/1dwf1kckrmIW-ddGn7vUxvezs7wxrh9kJ/view?usp=sharing)

# Deploy MySQL, Wayshub Backend, dan Wayshub Frontend

## 1. Install MySQL

Pada server Appserver, lakukan update sistem dan install MySQL.

```bash
sudo apt update
sudo apt install mysql-server -y
```

Aktifkan service MySQL:

```bash
sudo systemctl enable --now mysql
```

Periksa status MySQL:

```bash
sudo systemctl status mysql
```

## 2. Menjalankan MySQL Secure Installation

Jalankan perintah berikut untuk mengamankan instalasi MySQL:

```bash
sudo mysql_secure_installation
```

Ikuti prompt yang ditampilkan sesuai konfigurasi MySQL.

Periksa metode autentikasi user root:

```bash
sudo mysql
```

```sql
SELECT user, host, plugin
FROM mysql.user
WHERE user = 'root';
```

## 3. Membuat Database dan User Aplikasi

Masuk ke MySQL:

```bash
sudo mysql
```

Buat database `wayshub`:

```sql
CREATE DATABASE wayshub;
```

Buat user database:

```sql
CREATE USER 'wayshub_user'@'localhost'
IDENTIFIED BY 'GANTI_PASSWORD_KUAT';
```

Berikan hak akses kepada user untuk mengelola database `wayshub`:

```sql
GRANT ALL PRIVILEGES ON wayshub.*
TO 'wayshub_user'@'localhost';

FLUSH PRIVILEGES;
```

Ganti `GANTI_PASSWORD_KUAT` dengan password yang kuat.

## 4. Mengubah Bind Address MySQL

Edit konfigurasi MySQL:

```bash
sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf
```

Cari konfigurasi berikut:

```text
bind-address = 127.0.0.1
```

Ubah bind address agar MySQL dapat menerima koneksi melalui interface jaringan yang diperlukan.

Contoh jika private IP Appserver adalah `10.0.2.10`:

```text
bind-address = 10.0.2.10
```

Sesuaikan dengan private IP Appserver atau public IP jika memang diperlukan.

Restart MySQL:

```bash
sudo systemctl restart mysql
```

Periksa port MySQL:

```bash
sudo ss -lntp | grep 3306
```

## 5. Membuat Database Demo dan Tabel Transaction

Masuk ke MySQL:

```bash
sudo mysql
```

Buat database `demo`:

```sql
CREATE DATABASE demo;
```

Gunakan database tersebut:

```sql
USE demo;
```

Buat tabel `transaction`:

```sql
CREATE TABLE `transaction` (
    id INT AUTO_INCREMENT PRIMARY KEY,
    transaction_name VARCHAR(100) NOT NULL,
    amount DECIMAL(12,2) NOT NULL,
    transaction_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Masukkan data dummy:

```sql
INSERT INTO `transaction` (transaction_name, amount)
VALUES
('Pembelian Laptop', 7500000.00),
('Pembelian Keyboard', 350000.00),
('Pembelian Mouse', 150000.00);
```

Periksa data:

```sql
SELECT * FROM `transaction`;
```

## 6. Membuat Role Admin dan Guest

Buat dua role, yaitu `admin` dan `guest`:

```sql
CREATE ROLE 'admin', 'guest';
```

### Memberikan Hak Akses Admin

Role `admin` memiliki hak akses SELECT, INSERT, UPDATE, dan DELETE pada tabel `transaction`.

```sql
GRANT SELECT, INSERT, UPDATE, DELETE
ON demo.`transaction`
TO 'admin';
```

### Memberikan Hak Akses Guest

Role `guest` hanya memiliki hak akses SELECT:

```sql
GRANT SELECT
ON demo.`transaction`
TO 'guest';
```

## 7. Membuat User dan Memberikan Role

Buat user admin dengan username `your_name` dan password sesuai kebutuhan tugas:

```sql
CREATE USER 'your_name'@'%'
IDENTIFIED BY 'GANTI_PASSWORD_ANDA';
```

Buat user guest:

```sql
CREATE USER 'guest'@'%'
IDENTIFIED BY 'guest';
```

Berikan role kepada masing-masing user:

```sql
GRANT 'admin' TO 'your_name'@'%';

GRANT 'guest' TO 'guest'@'%';
```

Aktifkan role sebagai default role:

```sql
SET DEFAULT ROLE 'admin'
TO 'your_name'@'%';

SET DEFAULT ROLE 'guest'
TO 'guest'@'%';
```

Pastikan `your_name` dan `GANTI_PASSWORD_ANDA` disesuaikan dengan username dan password yang digunakan.

## 8. Pengujian User Admin

Login menggunakan user admin:

```bash
mysql -u your_name -p -D demo
```

### 8.1 Mengakses Data

```sql
SELECT * FROM `transaction`;
```

### 8.2 Menambahkan Data

```sql
INSERT INTO `transaction` (transaction_name, amount)
VALUES ('Testing Admin', 100000);
```

### 8.3 Mengubah Data

```sql
UPDATE `transaction`
SET amount = 125000
WHERE transaction_name = 'Testing Admin';
```

### 8.4 Menghapus Data

```sql
DELETE FROM `transaction`
WHERE transaction_name = 'Testing Admin';
```

Pastikan seluruh operasi SELECT, INSERT, UPDATE, dan DELETE berhasil dijalankan oleh user admin.

### 8.5 Pengujian User Guest

Login menggunakan user guest:

```bash
mysql -u guest -p -D demo
```

Uji membaca data:

```sql
SELECT * FROM `transaction`;
```

Uji menambahkan data:

```sql
INSERT INTO `transaction` (transaction_name, amount)
VALUES ('Testing Guest', 50000);
```

Uji mengubah data:

```sql
UPDATE `transaction`
SET amount = 50000
WHERE id = 1;
```

Uji menghapus data:

```sql
DELETE FROM `transaction`
WHERE id = 1;
```

# Deploy Wayshub Backend

## 1. Install Node.js Versi 14

Pada server Gateway, gunakan NVM untuk menginstal Node.js versi 14:

```bash
nvm install 14
nvm use 14
nvm alias default 14
```

Verifikasi versi Node.js dan npm:

```bash
node -v
npm -v
```

## 2. Clone Repository Backend

Clone repository Wayshub Backend:

```bash
git clone URL_REPOSITORY_WAYSHUB_BACKEND
```

Masuk ke direktori aplikasi:

```bash
cd dumbflix-backend
```

Ganti `URL_REPOSITORY_WAYSHUB_BACKEND` dengan URL repository yang digunakan.

## 3. Konfigurasi Database Backend

Buka file konfigurasi database:

```bash
vim config/config.json
```

Sesuaikan konfigurasi database dengan MySQL yang berjalan pada Appserver.

Contoh konfigurasi:

```json
{
  "development": {
    "username": "wayshub_user",
    "password": "GANTI_PASSWORD_KUAT",
    "database": "wayshub",
    "host": "10.0.2.10",
    "dialect": "mysql"
  }
}
```

Sesuaikan username, password, nama database, dan host dengan konfigurasi MySQL yang sudah dibuat.

Pastikan user database memiliki izin koneksi dari Gateway ke Appserver.

Gunakan struktur konfigurasi asli dari repository jika berbeda dengan contoh di atas.

## 4. Install Dependencies dan Sequelize CLI

Install dependencies aplikasi:

```bash
npm install
```

Install Sequelize CLI:

```bash
npm install --save-dev sequelize-cli
```

Jika repository memiliki `package-lock.json` yang sesuai, kamu juga dapat menggunakan:

```bash
npm ci
```

## 5. Menjalankan Database Creation dan Migration

Jalankan perintah berikut untuk membuat database berdasarkan konfigurasi Sequelize jika database tersebut belum tersedia:

```bash
npx sequelize-cli db:create
```

Kemudian jalankan migration:

```bash
npx sequelize-cli db:migrate
```

Periksa hasil migration dan pastikan tabel yang dibutuhkan aplikasi telah dibuat pada database MySQL.

Jika database `wayshub` sudah dibuat secara manual dan user aplikasi tidak memiliki izin membuat database, lewati `db:create` dan jalankan migration saja.

## 6. Deploy Backend Menggunakan PM2

Install PM2 secara global:

```bash
npm install -g pm2
```

Jalankan aplikasi backend menggunakan PM2. Contoh jika entry point aplikasi adalah `index.js`:

```bash
pm2 start index.js --name wayshub-backend
```

Sesuaikan `index.js` dengan entry point yang digunakan oleh aplikasi.

Periksa status aplikasi:

```bash
pm2 status
```

Periksa log aplikasi:

```bash
pm2 logs wayshub-backend
```

Simpan konfigurasi proses PM2:

```bash
pm2 save
```

Aktifkan startup otomatis:

```bash
pm2 startup
```

Jalankan perintah tambahan yang ditampilkan oleh `pm2 startup` agar aplikasi dapat berjalan kembali setelah server melakukan restart.

---

# Deploy Wayshub Frontend

## 1. Clone Repository Frontend

Gunakan Node.js versi 14:

```bash
nvm use 14
```

Clone repository Wayshub Frontend:

```bash
git clone URL_REPOSITORY_WAYSHUB_FRONTEND
```

Masuk ke direktori frontend sesuai nama folder hasil clone.

## 2. Konfigurasi URL Backend

Buka file konfigurasi API:

```bash
vim src/config/api.js
```

Sesuaikan URL API dengan alamat backend yang dapat diakses oleh browser.

Contoh:

```javascript
const API = "http://PUBLIC_IP_GATEWAY:5000";
export default API;
```

Ganti `PUBLIC_IP_GATEWAY` dengan alamat Gateway dan `5000` dengan port backend yang digunakan.

Pastikan backend dapat diakses dari frontend dan konfigurasi CORS sudah sesuai.


## 3. Deploy Frontend dengan PM2

Jika aplikasi menyediakan script `start` yang menjalankan server Node.js, jalankan:

```bash
pm2 start npm --name wayshub-frontend -- start
```

Jika frontend menggunakan React Scripts, kamu dapat membangun aplikasi terlebih dahulu:

```bash
npm run build
```

Hasil build dapat disajikan menggunakan web server seperti Nginx. Jika menggunakan Nginx untuk menyajikan file statis, PM2 tidak wajib menjalankan frontend secara langsung.

Periksa status proses:

```bash
pm2 status
```

Periksa log frontend jika dijalankan menggunakan PM2:

```bash
pm2 logs wayshub-frontend
```

Simpan konfigurasi PM2:

```bash
pm2 save
```

## 4. Verifikasi Frontend dan Backend

Periksa port yang digunakan aplikasi:

```bash
sudo ss -lntp
```
Seluruh proses pengerjaan didokumentasikan dalam repository GitHub sebagai bukti penyelesaian tugas.


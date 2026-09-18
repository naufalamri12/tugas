tugas 1
Dengan mendaftar akun free tier AWS/GCP/Azure, buatlah Infrastructre dengan terraform menggunakan registry yang sudah ada. dengan beberapa aturan berikut :
Buatlah 2 buah server dengan OS ubuntu 24 dan debian 11 (Untuk spec menyesuaikan)
attach vpc ke dalam server tersebut
attach ip static ke vm yang telah kalian buat
pasang firewall ke dalam server kalian dengan rule {allow all ip(0.0.0.0/0)}
buatlah 2 block storage di dalam terraform kalian, lalu attach block storage tersebut ke dalam server yang ingin kalian buat. (pasang 1 ke server ubuntu dan 1 di server debian)
test ssh ke server

jawaban 
(untuk main.tf berisi File ini berisi resource infrastructure yang ingin dibuat sedangkan providers.tf konfigurasi provider AWS) 
1. buat code yang dibutuhkan seperti:
VPC
Subnet
Internet Gateway
Route Table
Route Table Association
2. cari tau kita mau menggunakanserver dengan os apa contoh ubuntu 13 atau debian 11 ( pastikan di aws/gcp/azure terdapat pilihan tersebut)
3. jangan lupa memasukan elastic ip agar saat di offkan, tetap berjalan
4. setelah itu memasukan security group inbound dan outbound
5. masukan block storage sesuai yang kita mau semisal 10gb/ 15 gb
6. setelah itu lakukan terraform init,plan and apply,output
7. setelah itu bisa mengecheck di provider atau ssh -i ~/.ssh/aws-key.pem ubuntu@13.xxx.xxx.xxx
8. melakukan terraform destroy semisalnya ada yang gagal





pertanyaan
Buatlah ansible untuk :
Membuat user baru, gunakan login ssh key & password
Instalasi Docker
Deploy application frontend yang sudah kalian gunakan sebelumnya menggunakan ansible.
Instalasi Monitoring Server (node exporter, prometheus, grafana)
Setup reverse-proxy
Generated SSL certificate

catatan
jika ingin menghapus file kita bisa menjalankan uninstall aplikasi atau melalui ansible
jawaban
1. kita membuat file Inventory dimana itu berisi ip yang akan kita tuju
2. jangan lupa memasangkan ssh kita di autorized.key di server yang kita tuju
3. membuat ansible.cfg untuk mengetahui file kita berada dimana
[defaults]
inventory = Inventory
private_key_file = .......

4.untuk membuat user baru, instalasi doncker dan deploy application kita memakai pemahaman kita sebelumnya
contoh. untuk ssh kita membuat user ( id dan pass) , setelah itu memasukan  ssh public, melakukan perubahan /etc/ssh/sshd_config dimana PubkeyAuthentication yes dan  PasswordAuthentication yes
5. setelah itu baru kita mmengecheck dengan langsung masuk ke server
6. untuk install node exporter, prometheus, grafana kita membaca dokumentasi dan ada beberapa code yang harus disiapkan
7. untuk reverse proxy pertama kita masuk ke cloudfire untuk membuat dns dan menggunakan ip yang kita inginkan
8. setelah itu menulis code seperti seperti yang lainnay seperi menginstall nginx setelah itu membuat  Nginx configuration
9. untuk ssl saya menggunakan certbot dan tinggal mengikuti perintah yang ada diweb tetapi dimasukan ke ansible

[Monitoring Server]
Setup node-exporter, prometheus dan Grafana menggunakan docker / native diperbolehkan
monitoring seluruh server yang kalian buat di materi terraform dan yang kalian miliki di biznet.
Reverse Proxy
bebas ingin menggunakan nginx native / docker
SSL Cloufflare on / certbot SSL biasa / wildcard SSL diperbolehkan
Dengan Grafana, buatlah :
Dashboard untuk monitor resource server (CPU, RAM & Disk Usage) buatlah se freestyle kalian.
Buat dokumentasi tentang rumus promql yang kalian gunakan
Buat alerting dengan Contact Point pilihan kalian (discord, telegram, slack dkk)
Untuk alert :
Boleh menggunakan alert manager / alert rule dari grafana
Ketentuan alerting yang harus dibuat
CPU Usage over 20%
RAM Usage over 75%
Monitoring specific container
deploy application frontend di app-server
monitoring frontend container
untuk alerting bisa di check di server discord yaa, sudah di buatkan channel alerting

catatan
jangan lupa memasukan di config reverse proxy dibawah proxy_pass jika terjadi masalah origin not allowed
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

semisalnya ada masalah di targethealt tidak terbaca bisa mengganti file prometheus.yml dengan ipnya saja 

jika mau menggabungkan file bisa dengan ancible with in loob sehingga 1 file bisa untuk beberapa ssl atau custom seperti yang kita mau 


jawaban 
1. karena kita sudah setting menggunakan ansible kita tinggal membuka browser denga ip:9090 untuk membuka prometheus dan ip 3000/3001 untuk grafa
2. untuk prometheus tinggal pergi ke status dan target healt
3. reverse proxy dan ssl kita bisa menggabungkan file dengan ancible with in loob
4. sama login ke grafa akan dimintai admin, kita mengisi id = admin, pass=admin setelah itu kita diminta untuk merubah password
5. setelah kita masuk pertama kita pergi ke data sources
6. setelah itu kita pergi ke dasboard
7. kita bisa mengisi secara manual atau automatis melalui kode
8. untuk kode yang saya pakai
cpu :
1000 * (
  1 - avg(
    rate(node_cpu_seconds_total{
      instance="15.232.204.64:9100",
      mode="idle"
    }[5m])
  )
)

ram :
100 * (
  1 -
  (
    sum(node_memory_MemAvailable_bytes{
      instance="15.232.204.64:9100",
      job="dumbflix"
    })
    /
    sum(node_memory_MemTotal_bytes{
      instance="15.232.204.64:9100",
      job="dumbflix"
    })
  )
)

dish usage :
node_filesystem_size_bytes{
  instance="15.232.204.64:9100",
  job="dumbflix",
  mountpoint="/",
  fstype="ext4"
}
-
node_filesystem_avail_bytes{
  instance="15.232.204.64:9100",
  job="dumbflix",
  mountpoint="/",
  fstype="ext4"
}


9. alerting dengan Contact Point bisa pergi ke alerting rule untuk discord kalian harus membuat channel discord dan pergi kesetting dan integrasi dan klick webhooks setelah itu kita copy webhook dan paste di alerting
10. kita coba untuk monitoring frontend container dimana semisalnya container mati, dia akan mengirimkan pesan kita
code :
absent(container_last_seen{
  instance="15.232.204.64:8080",
  name="dumbflix-frontend"
})

12. 

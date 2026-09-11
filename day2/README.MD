vidio link (https://drive.google.com/file/d/1kv1HwS9YHiy1EZSzUcQL-AtVUbPmKQ_C/view?usp=sharing)

## docker

1. Buatlah bash script se freestyle mungkin untuk melakukan installasi docker. untuk skrip  install saya melihat di website resmi dockernya dan tidak menemukan kesulitan
2.Deploy aplikasi Web Server, Frontend, Backend, serta Database on top docker compose. untuk membuat docker compose cukup mudah karena kita hanya memasukan semua kode menjadi 1 folder, jadi deploynya tidak 1 per 1
3. Ketentuan pengerjaan:
Deploy database di server terpisah
Server Backend terpisah
Server Frontend terpisah
Web Server juga terpisah untuk reverse proxy kalian nantinya.
Untuk penamaan image, sesuaikan dengan environment masing masing, ex: mentor/dumbflx-frontend:production
Di dalam docker-compose file buat suatu custom network dengan nama team kalian, lalu pasang ke setiap service yang kalian miliki.
Deploy database terlebih dahulu menggunakan mysql dan jangan lupa untuk pasang volume di bagian database.
=untuk disini beluma ada kesulitan tetapi kadang bingung saat mencari file karena di luar tidak ada
4.gunakan multistage build . untuk multistage build saya menguranginnya dengan cara memakai nginx:alpine
5.Web Server buatlah configurasi reverse-proxy menggunakan nginx on top docker.
6. Push image ke docker registry kalian masing".
7.Aplikasi dapat berjalan dengan sesuai seperti melakukan login/register.

## Jenkins
1Installasi Jenkins on top Docker or native
2.Setup SSH-KEY di local server jenkins kalian, agar dapat login ke dalam server menggunakan SSH-KEY
3Reverse Proxy Jenkins
4gunakan domain ex. jenkins.mentor.studentdumbways.my.id
5.reverse proxy sesuaikan dengan ketentuan yang ada di dalam Jenkins documentation
6.Buatlah beberapa Job untuk aplikasi kalian yang telah kalian deploy di task sebelumnya (frontend && backend)
7.Untuk script CICD atur flow pengupdate an aplikasi se freestyle kalian dan harus mencangkup
8.Pull dari repository
9Dockerize/Build aplikasi kita
10Test application
11Push ke Docker Hub
12Deploy aplikasi on top Docker
13Auto trigger setiap ada perubahan di SCM
14Buat job notification ke discord
menurut saya reverse proxy jenkins tidak bisa disatuin sama yang lainnya karena saya sudah mencoba atau paling ada code saya yang salah atau belum dimasukin 

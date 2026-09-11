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



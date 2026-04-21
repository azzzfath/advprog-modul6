# Tutorial 6 - Web Server in Rust

**Nama:** Muhammad Azzam Fathurrahman
**NPM:** [Isi dengan NPM Genap kamu]

---

## Commit 1 Reflection notes
Pada tahap ini, saya menambahkan fungsi `handle_connection` yang bertugas untuk menangani koneksi masuk dari *browser* (klien). 
Di dalam fungsi ini, saya menggunakan `BufReader` untuk membaca data dari `TcpStream` secara efisien baris demi baris. Program akan membaca setiap baris dari HTTP *request* yang masuk dan memasukkannya ke dalam `Vec` (vektor) menggunakan iterator. Pembacaan berhenti ketika menemukan baris kosong (`line.is_empty()`), yang menandakan akhir dari header HTTP. Setelah itu, isi request tersebut dicetak ke terminal sehingga kita bisa melihat bagaimana browser berkomunikasi dengan server.

## Commit 2 Reflection notes
![Commit 2 screen capture](assets/images/commit2.png)

Pada tahap ini, server tidak lagi membiarkan browser dalam keadaan *loading* kosong, melainkan sudah membalas request dengan halaman web aktual (`hello.html`). 
Saya belajar bahwa untuk mengirimkan response HTTP yang valid, kita harus menyertakan *Status Line* (misal: `HTTP/1.1 200 OK`) dan *Headers*, khususnya `Content-Length` yang memberi tahu browser seberapa besar ukuran file HTML yang dikirim. Jika ukuran ini salah, browser mungkin tidak akan merender halaman dengan benar atau terpotong. Server akan membaca file HTML menjadi string menggunakan `fs::read_to_string`, mengukur panjangnya, menyusunnya dalam format HTTP response yang standar, lalu mengirimkannya kembali ke klien melalui `stream.write_all`.

## Commit 3 Reflection notes
*Refactoring* sangat penting di tahap ini karena sebelumnya server akan mengembalikan file `hello.html` terlepas dari apa pun URL yang diminta oleh pengguna (bahkan jika URL-nya ngawur). 
Dengan melakukan *refactoring* menggunakan percabangan `match` (atau `if/else`), server sekarang mengecek isi dari `request_line`. Jika permintaannya adalah `GET / HTTP/1.1` (halaman utama), server akan mengembalikan `hello.html` dengan status `200 OK`. Namun, jika permintaannya hal lain, server akan mengembalikan `404.html` dengan status `404 NOT FOUND`. *Refactoring* ini juga menyatukan logika pembacaan file dan pengiriman respons di bagian akhir fungsi, sehingga kode tidak berulang (prinsip DRY - *Don't Repeat Yourself*) dan lebih mudah untuk *maintenance*.

## Commit 4 Reflection notes
Pada *milestone* ini, saya menyimulasikan *slow response* dengan menambahkan rute `/sleep` yang akan menghentikan *thread* selama 10 detik (`thread::sleep`). 
Saat saya mencoba membuka `/sleep` di satu tab browser dan membuka halaman utama `/` di tab lain secara bersamaan, tab kedua baru bisa *loading* setelah tab pertama selesai (setelah 10 detik). Hal ini terjadi karena server yang kita buat masih bersifat *Single-Threaded*. Artinya, server hanya bisa memproses satu permintaan klien pada satu waktu. Klien berikutnya yang datang harus mengantre sampai urusan klien sebelumnya benar-benar selesai. Ini adalah masalah besar untuk server produksi (skalabilitas buruk).

## Commit 5 Reflection notes
Untuk mengatasi masalah antrean lambat pada *single-threaded server*, saya mengimplementasikan *Multithreaded Server* menggunakan konsep `ThreadPool`. 
Alih-alih membuat *thread* baru setiap kali ada koneksi masuk (yang berbahaya karena bisa membuat sistem kehabisan memori jika diserang jutaan *request* atau DDoS), `ThreadPool` hanya akan membuat sejumlah *worker threads* yang jumlahnya terbatas (misal: 4 *threads*) sejak awal. 
Mekanisme kerjanya menggunakan *channel* (`mpsc` - *multiple producer, single consumer*). `ThreadPool` bertindak sebagai *producer* yang mengirimkan pekerjaan (berupa *closure* atau fungsi tugas) ke dalam antrean. Para `Worker` (yang berjalan di dalam masing-masing *thread*) bertindak sebagai *consumer* yang akan berebut mendengarkan *channel* tersebut. Begitu ada tugas masuk, salah satu *worker* yang sedang menganggur akan mengambilnya dan mengeksekusinya. Dengan cara ini, server bisa menangani banyak *request* secara konkuren (bersamaan) tanpa takut server *crash* karena terlalu banyak *thread*.

## Bonus: Function improvement Reflection notes
Pembuatan fungsi `build` sebagai pengganti `new` pada `ThreadPool` adalah praktik idiomatik yang sangat disarankan di bahasa Rust untuk *error handling*.
Sebelumnya, pada fungsi `new`, jika kita memberikan ukuran kapasitas pool `0`, fungsi akan langsung menggunakan makro `assert!` yang menyebabkan program *panic* dan server langsung mati secara mendadak. 
Dengan menggantinya menjadi fungsi `build`, alih-alih melakukan *panic*, fungsi tersebut akan mengembalikan tipe data `Result::Err` jika inputnya tidak valid. Hal ini memberikan kebebasan kepada pemanggil fungsi (fungsi `main`) untuk menangani *error* secara anggun (*graceful*), misalnya dengan mencetak pesan *error* yang ramah atau mencoba melakukan pemulihan (*recovery*), sehingga kestabilan aplikasi lebih terjamin.
# Reflection

## 1. Apa perbedaan utama antara unary, server streaming, dan bidirectional streaming RPC, dan kapan masing-masing paling cocok digunakan?

Unary itu paling simpel, client kirim satu request, server balas satu response. Cocok buat operasi sederhana kayak submit pembayaran atau login.

Server streaming cocok kalau datanya banyak tapi requestnya cuma sekali, misalnya ambil riwayat transaksi. Server bisa kirim data sedikit-sedikit tanpa client harus request berkali-kali.

Bidirectional streaming cocok buat komunikasi dua arah yang real time kayak chat, di mana client dan server bisa saling kirim pesan kapan saja tanpa harus nunggu giliran.

## 2. Apa saja pertimbangan keamanan dalam implementasi gRPC di Rust, terutama soal autentikasi, otorisasi, dan enkripsi?

Di tutorial ini koneksinya masih pakai HTTP biasa, jadi data tidak terenkripsi. Di production harusnya pakai TLS supaya data aman saat dikirim.

Selain itu belum ada autentikasi sama sekali, jadi siapa saja bisa akses semua service. Idealnya ditambahkan JWT atau token di metadata gRPC. Setelah tahu siapa usernya, baru bisa dilakukan pengecekan otorisasi, misalnya user A tidak boleh lihat transaksi user B.

## 3. Apa saja tantangan yang mungkin muncul saat menangani bidirectional streaming di Rust gRPC, khususnya pada aplikasi chat?

Yang paling sering jadi masalah adalah ketika salah satu sisi disconnect tiba-tiba, stream bisa langsung berhenti tanpa error yang jelas. Selain itu kalau pesan masuk terlalu cepat dan buffer channel sudah penuh, bisa terjadi blocking atau pesan hilang. Pengelolaan task yang dispawn juga perlu diperhatikan supaya tidak ada task yang terus jalan padahal koneksi sudah putus.

## 4. Apa kelebihan dan kekurangan menggunakan `tokio_stream::wrappers::ReceiverStream` untuk streaming di Rust gRPC?

Kelebihannya, ReceiverStream memudahkan konversi dari channel tokio ke stream yang bisa langsung dipakai tonic. Integrasi dengan `tokio::spawn` juga natural dan backpressure sudah ditangani otomatis lewat buffer channel.

Kekurangannya, ukuran buffer harus dipilih dengan tepat. Terlalu kecil bisa blocking, terlalu besar boros memori. Kalau task sender panic, stream cuma berhenti begitu saja tanpa error yang informatif ke client.

## 5. Bagaimana kode Rust gRPC bisa distruktur agar lebih mudah di-maintain dan dikembangkan?

Setiap service bisa dipisah ke file masing-masing, misalnya `services/payment.rs`, `services/transaction.rs`, supaya tidak menumpuk dalam satu file. Logika bisnis sebaiknya dipisah dari handler gRPC lewat trait, sehingga lebih mudah di-test dan diganti implementasinya. Konfigurasi seperti port dan alamat server juga lebih baik dibaca dari environment variable, bukan dihardcode.

## 6. Langkah tambahan apa yang diperlukan di `MyPaymentService` untuk menangani logika pembayaran yang lebih kompleks?

Saat ini implementasinya langsung return `success: true` tanpa validasi apapun. Untuk kasus nyata, perlu ditambahkan validasi input (misal amount harus lebih dari 0), pengecekan apakah user valid, penyimpanan ke database, dan integrasi ke payment gateway seperti Midtrans atau Stripe. Mekanisme idempotency juga penting supaya pembayaran yang sama tidak diproses dua kali.

## 7. Apa dampak penggunaan gRPC terhadap arsitektur sistem terdistribusi, terutama soal interoperabilitas?

gRPC mendorong desain berbasis kontrak lewat file `.proto` yang jadi sumber kebenaran tunggal untuk semua service. Ini bagus untuk sistem microservice karena kode client dan server bisa digenerate otomatis untuk banyak bahasa. Tapi kelemahannya, gRPC kurang ramah untuk browser tanpa tambahan gateway, tidak seperti REST yang bisa langsung diakses dari mana saja.

## 8. Apa kelebihan dan kekurangan HTTP/2 yang dipakai gRPC dibanding HTTP/1.1 atau HTTP/1.1 dengan WebSocket?

HTTP/2 mendukung multiplexing sehingga banyak request bisa jalan bersamaan dalam satu koneksi TCP, lebih efisien dari HTTP/1.1 yang harus buka koneksi baru tiap request. Header juga dikompres sehingga lebih hemat bandwidth.

Kekurangannya, format binary HTTP/2 tidak bisa dibaca langsung, jadi debugging lebih susah. Browser juga tidak bisa langsung pakai gRPC tanpa lapisan tambahan, berbeda dengan WebSocket yang sudah didukung native.

## 9. Bagaimana model request-response REST berbeda dengan bi-directional streaming gRPC untuk komunikasi real-time?

REST mengharuskan client polling ke server untuk dapat update terbaru, yang artinya ada delay dan banyak request yang mungkin tidak perlu. gRPC dengan bidirectional streaming bisa langsung push data begitu ada yang baru tanpa client harus minta duluan, jadi lebih responsif dan efisien untuk use case real-time.

## 10. Apa implikasi penggunaan Protocol Buffers yang schema-based dibanding JSON yang schema-less di REST API?

Protobuf memaksa kedua sisi (client dan server) setuju dulu soal struktur data lewat file `.proto` sebelum bisa berkomunikasi. Ini bagus karena kesalahan kontrak ketahuan lebih awal dan ukuran data lebih kecil karena binary.

JSON lebih fleksibel dan mudah dibaca manusia, tapi tidak ada jaminan strukturnya konsisten. Kalau server ubah nama field, client bisa gagal diam-diam tanpa error yang jelas. Untuk komunikasi antar service internal, Protobuf lebih aman; untuk API publik, JSON lebih mudah diakses.

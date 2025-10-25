# 🛡️ Analisis Kasus Keamanan Siber Deloitte: Daikibo Industrials

Ini adalah dokumentasi analisis saya untuk studi kasus singkat dari Deloitte mengenai potensi pelanggaran keamanan di Daikibo Industrials.

## 1. Latar Belakang Kasus (Background)

Daikibo Industrials mengalami masalah produksi yang menghentikan lini perakitan. Ada dugaan bahwa *status board* baru mereka telah dibobol. Informasi sensitif klien juga telah bocor ke publik.

## 2. Objektif / Tugas (The Task)

Tugas saya adalah menganalisis file `web_activity.log` untuk:
1.  Menentukan apakah peretasan bisa terjadi dari internet langsung (tanpa akses VPN).
2.  Memeriksa log untuk menemukan permintaan (requests) yang mencurigakan.
3.  Mengidentifikasi ID pengguna yang terkait dengan aktivitas mencurigakan tersebut.

## 3. Alur Analisis (My Workflow)

Berikut adalah langkah-langkah yang saya lakukan untuk menganalisis log:

1.  **Inspeksi Awal:** Saya membuka file `web_activity.log` di VS Code.
2.  **Mencari Pola Login:** Saya mencari request `POST /login` untuk mengidentifikasi sesi pengguna yang aktif.
3.  **Menganalisis Alur Normal:** Alur wajar adalah: `POST /login` -> `GET /dashboard` -> `GET /css/...` -> `GET /js/...` -> `GET /api/machines`.
4.  **Mencari Anomali (Automated Requests):** Dashboard ini *tidak* memiliki fitur auto-refresh. Oleh karena itu, saya secara spesifik mencari:
    * Pola request ke API (`/api/factory/machine/status`) yang terjadi berulang kali.
    * Request yang memiliki **interval waktu yang tepat dan konsisten** (misalnya, terjadi setiap X detik/menit/jam secara presisi), yang mengindikasikan adanya skrip, bukan refresh manual oleh manusia.

## 4. Temuan (Findings) 🔍

Setelah menganalisis log, saya menemukan temuan berikut:

* **IP Mencurigakan:** Alamat IP **`192.168.0.101`** menunjukkan aktivitas yang sangat mencurigakan dan terotomatisasi.

* **Pola Serangan:** Saya menemukan pola permintaan API (`/api/factory/machine/status?...`) yang terjadi dengan interval **tepat 1 jam**, selalu pada detik ke-48.
    * **Bukti Otomatisasi:** Setiap jam, empat permintaan API ke semua pabrik (meiyo, seiko, shenzhen, berlin) dikirim pada **waktu milidetik yang persis sama**. Manusia tidak mungkin melakukan ini secara manual.
    * **Contoh Timestamp:** Pola ini terlihat jelas pada timestamp seperti `2021-06-25T17:00:48.000Z`, `2021-06-25T18:00:48.000Z`, `2021-06-25T19:00:48.000Z`, dan seterusnya.
    * **Aktivitas Berlanjut:** Aktivitas otomatis ini bahkan **terus berjalan setelah sesi kedaluwarsa**. Mulai pukul `2021-06-26T00:00:48.000Z`, skrip tersebut mulai menerima error `401 UNAUTHORIZED`, namun tetap mencoba melakukan request setiap jam.
    * **Login Ulang:** Pola otomatis ini berlanjut kembali setelah pengguna login ulang pada `2021-06-26T16:04:54.000Z`.

* **ID Pengguna Teridentifikasi:** Semua aktivitas otomatis dan mencurigakan dari IP `192.168.0.101` ini terkait dengan ID pengguna: **`mdB7yD2dp1BFZPontHBQ1Z`**.

## 5. Kesimpulan

Berdasarkan temuan di atas, saya menyimpulkan:

1.  **Apakah dari Internet?** **Tidak Dapat Ditentukan (dari log ini).** Log yang diberikan hanya berisi alamat IP internal (rentang `192.168.x.x`). Ini berarti penyerang **sudah berada di dalam jaringan**, kemungkinan besar menggunakan **VPN** yang valid atau merupakan ancaman dari dalam (*insider threat*). Log ini tidak memberikan bukti adanya serangan *langsung* dari internet (tanpa VPN).

2.  **Apakah Ada Pelanggaran?** **Ya.** Ada bukti kuat bahwa akun pengguna **`mdB7yD2dp1BFZPontHBQ1Z`** dari IP **`192.168.0.101`** menggunakan skrip otomatis untuk mengekstrak data status mesin secara periodik (setiap jam). Mengingat dashboard tidak memiliki fungsionalitas auto-refresh, ini adalah aktivitas terlarang yang sangat mungkin terkait dengan kebocoran data sensitif.
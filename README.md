# EXPLOIT-DB RCE TRIAGE CHALLENGE

## Studi Kasus: Laravel Debug Mode RCE pada myITS Thesis

**Tim:** Bug Bounty ITS\
**Sumber observasi:** Pengujian tambahan, 21–22 Oktober 2026\
**Tanggal penyusunan triage:** 11 November 2026\
**Kandidat exploit:** EDB-ID 49424 / CVE-2021-3129\
**Metode:** Analisis laporan terdahulu, pencocokan advisory, dan static source code review\
**Status:** Kandidat RCE, belum terkonfirmasi pada aset ITS

---

## 1. Latar Belakang

Pada kegiatan Bug Bounty ITS sebelumnya, kelompok melakukan pengujian keamanan aplikasi myITS Thesis melalui metode *error-based analysis*, manipulasi HTTP request menggunakan Burp Suite, dan peninjauan kode sumber yang tersedia.

Pengujian tersebut menghasilkan dua kelas temuan:

1. **Information Disclosure:** Aplikasi menampilkan informasi internal ketika terjadi kesalahan, termasuk stack trace, namespace, path server, dan detail query SQL Server.
2. **Weak File Type Validation:** Endpoint unggah berkas menerima file uji polyglot dengan karakteristik GIF yang mengandung konten PHP, berdasarkan respons yang didokumentasikan dalam laporan sebelumnya.

Kedua temuan menunjukkan area yang perlu diperiksa lebih lanjut. Namun, tidak satu pun secara otomatis membuktikan adanya Remote Code Execution (RCE).

Untuk aktivitas Exploit-DB RCE Triage Challenge, kelompok memilih CVE-2021-3129 sebagai kandidat karena berhubungan dengan framework Laravel dan fitur debugging yang memiliki keterkaitan teknis dengan temuan sebelumnya.

## 2. Identitas Exploit

| Komponen              | Informasi                                                       |
| --------------------- | --------------------------------------------------------------- |
| Nama exploit          | Laravel 8.4.2 Debug Mode — Remote Code Execution                |
| Exploit-DB ID         | 49424                                                           |
| CVE                   | CVE-2021-3129                                                   |
| Author PoC            | SunCSR Team                                                     |
| Tanggal publikasi PoC | 14 Januari 2021                                                 |
| Produk terkait        | Laravel / Facade Ignition                                       |
| Platform              | PHP                                                             |
| Bahasa PoC            | Python 3                                                        |
| Kategori              | Remote Code Execution                                           |
| Autentikasi           | Tidak diperlukan apabila seluruh prasyarat kerentanan terpenuhi |
| CVSS v3.1             | 9.8 — Critical                                                  |

CVE-2021-3129 berkaitan dengan penggunaan operasi file yang tidak aman pada komponen Facade Ignition. Pada kombinasi versi Laravel dan Ignition yang terdampak, dengan mode debug dan fungsi terkait yang dapat diakses, kerentanan ini dapat menyebabkan eksekusi kode tanpa autentikasi.

Menurut advisory paket `facade/ignition`, versi perbaikan pada masing-masing cabang meliputi 1.6.15, 1.16.14, 2.4.2, dan 2.5.2.

**Catatan:** Judul PoC Exploit-DB menyebut Laravel 8.4.2, sedangkan deskripsi NVD menyebut kondisi Laravel sebelum 8.4.2. Untuk keputusan aktual, verifikasi harus didasarkan pada versi dependency, kode yang terpasang, dan advisory yang relevan, bukan hanya judul PoC.

## 3. Observasi dari Pengujian Sebelumnya

### 3.1. Information Disclosure melalui Error Handling

Pada dokumentasi pengujian Oktober 2026, aplikasi menampilkan halaman *Internal Server Error* dengan informasi teknis yang seharusnya tidak perlu diberikan kepada pengguna akhir.

Informasi yang tampak meliputi:

- Jenis exception, termasuk `TypeError`.
- Identitas driver ODBC SQL Server.
- Struktur query SQL dan nama sejumlah kolom.
- Namespace serta nama class dan method aplikasi.
- Path absolut kode pada server.
- Informasi file controller dan nomor baris.

**Analisis:**

Observasi tersebut mendukung temuan kebocoran detail internal melalui error handling. Halaman error juga konsisten dengan kondisi debug yang terlalu terbuka.

Meskipun laporan terdahulu menyebut `APP_DEBUG=true`, nilai konfigurasi tersebut belum diverifikasi langsung melalui artefak konfigurasi. Kebocoran serupa juga dapat terjadi akibat custom exception handler yang tidak aman.

**Risiko:** Pengungkapan detail internal dapat membantu pemetaan arsitektur aplikasi dan mempermudah identifikasi kelemahan lainnya.

**Hubungan dengan CVE-2021-3129:** Temuan ini mendukung alasan melakukan pemeriksaan konfigurasi debugging, tetapi belum membuktikan penggunaan Facade Ignition yang rentan.

### 3.2. Weak File Type Validation

Pada endpoint unggah berkas yang diuji sebelumnya, laporan kelompok mencatat penerimaan file polyglot dengan karakteristik format GIF dan konten PHP.

Screenshot Burp Suite memperlihatkan respons HTTP 200 dan data JSON yang menunjukkan file berformat GIF diterima, dengan metadata seperti `file_ext` dan `file_mime`.

**Analisis:**

Hasil ini mendukung dugaan bahwa validasi isi file belum cukup ketat untuk menolak file polyglot tertentu.

Namun, penerimaan file tidak membuktikan bahwa konten PHP akan dieksekusi. Dampak RCE memerlukan kondisi tambahan, misalnya file diproses oleh interpreter atau mekanisme server lain yang tidak aman.

Jika file hanya disimpan sebagai objek statis pada S3, eksekusi PHP tidak terjadi hanya karena file tersebut mengandung sintaks PHP.

**Risiko:** Penyimpanan konten tidak diharapkan, kemungkinan penyalahgunaan pemrosesan file, atau risiko lanjutan sesuai konfigurasi penyimpanan dan penyajian konten.

**Hubungan dengan CVE-2021-3129:** Temuan upload dipertahankan sebagai observasi tambahan. Ini bukan bukti langsung eksploitasi Ignition karena CVE tersebut memiliki jalur kerentanan berbeda.

## 4. Matriks Kecocokan Kandidat RCE

| Prasyarat                                       | Bukti tersedia                                    | Status                              |
| ----------------------------------------------- | ------------------------------------------------- | ----------------------------------- |
| Aplikasi menggunakan Laravel                    | Namespace, struktur error, dan laporan sebelumnya | Indikasi kuat                       |
| Error menampilkan informasi internal            | Screenshot halaman error                          | Terobservasi                        |
| Konfigurasi debug yang efektif aktif            | Perilaku halaman error                            | Belum dikonfirmasi dari konfigurasi |
| Menggunakan Facade Ignition                     | Belum ada `composer.lock` atau SBOM               | Belum diketahui                     |
| Versi Ignition termasuk rentang rentan          | Belum ada informasi versi                         | Belum diketahui                     |
| Jalur kerentanan Ignition dapat diakses         | Belum ada bukti                                   | Belum diketahui                     |
| Aset dan metode validasi tercakup izin tertulis | Memerlukan peninjauan scope pengujian saat ini    | Belum diketahui                     |
| Eksekusi kode berhasil                          | Tidak ada bukti RCE                               | Tidak terbukti                      |

**Kesimpulan kecocokan:** CVE-2021-3129 relevan sebagai kandidat penelitian karena teknologi yang terindikasi adalah Laravel dan terdapat kebocoran error. Namun, informasi yang tersedia belum mencukupi untuk menyimpulkan bahwa myITS Thesis menggunakan versi Ignition yang rentan.

## 5. Static Source Code Review — EDB-ID 49424

Source code PoC Exploit-DB 49424 dianalisis untuk mengidentifikasi fungsi, perilaku berisiko, serta alasan PoC tidak cocok dieksekusi secara langsung pada lingkungan produksi.

| Komponen kode        | Hasil analisis                                                              |
| -------------------- | --------------------------------------------------------------------------- |
| `requests`           | Mengirim permintaan HTTP ke aplikasi target                                 |
| `clear_log()`        | Berupaya memodifikasi keadaan log aplikasi melalui operasi file yang rentan |
| `create_payload()`   | Mengirim data yang digunakan dalam rangkaian eksploitasi                    |
| `convert()`          | Melakukan tahap transformasi data terkait eksploitasi                       |
| `generate_payload()` | Menggunakan PHPGGC untuk membuat payload berbasis PHP                       |
| `os.system()`        | Menjalankan perintah pada komputer yang mengeksekusi PoC                    |
| `exploited()`        | Berusaha mendapatkan keluaran dari hasil eksploitasi                        |
| `verify=False`       | Menonaktifkan verifikasi sertifikat TLS pada permintaan Python              |

### 5.1. Koneksi Keluar

PoC melakukan komunikasi HTTP dengan alamat target yang ditentukan pengguna. Kode juga memuat mekanisme untuk mengambil dependency PHPGGC dari GitHub apabila belum tersedia pada lingkungan lokal.

Artinya, penggunaan PoC dapat melibatkan koneksi jaringan tambahan dan bergantung pada kode pihak ketiga.

### 5.2. Operasi File

Script menggunakan file sementara pada komputer penguji dan berusaha memanipulasi file log aplikasi yang terdampak.

Operasi ini merupakan salah satu alasan utama mengapa PoC tidak dikategorikan sebagai pemeriksaan read-only.

### 5.3. Eksekusi Command

PoC memiliki mekanisme untuk menghasilkan payload yang bertujuan mencapai eksekusi perintah. Penggunaan `os.system()` juga berarti script menjalankan proses lokal pada perangkat penguji.

Jika prasyarat kerentanan terpenuhi, rangkaian tersebut dapat menyebabkan eksekusi kode pada lingkungan server.

### 5.4. Persistence dan Perubahan Sistem

Tidak ditemukan rutinitas eksplisit untuk memasang backdoor permanen pada script yang ditinjau.

Namun, manipulasi log, pengiriman permintaan berulang, dan potensi eksekusi perintah merupakan perilaku yang memiliki risiko perubahan sistem.

### 5.5. Risiko Penggunaan PoC

PoC memiliki sejumlah karakteristik berisiko:

- Memodifikasi atau berupaya memodifikasi log aplikasi.
- Melakukan permintaan berulang, termasuk perulangan tanpa batas dalam kondisi tertentu.
- Menjalankan instruksi melalui shell lokal.
- Memerlukan dependency eksternal.
- Menonaktifkan pemeriksaan sertifikat TLS.
- Dapat memicu eksekusi perintah pada server jika kondisi eksploitasi terpenuhi.

**Kesimpulan source code review:** PoC EDB-49424 tidak layak dijalankan langsung pada sistem produksi untuk sekadar membuktikan kecocokan versi. Validasi awal harus dilakukan melalui analisis dependency dan konfigurasi.

## 6. Mini Exploit Register

| Field                      | Nilai                                                                                            |
| -------------------------- | ------------------------------------------------------------------------------------------------ |
| ID register                | ITS-RCE-TRIAGE-001                                                                               |
| Exploit                    | EDB-49424                                                                                        |
| CVE                        | CVE-2021-3129                                                                                    |
| Produk                     | Laravel / Facade Ignition                                                                        |
| Versi/build target         | Belum diketahui                                                                                  |
| Prasyarat                  | Paket dan versi rentan, debug aktif, serta jalur kerentanan yang dapat dicapai                   |
| Autentikasi                | Tidak diperlukan dalam kondisi rentan yang relevan                                               |
| Indikator keberhasilan RCE | Bukti eksekusi instruksi oleh proses aplikasi pada lingkungan yang diizinkan                     |
| Bukti yang sudah tersedia  | Detail error internal Laravel dan observasi upload polyglot                                      |
| Risiko menjalankan PoC     | Perubahan log, eksekusi perintah, gangguan aplikasi, dan perubahan sistem                        |
| Validasi dampak minimum    | Review dependency dan konfigurasi secara offline; lab terisolasi apabila secara khusus disetujui |
| Status RCE                 | Belum terbukti                                                                                   |
| Keputusan                  | Kandidat investigasi bersyarat                                                                   |

## 7. Rencana Validasi Minimal Impact

Tahapan yang direkomendasikan:

**Tahap 1 — Verifikasi administrasi**

Pastikan scope, tiket, NDA, izin tertulis, dan metode pemeriksaan yang diperbolehkan. Izin pengujian pada Oktober 2026 tidak otomatis mencakup pengujian baru pada November 2026.

**Tahap 2 — Verifikasi produk dan build**

Minta pemilik aplikasi menyediakan informasi dependency yang disanitasi, seperti `composer.lock`, SBOM, dan daftar versi package yang terpasang.

Periksa apakah `facade/ignition` digunakan dan apakah versinya termasuk rentang yang rentan.

**Tahap 3 — Pemeriksaan konfigurasi**

Periksa konfigurasi debug yang efektif, error handler, dan ketersediaan komponen terkait. Gunakan informasi dari administrator atau kode sumber yang secara resmi diberikan.

**Tahap 4 — Pemeriksaan penyimpanan upload**

Untuk temuan upload sebelumnya, tinjau aturan validasi, penyimpanan, visibilitas file, serta konfigurasi penyajian konten. Pastikan file upload diperlakukan sebagai data, bukan kode yang dapat dieksekusi.

**Tahap 5 — Validasi teknis terbatas**

Jika pemeriksaan offline belum mencukupi dan persetujuan tambahan diberikan, reproduksi dapat dilakukan pada lab terisolasi menggunakan salinan lingkungan yang diizinkan, tanpa data pengguna sebenarnya.

Hindari perubahan log produksi, persistence, backdoor, pengambilan data, koneksi keluar yang tidak diperlukan, serta gangguan layanan.

**Tahap 6 — Penghentian dan dokumentasi**

Pengujian dihentikan setelah bukti yang disetujui tersedia. Setiap hasil dicatat, termasuk batasan pengujian dan hal yang belum dapat dipastikan.

## 8. Rekomendasi Keamanan

### Temuan A — Information Disclosure

- Terapkan `APP_DEBUG=false` pada produksi dan pastikan konfigurasi efektif sesuai.
- Gunakan error handler yang menampilkan pesan generik kepada pengguna.
- Batasi detail exception, query SQL, namespace, serta path internal pada log yang hanya dapat diakses pihak berwenang.
- Terapkan validasi input sesuai tipe data, termasuk UUID dan integer.

### Temuan B — Weak File Type Validation

- Gunakan allowlist tipe file pada backend.
- Periksa MIME berdasarkan isi file, bukan hanya header dari klien.
- Terapkan pemeriksaan struktur file dan pemrosesan ulang gambar jika sesuai kebutuhan.
- Tolak file yang mengandung konten aktif yang tidak diperlukan.
- Simpan unggahan dengan nama yang dibangkitkan server pada lokasi yang tidak dapat menjalankan script.
- Terapkan pengaturan akses, batas ukuran, dan kebijakan penyajian konten yang aman.

### Kandidat CVE-2021-3129

- Inventarisasi paket Laravel dan Ignition.
- Identifikasi versi yang terkena dampak.
- Perbarui dependency ke rilis yang didukung dan telah diperbaiki.
- Batasi fungsi debugging agar tidak terekspos di produksi.
- Lakukan pengujian regresi setelah perbaikan.

## 9. Keputusan Triage

**Keputusan kelompok: Conditional Candidate — Further Verification Required.**

Alasan:

1. Exploit RCE tersedia secara publik dalam Exploit-DB.
2. Kerentanan memiliki tingkat keparahan Critical.
3. Laporan sebelumnya mengindikasikan teknologi Laravel dan kebocoran detail internal.
4. Namun, belum tersedia bukti versi Ignition, build Laravel, konfigurasi yang efektif, atau jalur eksploitasi yang rentan.
5. Belum terdapat bukti bahwa RCE pernah berhasil dijalankan terhadap aplikasi myITS Thesis.

Dengan demikian, EDB-49424 layak dipilih sebagai kandidat investigasi untuk aktivitas kelas, tetapi tidak boleh disebut sebagai kerentanan RCE ITS yang telah terkonfirmasi.

## 10. Kesimpulan

Daur ulang laporan Bug Bounty Oktober 2026 menghasilkan triage yang menghubungkan hasil observasi nyata dengan kandidat exploit yang relevan secara teknologi.

Temuan Information Disclosure dan Weak File Type Validation tetap dipertahankan sebagai dua kelas masalah yang terpisah. Kandidat CVE-2021-3129 digunakan untuk mengevaluasi kemungkinan risiko tambahan pada komponen debugging Laravel, tanpa menyimpulkan bahwa RCE benar-benar tersedia pada aset ITS.

Hasil analisis menunjukkan bahwa pendekatan paling tepat adalah memprioritaskan verifikasi dependency, versi/build, konfigurasi, dan scope tertulis sebelum mempertimbangkan reproduksi teknis.

**Hasil akhir: kandidat RCE relevan untuk ditriase, tetapi status kerentanan pada myITS Thesis masih belum terkonfirmasi.**

---

# Title
Critical Public Repository Exposure Leading to Full Source Code Disclosure and Target Verification of SIKOPJAK (CWE-547 / CWE-215)

## Issue Description
Kerentanan eksposur informasi kritikal berupa kebocoran repositori kode sumber (*Source Code Disclosure*) ditemukan pada infrastruktur Pemerintah Provinsi DKI Jakarta (`*.jakarta.go.id`). Melalui tahap *reconnaissance* yang sistematis menggunakan perangkat enumerasi otomatis seperti **Sublist3r, Amass, dan Subfinder**, berhasil diungkap keberadaan peladen GitLab internal di `git.jakarta.go.id` [cite: 4, 5]. Ini mengizinkan penemuan proyek secara anonim (*unauthenticated*) melalui *endpoint* API publiknya [cite: 5]. 

Melalui celah ini, keseluruhan pangkalan kode (*full code resource*) dari sistem SIMKoperasi terekspos ke publik. Repositori ini dipastikan merupakan kode sumber utama dari aplikasi produksi SIKOPJAK berdasarkan bukti berkas `README.md` (mengandung teks *"Stagging development-web-ppkukm"*) dan identitas *session cookie* `simkoperasi`.

Dari eksposur kode sumber ini, ditemukan bahwa pengembang menggunakan konfigurasi tidak aman `APP_DEBUG=true` pada `.env.example` di masa pengembangan. Untuk membuktikan apakah kelemahan konfigurasi dan logika ini terbawa hingga ke *server* produksi, analisis kode statis dilakukan dan menemukan *endpoint* `/sinkronisasi/berita/trx` yang tidak dilindungi autentikasi dan mengandung fungsi *debug* `dd()`. Saat *endpoint* ini dieksekusi di web server asli (*live*), sistem memicu halaman *debug* Laravel Ignition yang memvalidasi bahwa mode debug benar-benar aktif di produksi, sehingga mengekspos data internal yang sangat sensitif (kredensial *database*, kueri internal, dan *path server*).serta dikuatkan oleh berkas `README.md` di dalam repositori yang secara eksplisit mencantumkan keterangan *"Stagging development-web-ppkukm"* [cite: 4, 5] dan kecocokan struktur kuki sesi aplikasi *simkoperasi*. Eksposur ini mengakibatkan kerangka arsitektur aplikasi, logika bisnis, rute peladen (*routes*), hingga konfigurasi templat lingkungan pada berkas `.env.example` jatuh ke tangan publik tanpa proteksi [cite: 1, 4, 5].

## Affected URL/Area
- **Initial API Endpoint:** `https://git.jakarta.go.id/api/v4/projects` [cite: 5]
- **Target Repository:** `https://git.jakarta.go.id/miftah/SIMKoperasi` [cite: 1, 5]
- **Specific Branch/Tree:** `https://git.jakarta.go.id/miftah/SIMKoperasi/tree/sufi` [cite: 5]
- **Exposed Assets:** Seluruh pohon direktori aplikasi (`app`, `config`, `routes`, `database`), berkas `.env.example`, dan `README.md` 
- **Live Vulnerable Endpoint:** `https://disppkukm.jakarta.go.id/SIKOPJAK/sinkronisasi/berita/trx`
[cite: 4, 5].

## Risk Rating
- **Risk:** **Critical**
- **Difficulty to Exploit:** **Low** [cite: 1]
- **Authentication Required:** **No** [cite: 1]
- **User Interaction Required:** **No** [cite: 1]
- **CVSS 3.1 Score:** [7.5 (High/Critical) - AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N] [cite: 1]

### Impact
Penyerang eksternal anonim tanpa hak akses apa pun ke sistem dapat mengeksploitasi celah ini untuk mengunduh 100% *source code* aplikasi SIKOPJAK secara utuh [cite: 1, 5]. Tanpa memerlukan kredensial atau *privilege* peladen [cite: 1, 5], pelaku kejahatan dapat melakukan analisis kode statis secara *offline* untuk memetakan celah keamanan tersembunyi [cite: 1] serta membaca pola konfigurasi sensitif yang tercantum pada templat `.env.example` [cite: 1]. 

Lebih jauh lagi, kepemilikan penuh atas pangkalan kode ini berfungsi sebagai pengganda ancaman (*force multiplier*) melalui simulasi serangan *white-box*. Penyerang dapat memanfaatkan kode sumber yang bocor untuk mempelajari pengendali unggahan berkas secara mendalam guna merancang injeksi *web shell* (*Unauthenticated File Upload*), memetakan parameter sensitif untuk mengeksploitasi celah *Insecure Direct Object Reference* (IDOR) secara otomatis, hingga memilah rute mana saja yang luput dari proteksi *middleware* autentikasi tanpa harus melakukan pemindaian aktif di peladen produksi [cite: 1]. Hal ini mengancam integritas data operasional Dinas PPKUKM DKI Jakarta dan kerahasiaan informasi ribuan entitas koperasi yang dikelola oleh sistem [cite: 1].

### Attack Scenario
**Tahap 1: Enumerasi dan Eksposur Repositori**

![subfinder](reckon-subdomain.png)

1. Penyerang memulai *reconnaissance* pada lingkup target `*.jakarta.go.id` menggunakan perangkat pemindai otomatis seperti **Sublist3r, Amass, dan Subfinder**, yang kemudian berhasil memetakan keberadaan *subdomain* repositori di `git.jakarta.go.id` [cite: 4, 5].

<br>

![Git](git.jakarta.go.id.png)

2. Saat diakses melalui browser, antarmuka web GitLab mengharuskan pengguna melakukan *login* [cite: 5]. Namun, penyerang memanfaatkan standar arsitektur GitLab dengan menembak *endpoint* API publik bawaan secara anonim (`/api/v4/projects`) [cite: 5].

<br>

![informasi web](Bukti-nama-WEB.png)

3. Respons JSON dari API membocorkan Project ID 254 dengan nama `SIMKoperasi` [cite: 1, 4, 5]. Penyerang mengakses repositori tersebut tanpa kendala [cite: 1, 5]. Keabsahan bahwa repositori ini adalah kode sumber SIKOPJAK dikonfirmasi melalui teks `web-ppkukm` di dalam `README.md` [cite: 4, 5] serta kecocokan identitas kuki sesi.

<br>

4. Penyerang mengunduh seluruh isi direktori peladen untuk mempelajari struktur kontroler, basis data, dan celah logika aplikasi secara mendalam secara *offline*. 

<br>

**Tahap 2: Pembuktian Data Eksposur di Server Produksi**

6. Berdasarkan hasil tinjauan kode pada berkas `SinkronisasiController.php` baris 18, akses *endpoint* yang tidak terautentikasi di lingkungan produksi:
   `https://disppkukm.jakarta.go.id/SIKOPJAK/sinkronisasi/berita/trx`

<br>

![Fungsi Debug](fungsi-debug.png)

7. Amati *server* memicu halaman *debug* Laravel Ignition yang secara gamblang membocorkan informasi kredensial peladen internal.

## Steps to Reproduce/PoC

1. Jalankan proses pengumpulan informasi awal (*reconnaissance*) untuk memetakan domain GitLab instansi menggunakan alat enumerasi seperti Sublist3r atau Subfinder [cite: 4, 5]. 

2. Kirimkan HTTP GET *request* anonim ke API GitLab untuk memintas halaman pelindung *login* [cite: 5]:
   `curl -s "https://git.jakarta.go.id/api/v4/projects?per_page=20"` [cite: 5]

3. Ekstrak informasi dari respons JSON untuk menemukan *Project ID* 254 (`miftah/SIMKoperasi`) [cite: 1, 4, 5].

4. Buka tautan repositori secara langsung pada peramban: 
   `https://git.jakarta.go.id/miftah/SIMKoperasi/tree/sufi` [cite: 5]

5. Amati bahwa antarmuka repositori dapat dijelajahi secara penuh meskipun tombol *Sign in / Register* masih tertera di pojok kanan atas, membuktikan ketiadaan autentikasi [cite: 1, 5].

6. Buka berkas `README.md` untuk memvalidasi teks identifikasi penamaan peladen target (*web-ppkukm*) [cite: 4, 5] dan verifikasi kecocokan kuki sesi aplikasi `simkoperasi`.

7. Buka berkas `.env.example` untuk melihat struktur variabel lingkungan yang terekspos [cite: 1, 4, 5].

### Relevant Requests & Responses

*** Request ***
```http
GET /api/v4/projects?per_page=20 HTTP/1.1
Host: git.jakarta.go.id
Accept: application/json
```

*** Response ***
```http
HTTP/1.1 200 OK
Content-Type: application/json

[
  {
    "id": 254,
    "name": "SIMKoperasi",
    "path_with_namespace": "miftah/SIMKoperasi",
    "default_branch": "master",
    "web_url": "https://git.jakarta.go.id/miftah/SIMKoperasi"
  }
]
```
**Request 2 (Live Endpoint Exploitation):**
```http
GET /SIKOPJAK/sinkronisasi/berita/trx HTTP/1.1
Host: disppkukm.jakarta.go.id
Accept: text/html,application/xhtml+xml
```

**Response 2 (Data Leak Snippet):**
```http
HTTP/1.1 500 Internal Server Error
Content-Type: text/html; charset=UTF-8

SQLSTATE[HY000] [1045] Access denied for user 'forge'@'localhost' (using password: NO) 
(Connection: mysql2, SQL: select count(*) as aggregate from `berita` where `menu_id` = 11)
...
```

### Screenshots
![Bukti Akses Publik GitLab](Gitlab-code-source.png)

*Tangkapan layar di atas menunjukkan akses penuh tanpa batas ke repositori `miftah/SIMKoperasi` pada branch `sufi`. Keberadaan tombol "Sign in / Register" di pojok kanan atas membuktikan akses unauthenticated, mengekspos keseluruhan direktori (app, config, storage) ke ranah publik.*



## Affected Demographic/User Base
- Seluruh infrastruktur sistem informasi Dinas PPKUKM DKI Jakarta. Eksposur data repositori ini memberikan penyerang keuntungan strategis berupa pemahaman utuh terhadap celah logika aplikasi sebelum serangan lanjutan dilancarkan [cite: 1, 4, 5].

## Recommended Fix
1. **Ubah Visibilitas Repositori (Prioritas Utama):** Segera ubah setelan visibilitas proyek `miftah/SIMKoperasi` di peladen GitLab dari "Public" menjadi **"Private"** [cite: 1, 4, 5].
2. **Kunci Akses API Anonim:** Konfigurasikan peladen GitLab agar menonaktifkan pencantuman daftar proyek publik (*Public Directory*) bagi pengguna yang tidak terautentikasi [cite: 5].
3. **Audit Keamanan Pangkalan Kode:** Lakukan peninjauan menyeluruh terhadap riwayat komit (*commit history*) apabila terdapat kredensial sensitif atau token akses asli yang sempat terunggah sebelumnya.

## References
- [1] [CWE-547: Use of Hard-coded, Security-relevant Configuration Variables](https://cwe.mitre.org/data/definitions/547.html)
- [2] [CWE-215: Information Exposure Through Environmental Variables](https://cwe.mitre.org/data/definitions/215.html)
- [3] [GitLab Docs: Public Projects Visibility](https://docs.gitlab.com/ee/public_access/public_access.html) [cite: 5]
